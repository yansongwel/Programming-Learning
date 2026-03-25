# Datadog 与 StatsD — 用 Python 实现应用性能指标上报与全链路可观测

## 1. 是什么 & 为什么用它

**Datadog** 是一个 SaaS 形式的全栈可观测平台，提供指标（Metrics）、日志（Logs）、追踪（APM/Traces）三大支柱的统一管理。
**StatsD** 是一个轻量级的 UDP 指标上报协议，DogStatsD 是 Datadog 对其的扩展实现。

**核心价值：**
- 统一指标、日志、Trace 到一个平台，减少工具碎片化
- DogStatsD 基于 UDP，对应用性能几乎无影响
- 内置丰富的告警、异常检测和 Dashboard 能力
- 对 SRE 而言：开箱即用的基础设施监控 + 灵活的自定义指标

## 2. 安装与环境

```bash
# 安装 Datadog Python 库
pip install datadog

# 安装 DogStatsD 独立客户端（轻量版，只包含 StatsD 功能）
pip install datadog-api-client

# 安装 APM 追踪库
pip install ddtrace

# 验证安装
python -c "import datadog; print('datadog 安装成功')"
python -c "from ddtrace import tracer; print('ddtrace 安装成功')"
```

**环境要求：**
- Python >= 3.7
- Datadog Agent 已部署（接收 DogStatsD 数据，默认监听 UDP 8125）
- Datadog API Key 和 APP Key（用于 API 调用）

```bash
# Datadog Agent 快速安装（Linux）
DD_API_KEY=<你的API_KEY> DD_SITE="datadoghq.com" bash -c \
  "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"

# 或通过 Docker 运行 Agent
docker run -d --name dd-agent \
  -e DD_API_KEY=<你的API_KEY> \
  -e DD_DOGSTATSD_NON_LOCAL_TRAFFIC=true \
  -p 8125:8125/udp \
  -p 8126:8126/tcp \
  datadog/agent:latest
```

## 3. 核心概念与原理

### 3.1 数据流架构

```
+------------------+   UDP:8125   +------------------+   HTTPS   +------------------+
|  Python 应用     |------------->|  Datadog Agent   |---------->|  Datadog 后端    |
|  (DogStatsD 客户端)             |  (本地聚合+转发) |           |  (存储+分析+告警)|
+------------------+              +------------------+           +------------------+
        |                                |                              |
        | ddtrace                        | 日志收集                     |
        | (TCP:8126)                     | (文件/syslog)               v
        v                                v                      +------------------+
   APM Traces                      Log Pipeline               |  Dashboard       |
                                                               |  Alert           |
                                                               |  SLO             |
                                                               +------------------+
```

### 3.2 指标类型对比

```
类型              说明                          DogStatsD 方法
─────────────────────────────────────────────────────────────────
Count             事件计数                      statsd.increment()
Gauge             瞬时值                        statsd.gauge()
Histogram         分布统计（DD扩展）            statsd.histogram()
Distribution      全局分布（服务端聚合）        statsd.distribution()
Set               不重复元素计数                statsd.set()
Rate              每秒速率                      statsd.rate()
Timer             耗时统计                      statsd.timed()
```

### 3.3 DogStatsD 报文格式

```
# 格式：<METRIC_NAME>:<VALUE>|<TYPE>|#<TAG_KEY>:<TAG_VALUE>,<TAG>
# 示例：
myapp.request.count:1|c|#method:get,endpoint:/api/users
myapp.request.latency:0.25|h|#method:get,endpoint:/api/users
myapp.queue.size:42|g|#queue:orders
```

## 4. 基础用法

### 4.1 DogStatsD 指标上报

```python
"""
DogStatsD 基础示例：上报各类型指标
前提：本地运行 Datadog Agent（或用 socat 模拟接收 UDP 数据进行测试）
"""
from datadog import initialize, statsd
import time
import random

# 初始化 DogStatsD 客户端
initialize(
    statsd_host='127.0.0.1',  # Agent 地址
    statsd_port=8125,          # Agent DogStatsD 端口
    statsd_constant_tags=[     # 全局标签，自动附加到所有指标
        'env:production',
        'service:myapp',
        'team:sre'
    ]
)

# === Count：计数器 ===
def record_request(method, endpoint, status_code):
    """记录请求计数"""
    statsd.increment(
        'myapp.http.requests',           # 指标名
        tags=[                           # 标签
            f'method:{method}',
            f'endpoint:{endpoint}',
            f'status:{status_code}'
        ]
    )
    # 也可以一次增加多个
    # statsd.increment('myapp.http.requests', value=5)

    # 递减
    if status_code >= 500:
        statsd.increment('myapp.http.errors')

# === Gauge：瞬时值 ===
def report_queue_depth():
    """上报队列深度"""
    depth = random.randint(0, 100)
    statsd.gauge(
        'myapp.queue.depth',
        depth,
        tags=['queue:task_queue']
    )

# === Histogram：分布统计 ===
def record_latency(endpoint, latency):
    """记录请求延迟分布"""
    statsd.histogram(
        'myapp.request.latency',
        latency,
        tags=[f'endpoint:{endpoint}']
    )
    # Histogram 会自动生成：avg, count, max, median, 95percentile 等

# === Distribution：全局分布 ===
def record_response_size(size):
    """记录响应大小（全局聚合）"""
    statsd.distribution(
        'myapp.response.size',
        size,
        tags=['type:api']
    )

# === Set：不重复计数 ===
def track_unique_users(user_id):
    """统计独立用户数"""
    statsd.set(
        'myapp.unique_users',
        user_id,
        tags=['source:web']
    )

# === Timer：计时器 ===
@statsd.timed('myapp.process.duration', tags=['task:data_sync'])
def process_data():
    """使用装饰器自动计时"""
    time.sleep(random.uniform(0.1, 0.5))

# 上下文管理器方式
def process_order():
    """使用上下文管理器计时"""
    with statsd.timed('myapp.order.process_time', tags=['type:normal']):
        time.sleep(random.uniform(0.05, 0.3))

# === 模拟运行 ===
if __name__ == '__main__':
    print("开始上报指标到 DogStatsD...")
    while True:
        # 模拟请求
        method = random.choice(['GET', 'POST', 'PUT', 'DELETE'])
        endpoint = random.choice(['/api/users', '/api/orders', '/api/products'])
        status = random.choices([200, 201, 400, 404, 500], weights=[60, 10, 10, 10, 10])[0]
        latency = random.uniform(0.01, 2.0)

        record_request(method, endpoint, status)
        record_latency(endpoint, latency)
        report_queue_depth()
        record_response_size(random.randint(100, 10000))
        track_unique_users(f'user_{random.randint(1, 1000)}')
        process_data()

        time.sleep(0.5)
```

### 4.2 Datadog API 调用

```python
"""
通过 Datadog API 提交指标、事件和服务检查
适用于需要更精确控制的场景
"""
from datadog import initialize, api
import time

# 初始化 API 客户端
initialize(
    api_key='YOUR_API_KEY',           # Datadog API Key
    app_key='YOUR_APP_KEY',           # Datadog Application Key
    api_host='https://api.datadoghq.com'  # API 地址
)

# === 提交指标（通过 API）===
def submit_metrics():
    """通过 API 直接提交指标数据点"""
    now = int(time.time())

    api.Metric.send(
        metric='myapp.api.latency',
        points=[
            (now - 60, 0.5),    # 1 分钟前
            (now - 30, 0.3),    # 30 秒前
            (now, 0.2),         # 当前
        ],
        type='gauge',
        tags=['env:production', 'service:myapp'],
        host='web-server-01'
    )

    # 批量提交多个指标
    api.Metric.send([
        {
            'metric': 'myapp.cpu.usage',
            'points': [(now, 65.5)],
            'type': 'gauge',
            'tags': ['host:web-01']
        },
        {
            'metric': 'myapp.mem.usage',
            'points': [(now, 78.2)],
            'type': 'gauge',
            'tags': ['host:web-01']
        }
    ])
    print("指标已提交")

# === 发送事件 ===
def send_event():
    """发送事件到 Datadog Events 流"""
    api.Event.create(
        title='部署完成',
        text='myapp v2.1.0 部署到 production 环境\n\n变更内容：\n- 修复登录超时问题\n- 优化查询性能',
        alert_type='info',         # info / warning / error / success
        tags=[
            'env:production',
            'service:myapp',
            'version:2.1.0'
        ],
        source_type_name='deployment',
        aggregation_key='deploy_myapp'  # 同一 key 的事件会聚合
    )
    print("部署事件已发送")

# === 通过 DogStatsD 发送事件 ===
def send_event_via_statsd():
    """通过 DogStatsD 发送事件（无需 API Key）"""
    from datadog import statsd

    statsd.event(
        title='磁盘空间告警',
        message='服务器 web-01 磁盘使用率达到 90%',
        alert_type='warning',
        tags=['host:web-01', 'check:disk']
    )

# === 服务检查 ===
def send_service_check():
    """发送服务检查结果"""
    from datadog import statsd

    # 状态：0=OK, 1=WARNING, 2=CRITICAL, 3=UNKNOWN
    statsd.service_check(
        'myapp.database.health',
        status=0,  # OK
        tags=['db:primary', 'env:production'],
        message='数据库连接正常，延迟 2ms'
    )

    statsd.service_check(
        'myapp.redis.health',
        status=2,  # CRITICAL
        tags=['redis:cache', 'env:production'],
        message='Redis 连接超时'
    )

if __name__ == '__main__':
    submit_metrics()
    send_event()
    send_service_check()
```

### 4.3 APM 追踪（ddtrace）

```python
"""
ddtrace APM 追踪：自动和手动追踪请求链路
"""
from ddtrace import tracer, patch_all
import time
import random

# 配置 tracer
tracer.configure(
    hostname='127.0.0.1',      # Agent 地址
    port=8126,                  # Agent APM 端口
    enabled=True
)

# 自动 patch 常用库（requests, flask, redis, sqlalchemy 等）
patch_all()

# === 手动创建 Span ===
@tracer.wrap(service='myapp', resource='process_order')
def process_order(order_id):
    """使用装饰器创建 Span"""
    time.sleep(random.uniform(0.05, 0.2))

    # 创建子 Span
    with tracer.trace('validate_order', service='myapp') as span:
        span.set_tag('order.id', order_id)
        span.set_tag('order.type', 'normal')
        time.sleep(random.uniform(0.01, 0.05))

    with tracer.trace('charge_payment', service='payment') as span:
        span.set_tag('payment.method', 'alipay')
        time.sleep(random.uniform(0.1, 0.3))

        # 标记错误
        if random.random() < 0.1:
            span.set_tag('error', True)
            span.set_tag('error.msg', '支付网关超时')
            span.set_tag('error.type', 'TimeoutError')

    with tracer.trace('update_inventory', service='inventory') as span:
        time.sleep(random.uniform(0.02, 0.1))

# === 使用上下文管理器 ===
def handle_request(request_id):
    """手动管理 Span 生命周期"""
    with tracer.trace(
        'http.request',
        service='myapp',
        resource='GET /api/orders',
        span_type='web'
    ) as root_span:
        # 设置标签
        root_span.set_tag('http.method', 'GET')
        root_span.set_tag('http.url', '/api/orders')
        root_span.set_tag('request.id', request_id)

        # 调用下游服务
        process_order(order_id=request_id)

        # 设置响应信息
        root_span.set_tag('http.status_code', 200)

# === 添加自定义指标到 Span ===
def process_with_metrics():
    """在 Span 中附加自定义指标"""
    with tracer.trace('data_processing', service='myapp') as span:
        records = random.randint(100, 10000)
        start = time.time()

        # 模拟处理
        time.sleep(random.uniform(0.1, 0.5))

        duration = time.time() - start
        span.set_metric('records.processed', records)
        span.set_metric('processing.rate', records / duration)

if __name__ == '__main__':
    print("开始 APM 追踪...")
    for i in range(20):
        handle_request(f'req_{i}')
        time.sleep(0.5)
    print("追踪数据已上报")
```

## 5. 进阶用法

### 5.1 自定义 StatsD 客户端封装

```python
"""
封装一个 SRE 友好的指标上报客户端
支持：自动重试、批量发送、降级处理
"""
from datadog import initialize, statsd
import time
import logging
import functools
from contextlib import contextmanager

logger = logging.getLogger(__name__)

class MetricsClient:
    """
    SRE 指标上报客户端
    封装 DogStatsD，提供更便捷的 API
    """

    def __init__(self, service_name, env='production',
                 statsd_host='127.0.0.1', statsd_port=8125):
        """
        初始化指标客户端

        参数：
            service_name: 服务名称
            env: 环境（production/staging/development）
            statsd_host: DogStatsD 地址
            statsd_port: DogStatsD 端口
        """
        self.service_name = service_name
        self.env = env
        self._default_tags = [
            f'service:{service_name}',
            f'env:{env}',
        ]

        initialize(
            statsd_host=statsd_host,
            statsd_port=statsd_port,
            statsd_constant_tags=self._default_tags
        )
        logger.info(f"指标客户端已初始化：{service_name} ({env})")

    def _build_tags(self, extra_tags=None):
        """合并标签"""
        tags = list(self._default_tags)
        if extra_tags:
            tags.extend(extra_tags)
        return tags

    def count(self, metric, value=1, tags=None):
        """计数指标"""
        try:
            statsd.increment(
                f'{self.service_name}.{metric}',
                value=value,
                tags=tags
            )
        except Exception as e:
            logger.warning(f"指标上报失败：{metric} - {e}")

    def gauge(self, metric, value, tags=None):
        """仪表盘指标"""
        try:
            statsd.gauge(f'{self.service_name}.{metric}', value, tags=tags)
        except Exception as e:
            logger.warning(f"指标上报失败：{metric} - {e}")

    def latency(self, metric, value, tags=None):
        """延迟指标（histogram）"""
        try:
            statsd.histogram(f'{self.service_name}.{metric}', value, tags=tags)
        except Exception as e:
            logger.warning(f"指标上报失败：{metric} - {e}")

    @contextmanager
    def timer(self, metric, tags=None):
        """计时上下文管理器"""
        start = time.time()
        error_occurred = False
        try:
            yield
        except Exception:
            error_occurred = True
            raise
        finally:
            duration = time.time() - start
            final_tags = list(tags or [])
            final_tags.append(f'error:{error_occurred}')
            self.latency(metric, duration, tags=final_tags)

    def timed(self, metric, tags=None):
        """计时装饰器"""
        def decorator(func):
            @functools.wraps(func)
            def wrapper(*args, **kwargs):
                with self.timer(metric, tags=tags):
                    return func(*args, **kwargs)
            return wrapper
        return decorator

    def event(self, title, message, alert_type='info', tags=None):
        """发送事件"""
        try:
            statsd.event(
                title=title,
                message=message,
                alert_type=alert_type,
                tags=tags
            )
        except Exception as e:
            logger.warning(f"事件上报失败：{title} - {e}")

    def health_check(self, name, is_healthy, message='', tags=None):
        """上报服务检查结果"""
        try:
            status = 0 if is_healthy else 2  # 0=OK, 2=CRITICAL
            statsd.service_check(
                f'{self.service_name}.{name}',
                status=status,
                tags=tags,
                message=message
            )
        except Exception as e:
            logger.warning(f"健康检查上报失败：{name} - {e}")

# === 使用示例 ===
metrics = MetricsClient('order-service', env='production')

@metrics.timed('api.request', tags=['endpoint:/api/orders'])
def handle_order_request():
    """业务函数，自动记录耗时"""
    time.sleep(0.1)
    metrics.count('orders.created', tags=['type:normal'])
    return {'status': 'ok'}
```

### 5.2 Flask 自动埋点中间件

```python
"""
Flask 应用自动指标上报中间件
自动采集：请求计数、延迟、错误率、活跃连接数
"""
from flask import Flask, request, g
from datadog import initialize, statsd
import time

def init_datadog_middleware(app, service_name='flask-app'):
    """
    为 Flask 应用添加 Datadog 指标中间件

    参数：
        app: Flask 应用实例
        service_name: 服务名称
    """
    initialize(
        statsd_host='127.0.0.1',
        statsd_port=8125,
        statsd_constant_tags=[f'service:{service_name}']
    )

    @app.before_request
    def before_request():
        """请求开始：记录开始时间"""
        g.start_time = time.time()
        statsd.increment(
            f'{service_name}.request.active',
            tags=[f'endpoint:{request.path}']
        )

    @app.after_request
    def after_request(response):
        """请求结束：上报指标"""
        # 计算延迟
        duration = time.time() - g.get('start_time', time.time())

        tags = [
            f'method:{request.method}',
            f'endpoint:{request.path}',
            f'status:{response.status_code}',
        ]

        # 请求计数
        statsd.increment(f'{service_name}.request.count', tags=tags)

        # 请求延迟
        statsd.histogram(f'{service_name}.request.duration', duration, tags=tags)

        # 响应大小
        content_length = response.content_length or 0
        statsd.histogram(f'{service_name}.response.size', content_length, tags=tags)

        # 错误计数
        if response.status_code >= 500:
            statsd.increment(f'{service_name}.request.error', tags=tags)

        statsd.decrement(
            f'{service_name}.request.active',
            tags=[f'endpoint:{request.path}']
        )

        return response

    @app.errorhandler(Exception)
    def handle_exception(e):
        """异常处理：上报异常事件"""
        statsd.increment(
            f'{service_name}.exception',
            tags=[f'type:{type(e).__name__}']
        )
        statsd.event(
            title=f'{service_name} 异常',
            message=str(e),
            alert_type='error',
            tags=[f'type:{type(e).__name__}']
        )
        return {'error': str(e)}, 500

# === 使用示例 ===
app = Flask(__name__)
init_datadog_middleware(app, service_name='order-api')

@app.route('/api/orders')
def get_orders():
    return {'orders': [1, 2, 3]}

@app.route('/api/orders/<int:order_id>')
def get_order(order_id):
    return {'order_id': order_id}
```

### 5.3 Datadog API Client v2

```python
"""
使用 datadog-api-client（v2 API）进行高级操作
"""
from datadog_api_client import Configuration, ApiClient
from datadog_api_client.v1.api.monitors_api import MonitorsApi
from datadog_api_client.v1.model.monitor import Monitor
from datadog_api_client.v1.model.monitor_type import MonitorType
from datadog_api_client.v2.api.metrics_api import MetricsApi
import os

# 配置认证
configuration = Configuration()
configuration.api_key['apiKeyAuth'] = os.getenv('DD_API_KEY', 'your_api_key')
configuration.api_key['appKeyAuth'] = os.getenv('DD_APP_KEY', 'your_app_key')

def create_monitor():
    """创建告警 Monitor"""
    with ApiClient(configuration) as api_client:
        monitors_api = MonitorsApi(api_client)

        # 创建指标告警
        monitor = Monitor(
            name="[SRE] 应用错误率过高",
            type=MonitorType.METRIC_ALERT,
            query="sum(last_5m):sum:myapp.http.errors{env:production}.as_rate() / sum:myapp.http.requests{env:production}.as_rate() > 0.05",
            message="""
## 应用错误率超过 5%

当前错误率：{{value}}

### 处理步骤：
1. 检查应用日志：`kubectl logs -l app=myapp --tail=100`
2. 检查依赖服务状态
3. 如需回滚：`kubectl rollout undo deployment/myapp`

@slack-sre-alerts @pagerduty-sre
            """.strip(),
            tags=['env:production', 'service:myapp', 'team:sre'],
            options={
                'thresholds': {
                    'critical': 0.05,      # 5% 错误率触发 Critical
                    'warning': 0.02,       # 2% 触发 Warning
                },
                'notify_no_data': True,    # 无数据时通知
                'no_data_timeframe': 10,   # 无数据超时（分钟）
                'renotify_interval': 30,   # 重复通知间隔（分钟）
                'evaluation_delay': 60,    # 评估延迟（秒）
            }
        )

        result = monitors_api.create_monitor(body=monitor)
        print(f"Monitor 已创建：ID={result.id}, 名称={result.name}")
        return result

def list_monitors():
    """列出所有 Monitor"""
    with ApiClient(configuration) as api_client:
        monitors_api = MonitorsApi(api_client)
        monitors = monitors_api.list_monitors(
            tags='team:sre'
        )
        for m in monitors:
            print(f"[{m.overall_state}] {m.name} (ID: {m.id})")

if __name__ == '__main__':
    list_monitors()
```

## 6. SRE 实战案例：应用性能指标上报与告警系统

```python
"""
SRE 实战：完整的应用性能监控系统
功能：
  1. 自动采集应用 RED 指标（Rate/Error/Duration）
  2. 监控依赖服务健康状态
  3. 定期上报系统资源指标
  4. 异常自动上报事件
  5. SLI/SLO 指标计算
"""
import time
import random
import threading
import logging
import socket
import os
from datadog import initialize, statsd
from contextlib import contextmanager

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s'
)
logger = logging.getLogger('sre-metrics')

# ============================================================
# 第一部分：核心指标客户端
# ============================================================

class SREMetricsSystem:
    """
    SRE 指标监控系统
    集成 DogStatsD，提供完整的应用可观测能力
    """

    def __init__(self, service_name, env='production',
                 statsd_host='127.0.0.1', statsd_port=8125):
        self.service = service_name
        self.env = env
        self.hostname = socket.gethostname()

        # 初始化 DogStatsD
        initialize(
            statsd_host=statsd_host,
            statsd_port=statsd_port,
            statsd_constant_tags=[
                f'service:{service_name}',
                f'env:{env}',
                f'host:{self.hostname}'
            ]
        )

        # SLI 窗口数据
        self._request_count = 0
        self._error_count = 0
        self._sli_lock = threading.Lock()

        logger.info(f"SRE 指标系统已初始化：{service_name} @ {env}")

    # ---- RED 指标（Rate / Error / Duration）----

    def record_request(self, method, endpoint, status_code, duration,
                       request_size=0, response_size=0):
        """
        记录一次 HTTP 请求的完整指标

        参数：
            method: HTTP 方法
            endpoint: 请求路径
            status_code: 响应状态码
            duration: 请求耗时（秒）
            request_size: 请求体大小（字节）
            response_size: 响应体大小（字节）
        """
        tags = [
            f'method:{method}',
            f'endpoint:{endpoint}',
            f'status:{status_code}',
            f'status_class:{status_code // 100}xx'
        ]

        # Rate：请求速率
        statsd.increment(f'{self.service}.http.requests', tags=tags)

        # Error：错误计数
        if status_code >= 500:
            statsd.increment(f'{self.service}.http.errors', tags=tags)
            with self._sli_lock:
                self._error_count += 1

        # Duration：延迟分布
        statsd.histogram(f'{self.service}.http.duration', duration, tags=tags)

        # 请求/响应大小
        if request_size > 0:
            statsd.histogram(f'{self.service}.http.request_size', request_size, tags=tags)
        if response_size > 0:
            statsd.histogram(f'{self.service}.http.response_size', response_size, tags=tags)

        # 更新 SLI 计数
        with self._sli_lock:
            self._request_count += 1

    @contextmanager
    def track_request(self, method, endpoint):
        """
        请求追踪上下文管理器

        用法：
            with metrics.track_request('GET', '/api/users') as ctx:
                result = handle_request()
                ctx['status_code'] = 200
                ctx['response_size'] = len(result)
        """
        ctx = {
            'status_code': 200,
            'request_size': 0,
            'response_size': 0
        }
        start = time.time()
        try:
            yield ctx
        except Exception as e:
            ctx['status_code'] = 500
            self.record_error(e, endpoint=endpoint)
            raise
        finally:
            duration = time.time() - start
            self.record_request(
                method=method,
                endpoint=endpoint,
                status_code=ctx['status_code'],
                duration=duration,
                request_size=ctx['request_size'],
                response_size=ctx['response_size']
            )

    # ---- 依赖服务监控 ----

    def check_dependency(self, name, check_func, tags=None):
        """
        检查依赖服务健康状态

        参数：
            name: 依赖服务名称
            check_func: 检查函数（返回 True/False）
            tags: 额外标签
        """
        all_tags = [f'dependency:{name}']
        if tags:
            all_tags.extend(tags)

        start = time.time()
        try:
            is_healthy = check_func()
            latency = time.time() - start

            statsd.gauge(f'{self.service}.dependency.up',
                        1 if is_healthy else 0, tags=all_tags)
            statsd.histogram(f'{self.service}.dependency.latency',
                           latency, tags=all_tags)

            # 服务检查
            statsd.service_check(
                f'{self.service}.dependency.{name}',
                status=0 if is_healthy else 2,
                tags=all_tags,
                message=f'{name} 检查{"通过" if is_healthy else "失败"}'
            )

            if not is_healthy:
                self.send_alert(
                    f'依赖服务异常：{name}',
                    f'{name} 健康检查失败，延迟：{latency:.3f}s',
                    alert_type='warning'
                )

            return is_healthy

        except Exception as e:
            latency = time.time() - start
            statsd.gauge(f'{self.service}.dependency.up', 0, tags=all_tags)
            statsd.service_check(
                f'{self.service}.dependency.{name}',
                status=2,
                tags=all_tags,
                message=str(e)
            )
            self.send_alert(
                f'依赖服务异常：{name}',
                f'检查异常：{e}',
                alert_type='error'
            )
            return False

    # ---- 系统资源监控 ----

    def collect_system_metrics(self):
        """采集系统资源指标"""
        try:
            import psutil

            # CPU
            statsd.gauge(f'{self.service}.system.cpu.percent',
                        psutil.cpu_percent())

            # 内存
            mem = psutil.virtual_memory()
            statsd.gauge(f'{self.service}.system.memory.percent', mem.percent)
            statsd.gauge(f'{self.service}.system.memory.used', mem.used)
            statsd.gauge(f'{self.service}.system.memory.available', mem.available)

            # 磁盘
            for part in psutil.disk_partitions():
                try:
                    usage = psutil.disk_usage(part.mountpoint)
                    statsd.gauge(
                        f'{self.service}.system.disk.percent',
                        usage.percent,
                        tags=[f'mountpoint:{part.mountpoint}']
                    )
                    # 磁盘空间告警
                    if usage.percent > 90:
                        self.send_alert(
                            '磁盘空间告警',
                            f'{part.mountpoint} 使用率 {usage.percent}%',
                            alert_type='warning'
                        )
                except (PermissionError, OSError):
                    pass

            # 网络连接数
            connections = psutil.net_connections()
            conn_states = {}
            for conn in connections:
                state = conn.status
                conn_states[state] = conn_states.get(state, 0) + 1
            for state, count in conn_states.items():
                statsd.gauge(
                    f'{self.service}.system.connections',
                    count,
                    tags=[f'state:{state}']
                )

            # 进程信息
            proc = psutil.Process(os.getpid())
            statsd.gauge(f'{self.service}.process.threads', proc.num_threads())
            statsd.gauge(f'{self.service}.process.memory_rss', proc.memory_info().rss)
            try:
                statsd.gauge(f'{self.service}.process.fds', proc.num_fds())
            except AttributeError:
                pass

        except ImportError:
            logger.debug("psutil 未安装，跳过系统指标采集")

    # ---- SLI/SLO ----

    def report_sli(self):
        """上报 SLI（服务水平指标）"""
        with self._sli_lock:
            total = self._request_count
            errors = self._error_count

        if total > 0:
            availability = (total - errors) / total * 100
            error_rate = errors / total * 100
        else:
            availability = 100.0
            error_rate = 0.0

        statsd.gauge(f'{self.service}.sli.availability', availability)
        statsd.gauge(f'{self.service}.sli.error_rate', error_rate)
        statsd.gauge(f'{self.service}.sli.total_requests', total)

        logger.info(
            f"SLI 报告 - 可用性: {availability:.2f}%, "
            f"错误率: {error_rate:.2f}%, "
            f"总请求: {total}"
        )

    # ---- 告警事件 ----

    def record_error(self, exception, endpoint='unknown'):
        """记录异常"""
        statsd.increment(
            f'{self.service}.exceptions',
            tags=[
                f'type:{type(exception).__name__}',
                f'endpoint:{endpoint}'
            ]
        )

    def send_alert(self, title, message, alert_type='warning', tags=None):
        """发送告警事件"""
        statsd.event(
            title=f'[{self.service}] {title}',
            message=message,
            alert_type=alert_type,
            tags=tags or []
        )
        logger.warning(f"告警：{title} - {message}")

    def send_deploy_event(self, version, changelog=''):
        """发送部署事件"""
        statsd.event(
            title=f'{self.service} 部署 v{version}',
            message=f'版本：{version}\n\n变更：\n{changelog}',
            alert_type='info',
            tags=[f'version:{version}', 'action:deploy']
        )

    # ---- 后台任务 ----

    def start_background_collectors(self, system_interval=30, sli_interval=60,
                                     dependency_interval=30):
        """启动后台采集任务"""
        def system_loop():
            while True:
                self.collect_system_metrics()
                time.sleep(system_interval)

        def sli_loop():
            while True:
                self.report_sli()
                time.sleep(sli_interval)

        threading.Thread(target=system_loop, daemon=True).start()
        threading.Thread(target=sli_loop, daemon=True).start()

        logger.info("后台采集任务已启动")


# ============================================================
# 第二部分：演示运行
# ============================================================

def simulate_application():
    """模拟完整的应用运行场景"""

    # 初始化指标系统
    metrics = SREMetricsSystem(
        service_name='order-service',
        env='production'
    )

    # 发送部署事件
    metrics.send_deploy_event('2.1.0', '- 修复超时问题\n- 优化查询性能')

    # 启动后台采集
    metrics.start_background_collectors()

    # 定义依赖检查函数
    def check_db():
        time.sleep(random.uniform(0.001, 0.01))
        return random.random() > 0.05

    def check_redis():
        time.sleep(random.uniform(0.001, 0.005))
        return random.random() > 0.02

    # 定期检查依赖
    def dependency_check_loop():
        while True:
            metrics.check_dependency('database', check_db, tags=['type:mysql'])
            metrics.check_dependency('redis', check_redis, tags=['type:cache'])
            time.sleep(30)

    threading.Thread(target=dependency_check_loop, daemon=True).start()

    # 模拟请求流量
    endpoints = [
        ('GET', '/api/users', 0.05),
        ('GET', '/api/orders', 0.08),
        ('POST', '/api/orders', 0.15),
        ('GET', '/api/products', 0.03),
        ('POST', '/api/payment', 0.3),
    ]

    logger.info("开始模拟应用流量...")

    while True:
        method, endpoint, base_latency = random.choice(endpoints)

        with metrics.track_request(method, endpoint) as ctx:
            # 模拟业务处理
            latency = base_latency * random.uniform(0.5, 3.0)
            if random.random() < 0.03:
                latency *= 10  # 慢请求

            time.sleep(min(latency, 0.1))  # 限制实际等待时间

            # 模拟不同响应
            status = random.choices(
                [200, 201, 400, 404, 500, 502, 503],
                weights=[65, 10, 8, 7, 5, 3, 2]
            )[0]
            ctx['status_code'] = status
            ctx['response_size'] = random.randint(100, 10000)

            # 模拟异常
            if status >= 500 and random.random() < 0.3:
                raise Exception(f"模拟服务端错误：{status}")

        time.sleep(random.uniform(0.1, 0.3))

if __name__ == '__main__':
    try:
        simulate_application()
    except KeyboardInterrupt:
        logger.info("应用已停止")
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 指标丢失 | UDP 包被丢弃 | 增大 Agent 缓冲区；检查网络 |
| 指标延迟 | Agent 聚合周期 | 默认 10s 聚合一次，可调整 |
| 标签爆炸 | 高基数标签 | 避免 user_id 等唯一值作标签 |
| 采样不准 | Distribution 采样 | 使用 Distribution 替代 Histogram |
| APM Span 缺失 | 未 patch 库 | 确认 `patch_all()` 在导入后立即调用 |
| 事件被限流 | 超出 API 限额 | 使用 DogStatsD 发送事件 |

### 7.2 最佳实践

```
[✓] 使用 DogStatsD（UDP）而非 API 上报高频指标
[✓] 为所有指标设置 service / env / host 全局标签
[✓] 指标命名规范：service.subsystem.metric（点分隔）
[✓] 避免高基数标签（user_id、session_id、trace_id）
[✓] 使用 Distribution 获取准确的全局百分位数
[✓] 关键操作配合事件上报（部署、配置变更、故障）
[✓] 为核心链路配置 APM 追踪
[✓] 设置合理的 Monitor 告警阈值和通知策略
[✓] 定期审计指标使用量（Datadog 按指标数计费）
```

## 8. 速查表

```python
# === 安装 ===
# pip install datadog ddtrace

# === 初始化 ===
from datadog import initialize, statsd
initialize(statsd_host='127.0.0.1', statsd_port=8125,
           statsd_constant_tags=['service:myapp', 'env:prod'])

# === DogStatsD 指标 ===
statsd.increment('metric.name', tags=['k:v'])           # 计数 +1
statsd.increment('metric.name', value=5)                # 计数 +5
statsd.decrement('metric.name')                         # 计数 -1
statsd.gauge('metric.name', 42, tags=['k:v'])           # 瞬时值
statsd.histogram('metric.name', 0.5, tags=['k:v'])      # 分布统计
statsd.distribution('metric.name', 0.5)                 # 全局分布
statsd.set('metric.name', 'unique_value')               # 不重复计数

# === 计时 ===
@statsd.timed('metric.name')                            # 装饰器
def func(): pass

with statsd.timed('metric.name'):                       # 上下文管理器
    pass

# === 事件 ===
statsd.event('标题', '内容', alert_type='warning', tags=['k:v'])

# === 服务检查 ===
statsd.service_check('check.name', status=0)  # 0=OK 1=WARN 2=CRIT 3=UNKNOWN

# === APM ===
from ddtrace import tracer, patch_all
patch_all()

@tracer.wrap(service='svc', resource='op')
def func(): pass

with tracer.trace('op', service='svc') as span:
    span.set_tag('key', 'value')
    span.set_metric('key', 42)

# === API ===
from datadog import api
api.Metric.send(metric='name', points=[(time.time(), 42)], type='gauge')
api.Event.create(title='标题', text='内容', alert_type='info')

# === 自动埋点启动 ===
# ddtrace-run python myapp.py
# DD_SERVICE=myapp DD_ENV=prod ddtrace-run gunicorn app:app
```
