# Prometheus 客户端 — 用 Python 为应用埋点并暴露可观测指标

## 1. 是什么 & 为什么用它

`prometheus_client` 是 Prometheus 官方提供的 Python 客户端库，允许你在应用中定义和暴露自定义指标，
供 Prometheus Server 定时拉取（Pull 模型）。对 SRE 而言，它是实现"应用白盒监控"的基础组件。

**核心价值：**
- 让应用主动暴露内部运行状态（请求量、延迟、队列深度、错误率等）
- 配合 Prometheus + Grafana 形成完整的监控-告警-可视化链路
- 支持 Push 模式（通过 Pushgateway），适用于批处理 / CronJob 场景

## 2. 安装与环境

```bash
# 安装 prometheus_client
pip install prometheus-client

# 可选：用于演示的 HTTP 框架
pip install flask requests

# 验证安装
python -c "import prometheus_client; print(prometheus_client.__version__)"
```

**环境要求：**
- Python >= 3.7
- prometheus_client >= 0.17.0
- （可选）Prometheus Server、Grafana、Pushgateway

## 3. 核心概念与原理

### 3.1 Prometheus 指标采集架构

```
+-------------------+        +-----------------------+        +-------------+
|  Python 应用      |  HTTP  |  Prometheus Server     |  查询  |  Grafana    |
|  (埋点 + /metrics)|<-------|  (定时拉取 & 存储)    |<-------|  (可视化)   |
+-------------------+  Pull  +-----------------------+        +-------------+
        |                            |
        |  Push (可选)               |  告警规则
        v                            v
+-------------------+        +-----------------------+
|  Pushgateway      |        |  Alertmanager         |
|  (短生命周期任务) |        |  (告警路由 & 通知)    |
+-------------------+        +-----------------------+
```

### 3.2 四种核心指标类型

```
指标类型        特点                        典型场景
─────────────────────────────────────────────────────────────
Counter         只增不减的计数器            请求总数、错误总数
Gauge           可增可减的仪表盘            CPU 使用率、队列深度、连接数
Histogram       分桶统计分布               请求延迟分布、响应大小
Summary         客户端计算分位数            P50/P90/P99 延迟
```

### 3.3 指标数据模型

```
# 指标名 + 标签 = 唯一时间序列
http_requests_total{method="GET", endpoint="/api/v1/users", status="200"} 1027

# 格式：
# <metric_name>{<label_name>=<label_value>, ...} <value> [<timestamp>]
```

### 3.4 Pull vs Push 模型

```
Pull 模型（默认）：
  Prometheus Server --HTTP GET /metrics--> 应用
  适用：长期运行的服务

Push 模型：
  应用/脚本 --HTTP POST--> Pushgateway <--Pull-- Prometheus
  适用：CronJob、批处理、短生命周期任务
```

## 4. 基础用法

### 4.1 Counter — 计数器

```python
"""
Counter 基础示例：统计请求总数
Counter 只能递增，不能递减，重启后归零
"""
from prometheus_client import Counter, start_http_server
import time
import random

# 创建 Counter 指标
# 参数：指标名、描述、标签列表
request_counter = Counter(
    'myapp_http_requests_total',      # 指标名（推荐：应用名_指标_单位_total）
    'HTTP 请求总数',                   # 描述信息
    ['method', 'endpoint', 'status']  # 标签（用于多维度区分）
)

# 不带标签的简单计数器
error_counter = Counter(
    'myapp_errors_total',
    '应用错误总数'
)

def handle_request():
    """模拟处理请求"""
    method = random.choice(['GET', 'POST'])
    endpoint = random.choice(['/api/users', '/api/orders'])
    status = random.choice(['200', '200', '200', '500'])  # 75% 成功率

    # 使用 labels() 指定标签值，然后 inc() 递增
    request_counter.labels(
        method=method,
        endpoint=endpoint,
        status=status
    ).inc()  # 默认递增 1，也可以 inc(5) 递增指定值

    if status == '500':
        error_counter.inc()

if __name__ == '__main__':
    # 启动 HTTP 服务暴露指标，默认路径 /metrics
    start_http_server(8000)
    print("指标服务已启动：http://localhost:8000/metrics")

    # 模拟持续产生请求
    while True:
        handle_request()
        time.sleep(0.5)
```

### 4.2 Gauge — 仪表盘

```python
"""
Gauge 基础示例：监控系统资源
Gauge 可以任意增减，适合表示当前状态
"""
from prometheus_client import Gauge, start_http_server
import psutil
import time

# 创建 Gauge 指标
cpu_usage = Gauge(
    'myapp_cpu_usage_percent',
    'CPU 使用率百分比'
)

memory_usage = Gauge(
    'myapp_memory_usage_bytes',
    '内存使用量（字节）'
)

# 带标签的 Gauge
disk_usage = Gauge(
    'myapp_disk_usage_percent',
    '磁盘使用率百分比',
    ['mountpoint']
)

# 使用 Gauge 跟踪正在处理的请求数
in_progress = Gauge(
    'myapp_requests_in_progress',
    '正在处理的请求数'
)

def collect_system_metrics():
    """采集系统指标"""
    # set() 直接设置值
    cpu_usage.set(psutil.cpu_percent(interval=1))

    # 获取内存信息
    mem = psutil.virtual_memory()
    memory_usage.set(mem.used)

    # 采集所有磁盘分区的使用率
    for partition in psutil.disk_partitions():
        try:
            usage = psutil.disk_usage(partition.mountpoint)
            disk_usage.labels(mountpoint=partition.mountpoint).set(usage.percent)
        except PermissionError:
            pass

# Gauge 还支持 inc() / dec() 递增递减
# 以及 track_inprogress() 装饰器用于跟踪并发
@in_progress.track_inprogress()
def process_request():
    """模拟处理请求（自动跟踪并发数）"""
    time.sleep(random.uniform(0.1, 0.5))

if __name__ == '__main__':
    import random
    start_http_server(8000)
    print("系统指标服务已启动：http://localhost:8000/metrics")

    while True:
        collect_system_metrics()
        time.sleep(5)
```

### 4.3 Histogram — 直方图

```python
"""
Histogram 基础示例：统计请求延迟分布
自动生成 _bucket / _count / _sum 三组指标
"""
from prometheus_client import Histogram, start_http_server
import time
import random

# 创建 Histogram，指定分桶边界
request_latency = Histogram(
    'myapp_request_duration_seconds',
    'HTTP 请求延迟（秒）',
    ['endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
    # 默认 buckets: (.005, .01, .025, .05, .075, .1, .25, .5, .75, 1, 2.5, 5, 7.5, 10, +Inf)
)

# 方式一：使用 observe() 直接记录值
def handle_api_users():
    start = time.time()
    # ... 处理业务逻辑 ...
    time.sleep(random.uniform(0.01, 0.5))
    duration = time.time() - start
    request_latency.labels(endpoint='/api/users').observe(duration)

# 方式二：使用 time() 装饰器自动计时
@request_latency.labels(endpoint='/api/orders').time()
def handle_api_orders():
    time.sleep(random.uniform(0.05, 1.0))

# 方式三：使用上下文管理器
def handle_api_products():
    with request_latency.labels(endpoint='/api/products').time():
        time.sleep(random.uniform(0.02, 0.3))

if __name__ == '__main__':
    start_http_server(8000)
    print("延迟指标服务已启动：http://localhost:8000/metrics")

    handlers = [handle_api_users, handle_api_orders, handle_api_products]
    while True:
        random.choice(handlers)()
```

### 4.4 Summary — 摘要

```python
"""
Summary 示例：在客户端计算分位数
注意：Summary 的分位数无法跨实例聚合，生产中推荐用 Histogram
"""
from prometheus_client import Summary, start_http_server
import time
import random

# 创建 Summary
request_summary = Summary(
    'myapp_request_processing_seconds',
    '请求处理时间摘要'
)

# 使用装饰器自动计时
@request_summary.time()
def process_request():
    time.sleep(random.uniform(0.01, 0.5))

if __name__ == '__main__':
    start_http_server(8000)
    print("Summary 指标服务已启动：http://localhost:8000/metrics")
    while True:
        process_request()
```

## 5. 进阶用法

### 5.1 自定义 Collector（Exporter 模式）

```python
"""
自定义 Collector：适用于从外部系统采集指标
比如从数据库、Redis、消息队列等采集状态数据
"""
from prometheus_client.core import GaugeMetricFamily, CounterMetricFamily, REGISTRY
from prometheus_client import start_http_server
import time
import random

class DatabaseCollector:
    """
    自定义数据库指标采集器
    每次 Prometheus 抓取时，collect() 方法被调用
    """

    def collect(self):
        """
        必须实现 collect() 方法，返回指标对象的迭代器
        每次调用都应该获取最新数据
        """
        # 模拟从数据库获取连接池状态
        pool_stats = self._get_pool_stats()

        # Gauge 类型指标
        connections = GaugeMetricFamily(
            'mydb_pool_connections',
            '数据库连接池连接数',
            labels=['state']
        )
        connections.add_metric(['active'], pool_stats['active'])
        connections.add_metric(['idle'], pool_stats['idle'])
        connections.add_metric(['total'], pool_stats['total'])
        yield connections

        # Counter 类型指标
        queries = CounterMetricFamily(
            'mydb_queries_total',
            '数据库查询总数',
            labels=['type']
        )
        query_stats = self._get_query_stats()
        queries.add_metric(['select'], query_stats['select'])
        queries.add_metric(['insert'], query_stats['insert'])
        queries.add_metric(['update'], query_stats['update'])
        queries.add_metric(['delete'], query_stats['delete'])
        yield queries

        # 慢查询数量
        slow_queries = GaugeMetricFamily(
            'mydb_slow_queries',
            '当前慢查询数量'
        )
        slow_queries.add_metric([], random.randint(0, 5))
        yield slow_queries

    def _get_pool_stats(self):
        """模拟获取连接池状态"""
        active = random.randint(5, 20)
        idle = random.randint(10, 30)
        return {'active': active, 'idle': idle, 'total': active + idle}

    def _get_query_stats(self):
        """模拟获取查询统计"""
        return {
            'select': random.randint(10000, 50000),
            'insert': random.randint(1000, 5000),
            'update': random.randint(500, 2000),
            'delete': random.randint(100, 500),
        }

# 注册自定义采集器
REGISTRY.register(DatabaseCollector())

if __name__ == '__main__':
    start_http_server(8000)
    print("自定义 Exporter 已启动：http://localhost:8000/metrics")
    while True:
        time.sleep(1)
```

### 5.2 与 Flask 集成

```python
"""
将 Prometheus 指标集成到 Flask 应用中
"""
from flask import Flask, request, Response
from prometheus_client import (
    Counter, Histogram, Gauge,
    generate_latest, CONTENT_TYPE_LATEST, REGISTRY
)
import time

app = Flask(__name__)

# 定义指标
REQUEST_COUNT = Counter(
    'flask_http_requests_total',
    'Flask HTTP 请求总数',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'flask_http_request_duration_seconds',
    'Flask HTTP 请求延迟',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'flask_http_requests_active',
    '当前活跃请求数'
)

@app.before_request
def before_request():
    """请求前：记录开始时间，增加活跃计数"""
    request._start_time = time.time()
    ACTIVE_REQUESTS.inc()

@app.after_request
def after_request(response):
    """请求后：记录指标"""
    # 计算延迟
    latency = time.time() - request._start_time

    # 记录请求计数
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.path,
        status=response.status_code
    ).inc()

    # 记录延迟
    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.path
    ).observe(latency)

    ACTIVE_REQUESTS.dec()
    return response

@app.route('/metrics')
def metrics():
    """暴露 Prometheus 指标端点"""
    return Response(
        generate_latest(REGISTRY),
        mimetype=CONTENT_TYPE_LATEST
    )

@app.route('/api/users')
def get_users():
    time.sleep(0.1)  # 模拟业务处理
    return {'users': ['alice', 'bob']}

@app.route('/api/orders')
def get_orders():
    time.sleep(0.2)
    return {'orders': [1, 2, 3]}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 5.3 Pushgateway 推送指标

```python
"""
通过 Pushgateway 推送指标
适用于批处理任务、CronJob 等短生命周期场景
"""
from prometheus_client import (
    Counter, Gauge, Histogram,
    CollectorRegistry, push_to_gateway, pushadd_to_gateway
)
import time
import random

# 为 Pushgateway 创建独立的注册表（避免包含默认进程指标）
registry = CollectorRegistry()

# 定义指标，注册到独立的 registry
job_duration = Gauge(
    'batch_job_duration_seconds',
    '批处理任务执行时间',
    registry=registry
)

job_records_processed = Counter(
    'batch_job_records_processed_total',
    '批处理已处理记录数',
    registry=registry
)

job_last_success = Gauge(
    'batch_job_last_success_timestamp',
    '批处理上次成功时间戳',
    registry=registry
)

def run_batch_job():
    """模拟批处理任务"""
    start_time = time.time()

    # 模拟处理数据
    total_records = random.randint(100, 1000)
    for i in range(total_records):
        time.sleep(0.001)  # 模拟处理
        job_records_processed.inc()

    duration = time.time() - start_time
    job_duration.set(duration)
    job_last_success.set_to_current_time()

    print(f"处理完成：{total_records} 条记录，耗时 {duration:.2f}s")

def push_metrics():
    """推送指标到 Pushgateway"""
    pushgateway_url = 'localhost:9091'

    try:
        # push_to_gateway: 替换同 job 的所有指标
        push_to_gateway(
            pushgateway_url,
            job='daily_data_sync',          # 任务名称
            registry=registry,
            grouping_key={                   # 分组键（可选）
                'instance': 'worker-1',
                'env': 'production'
            }
        )
        print("指标已推送到 Pushgateway")

        # pushadd_to_gateway: 只更新指定指标，不删除已有的
        # pushadd_to_gateway(pushgateway_url, job='daily_data_sync', registry=registry)

    except Exception as e:
        print(f"推送失败：{e}")

if __name__ == '__main__':
    run_batch_job()
    push_metrics()
```

### 5.4 指标命名规范

```python
"""
Prometheus 指标命名最佳实践
"""
from prometheus_client import Counter, Gauge, Histogram, Summary

# === 命名规范 ===
# 格式：<namespace>_<subsystem>_<name>_<unit>
# 示例：myapp_http_requests_total
#       myapp_db_query_duration_seconds

# Counter 必须以 _total 结尾
http_requests_total = Counter('myapp_http_requests_total', 'HTTP 请求总数')
errors_total = Counter('myapp_errors_total', '错误总数')

# Gauge 使用当前状态描述
temperature_celsius = Gauge('myapp_temperature_celsius', '温度（摄氏度）')
queue_length = Gauge('myapp_queue_length', '队列长度')

# Histogram/Summary 使用 _seconds / _bytes 等单位后缀
request_duration_seconds = Histogram('myapp_request_duration_seconds', '请求延迟')
response_size_bytes = Histogram('myapp_response_size_bytes', '响应大小')

# === 标签规范 ===
# 标签名使用 snake_case
# 避免高基数标签（如 user_id、request_id）
# 推荐标签：method, endpoint, status, instance, env
```

## 6. SRE 实战案例：自定义应用指标 Exporter

```python
"""
SRE 实战：完整的应用指标 Exporter
功能：
  1. 监控应用核心业务指标
  2. 监控依赖服务健康状态（数据库、Redis、外部 API）
  3. 自动采集系统资源指标
  4. 提供 /metrics 和 /health 端点
"""
import time
import random
import threading
import logging
import socket
from http.server import HTTPServer, BaseHTTPRequestHandler
from prometheus_client import (
    Counter, Gauge, Histogram, Summary, Info,
    CollectorRegistry, generate_latest, CONTENT_TYPE_LATEST,
    REGISTRY
)
from prometheus_client.core import GaugeMetricFamily

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger(__name__)

# ============================================================
# 第一部分：业务指标定义
# ============================================================

# 应用信息（Info 类型，提供应用元数据）
APP_INFO = Info('myapp', '应用信息')
APP_INFO.info({
    'version': '2.1.0',
    'environment': 'production',
    'hostname': socket.gethostname()
})

# 请求指标
HTTP_REQUESTS = Counter(
    'myapp_http_requests_total',
    'HTTP 请求总数',
    ['method', 'endpoint', 'status_code']
)

HTTP_REQUEST_DURATION = Histogram(
    'myapp_http_request_duration_seconds',
    'HTTP 请求延迟分布',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

HTTP_REQUEST_SIZE = Summary(
    'myapp_http_request_size_bytes',
    'HTTP 请求体大小'
)

ACTIVE_CONNECTIONS = Gauge(
    'myapp_active_connections',
    '当前活跃连接数'
)

# 业务指标
ORDER_CREATED = Counter(
    'myapp_orders_created_total',
    '订单创建总数',
    ['payment_method']
)

ORDER_AMOUNT = Histogram(
    'myapp_order_amount_yuan',
    '订单金额分布（元）',
    buckets=[10, 50, 100, 500, 1000, 5000, 10000]
)

# 依赖服务健康状态
DEPENDENCY_UP = Gauge(
    'myapp_dependency_up',
    '依赖服务状态（1=正常，0=异常）',
    ['service']
)

DEPENDENCY_LATENCY = Gauge(
    'myapp_dependency_latency_seconds',
    '依赖服务延迟',
    ['service']
)

# ============================================================
# 第二部分：自定义 Collector — 系统资源采集
# ============================================================

class SystemResourceCollector:
    """采集系统资源指标（每次 scrape 时实时获取）"""

    def collect(self):
        """采集 CPU、内存、磁盘、网络指标"""
        try:
            import psutil

            # CPU 使用率
            cpu = GaugeMetricFamily(
                'myapp_system_cpu_percent',
                'CPU 使用率百分比',
                labels=['cpu']
            )
            # 每个 CPU 核心
            per_cpu = psutil.cpu_percent(percpu=True)
            for i, pct in enumerate(per_cpu):
                cpu.add_metric([f'cpu{i}'], pct)
            cpu.add_metric(['total'], psutil.cpu_percent())
            yield cpu

            # 内存
            mem = psutil.virtual_memory()
            memory = GaugeMetricFamily(
                'myapp_system_memory_bytes',
                '系统内存（字节）',
                labels=['type']
            )
            memory.add_metric(['total'], mem.total)
            memory.add_metric(['used'], mem.used)
            memory.add_metric(['available'], mem.available)
            memory.add_metric(['cached'], mem.cached)
            yield memory

            mem_pct = GaugeMetricFamily(
                'myapp_system_memory_percent',
                '内存使用率百分比'
            )
            mem_pct.add_metric([], mem.percent)
            yield mem_pct

            # 磁盘
            disk = GaugeMetricFamily(
                'myapp_system_disk_usage_percent',
                '磁盘使用率百分比',
                labels=['mountpoint']
            )
            for part in psutil.disk_partitions():
                try:
                    usage = psutil.disk_usage(part.mountpoint)
                    disk.add_metric([part.mountpoint], usage.percent)
                except (PermissionError, OSError):
                    pass
            yield disk

            # 打开文件描述符数
            fd_count = GaugeMetricFamily(
                'myapp_process_open_fds',
                '进程打开的文件描述符数'
            )
            import os
            try:
                proc = psutil.Process(os.getpid())
                fd_count.add_metric([], proc.num_fds())
            except Exception:
                pass
            yield fd_count

        except ImportError:
            logger.warning("psutil 未安装，跳过系统资源采集")

# 注册系统资源采集器
REGISTRY.register(SystemResourceCollector())

# ============================================================
# 第三部分：依赖服务健康检查
# ============================================================

class DependencyChecker:
    """定期检查依赖服务健康状态"""

    def __init__(self, check_interval=30):
        self.check_interval = check_interval
        self._running = False

    def start(self):
        self._running = True
        thread = threading.Thread(target=self._run, daemon=True)
        thread.start()
        logger.info("依赖服务健康检查已启动")

    def _run(self):
        while self._running:
            self._check_database()
            self._check_redis()
            self._check_external_api()
            time.sleep(self.check_interval)

    def _check_database(self):
        """检查数据库连接"""
        try:
            start = time.time()
            # 模拟数据库 ping（实际使用时替换为真实连接检查）
            time.sleep(random.uniform(0.001, 0.01))
            latency = time.time() - start

            # 模拟偶尔失败
            is_up = random.random() > 0.05  # 95% 成功率

            DEPENDENCY_UP.labels(service='database').set(1 if is_up else 0)
            DEPENDENCY_LATENCY.labels(service='database').set(latency)
        except Exception as e:
            DEPENDENCY_UP.labels(service='database').set(0)
            logger.error(f"数据库健康检查失败：{e}")

    def _check_redis(self):
        """检查 Redis 连接"""
        try:
            start = time.time()
            time.sleep(random.uniform(0.0005, 0.005))
            latency = time.time() - start

            is_up = random.random() > 0.02
            DEPENDENCY_UP.labels(service='redis').set(1 if is_up else 0)
            DEPENDENCY_LATENCY.labels(service='redis').set(latency)
        except Exception as e:
            DEPENDENCY_UP.labels(service='redis').set(0)
            logger.error(f"Redis 健康检查失败：{e}")

    def _check_external_api(self):
        """检查外部 API"""
        try:
            start = time.time()
            time.sleep(random.uniform(0.01, 0.1))
            latency = time.time() - start

            is_up = random.random() > 0.1
            DEPENDENCY_UP.labels(service='payment_api').set(1 if is_up else 0)
            DEPENDENCY_LATENCY.labels(service='payment_api').set(latency)
        except Exception as e:
            DEPENDENCY_UP.labels(service='payment_api').set(0)

# ============================================================
# 第四部分：HTTP 服务器（暴露 /metrics 和 /health）
# ============================================================

class MetricsHandler(BaseHTTPRequestHandler):
    """处理 /metrics 和 /health 请求"""

    def do_GET(self):
        if self.path == '/metrics':
            self._handle_metrics()
        elif self.path == '/health':
            self._handle_health()
        else:
            self.send_response(404)
            self.end_headers()

    def _handle_metrics(self):
        """返回 Prometheus 格式指标"""
        output = generate_latest(REGISTRY)
        self.send_response(200)
        self.send_header('Content-Type', CONTENT_TYPE_LATEST)
        self.end_headers()
        self.wfile.write(output)

    def _handle_health(self):
        """健康检查端点"""
        import json
        health = {
            'status': 'healthy',
            'timestamp': time.time(),
            'hostname': socket.gethostname()
        }
        body = json.dumps(health).encode()
        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format, *args):
        """静默日志（避免刷屏）"""
        pass

# ============================================================
# 第五部分：模拟业务流量
# ============================================================

def simulate_traffic():
    """模拟业务流量，产生指标数据"""
    endpoints = [
        ('/api/users', 'GET', 0.05),
        ('/api/orders', 'GET', 0.1),
        ('/api/orders', 'POST', 0.2),
        ('/api/products', 'GET', 0.03),
        ('/api/payment', 'POST', 0.5),
    ]

    while True:
        endpoint, method, base_latency = random.choice(endpoints)

        # 模拟请求延迟
        latency = base_latency * random.uniform(0.5, 3.0)
        # 偶尔出现慢请求
        if random.random() < 0.05:
            latency *= 10

        status = random.choices(
            ['200', '201', '400', '404', '500'],
            weights=[70, 10, 8, 7, 5]
        )[0]

        # 记录指标
        HTTP_REQUESTS.labels(method=method, endpoint=endpoint, status_code=status).inc()
        HTTP_REQUEST_DURATION.labels(method=method, endpoint=endpoint).observe(latency)
        HTTP_REQUEST_SIZE.observe(random.randint(100, 5000))

        # 模拟活跃连接数波动
        ACTIVE_CONNECTIONS.set(random.randint(10, 200))

        # 模拟订单创建
        if endpoint == '/api/orders' and method == 'POST' and status in ['200', '201']:
            payment = random.choice(['alipay', 'wechat', 'card'])
            ORDER_CREATED.labels(payment_method=payment).inc()
            ORDER_AMOUNT.observe(random.uniform(10, 5000))

        time.sleep(random.uniform(0.1, 0.5))

# ============================================================
# 第六部分：主入口
# ============================================================

def main():
    """启动 Exporter"""
    port = 8000

    # 启动依赖检查
    checker = DependencyChecker(check_interval=15)
    checker.start()

    # 启动流量模拟（后台线程）
    traffic_thread = threading.Thread(target=simulate_traffic, daemon=True)
    traffic_thread.start()

    # 启动 HTTP 服务
    server = HTTPServer(('0.0.0.0', port), MetricsHandler)
    logger.info(f"Exporter 已启动：http://0.0.0.0:{port}/metrics")
    logger.info(f"健康检查：http://0.0.0.0:{port}/health")

    try:
        server.serve_forever()
    except KeyboardInterrupt:
        logger.info("正在关闭 Exporter...")
        server.shutdown()

if __name__ == '__main__':
    main()
```

### 配套 Prometheus 配置

```yaml
# prometheus.yml — 配置 Prometheus 抓取上面的 Exporter
global:
  scrape_interval: 15s        # 全局抓取间隔
  evaluation_interval: 15s    # 告警规则评估间隔

scrape_configs:
  - job_name: 'myapp-exporter'
    static_configs:
      - targets: ['localhost:8000']
        labels:
          env: 'production'
          team: 'sre'

  # Pushgateway 配置
  - job_name: 'pushgateway'
    honor_labels: true         # 保留 push 时的标签
    static_configs:
      - targets: ['localhost:9091']
```

### 配套 Grafana Dashboard JSON 片段

```json
{
  "title": "应用指标大盘",
  "panels": [
    {
      "title": "QPS（每秒请求数）",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(myapp_http_requests_total[5m])",
          "legendFormat": "{{method}} {{endpoint}} {{status_code}}"
        }
      ]
    },
    {
      "title": "P99 延迟",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, rate(myapp_http_request_duration_seconds_bucket[5m]))",
          "legendFormat": "{{endpoint}}"
        }
      ]
    },
    {
      "title": "错误率",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(myapp_http_requests_total{status_code=~'5..'}[5m])) / sum(rate(myapp_http_requests_total[5m])) * 100"
        }
      ]
    }
  ]
}
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 指标爆炸（高基数） | 标签值过多（如 user_id） | 避免高基数标签，使用分桶代替 |
| 重启后计数器归零 | Counter 是进程内存指标 | 使用 `rate()` 而非绝对值 |
| 多进程模式指标重复 | fork 后各进程独立注册 | 使用 `multiprocess` 模式 |
| Summary 无法聚合 | 分位数不能跨实例聚合 | 用 Histogram 代替 |
| Pushgateway 指标残留 | push 后不会自动清理 | 任务完成后主动删除或设置 TTL |

### 7.2 多进程模式（Gunicorn）

```python
"""
Gunicorn 多进程模式下使用 prometheus_client
"""
import os
from prometheus_client import multiprocess, CollectorRegistry, generate_latest

# 必须设置环境变量
# export PROMETHEUS_MULTIPROC_DIR=/tmp/prometheus_multiproc

def app(environ, start_response):
    """WSGI 应用"""
    if environ['PATH_INFO'] == '/metrics':
        registry = CollectorRegistry()
        multiprocess.MultiProcessCollector(registry)
        data = generate_latest(registry)
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return [data]

# gunicorn 配置文件 gunicorn.conf.py
def child_exit(server, worker):
    """worker 退出时清理指标文件"""
    multiprocess.mark_process_dead(worker.pid)
```

### 7.3 最佳实践清单

```
[✓] 指标命名遵循 <namespace>_<subsystem>_<name>_<unit> 规范
[✓] Counter 以 _total 结尾，时间单位用 _seconds，大小用 _bytes
[✓] 避免高基数标签（user_id / request_id / IP 地址）
[✓] 使用 Histogram 而非 Summary（可跨实例聚合）
[✓] 为 Pushgateway 场景创建独立 Registry
[✓] 多进程部署使用 multiprocess 模式
[✓] 指标端点单独配置，不走业务认证
[✓] 为关键业务路径设置 SLI 指标（延迟、错误率、吞吐量）
```

## 8. 速查表

```python
# === 安装 ===
# pip install prometheus-client

# === 四种指标类型 ===
from prometheus_client import Counter, Gauge, Histogram, Summary

# Counter（只增计数器）
c = Counter('name_total', '描述', ['label1'])
c.labels(label1='val').inc()         # +1
c.labels(label1='val').inc(5)        # +5

# Gauge（可增减仪表）
g = Gauge('name', '描述', ['label1'])
g.labels(label1='val').set(42)       # 设置值
g.labels(label1='val').inc()         # +1
g.labels(label1='val').dec()         # -1
g.set_to_current_time()              # 设为当前时间戳

# Histogram（分桶直方图）
h = Histogram('name_seconds', '描述', buckets=[.01, .05, .1, .5, 1])
h.observe(0.3)                       # 记录值
with h.time(): pass                  # 上下文管理器计时
@h.time()                            # 装饰器计时
def func(): pass

# Summary（客户端分位数）
s = Summary('name_seconds', '描述')
s.observe(0.3)

# === 暴露指标 ===
from prometheus_client import start_http_server
start_http_server(8000)              # 启动独立 HTTP 服务

from prometheus_client import generate_latest, REGISTRY
output = generate_latest(REGISTRY)   # 生成指标文本

# === Pushgateway ===
from prometheus_client import push_to_gateway, CollectorRegistry
registry = CollectorRegistry()
push_to_gateway('localhost:9091', job='myjob', registry=registry)

# === 自定义 Collector ===
from prometheus_client.core import GaugeMetricFamily, REGISTRY
class MyCollector:
    def collect(self):
        g = GaugeMetricFamily('name', '描述', labels=['l'])
        g.add_metric(['v'], 42)
        yield g
REGISTRY.register(MyCollector())

# === 常用 PromQL ===
# rate(metric_total[5m])                    -- 每秒速率
# increase(metric_total[1h])               -- 1小时增量
# histogram_quantile(0.99, rate(m_bucket[5m])) -- P99
# sum by (label) (rate(m_total[5m]))       -- 按标签聚合
# absent(up{job="x"})                      -- 目标消失告警
```
