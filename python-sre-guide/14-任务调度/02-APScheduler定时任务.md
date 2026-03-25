# APScheduler 定时任务 — 构建灵活的定时巡检调度系统

## 1. 是什么 & 为什么用它

APScheduler（Advanced Python Scheduler）是 Python 的高级任务调度库，提供了灵活的定时任务能力。
对 SRE 来说，它是轻量级定时任务的首选：

- **比 crontab 更灵活**：支持动态添加/删除任务、任务持久化
- **比 Celery Beat 更轻量**：无需 Broker、单进程即可运行
- **多种触发器**：cron/interval/date，满足各种调度需求
- **任务持久化**：重启后任务不丢失

对比其他方案：
- `crontab`：静态配置，不支持动态调度
- `Celery Beat`：需要完整 Celery 栈（Broker/Worker）
- `schedule` 库：功能简单，无持久化
- `APScheduler`：功能完整、轻量、可嵌入

## 2. 安装与环境

```bash
# 安装 APScheduler
pip install APScheduler

# 可选：持久化支持
pip install sqlalchemy  # SQLAlchemy Job Store

# 验证
python3 -c "import apscheduler; print(apscheduler.__version__)"
```

## 3. 核心概念与原理

### 3.1 四大组件

```
Scheduler（调度器）
  ├── BlockingScheduler    — 独占主线程（适合独立调度服务）
  ├── BackgroundScheduler  — 后台线程运行（适合嵌入应用）
  ├── AsyncIOScheduler     — 异步调度器
  └── TornadoScheduler / GeventScheduler 等

Trigger（触发器）
  ├── cron       — cron 表达式（与 Linux crontab 类似）
  ├── interval   — 固定间隔（每 N 秒/分/时）
  └── date       — 一次性执行（指定时间）

Job Store（任务存储）
  ├── MemoryJobStore      — 内存（默认，重启丢失）
  ├── SQLAlchemyJobStore  — 数据库（持久化）
  ├── RedisJobStore       — Redis
  └── MongoDBJobStore     — MongoDB

Executor（执行器）
  ├── ThreadPoolExecutor  — 线程池（默认）
  └── ProcessPoolExecutor — 进程池（CPU 密集）
```

### 3.2 执行流程

```
添加 Job → Job Store 存储
              ↓
Scheduler 定期检查 → Trigger 判断是否到期
                        ↓ (到期)
                    Executor 执行任务
                        ↓
                    任务完成/失败 → 事件通知
```

### 3.3 触发器对比

| 触发器 | 用途 | 示例 |
|--------|------|------|
| `cron` | 类 crontab | 每天 8:00、每周一 |
| `interval` | 固定间隔 | 每 5 分钟、每 1 小时 |
| `date` | 一次性 | 2024-01-15 10:00:00 |

## 4. 基础用法

### 4.1 快速上手

```python
#!/usr/bin/env python3
"""APScheduler 快速上手"""

from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.background import BackgroundScheduler
from datetime import datetime
import time

# === Interval 触发器（固定间隔） ===
def health_check():
    """每 10 秒执行的健康检查"""
    print(f"[{datetime.now().strftime('%H:%M:%S')}] 健康检查: OK")

# === Cron 触发器（cron 表达式） ===
def daily_report():
    """每天 8:00 执行"""
    print(f"[{datetime.now().strftime('%H:%M:%S')}] 生成日报...")

# === Date 触发器（一次性） ===
def one_time_task():
    """只执行一次"""
    print(f"[{datetime.now().strftime('%H:%M:%S')}] 一次性任务执行")

# 创建调度器
scheduler = BackgroundScheduler()

# 添加任务
scheduler.add_job(
    health_check,
    trigger='interval',
    seconds=10,
    id='health_check',
    name='健康检查',
)

scheduler.add_job(
    daily_report,
    trigger='cron',
    hour=8,
    minute=0,
    id='daily_report',
    name='日报生成',
)

# 启动调度器
scheduler.start()
print("调度器已启动（后台运行）")

# 查看所有任务
print("\n=== 已注册的任务 ===")
for job in scheduler.get_jobs():
    print(f"  ID: {job.id}")
    print(f"  名称: {job.name}")
    print(f"  触发器: {job.trigger}")
    print(f"  下次执行: {job.next_run_time}")
    print()

# 运行一段时间
try:
    print("运行 25 秒后退出...")
    time.sleep(25)
except KeyboardInterrupt:
    pass
finally:
    scheduler.shutdown()
    print("调度器已关闭")
```

### 4.2 三种触发器详解

```python
#!/usr/bin/env python3
"""三种触发器详解"""

from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
from apscheduler.triggers.date import DateTrigger
from datetime import datetime, timedelta

scheduler = BackgroundScheduler()

def task_callback(name):
    """通用任务回调"""
    print(f"  [{datetime.now().strftime('%H:%M:%S')}] 执行: {name}")

# === Interval 触发器 ===
# 每 30 秒
scheduler.add_job(task_callback, 'interval', seconds=30,
                  args=['30秒巡检'], id='interval_30s')

# 每 5 分钟
scheduler.add_job(task_callback, 'interval', minutes=5,
                  args=['5分钟巡检'], id='interval_5m')

# 每 1 小时，错过的任务不补执行
scheduler.add_job(task_callback, 'interval', hours=1,
                  args=['1小时巡检'], id='interval_1h',
                  misfire_grace_time=60)  # 错过60秒内仍执行

# 带开始/结束时间
scheduler.add_job(task_callback, 'interval', minutes=10,
                  args=['限时巡检'],
                  start_date='2024-01-15 09:00:00',
                  end_date='2024-01-15 18:00:00',
                  id='interval_limited')

# === Cron 触发器 ===
# 对标 crontab 格式: 分 时 日 月 周

# 每天 8:00 (crontab: 0 8 * * *)
scheduler.add_job(task_callback, 'cron', hour=8, minute=0,
                  args=['每天8点'], id='cron_daily_8')

# 每 5 分钟 (crontab: */5 * * * *)
scheduler.add_job(task_callback, 'cron', minute='*/5',
                  args=['每5分钟'], id='cron_5min')

# 工作日 9-18 点每小时 (crontab: 0 9-18 * * 1-5)
scheduler.add_job(task_callback, 'cron',
                  day_of_week='mon-fri', hour='9-18', minute=0,
                  args=['工作时间巡检'], id='cron_workday')

# 每月 1 号 6:00 (crontab: 0 6 1 * *)
scheduler.add_job(task_callback, 'cron',
                  day=1, hour=6, minute=0,
                  args=['月报'], id='cron_monthly')

# 使用 CronTrigger.from_crontab() — 直接用 crontab 格式
scheduler.add_job(task_callback,
                  CronTrigger.from_crontab('30 6 * * 1'),  # 每周一 6:30
                  args=['周报'], id='cron_weekly')

# === Date 触发器（一次性） ===
run_time = datetime.now() + timedelta(seconds=5)
scheduler.add_job(task_callback, 'date', run_date=run_time,
                  args=['一次性任务'], id='date_once')

# 打印所有任务
print("=== 已注册的定时任务 ===")
for job in scheduler.get_jobs():
    print(f"  {job.id:25s} | {job.trigger}")

# 不实际启动
scheduler.shutdown(wait=False)

print("""
# === crontab 对照表 ===
# crontab 格式:       分  时  日  月  周
# APScheduler:         minute, hour, day, month, day_of_week
#
# */5 * * * *       → minute='*/5'
# 0 8 * * *         → hour=8, minute=0
# 0 9-18 * * 1-5    → hour='9-18', minute=0, day_of_week='mon-fri'
# 0 6 1 * *         → day=1, hour=6, minute=0
# 30 6 * * 1        → CronTrigger.from_crontab('30 6 * * 1')
""")
```

### 4.3 任务管理

```python
#!/usr/bin/env python3
"""任务动态管理"""

from apscheduler.schedulers.background import BackgroundScheduler
from datetime import datetime
import time

scheduler = BackgroundScheduler()

def my_task(name):
    print(f"  [{datetime.now().strftime('%H:%M:%S')}] {name}")

# 添加任务
scheduler.add_job(my_task, 'interval', seconds=5,
                  args=['任务A'], id='task_a', name='任务A')
scheduler.add_job(my_task, 'interval', seconds=8,
                  args=['任务B'], id='task_b', name='任务B')

scheduler.start()

# === 查看任务 ===
print("=== 当前任务 ===")
for job in scheduler.get_jobs():
    print(f"  {job.id}: {job.name} | 下次: {job.next_run_time}")

# === 暂停任务 ===
scheduler.pause_job('task_b')
print("\n任务B 已暂停")

# === 恢复任务 ===
time.sleep(3)
scheduler.resume_job('task_b')
print("任务B 已恢复")

# === 修改任务 ===
scheduler.reschedule_job('task_a', trigger='interval', seconds=3)
print("任务A 间隔修改为 3 秒")

# === 删除任务 ===
time.sleep(5)
scheduler.remove_job('task_b')
print("任务B 已删除")

# === 动态添加 ===
scheduler.add_job(my_task, 'interval', seconds=4,
                  args=['新任务C'], id='task_c')
print("新任务C 已添加")

time.sleep(10)
scheduler.shutdown()
```

## 5. 进阶用法

### 5.1 任务持久化

```python
#!/usr/bin/env python3
"""任务持久化 — SQLAlchemy Job Store"""

from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
from apscheduler.executors.pool import ThreadPoolExecutor, ProcessPoolExecutor
from datetime import datetime

# === 配置持久化 ===
jobstores = {
    'default': SQLAlchemyJobStore(url='sqlite:///tmp/sre_jobs.db'),
    # 也可以用其他数据库:
    # 'default': SQLAlchemyJobStore(url='postgresql://user:pass@localhost/scheduler')
    # 'redis': RedisJobStore(host='localhost', port=6379)
}

executors = {
    'default': ThreadPoolExecutor(20),         # 线程池，20 个线程
    'processpool': ProcessPoolExecutor(5),     # 进程池，5 个进程
}

job_defaults = {
    'coalesce': True,           # 错过的任务合并执行一次
    'max_instances': 3,         # 同一任务最多 3 个实例并行
    'misfire_grace_time': 60,   # 错过 60 秒内仍执行
}

scheduler = BackgroundScheduler(
    jobstores=jobstores,
    executors=executors,
    job_defaults=job_defaults,
    timezone='Asia/Shanghai',
)

def persistent_task():
    """持久化的任务 — 即使重启也会保留"""
    print(f"  [{datetime.now().strftime('%H:%M:%S')}] 持久化任务执行")

# 添加持久化任务（replace_existing=True 防止重复添加）
scheduler.add_job(
    persistent_task,
    'interval',
    minutes=5,
    id='persistent_check',
    name='持久化巡检',
    replace_existing=True,  # 重要：防止重启后重复添加
)

scheduler.start()

print("持久化调度器已启动")
print("任务存储在: /tmp/sre_jobs.db")
print("重启程序后任务仍会保留")

import time
time.sleep(3)
scheduler.shutdown()
```

### 5.2 事件监听

```python
#!/usr/bin/env python3
"""事件监听 — 任务执行监控"""

from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.events import (
    EVENT_JOB_EXECUTED, EVENT_JOB_ERROR, EVENT_JOB_MISSED,
    EVENT_JOB_ADDED, EVENT_JOB_REMOVED,
)
from datetime import datetime
import time

scheduler = BackgroundScheduler()

# === 事件监听器 ===
def job_executed_listener(event):
    """任务执行成功"""
    job = scheduler.get_job(event.job_id)
    job_name = job.name if job else event.job_id
    print(f"  [成功] {job_name} 执行完成 "
          f"(耗时: {event.retval if hasattr(event, 'retval') else 'N/A'})")

def job_error_listener(event):
    """任务执行失败"""
    print(f"  [失败] {event.job_id}: {event.exception}")

def job_missed_listener(event):
    """任务错过执行"""
    print(f"  [错过] {event.job_id} 错过了预定执行时间")

def job_lifecycle_listener(event):
    """任务生命周期"""
    if hasattr(event, 'job_id'):
        print(f"  [事件] Job {event.job_id}: {type(event).__name__}")

# 注册监听器
scheduler.add_listener(job_executed_listener, EVENT_JOB_EXECUTED)
scheduler.add_listener(job_error_listener, EVENT_JOB_ERROR)
scheduler.add_listener(job_missed_listener, EVENT_JOB_MISSED)
scheduler.add_listener(job_lifecycle_listener, EVENT_JOB_ADDED | EVENT_JOB_REMOVED)

# 正常任务
def normal_task():
    return "OK"

# 会失败的任务
def failing_task():
    raise RuntimeError("模拟错误")

scheduler.add_job(normal_task, 'interval', seconds=3, id='normal', name='正常任务')
scheduler.add_job(failing_task, 'interval', seconds=5, id='failing', name='失败任务')

scheduler.start()
print("监听器已注册，运行 15 秒...")

time.sleep(15)
scheduler.shutdown()
```

## 6. SRE 实战案例：定时巡检调度系统

```python
#!/usr/bin/env python3
"""
SRE 实战：定时巡检调度系统
功能：
1. 多种巡检任务注册
2. 灵活的调度策略（cron/interval）
3. 任务执行监控与告警
4. 执行历史记录
5. 动态任务管理（添加/暂停/删除）
6. 巡检报告生成
"""

import json
import time
import socket
import threading
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional, Callable
from dataclasses import dataclass, field, asdict
from collections import defaultdict

from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
from apscheduler.events import (
    EVENT_JOB_EXECUTED, EVENT_JOB_ERROR, EVENT_JOB_MISSED
)

# ============================================================
# 数据模型
# ============================================================
@dataclass
class InspectionResult:
    """巡检结果"""
    job_id: str
    job_name: str
    status: str           # ok / warning / critical / error
    message: str
    details: Dict = field(default_factory=dict)
    duration_ms: float = 0
    timestamp: str = ""

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

@dataclass
class InspectionConfig:
    """巡检任务配置"""
    job_id: str
    name: str
    description: str
    trigger_type: str     # cron / interval
    trigger_args: Dict    # 触发器参数
    task_func: str        # 任务函数名
    task_args: Dict = field(default_factory=dict)
    enabled: bool = True
    alert_on_failure: bool = True

# ============================================================
# 巡检任务集
# ============================================================
class InspectionTasks:
    """内置巡检任务"""

    @staticmethod
    def check_port(host: str, port: int, timeout: float = 5) -> dict:
        """端口连通性检查"""
        start = time.time()
        try:
            sock = socket.create_connection((host, port), timeout=timeout)
            sock.close()
            latency = (time.time() - start) * 1000
            return {
                'status': 'ok',
                'message': f'{host}:{port} 连通',
                'latency_ms': round(latency, 2),
            }
        except Exception as e:
            return {
                'status': 'critical',
                'message': f'{host}:{port} 不可达: {e}',
                'latency_ms': 0,
            }

    @staticmethod
    def check_disk(path: str = '/', threshold: float = 90) -> dict:
        """磁盘使用率检查"""
        import shutil
        total, used, free = shutil.disk_usage(path)
        usage = used / total * 100

        if usage >= threshold:
            status = 'critical'
        elif usage >= threshold - 10:
            status = 'warning'
        else:
            status = 'ok'

        return {
            'status': status,
            'message': f'磁盘使用率: {usage:.1f}%',
            'total_gb': round(total / (1024**3), 2),
            'used_gb': round(used / (1024**3), 2),
            'free_gb': round(free / (1024**3), 2),
            'usage_percent': round(usage, 1),
        }

    @staticmethod
    def check_dns(domain: str, expected_ip: str = None) -> dict:
        """DNS 解析检查"""
        try:
            import socket as sock_module
            start = time.time()
            result = sock_module.getaddrinfo(domain, None)
            latency = (time.time() - start) * 1000
            ips = list(set(addr[4][0] for addr in result))

            status = 'ok'
            message = f'{domain} → {ips}'

            if expected_ip and expected_ip not in ips:
                status = 'warning'
                message += f' (期望 {expected_ip} 但未找到)'

            return {
                'status': status,
                'message': message,
                'resolved_ips': ips,
                'latency_ms': round(latency, 2),
            }
        except Exception as e:
            return {
                'status': 'critical',
                'message': f'DNS 解析失败: {domain}: {e}',
            }

    @staticmethod
    def check_memory(threshold: float = 90) -> dict:
        """内存使用检查（简化版）"""
        try:
            with open('/proc/meminfo', 'r') as f:
                lines = f.readlines()
            mem_info = {}
            for line in lines:
                parts = line.split(':')
                if len(parts) == 2:
                    key = parts[0].strip()
                    value = int(parts[1].strip().split()[0])
                    mem_info[key] = value

            total = mem_info.get('MemTotal', 0)
            available = mem_info.get('MemAvailable', 0)
            used = total - available
            usage = (used / total * 100) if total else 0

            status = 'critical' if usage >= threshold else 'warning' if usage >= threshold - 10 else 'ok'
            return {
                'status': status,
                'message': f'内存使用率: {usage:.1f}%',
                'total_mb': round(total / 1024, 1),
                'used_mb': round(used / 1024, 1),
                'usage_percent': round(usage, 1),
            }
        except Exception as e:
            return {
                'status': 'error',
                'message': f'内存检查失败: {e}',
            }

# ============================================================
# 巡检调度系统
# ============================================================
class InspectionScheduler:
    """SRE 定时巡检调度系统"""

    def __init__(self):
        self.scheduler = BackgroundScheduler(
            timezone='Asia/Shanghai',
            job_defaults={
                'coalesce': True,
                'max_instances': 1,
                'misfire_grace_time': 30,
            }
        )
        self.tasks = InspectionTasks()
        self.history: List[InspectionResult] = []
        self.max_history = 10000
        self._lock = threading.Lock()

        # 注册事件监听
        self.scheduler.add_listener(self._on_job_executed, EVENT_JOB_EXECUTED)
        self.scheduler.add_listener(self._on_job_error, EVENT_JOB_ERROR)
        self.scheduler.add_listener(self._on_job_missed, EVENT_JOB_MISSED)

        # 任务函数映射
        self.task_registry: Dict[str, Callable] = {
            'check_port': self.tasks.check_port,
            'check_disk': self.tasks.check_disk,
            'check_dns': self.tasks.check_dns,
            'check_memory': self.tasks.check_memory,
        }

    def _record_result(self, result: InspectionResult):
        """记录巡检结果"""
        with self._lock:
            self.history.append(result)
            if len(self.history) > self.max_history:
                self.history = self.history[-self.max_history:]

    def _on_job_executed(self, event):
        """任务执行成功回调"""
        job = self.scheduler.get_job(event.job_id)
        if job and hasattr(event, 'retval') and isinstance(event.retval, dict):
            result = InspectionResult(
                job_id=event.job_id,
                job_name=job.name or event.job_id,
                status=event.retval.get('status', 'unknown'),
                message=event.retval.get('message', ''),
                details=event.retval,
            )
            self._record_result(result)
            if result.status in ('critical', 'error'):
                self._send_alert(result)

    def _on_job_error(self, event):
        """任务执行失败回调"""
        result = InspectionResult(
            job_id=event.job_id,
            job_name=event.job_id,
            status='error',
            message=f'任务异常: {event.exception}',
        )
        self._record_result(result)
        self._send_alert(result)

    def _on_job_missed(self, event):
        """任务错过执行回调"""
        result = InspectionResult(
            job_id=event.job_id,
            job_name=event.job_id,
            status='warning',
            message='任务错过预定执行时间',
        )
        self._record_result(result)

    def _send_alert(self, result: InspectionResult):
        """发送告警（模拟）"""
        print(f"  [告警] [{result.status.upper()}] {result.job_name}: {result.message}")

    def _create_wrapper(self, task_func: Callable, **task_args) -> Callable:
        """创建任务包装器"""
        def wrapper():
            return task_func(**task_args)
        return wrapper

    def add_inspection(self, config: InspectionConfig):
        """添加巡检任务"""
        if not config.enabled:
            return

        task_func = self.task_registry.get(config.task_func)
        if not task_func:
            print(f"  [错误] 未知任务: {config.task_func}")
            return

        wrapper = self._create_wrapper(task_func, **config.task_args)

        if config.trigger_type == 'interval':
            trigger = IntervalTrigger(**config.trigger_args)
        elif config.trigger_type == 'cron':
            trigger = CronTrigger(**config.trigger_args)
        else:
            print(f"  [错误] 未知触发器: {config.trigger_type}")
            return

        self.scheduler.add_job(
            wrapper,
            trigger=trigger,
            id=config.job_id,
            name=config.name,
            replace_existing=True,
        )
        print(f"  [添加] {config.job_id}: {config.name} ({config.trigger_type})")

    def remove_inspection(self, job_id: str):
        """删除巡检任务"""
        try:
            self.scheduler.remove_job(job_id)
            print(f"  [删除] {job_id}")
        except Exception as e:
            print(f"  [错误] 删除失败: {e}")

    def pause_inspection(self, job_id: str):
        """暂停巡检任务"""
        self.scheduler.pause_job(job_id)
        print(f"  [暂停] {job_id}")

    def resume_inspection(self, job_id: str):
        """恢复巡检任务"""
        self.scheduler.resume_job(job_id)
        print(f"  [恢复] {job_id}")

    def start(self):
        """启动调度器"""
        self.scheduler.start()
        print("巡检调度器已启动")

    def stop(self):
        """停止调度器"""
        self.scheduler.shutdown()
        print("巡检调度器已停止")

    def list_jobs(self) -> List[Dict]:
        """列出所有任务"""
        jobs = []
        for job in self.scheduler.get_jobs():
            jobs.append({
                'id': job.id,
                'name': job.name,
                'trigger': str(job.trigger),
                'next_run': str(job.next_run_time),
                'pending': job.pending,
            })
        return jobs

    def get_history(self, job_id: str = None, limit: int = 50) -> List[Dict]:
        """获取执行历史"""
        with self._lock:
            results = self.history
            if job_id:
                results = [r for r in results if r.job_id == job_id]
            return [asdict(r) for r in results[-limit:]]

    def report(self) -> str:
        """生成巡检报告"""
        lines = [
            f"{'='*60}",
            f"SRE 巡检调度报告",
            f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"{'='*60}",
        ]

        # 任务列表
        jobs = self.list_jobs()
        lines.append(f"\n--- 已注册任务 ({len(jobs)}) ---")
        for job in jobs:
            lines.append(f"  {job['id']:30s} | {job['trigger']:30s} | 下次: {job['next_run']}")

        # 历史统计
        with self._lock:
            if self.history:
                recent = self.history[-100:]
                status_counts = defaultdict(int)
                for r in recent:
                    status_counts[r.status] += 1

                lines.append(f"\n--- 最近 {len(recent)} 次执行 ---")
                for status, count in sorted(status_counts.items()):
                    pct = count / len(recent) * 100
                    lines.append(f"  {status}: {count} ({pct:.1f}%)")

                # 最近的异常
                errors = [r for r in recent if r.status in ('critical', 'error')]
                if errors:
                    lines.append(f"\n--- 最近异常 ({len(errors)}) ---")
                    for e in errors[-5:]:
                        lines.append(f"  [{e.timestamp}] {e.job_name}: {e.message}")

        return '\n'.join(lines)


# ============================================================
# 演示
# ============================================================
def main():
    # 初始化调度系统
    scheduler = InspectionScheduler()

    # 定义巡检任务
    inspections = [
        InspectionConfig(
            job_id='port_check_local',
            name='本地端口检查',
            description='检查本地 SSH 端口',
            trigger_type='interval',
            trigger_args={'seconds': 5},
            task_func='check_port',
            task_args={'host': '127.0.0.1', 'port': 22},
        ),
        InspectionConfig(
            job_id='disk_check_root',
            name='根目录磁盘检查',
            description='检查根分区使用率',
            trigger_type='interval',
            trigger_args={'seconds': 8},
            task_func='check_disk',
            task_args={'path': '/', 'threshold': 90},
        ),
        InspectionConfig(
            job_id='dns_check_google',
            name='DNS 解析检查',
            description='检查 google.com 解析',
            trigger_type='interval',
            trigger_args={'seconds': 10},
            task_func='check_dns',
            task_args={'domain': 'google.com'},
        ),
        InspectionConfig(
            job_id='memory_check',
            name='内存使用检查',
            description='检查内存使用率',
            trigger_type='interval',
            trigger_args={'seconds': 6},
            task_func='check_memory',
            task_args={'threshold': 90},
        ),
    ]

    # 添加所有巡检任务
    print("=== 注册巡检任务 ===")
    for config in inspections:
        scheduler.add_inspection(config)

    # 列出任务
    print(f"\n=== 任务列表 ===")
    for job in scheduler.list_jobs():
        print(f"  {job['id']}: {job['name']} → {job['next_run']}")

    # 启动调度器
    scheduler.start()

    # 运行一段时间
    print(f"\n运行 30 秒...")
    try:
        time.sleep(30)
    except KeyboardInterrupt:
        pass

    # 生成报告
    print(scheduler.report())

    # 查看历史
    print(f"\n=== 最近执行历史 ===")
    for entry in scheduler.get_history(limit=10):
        print(f"  [{entry['status']:8s}] {entry['job_name']}: {entry['message']}")

    # 停止
    scheduler.stop()

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **任务重复添加**：重启时任务可能重复注册，用 `replace_existing=True`
2. **时区问题**：默认 UTC，配置 `timezone='Asia/Shanghai'`
3. **内存 Job Store**：默认存内存，重启丢失，生产用 SQLAlchemy
4. **任务阻塞**：耗时任务阻塞线程池，设置合理的 `max_instances`
5. **错过执行**：系统休眠或负载高时任务可能错过，配置 `misfire_grace_time`

### 最佳实践

```python
# 1. 使用持久化 Job Store
jobstores = {'default': SQLAlchemyJobStore(url='sqlite:///jobs.db')}

# 2. 设置时区
scheduler = BackgroundScheduler(timezone='Asia/Shanghai')

# 3. 防止重复添加
scheduler.add_job(func, ..., replace_existing=True)

# 4. 合理设置并发和超时
job_defaults = {
    'coalesce': True,         # 错过的合并执行
    'max_instances': 1,        # 防止堆积
    'misfire_grace_time': 60,  # 错过容忍时间
}

# 5. 注册事件监听器做告警
# 6. 耗时任务用进程池执行器
```

## 8. 速查表

```python
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger

scheduler = BackgroundScheduler()

# === 触发器 ===
# Interval
scheduler.add_job(func, 'interval', seconds=30)
scheduler.add_job(func, 'interval', minutes=5, hours=1)

# Cron
scheduler.add_job(func, 'cron', hour=8, minute=0)
scheduler.add_job(func, 'cron', minute='*/5')
scheduler.add_job(func, 'cron', day_of_week='mon-fri', hour='9-18')
scheduler.add_job(func, CronTrigger.from_crontab('*/5 * * * *'))

# Date（一次性）
scheduler.add_job(func, 'date', run_date='2024-01-15 10:00:00')

# === 管理 ===
scheduler.start()
scheduler.shutdown()
scheduler.get_jobs()                     # 列出任务
scheduler.get_job('job_id')              # 获取任务
scheduler.remove_job('job_id')           # 删除
scheduler.pause_job('job_id')            # 暂停
scheduler.resume_job('job_id')           # 恢复
scheduler.reschedule_job('job_id', ...)  # 重新调度

# === 事件 ===
from apscheduler.events import EVENT_JOB_EXECUTED, EVENT_JOB_ERROR
scheduler.add_listener(callback, EVENT_JOB_EXECUTED | EVENT_JOB_ERROR)

# === 对比 crontab ===
# crontab -e → scheduler.add_job(...)
# crontab -l → scheduler.get_jobs()
# crontab -r → scheduler.remove_all_jobs()
```
