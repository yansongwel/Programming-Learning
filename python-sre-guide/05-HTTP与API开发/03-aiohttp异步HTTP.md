# aiohttp 异步 HTTP — 高并发网络请求与异步服务端

> 一句话说明：`aiohttp` 是 Python 异步 HTTP 框架，既能做高并发客户端（同时探测上千个 URL），也能做异步服务端和 WebSocket，是 SRE 大规模探测和实时监控的利器。

---

## 1. 是什么 & 为什么用它

### 对比 Shell 并发请求

```bash
# Shell：用 xargs 并发 curl（受限于进程数）
cat urls.txt | xargs -n1 -P10 curl -s -o /dev/null -w "%{url} -> %{http_code}\n"

# Shell：用 GNU parallel 并发
parallel -j20 'curl -s -o /dev/null -w "{} -> %{http_code}\n" {}' < urls.txt

# 缺点：
# 1. 每个请求一个进程，开销大
# 2. 结果收集/处理不方便
# 3. 超过几百并发就很吃力
# 4. 错误处理困难
```

### aiohttp vs 其他方案

| 方案 | 并发模型 | 1000 URL 耗时 | 内存占用 | 适用场景 |
|------|---------|--------------|---------|---------|
| curl + bash | 多进程 | 分钟级 | 高 | 简单脚本 |
| requests + 线程池 | 多线程 | 数十秒 | 中 | 中等并发 |
| httpx async | 协程 | 数秒 | 低 | 通用异步 |
| **aiohttp** | 协程 | 数秒 | **最低** | 大规模并发 |

**aiohttp 的独特优势：**
- 比 httpx 更底层，性能更好
- 内置服务端功能（无需 FastAPI/Flask）
- 原生 WebSocket 支持
- 连接池精细控制

---

## 2. 安装与环境

```bash
# 安装 aiohttp
pip install aiohttp

# 安装可选加速组件
pip install aiohttp[speedups]  # 包含 aiodns 和 cchardet

# 安装信号量限流
pip install aiohttp aiosignal

# 验证
python3 -c "import aiohttp; print(f'aiohttp 版本: {aiohttp.__version__}')"
```

**版本要求：**
- Python >= 3.8
- aiohttp >= 3.8

---

## 3. 核心概念与原理

### 异步 I/O 原理

```
同步模型（requests）：
  请求1 [====等待响应====] -> 请求2 [====等待响应====] -> 请求3 ...
  总耗时 = 请求1 + 请求2 + 请求3

异步模型（aiohttp）：
  请求1 [====等待响应====]
  请求2 [====等待响应====]   # 同时发出
  请求3 [====等待响应====]   # 同时发出
  总耗时 ≈ max(请求1, 请求2, 请求3)
```

### aiohttp 架构

```
asyncio 事件循环
    |
    v
aiohttp.ClientSession（会话管理、连接池）
    |
    v
TCPConnector（TCP 连接管理）
    |
    +-- 连接复用
    +-- DNS 缓存
    +-- SSL 配置
    +-- 并发限制
    |
    v
底层 socket（非阻塞 I/O）
```

### 关键概念

```python
# ClientSession = 连接池 + Cookie + Header 管理
# 一个 Session 应该在整个应用生命周期中复用

# TCPConnector = 控制并发连接数
# limit=100 表示同时最多 100 个 TCP 连接

# Semaphore = 信号量，限制并发任务数
# 防止一次性创建太多请求压垮目标
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 异步 GET 请求

```python
#!/usr/bin/env python3
"""aiohttp 基础 GET 请求"""

import asyncio
import aiohttp

async def basic_get():
    """最简单的异步 GET 请求"""

    # 创建 Session（类似 requests.Session）
    async with aiohttp.ClientSession() as session:

        # 发送 GET 请求
        # 等价于: curl https://httpbin.org/get
        async with session.get("https://httpbin.org/get") as response:
            # 状态码
            print(f"状态码: {response.status}")

            # 响应头
            print(f"Content-Type: {response.headers['Content-Type']}")

            # 读取 JSON
            data = await response.json()
            print(f"来源 IP: {data['origin']}")

            # 读取文本
            # text = await response.text()

async def get_with_params():
    """带参数的 GET 请求"""

    # 等价于: curl "https://httpbin.org/get?key=value&name=sre"
    params = {"key": "value", "name": "sre"}

    async with aiohttp.ClientSession() as session:
        async with session.get("https://httpbin.org/get", params=params) as resp:
            data = await resp.json()
            print(f"请求参数: {data['args']}")

async def get_with_headers():
    """带自定义 Header 的请求"""

    headers = {
        "User-Agent": "SRE-Bot/1.0",
        "X-Request-ID": "req-001"
    }

    async with aiohttp.ClientSession(headers=headers) as session:
        async with session.get("https://httpbin.org/headers") as resp:
            data = await resp.json()
            print(f"Headers: {data['headers']}")

# 运行
async def main():
    await basic_get()
    print("---")
    await get_with_params()
    print("---")
    await get_with_headers()

asyncio.run(main())
```

### 4.2 异步 POST 请求

```python
#!/usr/bin/env python3
"""aiohttp POST 请求示例"""

import asyncio
import aiohttp

async def post_json():
    """POST JSON 数据"""
    # 等价于: curl -X POST -H "Content-Type: application/json" \
    #         -d '{"service":"nginx","action":"restart"}' \
    #         https://httpbin.org/post

    payload = {"service": "nginx", "action": "restart"}

    async with aiohttp.ClientSession() as session:
        async with session.post("https://httpbin.org/post", json=payload) as resp:
            result = await resp.json()
            print(f"服务端收到: {result['json']}")

async def post_form():
    """POST 表单数据"""
    # 等价于: curl -X POST -d "user=admin&pass=123" https://httpbin.org/post

    data = {"user": "admin", "pass": "123"}

    async with aiohttp.ClientSession() as session:
        async with session.post("https://httpbin.org/post", data=data) as resp:
            result = await resp.json()
            print(f"表单数据: {result['form']}")

async def post_file():
    """上传文件"""
    # 等价于: curl -F "file=@/tmp/test.txt" https://httpbin.org/post

    # 准备测试文件
    import tempfile
    with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
        f.write("测试文件内容\n")
        filepath = f.name

    async with aiohttp.ClientSession() as session:
        # 使用 FormData 构建多部分表单
        form = aiohttp.FormData()
        form.add_field("file",
                       open(filepath, "rb"),
                       filename="test.txt",
                       content_type="text/plain")

        async with session.post("https://httpbin.org/post", data=form) as resp:
            print(f"上传状态: {resp.status}")

async def main():
    await post_json()
    print("---")
    await post_form()
    print("---")
    await post_file()

asyncio.run(main())
```

### 4.3 超时与错误处理

```python
#!/usr/bin/env python3
"""超时配置与异常处理"""

import asyncio
import aiohttp

async def timeout_demo():
    """超时控制"""

    # 全局超时配置
    timeout = aiohttp.ClientTimeout(
        total=30,        # 总超时（秒）
        connect=5,       # 连接超时
        sock_connect=5,  # Socket 连接超时
        sock_read=10     # Socket 读取超时
    )

    async with aiohttp.ClientSession(timeout=timeout) as session:
        try:
            # 请求一个延迟 2 秒的接口
            async with session.get("https://httpbin.org/delay/2") as resp:
                print(f"正常响应: {resp.status}")
        except asyncio.TimeoutError:
            print("请求超时！")
        except aiohttp.ClientError as e:
            print(f"客户端错误: {e}")

async def error_handling_demo():
    """完整的错误处理"""

    async with aiohttp.ClientSession() as session:
        try:
            async with session.get("https://httpbin.org/status/500",
                                   timeout=aiohttp.ClientTimeout(total=5)) as resp:
                # 检查状态码
                if resp.status >= 500:
                    print(f"服务器错误: {resp.status}")
                elif resp.status >= 400:
                    print(f"客户端错误: {resp.status}")
                else:
                    data = await resp.text()
                    print(f"成功: {data[:100]}")

        except aiohttp.ClientConnectorError:
            print("连接失败：目标不可达")
        except aiohttp.ClientResponseError as e:
            print(f"响应错误: {e.status} {e.message}")
        except asyncio.TimeoutError:
            print("超时")
        except Exception as e:
            print(f"未知错误: {type(e).__name__}: {e}")

async def main():
    await timeout_demo()
    print("---")
    await error_handling_demo()

asyncio.run(main())
```

---

## 5. 进阶用法

### 5.1 连接池与并发控制

```python
#!/usr/bin/env python3
"""连接池配置与并发控制"""

import asyncio
import aiohttp

async def connection_pool_demo():
    """精细化连接池控制"""

    # 创建 TCP 连接器
    connector = aiohttp.TCPConnector(
        limit=100,           # 总连接数上限
        limit_per_host=10,   # 每个主机的连接数上限
        ttl_dns_cache=300,   # DNS 缓存时间（秒）
        use_dns_cache=True,  # 启用 DNS 缓存
        ssl=False,           # 是否验证 SSL（False = 等价 curl -k）
        force_close=False,   # 是否在每个请求后关闭连接
        enable_cleanup_closed=True,  # 清理关闭的连接
    )

    async with aiohttp.ClientSession(connector=connector) as session:
        # 使用信号量限制并发数
        semaphore = asyncio.Semaphore(20)  # 最多同时 20 个请求

        async def limited_request(url):
            """带并发限制的请求"""
            async with semaphore:
                async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as resp:
                    return url, resp.status

        # 创建 50 个并发请求（但同时最多 20 个）
        urls = [f"https://httpbin.org/get?i={i}" for i in range(50)]
        tasks = [limited_request(url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        success = sum(1 for r in results if not isinstance(r, Exception))
        print(f"完成 {success}/{len(urls)} 个请求")

asyncio.run(connection_pool_demo())
```

### 5.2 流式下载

```python
#!/usr/bin/env python3
"""流式下载大文件"""

import asyncio
import aiohttp

async def stream_download(url: str, save_path: str):
    """
    流式下载文件（不占用大量内存）
    等价于: curl -o save_path url
    """
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            if resp.status != 200:
                print(f"下载失败: HTTP {resp.status}")
                return

            total = int(resp.headers.get("Content-Length", 0))
            downloaded = 0

            with open(save_path, "wb") as f:
                # 分块读取
                async for chunk in resp.content.iter_chunked(8192):
                    f.write(chunk)
                    downloaded += len(chunk)
                    if total:
                        pct = downloaded / total * 100
                        print(f"\r下载进度: {pct:.1f}%", end="")

            print(f"\n下载完成: {save_path} ({downloaded} bytes)")

# 示例
# asyncio.run(stream_download("https://example.com/file.tar.gz", "/tmp/file.tar.gz"))
```

### 5.3 aiohttp 服务端

```python
#!/usr/bin/env python3
"""aiohttp 异步 HTTP 服务端"""

from aiohttp import web
import json
from datetime import datetime

# ========== 路由处理函数 ==========

async def health(request):
    """健康检查"""
    return web.json_response({
        "status": "ok",
        "timestamp": datetime.now().isoformat()
    })

async def get_services(request):
    """获取服务列表"""
    # 获取查询参数
    status = request.query.get("status", "all")
    return web.json_response({
        "services": [
            {"name": "nginx", "status": "running"},
            {"name": "redis", "status": "running"},
            {"name": "mysql", "status": "stopped"},
        ]
    })

async def create_task(request):
    """创建任务"""
    try:
        data = await request.json()
    except json.JSONDecodeError:
        return web.json_response({"error": "无效的 JSON"}, status=400)

    return web.json_response({
        "task_id": "task-001",
        "command": data.get("command"),
        "status": "created"
    }, status=201)

# ========== 中间件 ==========

@web.middleware
async def log_middleware(request, handler):
    """请求日志中间件"""
    import time
    start = time.monotonic()
    response = await handler(request)
    duration = time.monotonic() - start
    print(f"{request.method} {request.path} -> {response.status} ({duration:.3f}s)")
    return response

# ========== 应用初始化 ==========

def create_app():
    app = web.Application(middlewares=[log_middleware])

    # 注册路由
    app.router.add_get("/health", health)
    app.router.add_get("/services", get_services)
    app.router.add_post("/tasks", create_task)

    return app

if __name__ == "__main__":
    app = create_app()
    # 启动服务: python3 server.py
    web.run_app(app, host="0.0.0.0", port=8080)
```

### 5.4 WebSocket 实时通信

```python
#!/usr/bin/env python3
"""WebSocket 实时日志推送服务"""

from aiohttp import web
import asyncio
import json
from datetime import datetime

# 存储所有 WebSocket 连接
ws_clients = set()

async def websocket_handler(request):
    """WebSocket 处理函数"""
    ws = web.WebSocketResponse()
    await ws.prepare(request)

    # 注册客户端
    ws_clients.add(ws)
    print(f"新客户端连接，当前 {len(ws_clients)} 个")

    try:
        async for msg in ws:
            if msg.type == web.WSMsgType.TEXT:
                # 收到客户端消息
                data = json.loads(msg.data)
                print(f"收到消息: {data}")

                # 回复
                await ws.send_json({
                    "type": "echo",
                    "data": data,
                    "timestamp": datetime.now().isoformat()
                })

            elif msg.type == web.WSMsgType.ERROR:
                print(f"WebSocket 错误: {ws.exception()}")

    finally:
        ws_clients.discard(ws)
        print(f"客户端断开，剩余 {len(ws_clients)} 个")

    return ws

async def broadcast_message(message: dict):
    """向所有连接的客户端广播消息"""
    if ws_clients:
        tasks = [ws.send_json(message) for ws in ws_clients if not ws.closed]
        await asyncio.gather(*tasks, return_exceptions=True)

async def simulate_log_stream(app):
    """模拟日志流推送"""
    while True:
        await asyncio.sleep(2)
        log_entry = {
            "type": "log",
            "level": "INFO",
            "message": f"系统运行正常 - {datetime.now().strftime('%H:%M:%S')}",
            "timestamp": datetime.now().isoformat()
        }
        await broadcast_message(log_entry)

async def start_background_tasks(app):
    """启动后台任务"""
    app["log_stream"] = asyncio.create_task(simulate_log_stream(app))

async def cleanup_background_tasks(app):
    """清理后台任务"""
    app["log_stream"].cancel()
    await app["log_stream"]

def create_app():
    app = web.Application()
    app.router.add_get("/ws", websocket_handler)
    app.on_startup.append(start_background_tasks)
    app.on_cleanup.append(cleanup_background_tasks)
    return app

# 客户端测试代码
async def ws_client_demo():
    """WebSocket 客户端测试"""
    import aiohttp

    async with aiohttp.ClientSession() as session:
        async with session.ws_connect("http://localhost:8080/ws") as ws:
            # 发送消息
            await ws.send_json({"type": "subscribe", "channel": "logs"})

            # 接收消息
            async for msg in ws:
                if msg.type == aiohttp.WSMsgType.TEXT:
                    data = json.loads(msg.data)
                    print(f"收到: {data}")
                elif msg.type == aiohttp.WSMsgType.CLOSED:
                    break

if __name__ == "__main__":
    app = create_app()
    web.run_app(app, host="0.0.0.0", port=8080)
```

---

## 6. SRE 实战案例：异步并发探测 1000 个 URL

```python
#!/usr/bin/env python3
"""
SRE 实战：异步并发探测大量 URL 的可用性
功能：
  1. 并发探测数百/数千个 URL
  2. 记录响应时间、状态码、错误信息
  3. 支持自定义并发数和超时
  4. 实时进度显示
  5. 生成探测报告（文本 + JSON）
  6. 对比 shell 方案的性能优势
"""

import asyncio
import aiohttp
import time
import json
import sys
from dataclasses import dataclass, asdict, field
from datetime import datetime
from typing import Optional, List
from collections import Counter


# ========== 配置 ==========
# 并发控制
MAX_CONCURRENT = 50       # 最大并发数
REQUEST_TIMEOUT = 10      # 单个请求超时（秒）
TOTAL_TIMEOUT = 300       # 总超时（秒）
DNS_CACHE_TTL = 600       # DNS 缓存时间

# 生成测试 URL 列表（实际使用时从文件或 CMDB 读取）
def generate_test_urls(count=100):
    """生成测试 URL 列表"""
    urls = []

    # httpbin 提供的各种测试端点
    base_urls = [
        "https://httpbin.org/status/200",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/get",
        "https://httpbin.org/json",
    ]

    for i in range(count):
        urls.append(base_urls[i % len(base_urls)] + f"?probe_id={i}")

    return urls


# ========== 数据模型 ==========
@dataclass
class ProbeResult:
    """探测结果"""
    url: str
    status_code: Optional[int] = None
    response_time: float = 0.0
    is_available: bool = False
    error: Optional[str] = None
    content_length: int = 0
    ssl_valid: bool = True
    redirect_url: Optional[str] = None
    probed_at: str = field(default_factory=lambda: datetime.now().isoformat())


# ========== 核心探测逻辑 ==========
class URLProber:
    """URL 可用性探测器"""

    def __init__(
        self,
        max_concurrent: int = MAX_CONCURRENT,
        timeout: int = REQUEST_TIMEOUT,
        headers: dict = None
    ):
        self.max_concurrent = max_concurrent
        self.timeout = timeout
        self.headers = headers or {"User-Agent": "SRE-Prober/1.0"}
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.completed = 0
        self.total = 0

    async def probe_single(
        self,
        session: aiohttp.ClientSession,
        url: str
    ) -> ProbeResult:
        """
        探测单个 URL

        参数:
            session: aiohttp 会话
            url: 目标 URL

        返回:
            ProbeResult: 探测结果
        """
        result = ProbeResult(url=url)

        async with self.semaphore:  # 控制并发
            try:
                start = time.monotonic()

                async with session.get(
                    url,
                    timeout=aiohttp.ClientTimeout(total=self.timeout),
                    allow_redirects=True,
                    ssl=True  # 验证 SSL
                ) as response:
                    result.response_time = round(time.monotonic() - start, 3)
                    result.status_code = response.status
                    result.is_available = 200 <= response.status < 400
                    result.content_length = int(
                        response.headers.get("Content-Length", 0)
                    )

                    # 检查是否有重定向
                    if response.history:
                        result.redirect_url = str(response.url)

            except asyncio.TimeoutError:
                result.response_time = self.timeout
                result.error = "超时"
            except aiohttp.ClientConnectorSSLError as e:
                result.error = f"SSL 证书错误: {str(e)[:80]}"
                result.ssl_valid = False
            except aiohttp.ClientConnectorError as e:
                result.error = f"连接失败: {str(e)[:80]}"
            except aiohttp.InvalidURL:
                result.error = "无效的 URL"
            except Exception as e:
                result.error = f"{type(e).__name__}: {str(e)[:80]}"

            # 更新进度
            self.completed += 1
            if self.completed % 10 == 0 or self.completed == self.total:
                pct = self.completed / self.total * 100
                sys.stdout.write(
                    f"\r进度: {self.completed}/{self.total} ({pct:.1f}%)"
                )
                sys.stdout.flush()

        return result

    async def probe_all(self, urls: List[str]) -> List[ProbeResult]:
        """
        批量并发探测所有 URL

        参数:
            urls: URL 列表

        返回:
            List[ProbeResult]: 探测结果列表
        """
        self.total = len(urls)
        self.completed = 0

        # 创建连接器
        connector = aiohttp.TCPConnector(
            limit=self.max_concurrent,
            limit_per_host=10,
            ttl_dns_cache=DNS_CACHE_TTL,
            use_dns_cache=True,
            enable_cleanup_closed=True,
        )

        async with aiohttp.ClientSession(
            connector=connector,
            headers=self.headers
        ) as session:
            tasks = [self.probe_single(session, url) for url in urls]
            results = await asyncio.gather(*tasks, return_exceptions=True)

        # 处理异常
        final_results = []
        for i, r in enumerate(results):
            if isinstance(r, Exception):
                final_results.append(ProbeResult(
                    url=urls[i],
                    error=f"任务异常: {str(r)[:80]}"
                ))
            else:
                final_results.append(r)

        print()  # 换行
        return final_results


# ========== 报告生成 ==========
class ProbeReporter:
    """探测报告生成器"""

    @staticmethod
    def generate_text_report(results: List[ProbeResult], elapsed: float) -> str:
        """生成文本报告"""
        lines = []
        lines.append("=" * 70)
        lines.append("          SRE URL 可用性探测报告")
        lines.append("=" * 70)
        lines.append(f"探测时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        lines.append(f"URL 总数: {len(results)}")
        lines.append(f"总耗时:   {elapsed:.2f}s")
        lines.append(f"并发数:   {MAX_CONCURRENT}")

        # 统计
        available = [r for r in results if r.is_available]
        unavailable = [r for r in results if not r.is_available]
        response_times = [r.response_time for r in results if r.response_time > 0]

        lines.append("-" * 70)
        lines.append(f"可用: {len(available)}  |  不可用: {len(unavailable)}  |  "
                      f"可用率: {len(available)/len(results)*100:.1f}%")

        if response_times:
            avg_time = sum(response_times) / len(response_times)
            max_time = max(response_times)
            min_time = min(response_times)
            # 计算 P95
            sorted_times = sorted(response_times)
            p95_idx = int(len(sorted_times) * 0.95)
            p95_time = sorted_times[p95_idx] if p95_idx < len(sorted_times) else max_time

            lines.append(f"响应时间: 最小={min_time:.3f}s  平均={avg_time:.3f}s  "
                          f"P95={p95_time:.3f}s  最大={max_time:.3f}s")

        # 状态码分布
        status_codes = Counter(r.status_code for r in results if r.status_code)
        lines.append(f"状态码分布: {dict(status_codes)}")

        # 错误分布
        errors = Counter(r.error for r in results if r.error)
        if errors:
            lines.append("\n错误类型分布:")
            for err, count in errors.most_common(10):
                lines.append(f"  [{count:>4}次] {err}")

        # 不可用的 URL 列表
        if unavailable:
            lines.append(f"\n不可用 URL 列表（共 {len(unavailable)} 个）:")
            for r in unavailable[:20]:  # 只显示前 20 个
                lines.append(
                    f"  [!] {r.url[:60]:<60} "
                    f"HTTP {r.status_code or 'N/A':>3}  "
                    f"{r.response_time:.3f}s  "
                    f"{r.error or ''}"
                )
            if len(unavailable) > 20:
                lines.append(f"  ... 还有 {len(unavailable)-20} 个")

        # 响应最慢的 URL
        slowest = sorted(results, key=lambda r: r.response_time, reverse=True)[:5]
        lines.append("\n响应最慢 Top 5:")
        for r in slowest:
            lines.append(
                f"  {r.response_time:.3f}s  HTTP {r.status_code or 'N/A':>3}  {r.url[:60]}"
            )

        lines.append("=" * 70)
        return "\n".join(lines)

    @staticmethod
    def export_json(results: List[ProbeResult], filepath: str):
        """导出 JSON 结果"""
        data = {
            "probe_time": datetime.now().isoformat(),
            "total": len(results),
            "available": sum(1 for r in results if r.is_available),
            "unavailable": sum(1 for r in results if not r.is_available),
            "results": [asdict(r) for r in results]
        }
        with open(filepath, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        print(f"JSON 报告已导出: {filepath}")


# ========== 对比：同步方案（演示差异） ==========
def sync_probe_demo(urls: List[str]):
    """
    同步探测（对比用）
    等价于 Shell:
      for url in urls; do curl -s -o /dev/null -w "%{http_code}" $url; done
    """
    import requests

    start = time.time()
    results = []
    for url in urls[:10]:  # 只测 10 个
        try:
            resp = requests.get(url, timeout=5)
            results.append((url, resp.status_code, resp.elapsed.total_seconds()))
        except Exception as e:
            results.append((url, None, 0))
    elapsed = time.time() - start
    print(f"同步探测 {len(urls[:10])} 个 URL 耗时: {elapsed:.2f}s")
    return elapsed


# ========== 主函数 ==========
async def main():
    """主入口"""
    # 生成测试 URL
    urls = generate_test_urls(20)  # 用 20 个 URL 演示（实际可以 1000+）
    print(f"准备探测 {len(urls)} 个 URL...")
    print(f"并发数: {MAX_CONCURRENT}, 超时: {REQUEST_TIMEOUT}s\n")

    # 创建探测器
    prober = URLProber(
        max_concurrent=MAX_CONCURRENT,
        timeout=REQUEST_TIMEOUT
    )

    # 执行探测
    start = time.monotonic()
    results = await prober.probe_all(urls)
    elapsed = time.monotonic() - start

    # 生成报告
    reporter = ProbeReporter()
    report = reporter.generate_text_report(results, elapsed)
    print(report)

    # 导出 JSON
    reporter.export_json(results, "/tmp/probe_results.json")

    # 性能对比提示
    print(f"\n性能对比:")
    print(f"  异步探测 {len(urls)} 个 URL: {elapsed:.2f}s")
    print(f"  同步串行预估: ~{len(urls) * 1.5:.0f}s（每个约 1.5s）")
    print(f"  加速比: ~{len(urls) * 1.5 / max(elapsed, 0.1):.1f}x")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：每个请求创建新 Session

```python
# 错误：每次都创建新 Session（无法复用连接）
async def bad():
    for url in urls:
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                pass

# 正确：复用 Session
async def good():
    async with aiohttp.ClientSession() as session:
        for url in urls:
            async with session.get(url) as resp:
                pass
```

### 坑点 2：不限制并发数导致目标崩溃

```python
# 错误：无限制并发
tasks = [session.get(url) for url in thousand_urls]
await asyncio.gather(*tasks)  # 可能同时发 1000 个请求！

# 正确：用信号量限制
sem = asyncio.Semaphore(50)
async def limited_get(url):
    async with sem:
        async with session.get(url) as resp:
            return resp.status

tasks = [limited_get(url) for url in thousand_urls]
await asyncio.gather(*tasks)
```

### 坑点 3：忘记读取响应体

```python
# 错误：只检查状态码就关闭（可能导致连接不能复用）
async with session.get(url) as resp:
    status = resp.status
    # 没读取 body 就退出了

# 正确：读取响应体（至少调用 release）
async with session.get(url) as resp:
    status = resp.status
    await resp.read()  # 读取并丢弃
```

### 坑点 4：在异步函数中用同步操作

```python
# 错误：在 async 函数中用 time.sleep
async def bad():
    time.sleep(1)  # 阻塞整个事件循环！

# 正确：用 asyncio.sleep
async def good():
    await asyncio.sleep(1)

# 如果必须调用同步阻塞函数
import functools
async def run_sync():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_function, args)
```

### 最佳实践

1. **复用 ClientSession**：整个应用生命周期一个 Session
2. **限制并发**：用 Semaphore 或 TCPConnector.limit
3. **设置超时**：connect + read 都要设
4. **启用 DNS 缓存**：减少 DNS 查询
5. **优雅关闭**：await session.close()
6. **SSL 验证**：生产环境不要关闭

---

## 8. 速查表

### aiohttp 客户端速查

```python
import aiohttp, asyncio

# 创建 Session
async with aiohttp.ClientSession() as session:
    # GET
    async with session.get(url) as resp: ...

    # POST JSON
    async with session.post(url, json=data) as resp: ...

    # POST 表单
    async with session.post(url, data=form) as resp: ...

    # 带超时
    timeout = aiohttp.ClientTimeout(total=10)
    async with session.get(url, timeout=timeout) as resp: ...

    # 带 Header
    async with session.get(url, headers={"X-Key": "val"}) as resp: ...

    # 读取响应
    status = resp.status           # 状态码
    text = await resp.text()       # 文本
    data = await resp.json()       # JSON
    body = await resp.read()       # 字节
```

### 并发控制速查

```python
# 方式 1: Semaphore（推荐）
sem = asyncio.Semaphore(50)
async with sem:
    await do_request()

# 方式 2: TCPConnector
conn = aiohttp.TCPConnector(limit=100, limit_per_host=10)
async with aiohttp.ClientSession(connector=conn) as s: ...

# 方式 3: asyncio.wait 批次执行
batch_size = 50
for i in range(0, len(tasks), batch_size):
    batch = tasks[i:i+batch_size]
    await asyncio.gather(*batch)
```

### 对比 curl 命令

| curl 命令 | aiohttp 等价 |
|-----------|-------------|
| `curl url` | `await session.get(url)` |
| `curl -X POST -d '...' url` | `await session.post(url, json=data)` |
| `curl -H "Key: Val" url` | `await session.get(url, headers=h)` |
| `curl -k url` | `session.get(url, ssl=False)` |
| `curl --max-time 5 url` | `session.get(url, timeout=ClientTimeout(total=5))` |
| `curl -o file url` | 流式下载 + `resp.content.iter_chunked()` |
| `xargs -P10 curl` | `asyncio.gather() + Semaphore(10)` |
