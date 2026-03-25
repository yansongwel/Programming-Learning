# Sentry 错误追踪 — 用 Python 实现运维脚本与服务的异常自动捕获与追踪

## 1. 是什么 & 为什么用它

**Sentry** 是一个开源的实时错误追踪和性能监控平台。`sentry-sdk` 是其官方 Python 客户端，
能自动捕获未处理异常、记录上下文信息（面包屑）、追踪性能瓶颈。

**对 SRE 的价值：**
- 自动捕获运维脚本、自动化工具中的异常，避免"静默失败"
- 丰富的上下文信息（堆栈、变量、环境）加速故障定位
- Release 跟踪：将错误与版本关联，快速判断回归问题
- 性能监控：识别慢事务和瓶颈

## 2. 安装与环境

```bash
# 安装 sentry-sdk
pip install sentry-sdk

# 可选：特定框架集成
pip install sentry-sdk[flask]      # Flask 集成
pip install sentry-sdk[celery]     # Celery 集成
pip install sentry-sdk[sqlalchemy] # SQLAlchemy 集成

# 验证安装
python -c "import sentry_sdk; print(sentry_sdk.VERSION)"
```

**环境要求：**
- Python >= 3.6
- Sentry 服务端（SaaS: sentry.io / 自建: self-hosted）
- DSN（Data Source Name）：项目的唯一标识 URL

```
DSN 格式：https://<public_key>@<host>/<project_id>
示例：    https://abc123@o123456.ingest.sentry.io/789
```

## 3. 核心概念与原理

### 3.1 Sentry 数据流架构

```
+-------------------+    HTTPS    +-------------------+    处理    +------------------+
|  Python 应用      |------------>|  Sentry Server    |---------->|  存储 & 索引     |
|  (sentry-sdk)     |   上报事件  |  (Relay 中继)     |           |  (PostgreSQL +   |
+-------------------+             +-------------------+           |   Clickhouse)    |
        |                                |                        +------------------+
        | 自动捕获                       |                               |
        | - 未处理异常                   v                               v
        | - 面包屑                +-------------------+          +------------------+
        | - 性能事务              |  告警规则引擎     |          |  Web UI          |
        |                        |  (Issue Alert /   |          |  - Issue 列表    |
        v                        |   Metric Alert)   |          |  - 性能监控      |
  错误 / 事务 / 面包屑           +-------------------+          |  - Release       |
                                        |                       +------------------+
                                        v
                                 通知（Slack/邮件/PagerDuty/Webhook）
```

### 3.2 核心概念

```
概念              说明
─────────────────────────────────────────────────────────
Event             一次错误或异常的完整记录
Issue             相同错误的聚合（去重分组）
Breadcrumb        面包屑：事件发生前的操作日志轨迹
Scope             作用域：附加到事件上的上下文（标签、用户、额外数据）
Transaction       性能事务：一次完整操作的耗时追踪
Span              事务中的子操作（如数据库查询、HTTP 请求）
Release           版本标识：关联错误与代码版本
Environment       环境标识：production / staging / development
DSN               数据源名称：项目的上报地址
Sample Rate       采样率：控制上报频率（0.0~1.0）
```

### 3.3 事件处理流程

```
异常发生 --> SDK 捕获 --> 附加上下文 --> 采样判断 --> 序列化 --> 发送到 Sentry
                |              |              |
                |              |              +-- before_send 钩子（可修改/丢弃）
                |              +-- Scope（标签、用户、面包屑）
                +-- 自动（未处理异常）/ 手动（capture_exception/capture_message）
```

## 4. 基础用法

### 4.1 初始化配置

```python
"""
Sentry SDK 初始化：推荐在应用启动时最早执行
"""
import sentry_sdk
from sentry_sdk.integrations.logging import LoggingIntegration
import logging

# 基础初始化
sentry_sdk.init(
    dsn="https://your-public-key@o123456.ingest.sentry.io/project-id",

    # === 基础配置 ===
    environment="production",          # 环境标识
    release="myapp@2.1.0",           # 版本号（推荐：包名@版本）
    server_name="web-server-01",      # 服务器名称

    # === 采样率 ===
    sample_rate=1.0,                  # 错误采样率（1.0 = 100% 上报）
    traces_sample_rate=0.2,           # 性能追踪采样率（20%）
    profiles_sample_rate=0.1,         # 性能剖析采样率（10%）

    # === 数据控制 ===
    send_default_pii=False,           # 是否发送个人身份信息（PII）
    max_breadcrumbs=50,               # 最大面包屑数量
    attach_stacktrace=True,           # 非异常事件也附加堆栈

    # === 钩子函数 ===
    before_send=before_send_handler,  # 发送前处理（见下方定义）
    before_breadcrumb=before_breadcrumb_handler,  # 面包屑过滤

    # === 集成配置 ===
    integrations=[
        LoggingIntegration(
            level=logging.INFO,        # 捕获 INFO 及以上级别的日志为面包屑
            event_level=logging.ERROR   # ERROR 及以上级别的日志触发事件
        ),
    ],

    # === 传输配置 ===
    # transport 默认使用后台线程异步发送
    shutdown_timeout=5,               # 关闭时等待发送完成的超时（秒）
)

def before_send_handler(event, hint):
    """
    发送前钩子：可以修改或丢弃事件
    返回 None 表示丢弃，返回 event 表示继续发送
    """
    # 过滤掉特定异常
    if 'exc_info' in hint:
        exc_type, exc_value, tb = hint['exc_info']
        # 忽略 KeyboardInterrupt
        if isinstance(exc_value, KeyboardInterrupt):
            return None
        # 忽略特定错误消息
        if 'Connection refused' in str(exc_value):
            return None

    # 清除敏感数据
    if 'request' in event:
        headers = event['request'].get('headers', {})
        if 'Authorization' in headers:
            headers['Authorization'] = '[已过滤]'

    return event

def before_breadcrumb_handler(crumb, hint):
    """面包屑过滤：过滤不需要的面包屑"""
    # 过滤掉健康检查请求的面包屑
    if crumb.get('category') == 'httplib':
        url = crumb.get('data', {}).get('url', '')
        if '/health' in url or '/ping' in url:
            return None
    return crumb

print("Sentry 已初始化")
```

### 4.2 自动捕获异常

```python
"""
Sentry 自动捕获未处理异常
SDK 初始化后，所有未捕获的异常都会自动上报
"""
import sentry_sdk

sentry_sdk.init(dsn="https://your-key@sentry.io/project-id")

def divide(a, b):
    """会抛出 ZeroDivisionError 的函数"""
    return a / b

def process_data(data):
    """会抛出 KeyError 的函数"""
    return data['nonexistent_key']

# 这些异常会被自动捕获并上报到 Sentry
try:
    divide(1, 0)
except ZeroDivisionError:
    # 如果你 catch 了异常，Sentry 不会自动上报
    # 需要手动上报（见下一节）
    pass

# 未捕获的异常会自动上报
# divide(1, 0)  # 取消注释会触发自动上报
```

### 4.3 手动上报

```python
"""
手动上报异常和消息
"""
import sentry_sdk

sentry_sdk.init(dsn="https://your-key@sentry.io/project-id")

# === 手动捕获异常 ===
def safe_operation():
    try:
        result = 1 / 0
    except ZeroDivisionError as e:
        # 手动上报捕获的异常
        sentry_sdk.capture_exception(e)
        print("错误已上报到 Sentry")

# === 上报消息（非异常） ===
def report_warning():
    sentry_sdk.capture_message(
        "磁盘空间不足：/data 分区使用率达到 85%",
        level="warning"  # 级别：fatal/error/warning/info/debug
    )

# === 带上下文的手动上报 ===
def report_with_context():
    with sentry_sdk.push_scope() as scope:
        # 设置标签（用于搜索和过滤）
        scope.set_tag("component", "data-pipeline")
        scope.set_tag("pipeline_id", "pipe_001")

        # 设置额外数据（附加到事件）
        scope.set_extra("records_processed", 5000)
        scope.set_extra("batch_size", 100)

        # 设置用户信息
        scope.set_user({
            "id": "sre-bot",
            "username": "automation",
            "email": "sre@example.com"
        })

        # 设置级别
        scope.level = "warning"

        # 上报
        sentry_sdk.capture_message("数据管道处理延迟超过阈值")

# === 使用装饰器简化 ===
def with_sentry_context(**tags):
    """自定义装饰器：为函数添加 Sentry 上下文"""
    import functools
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            with sentry_sdk.push_scope() as scope:
                for key, value in tags.items():
                    scope.set_tag(key, value)
                scope.set_tag("function", func.__name__)
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    sentry_sdk.capture_exception(e)
                    raise
        return wrapper
    return decorator

@with_sentry_context(component="backup", priority="high")
def run_backup():
    """带自动上下文的备份函数"""
    raise RuntimeError("备份目标不可达")

if __name__ == '__main__':
    safe_operation()
    report_warning()
    report_with_context()
    try:
        run_backup()
    except RuntimeError:
        pass
```

### 4.4 面包屑（Breadcrumbs）

```python
"""
面包屑：记录事件发生前的操作轨迹
帮助重现异常发生时的上下文
"""
import sentry_sdk
import time

sentry_sdk.init(
    dsn="https://your-key@sentry.io/project-id",
    max_breadcrumbs=100  # 最多保留 100 条面包屑
)

def process_order(order_id):
    """处理订单：通过面包屑记录关键步骤"""

    # 手动添加面包屑
    sentry_sdk.add_breadcrumb(
        category='order',
        message=f'开始处理订单 {order_id}',
        level='info',
        data={'order_id': order_id}
    )

    # 步骤 1：验证订单
    sentry_sdk.add_breadcrumb(
        category='order.validation',
        message='订单验证通过',
        level='info',
        data={'order_id': order_id, 'items': 3}
    )

    # 步骤 2：扣减库存
    sentry_sdk.add_breadcrumb(
        category='inventory',
        message='库存扣减成功',
        level='info',
        data={'sku': 'PROD-001', 'quantity': 2}
    )

    # 步骤 3：调用支付（模拟失败）
    sentry_sdk.add_breadcrumb(
        category='payment',
        message='发起支付请求',
        level='info',
        data={
            'amount': 299.00,
            'method': 'alipay',
            'gateway': 'payment-service:8080'
        }
    )

    # 支付失败，触发异常
    # 此时 Sentry 会将上面所有面包屑一起上报
    raise ConnectionError("支付网关连接超时：payment-service:8080")

if __name__ == '__main__':
    try:
        process_order('ORD-20260101-001')
    except ConnectionError:
        sentry_sdk.capture_exception()
        print("订单处理失败，错误已上报")
```

### 4.5 上下文信息

```python
"""
Sentry 上下文：为事件附加结构化的额外信息
"""
import sentry_sdk

sentry_sdk.init(dsn="https://your-key@sentry.io/project-id")

# === 全局上下文（影响后续所有事件）===

# 设置用户
sentry_sdk.set_user({
    "id": "sre-automation",
    "username": "sre-bot",
    "email": "sre@example.com",
    "ip_address": "10.0.1.100"
})

# 设置标签（可搜索、可过滤）
sentry_sdk.set_tag("datacenter", "us-east-1")
sentry_sdk.set_tag("cluster", "prod-cluster-01")

# 设置上下文（结构化数据，不可搜索但会展示在事件详情中）
sentry_sdk.set_context("server", {
    "hostname": "web-server-01",
    "os": "Ubuntu 22.04",
    "cpu_cores": 8,
    "memory_gb": 32,
    "kernel": "5.15.0-78"
})

sentry_sdk.set_context("deployment", {
    "version": "2.1.0",
    "git_sha": "abc123def",
    "deployer": "jenkins",
    "deploy_time": "2026-03-25T10:00:00Z"
})

# === 局部上下文（仅影响 scope 内的事件）===

def handle_task(task_id, task_type):
    """使用局部 scope 避免污染全局上下文"""
    with sentry_sdk.push_scope() as scope:
        scope.set_tag("task_id", task_id)
        scope.set_tag("task_type", task_type)
        scope.set_context("task", {
            "id": task_id,
            "type": task_type,
            "retry_count": 3,
            "queue": "high-priority"
        })

        # 此 scope 内的异常会自动携带上述上下文
        try:
            raise ValueError(f"任务 {task_id} 参数校验失败")
        except ValueError:
            sentry_sdk.capture_exception()

# === 使用 scope 设置事件级别和指纹 ===
def report_custom_grouped():
    """自定义事件分组（指纹）"""
    with sentry_sdk.push_scope() as scope:
        # 自定义指纹：相同指纹的事件会合并为一个 Issue
        scope.fingerprint = ['database-timeout', 'primary-db']
        scope.level = 'error'

        sentry_sdk.capture_message("数据库查询超时")

if __name__ == '__main__':
    handle_task("task_001", "data_sync")
    report_custom_grouped()
```

## 5. 进阶用法

### 5.1 性能监控（Transactions & Spans）

```python
"""
Sentry 性能监控：追踪事务和子操作耗时
"""
import sentry_sdk
import time
import random

sentry_sdk.init(
    dsn="https://your-key@sentry.io/project-id",
    traces_sample_rate=1.0,  # 开发环境 100% 采样，生产建议 0.1~0.3
    profiles_sample_rate=0.1  # 性能剖析采样率
)

def process_api_request():
    """追踪完整的 API 请求处理过程"""

    # 开启一个事务（Transaction）
    with sentry_sdk.start_transaction(
        op="http.server",                 # 操作类型
        name="GET /api/orders",           # 事务名称
        description="获取订单列表"        # 描述
    ) as transaction:

        # 设置事务级别标签
        transaction.set_tag("endpoint", "/api/orders")
        transaction.set_tag("method", "GET")

        # 子操作 1：参数验证
        with sentry_sdk.start_span(
            op="validation",
            description="请求参数验证"
        ) as span:
            time.sleep(random.uniform(0.001, 0.01))
            span.set_data("params", {"page": 1, "size": 20})

        # 子操作 2：数据库查询
        with sentry_sdk.start_span(
            op="db.query",
            description="SELECT * FROM orders WHERE ..."
        ) as span:
            time.sleep(random.uniform(0.01, 0.1))
            span.set_data("db.system", "mysql")
            span.set_data("db.name", "order_db")
            span.set_data("rows_returned", 20)

        # 子操作 3：缓存查询
        with sentry_sdk.start_span(
            op="cache.get",
            description="Redis GET user_preferences"
        ) as span:
            time.sleep(random.uniform(0.001, 0.005))
            span.set_data("cache.hit", True)

        # 子操作 4：序列化响应
        with sentry_sdk.start_span(
            op="serialize",
            description="JSON 序列化响应"
        ) as span:
            time.sleep(random.uniform(0.001, 0.005))

        # 设置最终状态
        transaction.set_http_status(200)

def batch_processing():
    """追踪批处理任务"""
    with sentry_sdk.start_transaction(
        op="task",
        name="daily_data_sync"
    ) as transaction:

        total_records = 1000
        transaction.set_data("total_records", total_records)

        # 分批处理
        batch_size = 100
        for offset in range(0, total_records, batch_size):
            with sentry_sdk.start_span(
                op="batch.process",
                description=f"处理记录 {offset}-{offset+batch_size}"
            ) as span:
                span.set_data("offset", offset)
                span.set_data("batch_size", batch_size)
                time.sleep(random.uniform(0.05, 0.2))

        transaction.set_status("ok")

if __name__ == '__main__':
    for _ in range(5):
        process_api_request()
    batch_processing()
    print("性能数据已上报")
```

### 5.2 Release 跟踪

```python
"""
Release 跟踪：将错误与代码版本关联
"""
import sentry_sdk
import subprocess

def get_git_sha():
    """获取当前 Git commit SHA"""
    try:
        return subprocess.check_output(
            ['git', 'rev-parse', 'HEAD']
        ).decode().strip()
    except Exception:
        return 'unknown'

# 使用 release 标识版本
sentry_sdk.init(
    dsn="https://your-key@sentry.io/project-id",
    release=f"myapp@2.1.0+{get_git_sha()[:8]}",  # 包名@版本+commit
    environment="production"
)

# === 通过 Sentry CLI 创建 Release（推荐在 CI/CD 中执行）===
# sentry-cli releases new myapp@2.1.0
# sentry-cli releases set-commits myapp@2.1.0 --auto
# sentry-cli releases deploys myapp@2.1.0 new -e production
# sentry-cli releases finalize myapp@2.1.0

# === 通过 API 管理 Release ===
def create_release_via_api():
    """通过 Sentry API 创建 Release"""
    import requests

    org_slug = "my-org"
    project_slug = "my-project"
    auth_token = "your-auth-token"

    headers = {
        "Authorization": f"Bearer {auth_token}",
        "Content-Type": "application/json"
    }

    # 创建 Release
    response = requests.post(
        f"https://sentry.io/api/0/organizations/{org_slug}/releases/",
        headers=headers,
        json={
            "version": "myapp@2.1.0",
            "projects": [project_slug],
            "refs": [{
                "repository": "my-org/my-repo",
                "commit": get_git_sha()
            }]
        }
    )
    print(f"Release 创建结果：{response.status_code}")

    # 标记部署
    response = requests.post(
        f"https://sentry.io/api/0/organizations/{org_slug}/releases/myapp@2.1.0/deploys/",
        headers=headers,
        json={
            "environment": "production",
            "name": "v2.1.0 production deploy"
        }
    )
    print(f"Deploy 记录结果：{response.status_code}")
```

### 5.3 自定义集成与过滤

```python
"""
高级配置：自定义集成、错误过滤、采样策略
"""
import sentry_sdk
from sentry_sdk.integrations.logging import LoggingIntegration
from sentry_sdk.integrations.threading import ThreadingIntegration
import logging
import random

def traces_sampler(sampling_context):
    """
    动态采样策略：根据事务类型决定采样率
    返回 0.0~1.0 的采样率，或 True/False
    """
    transaction_name = sampling_context.get("transaction_context", {}).get("name", "")

    # 健康检查不采样
    if "/health" in transaction_name or "/ping" in transaction_name:
        return 0.0

    # 关键接口 100% 采样
    if "/api/payment" in transaction_name:
        return 1.0

    # 其他接口 10% 采样
    return 0.1

def before_send_transaction(event, hint):
    """事务发送前处理"""
    # 过滤过短的事务（可能是健康检查）
    duration = event.get("timestamp", 0) - event.get("start_timestamp", 0)
    if duration < 0.001:  # 小于 1ms
        return None
    return event

sentry_sdk.init(
    dsn="https://your-key@sentry.io/project-id",
    environment="production",
    release="myapp@2.1.0",

    # 使用动态采样函数
    traces_sampler=traces_sampler,

    # 事务发送前处理
    before_send_transaction=before_send_transaction,

    # 忽略特定异常
    ignore_errors=[
        KeyboardInterrupt,
        SystemExit,
        ConnectionResetError,
    ],

    integrations=[
        LoggingIntegration(
            level=logging.INFO,
            event_level=logging.ERROR
        ),
        ThreadingIntegration(propagate_hub=True),  # 线程间传播上下文
    ],

    # 数据清洗
    before_send=lambda event, hint: scrub_sensitive_data(event),
)

def scrub_sensitive_data(event):
    """清洗敏感数据"""
    # 清除密码相关字段
    if 'extra' in event:
        for key in list(event['extra'].keys()):
            if any(s in key.lower() for s in ['password', 'secret', 'token', 'key']):
                event['extra'][key] = '[已过滤]'

    # 清除请求中的敏感 header
    if 'request' in event and 'headers' in event['request']:
        sensitive_headers = ['authorization', 'cookie', 'x-api-key']
        for header in sensitive_headers:
            if header in event['request']['headers']:
                event['request']['headers'][header] = '[已过滤]'

    return event
```

## 6. SRE 实战案例：为运维脚本接入 Sentry 错误追踪

```python
"""
SRE 实战：完整的运维脚本 Sentry 集成框架
功能：
  1. 统一的 Sentry 初始化（支持多环境）
  2. 自动捕获异常并附加运维上下文
  3. 关键操作的性能追踪
  4. 定期巡检任务的错误上报
  5. 面包屑记录操作步骤
  6. 告警事件与 Release 关联
"""
import sentry_sdk
import time
import random
import socket
import os
import sys
import logging
import functools
from datetime import datetime
from contextlib import contextmanager

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s'
)
logger = logging.getLogger('sre-tools')

# ============================================================
# 第一部分：Sentry 初始化框架
# ============================================================

class SentrySREInit:
    """
    SRE 专用 Sentry 初始化器
    封装初始化配置，适配运维脚本场景
    """

    # 不需要上报的异常类型
    IGNORED_EXCEPTIONS = [
        KeyboardInterrupt,
        SystemExit,
    ]

    # 敏感字段关键词
    SENSITIVE_KEYWORDS = [
        'password', 'secret', 'token', 'key', 'credential',
        'authorization', 'cookie', 'session'
    ]

    @classmethod
    def init(cls, dsn, service_name, version='0.0.0',
             environment=None, extra_tags=None):
        """
        初始化 Sentry

        参数：
            dsn: Sentry DSN
            service_name: 服务/脚本名称
            version: 版本号
            environment: 环境（自动检测或手动指定）
            extra_tags: 额外全局标签
        """
        # 自动检测环境
        if environment is None:
            environment = os.getenv('SENTRY_ENVIRONMENT',
                                   os.getenv('ENV', 'development'))

        sentry_sdk.init(
            dsn=dsn,
            release=f"{service_name}@{version}",
            environment=environment,
            server_name=socket.gethostname(),

            # 运维脚本推荐 100% 采样（调用量低）
            sample_rate=1.0,
            traces_sample_rate=1.0 if environment != 'production' else 0.5,

            # 配置
            max_breadcrumbs=200,
            attach_stacktrace=True,
            send_default_pii=False,
            shutdown_timeout=10,

            # 钩子
            before_send=cls._before_send,
            ignore_errors=cls.IGNORED_EXCEPTIONS,
        )

        # 设置全局上下文
        sentry_sdk.set_tag("service", service_name)
        sentry_sdk.set_tag("hostname", socket.gethostname())
        sentry_sdk.set_tag("pid", str(os.getpid()))
        sentry_sdk.set_tag("python_version", sys.version.split()[0])

        if extra_tags:
            for key, value in extra_tags.items():
                sentry_sdk.set_tag(key, value)

        sentry_sdk.set_context("runtime", {
            "hostname": socket.gethostname(),
            "python": sys.version,
            "pid": os.getpid(),
            "cwd": os.getcwd(),
            "user": os.getenv('USER', 'unknown'),
        })

        logger.info(f"Sentry 已初始化：{service_name}@{version} ({environment})")

    @classmethod
    def _before_send(cls, event, hint):
        """发送前清洗敏感数据"""
        cls._scrub_event(event)
        return event

    @classmethod
    def _scrub_event(cls, event):
        """递归清洗事件中的敏感数据"""
        if isinstance(event, dict):
            for key in list(event.keys()):
                if any(kw in key.lower() for kw in cls.SENSITIVE_KEYWORDS):
                    event[key] = '[FILTERED]'
                else:
                    cls._scrub_event(event[key])
        elif isinstance(event, list):
            for item in event:
                cls._scrub_event(item)


# ============================================================
# 第二部分：运维任务装饰器
# ============================================================

def sre_task(task_name, tags=None):
    """
    SRE 任务装饰器：自动追踪执行过程

    用法：
        @sre_task("数据库备份", tags={"db": "primary"})
        def backup_database():
            ...
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # 创建 Sentry 事务
            with sentry_sdk.start_transaction(
                op="sre.task",
                name=task_name,
                description=func.__doc__ or task_name
            ) as transaction:
                # 设置标签
                transaction.set_tag("task_name", task_name)
                transaction.set_tag("function", func.__name__)
                if tags:
                    for k, v in tags.items():
                        transaction.set_tag(k, v)

                # 添加面包屑
                sentry_sdk.add_breadcrumb(
                    category='sre.task',
                    message=f'开始执行：{task_name}',
                    level='info',
                    data={
                        'function': func.__name__,
                        'args': str(args)[:200],
                        'start_time': datetime.now().isoformat()
                    }
                )

                start_time = time.time()
                try:
                    result = func(*args, **kwargs)
                    duration = time.time() - start_time

                    # 记录成功
                    transaction.set_status("ok")
                    sentry_sdk.add_breadcrumb(
                        category='sre.task',
                        message=f'执行成功：{task_name}（耗时 {duration:.2f}s）',
                        level='info'
                    )

                    logger.info(f"[{task_name}] 执行成功，耗时 {duration:.2f}s")
                    return result

                except Exception as e:
                    duration = time.time() - start_time
                    transaction.set_status("internal_error")

                    # 附加错误上下文
                    with sentry_sdk.push_scope() as scope:
                        scope.set_tag("task_name", task_name)
                        scope.set_tag("task_status", "failed")
                        scope.set_context("task_execution", {
                            "task_name": task_name,
                            "duration_seconds": duration,
                            "function": func.__name__,
                            "error_type": type(e).__name__,
                            "error_message": str(e),
                            "timestamp": datetime.now().isoformat()
                        })
                        sentry_sdk.capture_exception(e)

                    logger.error(f"[{task_name}] 执行失败：{e}（耗时 {duration:.2f}s）")
                    raise

        return wrapper
    return decorator

@contextmanager
def sre_step(step_name, data=None):
    """
    运维步骤上下文管理器：自动记录面包屑和 Span

    用法：
        with sre_step("连接数据库", data={"host": "db-primary"}):
            connect_to_db()
    """
    sentry_sdk.add_breadcrumb(
        category='sre.step',
        message=f'开始步骤：{step_name}',
        level='info',
        data=data or {}
    )

    with sentry_sdk.start_span(op="sre.step", description=step_name) as span:
        if data:
            for k, v in data.items():
                span.set_data(k, v)

        start = time.time()
        try:
            yield span
            duration = time.time() - start
            span.set_data("duration", f"{duration:.3f}s")
            span.set_data("status", "success")
            logger.info(f"  步骤完成：{step_name}（{duration:.3f}s）")
        except Exception as e:
            duration = time.time() - start
            span.set_data("duration", f"{duration:.3f}s")
            span.set_data("status", "failed")
            span.set_data("error", str(e))
            logger.error(f"  步骤失败：{step_name}（{duration:.3f}s）- {e}")
            raise


# ============================================================
# 第三部分：实战 — 服务器巡检脚本
# ============================================================

class ServerInspector:
    """
    服务器巡检工具
    检查：磁盘空间、服务状态、证书过期、日志错误
    """

    def __init__(self):
        self.results = []
        self.warnings = []
        self.errors = []

    @sre_task("服务器巡检", tags={"type": "inspection"})
    def run_inspection(self, servers=None):
        """执行完整的服务器巡检"""
        servers = servers or [
            {"host": "web-01", "ip": "10.0.1.10", "role": "web"},
            {"host": "web-02", "ip": "10.0.1.11", "role": "web"},
            {"host": "db-01", "ip": "10.0.2.10", "role": "database"},
            {"host": "redis-01", "ip": "10.0.3.10", "role": "cache"},
        ]

        sentry_sdk.set_context("inspection", {
            "total_servers": len(servers),
            "start_time": datetime.now().isoformat(),
            "inspector": os.getenv('USER', 'sre-bot')
        })

        for server in servers:
            self._inspect_server(server)

        # 汇总结果
        summary = self._generate_summary()

        # 如果有错误，上报到 Sentry
        if self.errors:
            sentry_sdk.capture_message(
                f"巡检发现 {len(self.errors)} 个错误",
                level="error"
            )

        return summary

    def _inspect_server(self, server):
        """巡检单台服务器"""
        host = server['host']
        logger.info(f"正在巡检：{host}")

        with sentry_sdk.push_scope() as scope:
            scope.set_tag("target_host", host)
            scope.set_tag("target_role", server['role'])

            # 检查磁盘空间
            with sre_step("检查磁盘空间", data={"host": host}):
                self._check_disk(server)

            # 检查服务状态
            with sre_step("检查服务状态", data={"host": host}):
                self._check_services(server)

            # 检查日志错误
            with sre_step("检查日志错误", data={"host": host}):
                self._check_logs(server)

            # 检查证书
            if server['role'] == 'web':
                with sre_step("检查 SSL 证书", data={"host": host}):
                    self._check_certificates(server)

    def _check_disk(self, server):
        """检查磁盘空间"""
        # 模拟检查结果
        usage = random.uniform(30, 95)
        result = {
            "host": server['host'],
            "check": "disk",
            "usage_percent": round(usage, 1),
            "status": "ok" if usage < 80 else ("warning" if usage < 90 else "error")
        }

        if usage >= 90:
            self.errors.append(result)
            sentry_sdk.add_breadcrumb(
                category='inspection',
                message=f'{server["host"]} 磁盘使用率 {usage:.1f}%（危险）',
                level='error'
            )
        elif usage >= 80:
            self.warnings.append(result)
            sentry_sdk.add_breadcrumb(
                category='inspection',
                message=f'{server["host"]} 磁盘使用率 {usage:.1f}%（警告）',
                level='warning'
            )

        self.results.append(result)
        time.sleep(0.1)  # 模拟网络延迟

    def _check_services(self, server):
        """检查关键服务状态"""
        services = {
            'web': ['nginx', 'gunicorn'],
            'database': ['mysql', 'mysqld_exporter'],
            'cache': ['redis-server', 'redis_exporter']
        }

        for svc in services.get(server['role'], []):
            is_running = random.random() > 0.1  # 90% 概率运行中
            result = {
                "host": server['host'],
                "check": "service",
                "service": svc,
                "status": "running" if is_running else "stopped"
            }

            if not is_running:
                self.errors.append(result)
                sentry_sdk.capture_message(
                    f"服务异常：{server['host']} 上的 {svc} 未运行",
                    level="error"
                )

            self.results.append(result)

    def _check_logs(self, server):
        """检查最近的错误日志"""
        error_count = random.randint(0, 50)
        result = {
            "host": server['host'],
            "check": "logs",
            "error_count": error_count,
            "period": "last_1h"
        }

        if error_count > 20:
            self.warnings.append(result)
            sentry_sdk.add_breadcrumb(
                category='inspection',
                message=f'{server["host"]} 最近 1 小时 {error_count} 条错误日志',
                level='warning'
            )

        self.results.append(result)

    def _check_certificates(self, server):
        """检查 SSL 证书过期时间"""
        days_remaining = random.randint(5, 365)
        result = {
            "host": server['host'],
            "check": "certificate",
            "days_remaining": days_remaining,
            "status": "ok" if days_remaining > 30 else (
                "warning" if days_remaining > 7 else "error"
            )
        }

        if days_remaining <= 7:
            self.errors.append(result)
            sentry_sdk.capture_message(
                f"证书即将过期：{server['host']} 剩余 {days_remaining} 天",
                level="error"
            )
        elif days_remaining <= 30:
            self.warnings.append(result)

        self.results.append(result)

    def _generate_summary(self):
        """生成巡检报告"""
        summary = {
            "timestamp": datetime.now().isoformat(),
            "total_checks": len(self.results),
            "errors": len(self.errors),
            "warnings": len(self.warnings),
            "status": "healthy" if not self.errors else "unhealthy"
        }

        sentry_sdk.set_context("inspection_summary", summary)

        logger.info(
            f"巡检完成 - 总检查项: {summary['total_checks']}, "
            f"错误: {summary['errors']}, 警告: {summary['warnings']}"
        )

        return summary


# ============================================================
# 第四部分：主入口
# ============================================================

def main():
    """运行巡检脚本"""

    # 初始化 Sentry
    SentrySREInit.init(
        dsn=os.getenv('SENTRY_DSN', 'https://your-key@sentry.io/project-id'),
        service_name='server-inspector',
        version='1.0.0',
        environment=os.getenv('ENV', 'production'),
        extra_tags={
            "team": "sre",
            "schedule": "every-5min",
        }
    )

    # 执行巡检
    inspector = ServerInspector()
    try:
        summary = inspector.run_inspection()
        print(f"\n巡检结果：{summary}")
    except Exception as e:
        logger.error(f"巡检脚本执行失败：{e}")
        # 异常已由 @sre_task 装饰器自动上报到 Sentry
        sys.exit(1)
    finally:
        # 确保所有事件都发送完毕
        sentry_sdk.flush(timeout=10)

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 异常未上报 | 异常被 try/except 捕获 | 手动调用 `capture_exception()` |
| 事件丢失 | 采样率过低 | 调整 `sample_rate` |
| 敏感数据泄露 | 未配置数据清洗 | 使用 `before_send` 过滤 |
| 进程退出前事件丢失 | 异步发送未完成 | 退出前调用 `sentry_sdk.flush()` |
| 同类错误被拆分 | Sentry 分组算法不准 | 使用自定义 `fingerprint` |
| 面包屑过多影响性能 | 默认记录所有操作 | 限制 `max_breadcrumbs` |
| 多线程上下文丢失 | Hub 未传播 | 使用 `ThreadingIntegration` |

### 7.2 最佳实践

```
[✓] 在应用最早期初始化 Sentry（import 之后立即初始化）
[✓] 始终设置 release 和 environment
[✓] 使用 before_send 过滤敏感数据
[✓] 运维脚本 sample_rate=1.0（不采样）
[✓] 生产服务 traces_sample_rate 设置为 0.1~0.3
[✓] 退出前调用 sentry_sdk.flush() 确保发送完成
[✓] 使用 push_scope 隔离上下文，避免污染全局
[✓] 关键步骤添加面包屑，便于事后追踪
[✓] 配置 ignore_errors 过滤无意义的异常
[✓] 在 CI/CD 中集成 Release 和 Deploy 追踪
```

## 8. 速查表

```python
# === 安装 ===
# pip install sentry-sdk

# === 初始化 ===
import sentry_sdk
sentry_sdk.init(
    dsn="https://key@sentry.io/id",
    release="app@1.0.0",
    environment="production",
    sample_rate=1.0,              # 错误采样率
    traces_sample_rate=0.2,       # 性能采样率
    before_send=my_filter,        # 发送前钩子
    ignore_errors=[KeyboardInterrupt],
)

# === 手动上报 ===
sentry_sdk.capture_exception(e)                         # 上报异常
sentry_sdk.capture_message("消息", level="warning")     # 上报消息

# === 上下文 ===
sentry_sdk.set_user({"id": "123", "email": "x@x.com"})  # 设置用户
sentry_sdk.set_tag("key", "value")                       # 设置标签
sentry_sdk.set_context("name", {"key": "value"})         # 设置上下文

# === Scope ===
with sentry_sdk.push_scope() as scope:
    scope.set_tag("key", "value")
    scope.set_extra("key", "value")
    scope.set_user({"id": "123"})
    scope.level = "warning"
    scope.fingerprint = ["custom-group"]
    sentry_sdk.capture_message("分组消息")

# === 面包屑 ===
sentry_sdk.add_breadcrumb(
    category='http', message='GET /api', level='info',
    data={'url': '/api', 'status': 200}
)

# === 性能监控 ===
with sentry_sdk.start_transaction(op="task", name="任务") as txn:
    with sentry_sdk.start_span(op="db", description="查询") as span:
        span.set_data("key", "value")
    txn.set_status("ok")

# === 清理 ===
sentry_sdk.flush(timeout=5)    # 等待发送完成

# === 环境变量 ===
# SENTRY_DSN=https://...        自动读取 DSN
# SENTRY_ENVIRONMENT=prod       自动设置环境
# SENTRY_RELEASE=app@1.0.0     自动设置版本
```
