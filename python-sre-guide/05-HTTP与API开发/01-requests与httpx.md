# requests 与 httpx — Python SRE 必备的 HTTP 请求库

> 一句话说明：`requests` 是 Python 最流行的同步 HTTP 库，`httpx` 是其异步升级版，两者是 SRE 日常调用 API、健康检查、Webhook 通知的核心工具。

---

## 1. 是什么 & 为什么用它

### 对比 Shell/curl

在传统运维中，我们习惯用 `curl` 做 HTTP 请求：

```bash
# Shell: 用 curl 检查服务健康
curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health

# Shell: POST JSON 数据
curl -X POST https://api.example.com/deploy \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token123" \
  -d '{"service": "web", "version": "1.2.3"}'

# Shell: 批量检查多个 URL（串行，效率低）
for url in https://a.com https://b.com https://c.com; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  echo "$url -> $code"
done
```

**为什么要用 Python 的 HTTP 库？**

| 特性 | curl/Shell | requests/httpx |
|------|-----------|----------------|
| 错误处理 | 靠 `$?` 和字符串解析 | 结构化异常 + 状态码对象 |
| JSON 处理 | 需要 `jq` 辅助 | 内置 `.json()` 解析 |
| Session/Cookie | 手动管理 | 自动保持会话 |
| 认证方式 | 手动拼 Header | 内置 Auth 对象 |
| 重试机制 | 自己写循环 | 内置 Retry 适配器 |
| 异步并发 | 需要 `xargs` / `parallel` | httpx 原生 async |
| 证书处理 | `-k` 忽略 | 精细化 SSL 配置 |
| 可编程性 | 复杂逻辑难以维护 | 完整编程语言支持 |

---

## 2. 安装与环境

```bash
# 创建虚拟环境（推荐）
python3 -m venv ~/sre-venv
source ~/sre-venv/bin/activate

# 安装 requests
pip install requests

# 安装 httpx（支持异步）
pip install httpx

# 安装重试库（配合 requests 使用）
pip install urllib3

# 验证安装
python3 -c "import requests; print(f'requests 版本: {requests.__version__}')"
python3 -c "import httpx; print(f'httpx 版本: {httpx.__version__}')"
```

**版本要求：**
- Python >= 3.7
- requests >= 2.28
- httpx >= 0.24

---

## 3. 核心概念与原理

### HTTP 请求生命周期

```
客户端构造请求 -> DNS 解析 -> TCP 连接 -> TLS 握手(HTTPS)
    -> 发送请求 -> 等待响应 -> 接收响应 -> 连接复用/关闭
```

### requests 架构

```
requests.get(url)
    |
    v
PreparedRequest（预处理请求对象）
    |
    v
Session（会话层：Cookie、Auth、连接池）
    |
    v
HTTPAdapter（传输适配器：重试、SSL、代理）
    |
    v
urllib3（底层连接池管理）
    |
    v
socket（系统套接字）
```

### httpx vs requests

```
requests  = 同步请求（一个请求完成后才能发下一个）
httpx     = 同步 + 异步请求（可并发发送多个请求）

# httpx 的 API 几乎与 requests 一致，迁移成本极低
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 GET 请求

```python
#!/usr/bin/env python3
"""GET 请求基础示例"""

import requests

# === 最简单的 GET 请求 ===
# 等价于: curl https://httpbin.org/get
response = requests.get("https://httpbin.org/get")

# 查看状态码（等价于 curl -w "%{http_code}"）
print(f"状态码: {response.status_code}")  # 200

# 查看响应头
print(f"Content-Type: {response.headers['Content-Type']}")

# 查看响应体（文本）
print(f"响应文本: {response.text[:100]}...")

# 解析 JSON 响应（等价于 curl ... | jq .）
data = response.json()
print(f"来源IP: {data['origin']}")

# === 带查询参数的 GET ===
# 等价于: curl "https://httpbin.org/get?name=sre&role=ops"
params = {
    "name": "sre",
    "role": "ops"
}
response = requests.get("https://httpbin.org/get", params=params)
print(f"请求URL: {response.url}")  # 自动编码参数
print(f"参数回显: {response.json()['args']}")

# === 带自定义 Header 的 GET ===
# 等价于: curl -H "X-Custom: hello" -H "User-Agent: SRE-Bot" ...
headers = {
    "User-Agent": "SRE-Monitor/1.0",
    "X-Request-ID": "req-001",
    "Accept": "application/json"
}
response = requests.get("https://httpbin.org/headers", headers=headers)
print(f"服务端收到的 Headers: {response.json()['headers']}")
```

### 4.2 POST 请求

```python
#!/usr/bin/env python3
"""POST 请求基础示例"""

import requests
import json

# === POST JSON 数据 ===
# 等价于: curl -X POST https://httpbin.org/post \
#         -H "Content-Type: application/json" \
#         -d '{"service": "nginx", "action": "restart"}'
payload = {
    "service": "nginx",
    "action": "restart",
    "operator": "sre-bot"
}
response = requests.post("https://httpbin.org/post", json=payload)
print(f"状态码: {response.status_code}")
print(f"服务端收到的 JSON: {response.json()['json']}")

# === POST 表单数据 ===
# 等价于: curl -X POST https://httpbin.org/post \
#         -d "username=admin&password=secret"
form_data = {
    "username": "admin",
    "password": "secret"
}
response = requests.post("https://httpbin.org/post", data=form_data)
print(f"服务端收到的表单: {response.json()['form']}")

# === POST 上传文件 ===
# 等价于: curl -X POST https://httpbin.org/post -F "file=@/tmp/test.txt"
# 先创建一个测试文件
with open("/tmp/test_upload.txt", "w") as f:
    f.write("这是一个测试文件内容\n服务器日志片段...")

with open("/tmp/test_upload.txt", "rb") as f:
    files = {"file": ("test.txt", f, "text/plain")}
    response = requests.post("https://httpbin.org/post", files=files)
    print(f"上传结果: {response.status_code}")
```

### 4.3 PUT / DELETE / PATCH

```python
#!/usr/bin/env python3
"""其他 HTTP 方法示例"""

import requests

# === PUT 更新资源 ===
# 等价于: curl -X PUT https://httpbin.org/put -d '{"status": "active"}'
response = requests.put(
    "https://httpbin.org/put",
    json={"status": "active", "version": "2.0"}
)
print(f"PUT 状态码: {response.status_code}")

# === DELETE 删除资源 ===
# 等价于: curl -X DELETE https://httpbin.org/delete
response = requests.delete("https://httpbin.org/delete")
print(f"DELETE 状态码: {response.status_code}")

# === PATCH 部分更新 ===
# 等价于: curl -X PATCH https://httpbin.org/patch -d '{"replicas": 3}'
response = requests.patch(
    "https://httpbin.org/patch",
    json={"replicas": 3}
)
print(f"PATCH 状态码: {response.status_code}")
```

---

## 5. 进阶用法

### 5.1 Session 会话保持

```python
#!/usr/bin/env python3
"""Session 会话管理 — 复用连接、保持 Cookie"""

import requests

# 创建 Session 对象（底层复用 TCP 连接，性能更好）
# 类似于 curl 的 -c cookie.txt -b cookie.txt
session = requests.Session()

# 设置全局 Header（所有请求都会带上）
session.headers.update({
    "User-Agent": "SRE-Monitor/1.0",
    "Authorization": "Bearer my-token-123"
})

# 设置全局超时（需要通过适配器或每次请求指定）
# Session 不直接支持全局超时，但可以封装

# 第一个请求（建立连接）
resp1 = session.get("https://httpbin.org/cookies/set/session_id/abc123")
print(f"设置 Cookie 后: {session.cookies.get_dict()}")

# 第二个请求（自动携带 Cookie，复用连接）
resp2 = session.get("https://httpbin.org/cookies")
print(f"Cookie 回显: {resp2.json()}")

# 使用完毕关闭 Session（释放连接池）
session.close()

# 推荐使用 with 语句自动管理
with requests.Session() as s:
    s.headers["X-API-Key"] = "key-456"
    resp = s.get("https://httpbin.org/headers")
    print(f"API Key 已发送: {resp.json()['headers'].get('X-Api-Key')}")
```

### 5.2 超时与重试

```python
#!/usr/bin/env python3
"""超时控制与自动重试机制"""

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# === 超时控制 ===
# connect_timeout: TCP 连接超时
# read_timeout: 等待响应超时
try:
    # 等价于: curl --connect-timeout 3 --max-time 10 ...
    response = requests.get(
        "https://httpbin.org/delay/2",
        timeout=(3, 10)  # (连接超时秒数, 读取超时秒数)
    )
    print(f"响应耗时: {response.elapsed.total_seconds():.2f}s")
except requests.exceptions.ConnectTimeout:
    print("连接超时！服务器可能宕机")
except requests.exceptions.ReadTimeout:
    print("读取超时！服务器响应太慢")
except requests.exceptions.Timeout:
    print("请求超时！")

# === 自动重试机制 ===
# 配置重试策略
retry_strategy = Retry(
    total=3,                    # 最多重试 3 次
    backoff_factor=1,           # 退避因子：1s, 2s, 4s
    status_forcelist=[429, 500, 502, 503, 504],  # 这些状态码触发重试
    allowed_methods=["GET", "POST"],  # 允许重试的 HTTP 方法
    raise_on_status=False       # 重试耗尽后不抛异常
)

# 创建带重试的 Session
session = requests.Session()
adapter = HTTPAdapter(
    max_retries=retry_strategy,
    pool_connections=10,        # 连接池大小
    pool_maxsize=20             # 每个主机最大连接数
)
session.mount("http://", adapter)   # 对 http:// 开头的 URL 生效
session.mount("https://", adapter)  # 对 https:// 开头的 URL 生效

# 使用带重试的 Session 发请求
response = session.get("https://httpbin.org/status/200", timeout=5)
print(f"带重试的请求结果: {response.status_code}")

session.close()
```

### 5.3 认证方式

```python
#!/usr/bin/env python3
"""多种认证方式示例"""

import requests
from requests.auth import HTTPBasicAuth, HTTPDigestAuth

# === Basic 认证 ===
# 等价于: curl -u admin:password https://httpbin.org/basic-auth/admin/password
response = requests.get(
    "https://httpbin.org/basic-auth/admin/password",
    auth=HTTPBasicAuth("admin", "password")
)
print(f"Basic Auth: {response.status_code} - {response.json()}")

# 简写形式（元组）
response = requests.get(
    "https://httpbin.org/basic-auth/admin/password",
    auth=("admin", "password")
)
print(f"Basic Auth 简写: {response.json()['authenticated']}")

# === Bearer Token 认证 ===
# 等价于: curl -H "Authorization: Bearer mytoken" ...
headers = {"Authorization": "Bearer my-jwt-token-here"}
response = requests.get("https://httpbin.org/bearer", headers=headers)
print(f"Bearer Auth: {response.status_code}")

# === 自定义认证类 ===
class TokenAuth(requests.auth.AuthBase):
    """自定义 Token 认证"""
    def __init__(self, token):
        self.token = token

    def __call__(self, r):
        # 在请求中添加认证头
        r.headers["X-API-Token"] = self.token
        r.headers["X-Timestamp"] = str(int(__import__("time").time()))
        return r

response = requests.get(
    "https://httpbin.org/headers",
    auth=TokenAuth("secret-token-789")
)
print(f"自定义认证 Headers: {response.json()['headers']}")
```

### 5.4 SSL/TLS 证书处理

```python
#!/usr/bin/env python3
"""SSL 证书验证与配置"""

import requests
import warnings

# === 默认验证 SSL 证书（推荐） ===
response = requests.get("https://httpbin.org/get")
print(f"SSL 验证通过: {response.status_code}")

# === 跳过 SSL 验证（等价于 curl -k） ===
# 注意：仅在测试环境使用，生产环境严禁关闭！
warnings.filterwarnings("ignore", message="Unverified HTTPS request")
response = requests.get("https://httpbin.org/get", verify=False)
print(f"跳过 SSL: {response.status_code}")

# === 指定 CA 证书 ===
# 等价于: curl --cacert /path/to/ca-bundle.crt ...
# response = requests.get(
#     "https://internal-api.company.com",
#     verify="/etc/ssl/certs/company-ca.crt"
# )

# === 客户端证书认证（mTLS 双向认证） ===
# 等价于: curl --cert client.crt --key client.key ...
# response = requests.get(
#     "https://secure-api.company.com",
#     cert=("/path/to/client.crt", "/path/to/client.key")
# )
```

### 5.5 httpx 异步请求

```python
#!/usr/bin/env python3
"""httpx 异步 HTTP 请求"""

import httpx
import asyncio
import time

# === httpx 同步用法（API 与 requests 几乎一致） ===
response = httpx.get("https://httpbin.org/get")
print(f"httpx 同步: {response.status_code}")
print(f"JSON: {response.json()['url']}")

# === httpx 异步用法 ===
async def async_demo():
    """演示异步请求的优势"""

    # 创建异步客户端（类似 requests.Session）
    async with httpx.AsyncClient(
        timeout=httpx.Timeout(10.0),    # 全局超时
        follow_redirects=True,           # 自动跟随重定向
        headers={"User-Agent": "SRE-AsyncBot/1.0"}
    ) as client:

        # 单个异步请求
        resp = await client.get("https://httpbin.org/get")
        print(f"异步单请求: {resp.status_code}")

        # === 并发请求（核心优势） ===
        urls = [
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
        ]

        start = time.time()

        # 同时发起 3 个请求（每个延迟 1 秒）
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)

        elapsed = time.time() - start

        for resp in responses:
            print(f"  状态: {resp.status_code}")

        # 串行需要 3 秒，并发只需约 1 秒
        print(f"3 个并发请求总耗时: {elapsed:.2f}s（串行需要 ~3s）")

# 运行异步函数
asyncio.run(async_demo())
```

### 5.6 流式下载大文件

```python
#!/usr/bin/env python3
"""流式下载大文件（不会撑爆内存）"""

import requests

def download_file(url, save_path, chunk_size=8192):
    """
    流式下载文件
    等价于: curl -o save_path -L url
    """
    with requests.get(url, stream=True, timeout=30) as resp:
        resp.raise_for_status()  # 检查状态码

        # 获取文件大小
        total = int(resp.headers.get("Content-Length", 0))
        downloaded = 0

        with open(save_path, "wb") as f:
            for chunk in resp.iter_content(chunk_size=chunk_size):
                f.write(chunk)
                downloaded += len(chunk)
                if total:
                    pct = downloaded / total * 100
                    print(f"\r下载进度: {pct:.1f}% ({downloaded}/{total})", end="")

    print(f"\n下载完成: {save_path}")

# 示例：下载一个测试文件
# download_file("https://example.com/large-file.tar.gz", "/tmp/large-file.tar.gz")
```

---

## 6. SRE 实战案例：批量 API 健康检查 + Webhook 通知

```python
#!/usr/bin/env python3
"""
SRE 实战：批量 API 健康检查 + Webhook 通知
功能：
  1. 并发检查多个服务的健康端点
  2. 记录响应时间和状态
  3. 异常时发送 Webhook 告警（支持企业微信/钉钉/飞书）
  4. 生成检查报告
"""

import httpx
import asyncio
import time
import json
from datetime import datetime
from dataclasses import dataclass, field, asdict
from typing import Optional


# ========== 配置 ==========
# 待检查的服务列表
SERVICES = [
    {
        "name": "httpbin-主站",
        "url": "https://httpbin.org/status/200",
        "method": "GET",
        "expected_status": 200,
        "timeout": 5,
    },
    {
        "name": "httpbin-延迟接口",
        "url": "https://httpbin.org/delay/1",
        "method": "GET",
        "expected_status": 200,
        "timeout": 5,
    },
    {
        "name": "httpbin-JSON接口",
        "url": "https://httpbin.org/json",
        "method": "GET",
        "expected_status": 200,
        "timeout": 5,
    },
    {
        "name": "模拟故障服务",
        "url": "https://httpbin.org/status/503",
        "method": "GET",
        "expected_status": 200,  # 期望200但实际503
        "timeout": 5,
    },
]

# Webhook 配置（企业微信示例）
WEBHOOK_URL = "https://httpbin.org/post"  # 实际使用时替换为真实 Webhook 地址
ALERT_ENABLED = True  # 是否启用告警


# ========== 数据模型 ==========
@dataclass
class CheckResult:
    """健康检查结果"""
    name: str                           # 服务名称
    url: str                            # 检查 URL
    status_code: Optional[int] = None   # HTTP 状态码
    response_time: float = 0.0          # 响应时间（秒）
    is_healthy: bool = False            # 是否健康
    error: Optional[str] = None         # 错误信息
    checked_at: str = ""                # 检查时间

    def __post_init__(self):
        if not self.checked_at:
            self.checked_at = datetime.now().strftime("%Y-%m-%d %H:%M:%S")


# ========== 核心检查逻辑 ==========
async def check_service(
    client: httpx.AsyncClient,
    service: dict
) -> CheckResult:
    """
    检查单个服务的健康状态

    参数:
        client: httpx 异步客户端
        service: 服务配置字典

    返回:
        CheckResult: 检查结果
    """
    result = CheckResult(
        name=service["name"],
        url=service["url"]
    )

    try:
        start_time = time.monotonic()

        # 发送请求
        response = await client.request(
            method=service.get("method", "GET"),
            url=service["url"],
            timeout=service.get("timeout", 5),
            headers=service.get("headers", {}),
        )

        # 计算响应时间
        result.response_time = round(time.monotonic() - start_time, 3)
        result.status_code = response.status_code

        # 判断健康状态
        expected = service.get("expected_status", 200)
        result.is_healthy = response.status_code == expected

        if not result.is_healthy:
            result.error = f"期望状态码 {expected}，实际 {response.status_code}"

    except httpx.TimeoutException:
        result.response_time = service.get("timeout", 5)
        result.error = "请求超时"
    except httpx.ConnectError as e:
        result.error = f"连接失败: {str(e)[:100]}"
    except Exception as e:
        result.error = f"未知错误: {str(e)[:100]}"

    return result


async def batch_check(services: list) -> list:
    """
    批量并发检查所有服务

    参数:
        services: 服务配置列表

    返回:
        list[CheckResult]: 检查结果列表
    """
    async with httpx.AsyncClient(
        follow_redirects=True,
        verify=True,  # 生产环境务必开启 SSL 验证
    ) as client:
        # 创建并发任务
        tasks = [check_service(client, svc) for svc in services]
        # 并发执行所有检查
        results = await asyncio.gather(*tasks, return_exceptions=True)

    # 处理异常结果
    final_results = []
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            final_results.append(CheckResult(
                name=services[i]["name"],
                url=services[i]["url"],
                error=f"任务异常: {str(r)}"
            ))
        else:
            final_results.append(r)

    return final_results


# ========== Webhook 告警 ==========
async def send_webhook_alert(unhealthy_results: list):
    """
    发送 Webhook 告警通知

    支持格式：企业微信 / 钉钉 / 飞书
    这里以通用 JSON 格式演示
    """
    if not ALERT_ENABLED or not unhealthy_results:
        return

    # 构造告警消息
    alert_lines = []
    for r in unhealthy_results:
        alert_lines.append(
            f"  - {r.name}: 状态码={r.status_code}, "
            f"耗时={r.response_time}s, 错误={r.error}"
        )

    message = {
        "msgtype": "text",
        "text": {
            "content": (
                f"[SRE健康检查告警]\n"
                f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n"
                f"异常服务数: {len(unhealthy_results)}\n"
                f"详情:\n" + "\n".join(alert_lines)
            )
        }
    }

    try:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                WEBHOOK_URL,
                json=message,
                timeout=10
            )
            print(f"[告警] Webhook 发送结果: {resp.status_code}")
    except Exception as e:
        print(f"[告警] Webhook 发送失败: {e}")


# ========== 报告生成 ==========
def generate_report(results: list) -> str:
    """生成检查报告"""
    healthy = [r for r in results if r.is_healthy]
    unhealthy = [r for r in results if not r.is_healthy]

    report = []
    report.append("=" * 60)
    report.append("        SRE 服务健康检查报告")
    report.append("=" * 60)
    report.append(f"检查时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    report.append(f"检查总数: {len(results)}")
    report.append(f"健康: {len(healthy)}  |  异常: {len(unhealthy)}")
    report.append("-" * 60)

    # 详细结果
    for r in results:
        status = "OK" if r.is_healthy else "FAIL"
        icon = "[+]" if r.is_healthy else "[!]"
        report.append(
            f"{icon} {r.name:<20} | {status:<4} | "
            f"HTTP {r.status_code or 'N/A':<3} | "
            f"{r.response_time:.3f}s"
        )
        if r.error:
            report.append(f"    错误: {r.error}")

    report.append("-" * 60)

    # 统计信息
    response_times = [r.response_time for r in results if r.response_time > 0]
    if response_times:
        avg_time = sum(response_times) / len(response_times)
        max_time = max(response_times)
        report.append(f"平均响应时间: {avg_time:.3f}s")
        report.append(f"最大响应时间: {max_time:.3f}s")

    report.append("=" * 60)
    return "\n".join(report)


# ========== 主函数 ==========
async def main():
    """主入口"""
    print("开始批量健康检查...\n")

    start = time.monotonic()

    # 执行批量检查
    results = await batch_check(SERVICES)

    elapsed = time.monotonic() - start

    # 生成并打印报告
    report = generate_report(results)
    print(report)
    print(f"\n总检查耗时: {elapsed:.2f}s（并发执行）")

    # 发送告警
    unhealthy = [r for r in results if not r.is_healthy]
    if unhealthy:
        print(f"\n发现 {len(unhealthy)} 个异常服务，发送告警...")
        await send_webhook_alert(unhealthy)

    # 导出 JSON 结果（方便后续处理）
    json_results = [asdict(r) for r in results]
    output_path = "/tmp/health_check_results.json"
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(json_results, f, ensure_ascii=False, indent=2)
    print(f"\n检查结果已导出: {output_path}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：不设超时导致请求挂死

```python
# 错误：没有超时，请求可能永远不返回
response = requests.get("https://slow-api.com")  # 可能挂死！

# 正确：始终设置超时
response = requests.get("https://slow-api.com", timeout=(3, 10))
```

### 坑点 2：不检查状态码

```python
# 错误：假设请求一定成功
data = requests.get(url).json()  # 如果 500 了呢？

# 正确：检查后再处理
response = requests.get(url, timeout=5)
response.raise_for_status()  # 非 2xx 抛异常
data = response.json()
```

### 坑点 3：不关闭 Session

```python
# 错误：Session 泄漏
session = requests.Session()
session.get(url)
# 忘记 close()

# 正确：用 with 自动关闭
with requests.Session() as session:
    session.get(url)
```

### 坑点 4：生产环境关闭 SSL 验证

```python
# 严禁在生产环境使用！
requests.get(url, verify=False)  # 中间人攻击风险

# 正确：配置正确的 CA 证书
requests.get(url, verify="/etc/ssl/certs/ca-certificates.crt")
```

### 坑点 5：不处理重定向

```python
# requests 默认跟随重定向，但有时需要禁止
response = requests.get(url, allow_redirects=False)
if response.status_code in (301, 302):
    print(f"重定向到: {response.headers['Location']}")
```

### 最佳实践总结

```python
"""requests/httpx 最佳实践模板"""

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_robust_session():
    """创建一个生产级别的 HTTP Session"""
    session = requests.Session()

    # 1. 配置重试
    retry = Retry(
        total=3,
        backoff_factor=0.5,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry, pool_maxsize=20)
    session.mount("http://", adapter)
    session.mount("https://", adapter)

    # 2. 设置通用 Headers
    session.headers.update({
        "User-Agent": "SRE-Tool/1.0",
        "Accept": "application/json",
    })

    return session

def safe_request(session, method, url, **kwargs):
    """安全的 HTTP 请求封装"""
    kwargs.setdefault("timeout", (3, 10))  # 默认超时

    try:
        resp = session.request(method, url, **kwargs)
        resp.raise_for_status()
        return resp
    except requests.exceptions.HTTPError as e:
        print(f"HTTP 错误 {e.response.status_code}: {url}")
        raise
    except requests.exceptions.ConnectionError:
        print(f"连接失败: {url}")
        raise
    except requests.exceptions.Timeout:
        print(f"请求超时: {url}")
        raise
```

---

## 8. 速查表

### requests 速查

| 操作 | 代码 | 等价 curl |
|------|------|-----------|
| GET 请求 | `requests.get(url)` | `curl url` |
| POST JSON | `requests.post(url, json=data)` | `curl -X POST -H "Content-Type: application/json" -d '...' url` |
| POST 表单 | `requests.post(url, data=form)` | `curl -X POST -d "k=v" url` |
| 带 Header | `requests.get(url, headers=h)` | `curl -H "Key: Val" url` |
| Basic Auth | `requests.get(url, auth=(u, p))` | `curl -u user:pass url` |
| 超时 | `requests.get(url, timeout=5)` | `curl --max-time 5 url` |
| 跳过 SSL | `requests.get(url, verify=False)` | `curl -k url` |
| 下载文件 | `requests.get(url, stream=True)` | `curl -O url` |
| 上传文件 | `requests.post(url, files=f)` | `curl -F "file=@path" url` |
| 查看状态码 | `resp.status_code` | `curl -w "%{http_code}"` |
| 解析 JSON | `resp.json()` | `curl ... \| jq .` |
| 响应头 | `resp.headers['Key']` | `curl -I url` |

### httpx 异步速查

```python
# 创建异步客户端
async with httpx.AsyncClient() as client:
    # 单个请求
    resp = await client.get(url)

    # 并发请求
    tasks = [client.get(u) for u in urls]
    results = await asyncio.gather(*tasks)

    # POST JSON
    resp = await client.post(url, json=data)

    # 带超时
    resp = await client.get(url, timeout=5.0)

    # 流式下载
    async with client.stream("GET", url) as resp:
        async for chunk in resp.aiter_bytes():
            f.write(chunk)
```

### 常用状态码含义（SRE 视角）

| 状态码 | 含义 | SRE 关注点 |
|--------|------|------------|
| 200 | 成功 | 正常 |
| 301/302 | 重定向 | 检查配置是否正确 |
| 401 | 未认证 | Token 过期/错误 |
| 403 | 禁止访问 | 权限/IP 白名单问题 |
| 404 | 未找到 | 服务路径变更 |
| 429 | 请求过多 | 触发限流，需要退避重试 |
| 500 | 服务器错误 | 应用 Bug，查日志 |
| 502 | 网关错误 | 上游服务挂了 |
| 503 | 服务不可用 | 服务过载/维护中 |
| 504 | 网关超时 | 上游响应太慢 |
