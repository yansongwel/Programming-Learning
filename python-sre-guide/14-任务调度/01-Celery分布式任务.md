# Celery 分布式任务 — 构建分布式运维任务执行平台

## 1. 是什么 & 为什么用它

Celery 是 Python 最流行的分布式任务队列系统，支持实时处理和定时调度。
对 SRE 来说，Celery 可以：

- **分布式运维任务执行**：在多台机器上并行执行巡检、部署
- **异步任务处理**：耗时操作（日志分析、报告生成）异步执行
- **定时任务调度**：替代 crontab，支持动态调度
- **任务编排**：链式执行、并行执行、工作流

核心架构：
```
Producer(生产者) → Broker(消息队列) → Worker(消费者)
                   ↑                    ↓
                   └── Backend(结果存储) ←┘
```

## 2. 安装与环境

```bash
# 安装 Celery
pip install celery[redis]    # 使用 Redis 作为 Broker

# 或使用 RabbitMQ
# pip install celery[rabbitmq]

# 安装 Redis（Broker + Backend）
# Ubuntu: sudo apt install redis-server
# macOS: brew install redis
# Docker: docker run -d -p 6379:6379 redis:7

# 验证
python3 -c "import celery; print(celery.__version__)"
redis-cli ping  # 应返回 PONG

# 可选：监控工具
pip install flower  # Celery 的 Web 监控界面
```

## 3. 核心概念与原理

### 3.1 架构组件

| 组件 | 说明 | 常用选择 |
|------|------|----------|
| Broker | 消息队列，存储任务消息 | Redis / RabbitMQ |
| Worker | 任务执行者，消费并执行任务 | Celery Worker 进程 |
| Backend | 结果存储，保存任务返回值 | Redis / PostgreSQL |
| Beat | 定时调度器，按计划发送任务 | Celery Beat |
| Flower | Web 监控面板 | Flower |

### 3.2 任务执行流程

```
1. 客户端调用 task.delay(args)
   ↓
2. 任务被序列化并发送到 Broker（Redis/RabbitMQ）
   ↓
3. Worker 从 Broker 获取任务
   ↓
4. Worker 反序列化并执行任务
   ↓
5. 执行结果存储到 Backend
   ↓
6. 客户端通过 AsyncResult 获取结果
```

### 3.3 任务状态

```
PENDING → STARTED → SUCCESS
                  → FAILURE
                  → RETRY
          → REVOKED（取消）
```

## 4. 基础用法

### 4.1 定义 Celery 应用和任务

```python
#!/usr/bin/env python3
"""
Celery 应用定义
文件名: celery_app.py

启动 Worker:
  celery -A celery_app worker --loglevel=info

启动 Beat（定时任务）:
  celery -A celery_app beat --loglevel=info
"""

from celery import Celery
from datetime import timedelta

# ============================================================
# Celery 应用配置
# ============================================================
app = Celery(
    'sre_tasks',
    broker='redis://localhost:6379/0',     # 消息队列
    backend='redis://localhost:6379/1',    # 结果存储
)

# 配置
app.conf.update(
    # 序列化方式
    task_serializer='json',
    result_serializer='json',
    accept_content=['json'],

    # 时区
    timezone='Asia/Shanghai',
    enable_utc=True,

    # 任务结果过期时间
    result_expires=3600,  # 1小时

    # Worker 配置
    worker_prefetch_multiplier=1,   # 每次预取1个任务
    worker_max_tasks_per_child=100, # 每个子进程执行100个任务后重启
    worker_concurrency=4,           # 并发 Worker 数

    # 任务超时
    task_soft_time_limit=300,       # 软超时（秒）
    task_time_limit=600,            # 硬超时（秒）

    # 重试配置
    task_acks_late=True,            # 任务完成后才确认
    task_reject_on_worker_lost=True,
)

# ============================================================
# 任务定义
# ============================================================

@app.task(bind=True, name='sre.health_check')
def health_check(self, host: str, port: int = 80) -> dict:
    """健康检查任务"""
    import socket
    import time

    start = time.time()
    try:
        sock = socket.create_connection((host, port), timeout=5)
        sock.close()
        latency = (time.time() - start) * 1000
        return {
            'host': host,
            'port': port,
            'status': 'ok',
            'latency_ms': round(latency, 2),
        }
    except Exception as e:
        return {
            'host': host,
            'port': port,
            'status': 'error',
            'error': str(e),
        }

@app.task(bind=True, name='sre.disk_check',
          max_retries=3, default_retry_delay=60)
def disk_check(self, hostname: str) -> dict:
    """磁盘检查任务（带重试）"""
    import shutil
    import os

    try:
        # 实际应通过 SSH 远程执行
        total, used, free = shutil.disk_usage("/")
        return {
            'hostname': hostname,
            'total_gb': round(total / (1024**3), 2),
            'used_gb': round(used / (1024**3), 2),
            'free_gb': round(free / (1024**3), 2),
            'usage_percent': round(used / total * 100, 1),
        }
    except Exception as exc:
        # 自动重试
        raise self.retry(exc=exc)

@app.task(name='sre.send_alert')
def send_alert(channel: str, message: str, severity: str = 'warning') -> dict:
    """发送告警（模拟）"""
    print(f"[{severity.upper()}] → {channel}: {message}")
    return {
        'channel': channel,
        'message': message,
        'severity': severity,
        'sent': True,
    }

@app.task(name='sre.generate_report')
def generate_report(data: list, report_type: str = 'daily') -> dict:
    """生成报告"""
    import json
    from datetime import datetime

    report = {
        'type': report_type,
        'generated_at': datetime.now().isoformat(),
        'total_checks': len(data),
        'healthy': sum(1 for d in data if d.get('status') == 'ok'),
        'unhealthy': sum(1 for d in data if d.get('status') != 'ok'),
        'data': data,
    }

    # 保存报告
    report_path = f"/tmp/sre_report_{report_type}.json"
    with open(report_path, 'w') as f:
        json.dump(report, f, ensure_ascii=False, indent=2)

    return {'report_path': report_path, 'summary': report}

print("Celery 应用已定义")
print("启动 Worker: celery -A celery_app worker --loglevel=info")
```

### 4.2 调用任务

```python
#!/usr/bin/env python3
"""调用 Celery 任务"""

# 注意：以下代码需要 Redis 和 Worker 运行才能真正执行
# 这里展示调用方式

print("""
# === 异步调用 ===
from celery_app import health_check, disk_check, send_alert

# delay() — 最简单的异步调用
result = health_check.delay('10.0.0.1', 80)
print(f"任务 ID: {result.id}")
print(f"状态: {result.status}")
print(f"结果: {result.get(timeout=10)}")  # 阻塞等待结果

# apply_async() — 更多选项
result = health_check.apply_async(
    args=['10.0.0.1', 80],
    countdown=10,        # 10秒后执行
    expires=300,         # 5分钟内必须执行
    retry=True,          # 失败重试
    retry_policy={
        'max_retries': 3,
        'interval_start': 1,
        'interval_step': 2,
    },
)

# === 检查任务状态 ===
from celery.result import AsyncResult
result = AsyncResult('task-id-here')
print(f"状态: {result.status}")     # PENDING/STARTED/SUCCESS/FAILURE
print(f"是否就绪: {result.ready()}")
print(f"是否成功: {result.successful()}")
if result.ready():
    print(f"结果: {result.result}")
""")
```

### 4.3 任务链与工作流

```python
#!/usr/bin/env python3
"""Celery 任务编排：链/组/chord"""

print("""
from celery import chain, group, chord
from celery_app import health_check, send_alert, generate_report

# === Chain（链）— 顺序执行，前一个的结果传给后一个 ===
# 检查 → 告警
workflow = chain(
    health_check.s('10.0.0.1', 80),  # .s() 创建 signature
    send_alert.s('slack', severity='info'),
)
result = workflow.apply_async()

# === Group（组）— 并行执行 ===
# 同时检查多台服务器
hosts = ['10.0.0.1', '10.0.0.2', '10.0.0.3', '10.0.0.4']
check_group = group(
    health_check.s(host, 80) for host in hosts
)
result = check_group.apply_async()
# result.get() 返回所有结果的列表

# === Chord（和弦）— 并行执行 + 回调 ===
# 并行检查所有服务器，全部完成后生成报告
workflow = chord(
    [health_check.s(host, 80) for host in hosts],
    generate_report.s(report_type='health_check')  # 回调
)
result = workflow.apply_async()

# === 复合工作流 ===
# 检查 → 并行（多台服务器） → 汇总 → 告警
complex_workflow = chain(
    # 先执行准备任务
    prepare_task.s(),
    # 然后并行检查 + 汇总
    chord(
        [health_check.s(host, 80) for host in hosts],
        generate_report.s()
    ),
    # 最后发告警
    send_alert.s('slack')
)
""")
```

## 5. 进阶用法

### 5.1 定时任务（Beat）

```python
#!/usr/bin/env python3
"""Celery Beat 定时任务配置"""

from celery.schedules import crontab
from datetime import timedelta

# 在 Celery 配置中添加定时任务
CELERY_BEAT_SCHEDULE = {
    # 每 5 分钟执行健康检查
    'health-check-every-5min': {
        'task': 'sre.health_check',
        'schedule': timedelta(minutes=5),
        'args': ('10.0.0.1', 80),
    },

    # 每小时检查磁盘
    'disk-check-hourly': {
        'task': 'sre.disk_check',
        'schedule': timedelta(hours=1),
        'args': ('web-01',),
    },

    # 每天早上 8 点生成日报
    'daily-report-8am': {
        'task': 'sre.generate_report',
        'schedule': crontab(hour=8, minute=0),
        'args': ([], 'daily'),
    },

    # 工作日每 30 分钟执行（周一到周五）
    'workday-check': {
        'task': 'sre.health_check',
        'schedule': crontab(minute='*/30', hour='9-18', day_of_week='1-5'),
        'args': ('api.example.com', 443),
    },

    # 每月 1 号生成月报
    'monthly-report': {
        'task': 'sre.generate_report',
        'schedule': crontab(day_of_month=1, hour=6, minute=0),
        'args': ([], 'monthly'),
    },
}

print("=== Celery Beat 定时任务 ===")
for name, config in CELERY_BEAT_SCHEDULE.items():
    print(f"  {name}: {config['task']} | {config['schedule']}")

print("""
# 启动 Beat:
# celery -A celery_app beat --loglevel=info

# 同时启动 Worker + Beat:
# celery -A celery_app worker --beat --loglevel=info

# 使用 crontab 对照:
# crontab       → Celery crontab
# */5 * * * *   → crontab(minute='*/5')
# 0 8 * * *     → crontab(hour=8, minute=0)
# 0 */1 * * 1-5 → crontab(minute=0, hour='*/1', day_of_week='1-5')
# 0 6 1 * *     → crontab(hour=6, minute=0, day_of_month=1)
""")
```

### 5.2 监控 Flower

```python
#!/usr/bin/env python3
"""Celery 监控：Flower"""

print("""
# === 启动 Flower ===
# celery -A celery_app flower --port=5555

# 访问: http://localhost:5555

# Flower 提供:
# - Worker 状态监控
# - 任务执行历史
# - 任务成功/失败率
# - 实时任务追踪
# - 手动重试/撤销任务

# === API 访问 ===
import requests

# 获取 Worker 列表
resp = requests.get('http://localhost:5555/api/workers')
workers = resp.json()
for name, info in workers.items():
    print(f"Worker: {name}")
    print(f"  状态: {'在线' if info.get('status') else '离线'}")
    print(f"  并发: {info.get('concurrency', 'N/A')}")
    print(f"  已处理: {info.get('completed_tasks', 0)}")

# 获取任务列表
resp = requests.get('http://localhost:5555/api/tasks')
tasks = resp.json()
""")
```

## 6. SRE 实战案例：分布式运维任务执行平台

```python
#!/usr/bin/env python3
"""
SRE 实战：分布式运维任务执行平台
功能：
1. 批量服务器健康巡检
2. 分布式日志收集
3. 配置下发与验证
4. 定时巡检调度
5. 告警集成
6. 执行结果汇总报告
"""

import json
import time
import uuid
from datetime import datetime
from typing import Dict, List, Optional
from dataclasses import dataclass, field, asdict
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor

# ============================================================
# 数据模型
# ============================================================
@dataclass
class TaskResult:
    """任务执行结果"""
    task_id: str
    task_name: str
    target: str
    status: str       # success / failure / timeout / skipped
    result: Dict = field(default_factory=dict)
    error: str = ""
    duration_seconds: float = 0
    timestamp: str = ""

    def __post_init__(self):
        if not self.task_id:
            self.task_id = str(uuid.uuid4())[:8]
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

@dataclass
class BatchExecution:
    """批量执行记录"""
    batch_id: str
    task_name: str
    targets: List[str]
    total: int = 0
    succeeded: int = 0
    failed: int = 0
    results: List[TaskResult] = field(default_factory=list)
    start_time: str = ""
    end_time: str = ""
    duration_seconds: float = 0

    def __post_init__(self):
        if not self.batch_id:
            self.batch_id = str(uuid.uuid4())[:8]
        if not self.start_time:
            self.start_time = datetime.now().isoformat()

# ============================================================
# 运维任务定义（模拟 Celery 任务）
# ============================================================
class SRETaskExecutor:
    """运维任务执行器（模拟分布式执行）"""

    @staticmethod
    def health_check(host: str, port: int = 80) -> dict:
        """健康检查"""
        import random
        time.sleep(random.uniform(0.01, 0.05))  # 模拟网络延迟
        alive = random.random() > 0.1  # 90% 存活率
        return {
            'host': host,
            'port': port,
            'status': 'ok' if alive else 'error',
            'latency_ms': round(random.uniform(1, 100), 2) if alive else 0,
        }

    @staticmethod
    def disk_check(hostname: str) -> dict:
        """磁盘检查"""
        import random
        usage = random.uniform(30, 95)
        return {
            'hostname': hostname,
            'total_gb': 500,
            'usage_percent': round(usage, 1),
            'status': 'critical' if usage > 90 else 'warning' if usage > 80 else 'ok',
        }

    @staticmethod
    def service_check(hostname: str, service: str) -> dict:
        """服务检查"""
        import random
        running = random.random() > 0.05
        return {
            'hostname': hostname,
            'service': service,
            'status': 'running' if running else 'stopped',
            'pid': random.randint(1000, 65535) if running else None,
            'memory_mb': round(random.uniform(50, 500), 1) if running else 0,
        }

# ============================================================
# 分布式任务执行平台
# ============================================================
class SRETaskPlatform:
    """SRE 分布式运维任务执行平台"""

    def __init__(self, max_workers: int = 10):
        self.max_workers = max_workers
        self.executor = SRETaskExecutor()
        self.executions: List[BatchExecution] = []

    def _execute_single(self, task_func, target: str, task_name: str, **kwargs) -> TaskResult:
        """执行单个任务"""
        start = time.time()
        try:
            result = task_func(target, **kwargs)
            duration = time.time() - start
            status = 'success' if result.get('status') in ('ok', 'running') else 'failure'
            return TaskResult(
                task_id="",
                task_name=task_name,
                target=target,
                status=status,
                result=result,
                duration_seconds=round(duration, 3),
            )
        except Exception as e:
            return TaskResult(
                task_id="",
                task_name=task_name,
                target=target,
                status='failure',
                error=str(e),
                duration_seconds=round(time.time() - start, 3),
            )

    def batch_execute(self, task_name: str, task_func, targets: list,
                      **kwargs) -> BatchExecution:
        """批量并行执行任务"""
        batch = BatchExecution(
            batch_id="",
            task_name=task_name,
            targets=targets,
            total=len(targets),
        )

        print(f"\n{'='*60}")
        print(f"批量执行: {task_name}")
        print(f"目标数: {len(targets)}, 并发: {self.max_workers}")
        print(f"{'='*60}")

        start = time.time()

        # 使用线程池并行执行
        with ThreadPoolExecutor(max_workers=self.max_workers) as pool:
            futures = {
                pool.submit(self._execute_single, task_func, target, task_name, **kwargs): target
                for target in targets
            }

            for future in futures:
                result = future.result()
                batch.results.append(result)
                if result.status == 'success':
                    batch.succeeded += 1
                    icon = '+'
                else:
                    batch.failed += 1
                    icon = 'X'
                print(f"  [{icon}] {result.target}: {result.status} ({result.duration_seconds:.3f}s)")

        batch.duration_seconds = round(time.time() - start, 2)
        batch.end_time = datetime.now().isoformat()
        self.executions.append(batch)

        print(f"\n完成: {batch.succeeded}/{batch.total} 成功, "
              f"耗时 {batch.duration_seconds}s")

        return batch

    def run_full_inspection(self, hosts: list) -> dict:
        """运行完整巡检"""
        print(f"\n{'#'*60}")
        print(f"# SRE 全面巡检")
        print(f"# 时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"# 目标: {len(hosts)} 台主机")
        print(f"{'#'*60}")

        results = {}

        # 1. 健康检查
        results['health'] = self.batch_execute(
            '健康检查', self.executor.health_check, hosts
        )

        # 2. 磁盘检查
        results['disk'] = self.batch_execute(
            '磁盘检查', self.executor.disk_check, hosts
        )

        # 3. 服务检查
        services = ['nginx', 'docker', 'node_exporter']
        for svc in services:
            results[f'service_{svc}'] = self.batch_execute(
                f'服务检查({svc})', self.executor.service_check, hosts,
                service=svc
            )

        return results

    def generate_report(self) -> str:
        """生成巡检报告"""
        lines = [
            f"\n{'='*60}",
            f"SRE 巡检报告",
            f"{'='*60}",
            f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"总批次: {len(self.executions)}",
        ]

        # 汇总统计
        total_tasks = sum(e.total for e in self.executions)
        total_success = sum(e.succeeded for e in self.executions)
        total_failed = sum(e.failed for e in self.executions)
        total_duration = sum(e.duration_seconds for e in self.executions)

        lines.append(f"\n--- 总览 ---")
        lines.append(f"  总任务数: {total_tasks}")
        lines.append(f"  成功: {total_success} ({total_success/total_tasks*100:.1f}%)")
        lines.append(f"  失败: {total_failed} ({total_failed/total_tasks*100:.1f}%)")
        lines.append(f"  总耗时: {total_duration:.2f}s")

        # 每个批次详情
        lines.append(f"\n--- 批次详情 ---")
        for batch in self.executions:
            rate = batch.succeeded / batch.total * 100 if batch.total else 0
            lines.append(f"  [{batch.task_name}] {batch.succeeded}/{batch.total} "
                        f"({rate:.0f}%) {batch.duration_seconds:.2f}s")

        # 失败详情
        failures = []
        for batch in self.executions:
            for r in batch.results:
                if r.status != 'success':
                    failures.append(f"  {batch.task_name} @ {r.target}: {r.error or r.result}")

        if failures:
            lines.append(f"\n--- 失败详情 ({len(failures)}) ---")
            lines.extend(failures[:20])  # 最多显示 20 条
            if len(failures) > 20:
                lines.append(f"  ... 还有 {len(failures) - 20} 条")

        return '\n'.join(lines)


# ============================================================
# 演示
# ============================================================
def main():
    # 初始化平台
    platform = SRETaskPlatform(max_workers=5)

    # 模拟服务器列表
    hosts = [f"web-{i:02d}" for i in range(1, 6)] + \
            [f"db-{i:02d}" for i in range(1, 3)] + \
            [f"cache-{i:02d}" for i in range(1, 3)]

    # 运行完整巡检
    platform.run_full_inspection(hosts)

    # 生成报告
    report = platform.generate_report()
    print(report)

    # 保存报告
    from pathlib import Path
    Path("/tmp/sre_inspection_report.txt").write_text(report)
    print(f"\n报告已保存: /tmp/sre_inspection_report.txt")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **Broker 挂掉导致任务丢失**：Redis 默认不持久化，RabbitMQ 更可靠
2. **序列化问题**：默认 JSON，不能传递 Python 对象
3. **内存泄漏**：Worker 长时间运行可能泄漏，用 `max_tasks_per_child`
4. **时区问题**：Celery 默认 UTC，配置 `timezone` 和 `enable_utc`
5. **任务幂等性**：网络问题可能导致任务重复执行，需设计幂等任务

### 最佳实践

```python
# 1. 任务要幂等（可重复执行不产生副作用）
# 2. 设置合理超时
task_soft_time_limit = 300
task_time_limit = 600

# 3. 使用 task_acks_late 防止任务丢失
task_acks_late = True

# 4. 生产环境使用 RabbitMQ 而非 Redis
# 5. 使用 Flower 监控 Worker 状态
# 6. 敏感数据不要放在任务参数中
```

## 8. 速查表

```python
from celery import Celery, chain, group, chord

# === 定义 ===
app = Celery('name', broker='redis://localhost:6379/0')

@app.task
def my_task(arg):
    return result

# === 调用 ===
my_task.delay(arg)                           # 异步调用
my_task.apply_async(args=[arg], countdown=10)  # 延迟执行
result = my_task.apply_async(...)
result.get(timeout=30)                       # 获取结果
result.status                                 # 任务状态

# === 编排 ===
chain(task1.s(arg), task2.s())()             # 顺序执行
group(task.s(a) for a in args)()             # 并行执行
chord([task.s(a) for a in args], callback.s())()  # 并行+回调

# === Beat 定时 ===
from celery.schedules import crontab
crontab(minute='*/5')                        # 每5分钟
crontab(hour=8, minute=0)                    # 每天8:00
crontab(day_of_week='1-5', hour='9-18')      # 工作日

# === 启动命令 ===
# celery -A app worker --loglevel=info       # Worker
# celery -A app beat --loglevel=info         # Beat
# celery -A app flower --port=5555           # 监控
```
