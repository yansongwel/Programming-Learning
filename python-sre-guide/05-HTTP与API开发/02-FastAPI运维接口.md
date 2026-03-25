# FastAPI 运维接口 — 用 Python 快速搭建 SRE 运维 API

> 一句话说明：FastAPI 是 Python 高性能异步 Web 框架，自带参数校验和自动文档，特别适合 SRE 快速搭建运维操作 API（重启服务/查看状态/执行命令）。

---

## 1. 是什么 & 为什么用它

### 传统运维 API 的痛点

```bash
# Shell：用 nc/socat 写简易 HTTP 服务？太原始
while true; do
  echo -e "HTTP/1.1 200 OK\r\n\r\n$(date)" | nc -l -p 8080 -q 1
done

# Shell：用 CGI/Flask 写？需要大量样板代码
# 参数校验？手动写。文档？手动写。异步？不支持。
```

### FastAPI 的优势

| 特性 | Flask（老牌） | FastAPI |
|------|-------------|---------|
| 性能 | 同步，较慢 | 异步，接近 Go/Node |
| 参数校验 | 手动或 WTForms | 自动（Pydantic） |
| API 文档 | 手动写 Swagger | 自动生成 OpenAPI |
| 类型提示 | 可选 | 核心特性 |
| 异步支持 | 需 gevent | 原生 async/await |
| 学习曲线 | 低 | 低（Flask 经验可迁移） |

---

## 2. 安装与环境

```bash
# 创建虚拟环境
python3 -m venv ~/sre-venv
source ~/sre-venv/bin/activate

# 安装 FastAPI 和 ASGI 服务器
pip install fastapi uvicorn[standard]

# 安装常用扩展
pip install pydantic python-multipart  # 表单/文件上传支持
pip install python-jose[cryptography]  # JWT 认证
pip install passlib[bcrypt]            # 密码加密

# 验证安装
python3 -c "import fastapi; print(f'FastAPI 版本: {fastapi.__version__}')"
```

**启动方式：**
```bash
# 开发模式（自动重载）
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# 生产模式（多 worker）
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# 等价的 curl 测试
curl http://localhost:8000/docs   # 自动生成的 Swagger UI
curl http://localhost:8000/redoc  # ReDoc 文档
```

---

## 3. 核心概念与原理

### FastAPI 请求处理流程

```
HTTP 请求
    |
    v
Uvicorn (ASGI 服务器) -- 接收连接、解析 HTTP
    |
    v
FastAPI 路由匹配 -- 根据路径和方法找到处理函数
    |
    v
中间件链 -- 认证、日志、CORS 等
    |
    v
依赖注入 -- 数据库连接、当前用户等
    |
    v
参数解析 & 校验 -- Pydantic 自动校验
    |
    v
路由处理函数 -- 你写的业务逻辑
    |
    v
响应序列化 -- Pydantic 模型转 JSON
    |
    v
HTTP 响应
```

### 关键概念

```python
# 路径操作 = HTTP 方法 + URL 路径 + 处理函数
@app.get("/services")          # GET /services
@app.post("/services/restart") # POST /services/restart

# 依赖注入 = 自动提供函数需要的参数
def get_db():  # 依赖
    return database_connection

@app.get("/data")
def read_data(db=Depends(get_db)):  # 自动注入
    return db.query()

# Pydantic 模型 = 数据校验 + 序列化
class Service(BaseModel):
    name: str           # 必填字符串
    port: int           # 必填整数
    replicas: int = 1   # 可选，默认 1
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 Hello World

```python
#!/usr/bin/env python3
"""最简 FastAPI 应用"""
# 文件名: main.py
# 启动: uvicorn main:app --reload

from fastapi import FastAPI

# 创建应用实例
app = FastAPI(
    title="SRE 运维 API",
    description="运维自动化接口服务",
    version="1.0.0"
)

# 定义路由 — GET 方法
@app.get("/")
def root():
    """根路径 — 返回服务信息"""
    return {"service": "SRE Ops API", "status": "running"}

# 定义路由 — 带路径参数
@app.get("/services/{service_name}")
def get_service(service_name: str):
    """
    获取服务信息
    等价于: curl http://localhost:8000/services/nginx
    """
    return {
        "name": service_name,
        "status": "active",
        "uptime": "3d 12h"
    }

# 定义路由 — 带查询参数
@app.get("/services")
def list_services(status: str = "all", limit: int = 10):
    """
    列出服务列表
    等价于: curl "http://localhost:8000/services?status=active&limit=5"
    """
    return {
        "filter": status,
        "limit": limit,
        "services": ["nginx", "redis", "mysql"][:limit]
    }
```

### 4.2 请求体与 Pydantic 模型

```python
#!/usr/bin/env python3
"""Pydantic 模型验证示例"""

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field, validator
from typing import Optional, List
from enum import Enum
from datetime import datetime

app = FastAPI(title="SRE Ops API")

# ========== 数据模型 ==========

class ServiceAction(str, Enum):
    """服务操作枚举"""
    START = "start"
    STOP = "stop"
    RESTART = "restart"
    RELOAD = "reload"

class ServiceCommand(BaseModel):
    """服务操作请求体"""
    service_name: str = Field(
        ...,  # ... 表示必填
        min_length=1,
        max_length=50,
        description="服务名称",
        examples=["nginx"]
    )
    action: ServiceAction = Field(
        ...,
        description="操作类型"
    )
    operator: str = Field(
        ...,
        description="操作人"
    )
    reason: Optional[str] = Field(
        None,
        description="操作原因"
    )
    force: bool = Field(
        False,
        description="是否强制执行"
    )

    # 自定义校验器
    @validator("service_name")
    def validate_service_name(cls, v):
        """校验服务名：只允许字母、数字、横线、下划线"""
        import re
        if not re.match(r'^[a-zA-Z0-9_-]+$', v):
            raise ValueError("服务名只能包含字母、数字、横线、下划线")
        return v

class ServiceResponse(BaseModel):
    """服务操作响应"""
    success: bool
    message: str
    service_name: str
    action: str
    timestamp: datetime = Field(default_factory=datetime.now)

# ========== 路由 ==========

@app.post("/ops/service", response_model=ServiceResponse)
def operate_service(cmd: ServiceCommand):
    """
    对服务执行操作
    等价于:
    curl -X POST http://localhost:8000/ops/service \
      -H "Content-Type: application/json" \
      -d '{"service_name":"nginx","action":"restart","operator":"admin"}'
    """
    # 模拟服务操作
    # 实际场景调用 subprocess 或 API
    return ServiceResponse(
        success=True,
        message=f"服务 {cmd.service_name} 已执行 {cmd.action.value}",
        service_name=cmd.service_name,
        action=cmd.action.value,
    )
```

### 4.3 路径参数、查询参数、请求头

```python
#!/usr/bin/env python3
"""参数类型详解"""

from fastapi import FastAPI, Header, Query, Path, HTTPException
from typing import Optional, List

app = FastAPI()

@app.get("/hosts/{host_id}/metrics")
def get_host_metrics(
    # 路径参数（必填）
    host_id: int = Path(
        ...,
        title="主机 ID",
        ge=1,          # 大于等于 1
        le=100000      # 小于等于 100000
    ),
    # 查询参数（可选）
    metric: str = Query(
        "cpu",
        description="指标类型",
        regex="^(cpu|memory|disk|network)$"
    ),
    # 查询参数（可选，带范围校验）
    duration: int = Query(
        3600,
        description="时间范围（秒）",
        ge=60,
        le=86400
    ),
    # 请求头参数
    x_api_key: Optional[str] = Header(None, description="API 密钥"),
):
    """
    获取主机监控指标
    等价于:
    curl -H "X-API-Key: mykey" \
      "http://localhost:8000/hosts/42/metrics?metric=cpu&duration=3600"
    """
    if x_api_key != "secret-key" and x_api_key is not None:
        raise HTTPException(status_code=401, detail="无效的 API Key")

    return {
        "host_id": host_id,
        "metric": metric,
        "duration": duration,
        "data": [
            {"timestamp": "2024-01-01T00:00:00", "value": 45.2},
            {"timestamp": "2024-01-01T00:05:00", "value": 52.1},
        ]
    }
```

---

## 5. 进阶用法

### 5.1 中间件

```python
#!/usr/bin/env python3
"""中间件：请求日志、耗时统计、异常处理"""

import time
import logging
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("sre-api")

# ========== 中间件 1: 请求日志与耗时统计 ==========
@app.middleware("http")
async def log_requests(request: Request, call_next):
    """记录每个请求的方法、路径、耗时、状态码"""
    start_time = time.monotonic()
    request_id = request.headers.get("X-Request-ID", "N/A")

    # 记录请求开始
    logger.info(f"[{request_id}] 开始 {request.method} {request.url.path}")

    # 处理请求
    response = await call_next(request)

    # 计算耗时
    duration = time.monotonic() - start_time
    logger.info(
        f"[{request_id}] 完成 {request.method} {request.url.path} "
        f"-> {response.status_code} ({duration:.3f}s)"
    )

    # 添加响应头
    response.headers["X-Process-Time"] = f"{duration:.3f}"
    response.headers["X-Request-ID"] = request_id

    return response

# ========== 中间件 2: CORS 跨域支持 ==========
from starlette.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],         # 允许的来源（生产环境应限制）
    allow_credentials=True,
    allow_methods=["*"],         # 允许的 HTTP 方法
    allow_headers=["*"],         # 允许的请求头
)

@app.get("/")
def root():
    return {"message": "带中间件的 API"}
```

### 5.2 后台任务

```python
#!/usr/bin/env python3
"""后台任务：异步执行耗时操作"""

from fastapi import FastAPI, BackgroundTasks
from datetime import datetime
import time

app = FastAPI()

# 模拟任务存储
task_results = {}

def long_running_task(task_id: str, command: str):
    """
    耗时的后台任务
    实际场景：部署、备份、批量操作等
    """
    task_results[task_id] = {"status": "running", "started": datetime.now().isoformat()}

    # 模拟耗时操作
    time.sleep(5)

    task_results[task_id] = {
        "status": "completed",
        "started": task_results[task_id]["started"],
        "completed": datetime.now().isoformat(),
        "command": command,
        "output": f"命令 '{command}' 执行完成"
    }

@app.post("/ops/async-task")
def create_async_task(
    command: str,
    background_tasks: BackgroundTasks
):
    """
    创建异步任务（立即返回，后台执行）
    等价于: nohup command &
    """
    import uuid
    task_id = str(uuid.uuid4())[:8]

    # 添加后台任务
    background_tasks.add_task(long_running_task, task_id, command)

    return {
        "task_id": task_id,
        "status": "accepted",
        "message": "任务已提交，后台执行中"
    }

@app.get("/ops/async-task/{task_id}")
def get_task_status(task_id: str):
    """查询异步任务状态"""
    if task_id not in task_results:
        return {"task_id": task_id, "status": "not_found"}
    return {"task_id": task_id, **task_results[task_id]}
```

### 5.3 认证鉴权

```python
#!/usr/bin/env python3
"""API 认证鉴权：API Key + JWT Token"""

from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.security.api_key import APIKeyHeader
from datetime import datetime, timedelta
from typing import Optional
import hashlib
import hmac
import json
import base64
import time

app = FastAPI()

# ========== 方式 1: API Key 认证 ==========
API_KEYS = {
    "key-admin-001": {"role": "admin", "name": "管理员"},
    "key-viewer-002": {"role": "viewer", "name": "只读用户"},
}

api_key_header = APIKeyHeader(name="X-API-Key")

def verify_api_key(api_key: str = Security(api_key_header)):
    """验证 API Key"""
    if api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="无效的 API Key")
    return API_KEYS[api_key]

@app.get("/api/v1/services")
def list_services(user=Depends(verify_api_key)):
    """
    需要 API Key 认证的接口
    curl -H "X-API-Key: key-admin-001" http://localhost:8000/api/v1/services
    """
    return {
        "user": user["name"],
        "role": user["role"],
        "services": ["nginx", "redis", "mysql"]
    }

# ========== 方式 2: Bearer Token 认证 ==========
SECRET_KEY = "your-secret-key-change-in-production"
security = HTTPBearer()

def create_simple_token(username: str, role: str, expires_hours: int = 24) -> str:
    """创建简单 Token（生产环境请用 python-jose JWT）"""
    payload = {
        "sub": username,
        "role": role,
        "exp": int(time.time()) + expires_hours * 3600
    }
    payload_json = json.dumps(payload)
    payload_b64 = base64.b64encode(payload_json.encode()).decode()
    signature = hmac.new(SECRET_KEY.encode(), payload_b64.encode(), hashlib.sha256).hexdigest()
    return f"{payload_b64}.{signature}"

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    """验证 Bearer Token"""
    token = credentials.credentials
    try:
        parts = token.split(".")
        if len(parts) != 2:
            raise ValueError("Token 格式错误")
        payload_b64, signature = parts
        expected_sig = hmac.new(SECRET_KEY.encode(), payload_b64.encode(), hashlib.sha256).hexdigest()
        if not hmac.compare_digest(signature, expected_sig):
            raise ValueError("签名验证失败")
        payload = json.loads(base64.b64decode(payload_b64))
        if payload["exp"] < time.time():
            raise ValueError("Token 已过期")
        return payload
    except Exception as e:
        raise HTTPException(status_code=401, detail=f"Token 验证失败: {str(e)}")

@app.post("/auth/token")
def login(username: str, password: str):
    """获取 Token"""
    # 简化版：实际应查数据库
    if username == "admin" and password == "admin123":
        token = create_simple_token(username, "admin")
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="用户名或密码错误")

@app.get("/api/v2/services")
def list_services_v2(user=Depends(verify_token)):
    """
    需要 Bearer Token 认证的接口
    curl -H "Authorization: Bearer <token>" http://localhost:8000/api/v2/services
    """
    return {
        "user": user["sub"],
        "role": user["role"],
        "services": ["nginx", "redis", "mysql"]
    }
```

### 5.4 依赖注入

```python
#!/usr/bin/env python3
"""依赖注入：数据库连接、配置管理"""

from fastapi import FastAPI, Depends
from typing import Generator

app = FastAPI()

# ========== 模拟数据库连接 ==========
class FakeDB:
    """模拟数据库连接"""
    def __init__(self):
        self.connected = True
        print("数据库连接已创建")

    def query(self, sql):
        return [{"id": 1, "name": "nginx"}]

    def close(self):
        self.connected = False
        print("数据库连接已关闭")

def get_db() -> Generator:
    """
    数据库连接依赖
    每个请求创建一个连接，请求结束自动关闭
    """
    db = FakeDB()
    try:
        yield db  # 提供给路由函数使用
    finally:
        db.close()  # 请求结束后自动执行

# ========== 配置依赖 ==========
class Settings:
    """应用配置"""
    app_name: str = "SRE Ops API"
    debug: bool = False
    max_workers: int = 4

def get_settings() -> Settings:
    return Settings()

# ========== 使用依赖注入 ==========
@app.get("/db/services")
def list_from_db(
    db=Depends(get_db),
    settings=Depends(get_settings)
):
    """路由函数自动获得数据库连接和配置"""
    services = db.query("SELECT * FROM services")
    return {
        "app": settings.app_name,
        "services": services
    }
```

---

## 6. SRE 实战案例：运维操作 API

```python
#!/usr/bin/env python3
"""
SRE 实战：运维操作 API 服务
功能：
  1. 查看服务状态（systemctl status）
  2. 重启/停止服务
  3. 执行安全命令
  4. 查看系统信息
  5. 操作审计日志
文件名: ops_api.py
启动: uvicorn ops_api:app --host 0.0.0.0 --port 8000
"""

import subprocess
import platform
import psutil  # pip install psutil; 如果没有可以用 subprocess 替代
import os
import time
import json
import logging
from datetime import datetime
from typing import Optional, List
from enum import Enum

from fastapi import FastAPI, HTTPException, Depends, Security, BackgroundTasks
from fastapi.security.api_key import APIKeyHeader
from pydantic import BaseModel, Field, validator

# ========== 日志配置 ==========
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger("ops-api")

# ========== 应用初始化 ==========
app = FastAPI(
    title="SRE 运维操作 API",
    description="提供服务管理、系统信息查询、命令执行等运维操作接口",
    version="1.0.0",
    docs_url="/docs",      # Swagger UI 地址
    redoc_url="/redoc",    # ReDoc 地址
)

# ========== 认证配置 ==========
API_KEYS = {
    "ops-admin-key-001": {"role": "admin", "name": "运维管理员"},
    "ops-viewer-key-002": {"role": "viewer", "name": "只读用户"},
}

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def get_current_user(api_key: str = Security(api_key_header)):
    """获取当前用户"""
    if not api_key or api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="无效的 API Key")
    return API_KEYS[api_key]

def require_admin(user=Depends(get_current_user)):
    """要求管理员权限"""
    if user["role"] != "admin":
        raise HTTPException(status_code=403, detail="需要管理员权限")
    return user

# ========== 审计日志 ==========
AUDIT_LOG_FILE = "/tmp/ops_audit.log"

def write_audit_log(operator: str, action: str, target: str, result: str):
    """写入审计日志"""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "operator": operator,
        "action": action,
        "target": target,
        "result": result
    }
    logger.info(f"审计: {json.dumps(log_entry, ensure_ascii=False)}")
    with open(AUDIT_LOG_FILE, "a") as f:
        f.write(json.dumps(log_entry, ensure_ascii=False) + "\n")

# ========== 数据模型 ==========

class ServiceAction(str, Enum):
    START = "start"
    STOP = "stop"
    RESTART = "restart"
    RELOAD = "reload"
    STATUS = "status"

class ServiceRequest(BaseModel):
    """服务操作请求"""
    service_name: str = Field(..., min_length=1, max_length=50, description="服务名称")
    action: ServiceAction = Field(..., description="操作类型")
    reason: Optional[str] = Field(None, description="操作原因")

    @validator("service_name")
    def validate_name(cls, v):
        import re
        if not re.match(r'^[a-zA-Z0-9_.-]+$', v):
            raise ValueError("服务名包含非法字符")
        return v

class CommandRequest(BaseModel):
    """命令执行请求"""
    command: str = Field(..., description="要执行的命令")
    timeout: int = Field(30, ge=1, le=300, description="超时时间（秒）")

    @validator("command")
    def validate_command(cls, v):
        """安全校验：禁止危险命令"""
        dangerous = ["rm -rf", "mkfs", "dd if=", ":(){ :", "shutdown", "reboot",
                      "init 0", "init 6", "> /dev/sd"]
        for d in dangerous:
            if d in v.lower():
                raise ValueError(f"禁止执行危险命令: {d}")
        return v

class ServiceStatusResponse(BaseModel):
    """服务状态响应"""
    service_name: str
    active: bool
    status: str
    pid: Optional[int] = None
    uptime: Optional[str] = None
    memory: Optional[str] = None

class SystemInfo(BaseModel):
    """系统信息"""
    hostname: str
    os: str
    kernel: str
    cpu_count: int
    cpu_percent: float
    memory_total_gb: float
    memory_used_percent: float
    disk_usage: dict
    uptime: str
    load_average: list

# ========== API 路由 ==========

# --- 健康检查（无需认证） ---
@app.get("/health")
def health_check():
    """
    服务健康检查
    curl http://localhost:8000/health
    """
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "version": "1.0.0"
    }

# --- 系统信息 ---
@app.get("/system/info", response_model=SystemInfo)
def get_system_info(user=Depends(get_current_user)):
    """
    获取系统信息
    curl -H "X-API-Key: ops-admin-key-001" http://localhost:8000/system/info
    """
    try:
        # 获取运行时间
        boot_time = datetime.fromtimestamp(psutil.boot_time())
        uptime = str(datetime.now() - boot_time).split(".")[0]

        # 获取磁盘信息
        disk = psutil.disk_usage("/")
        disk_info = {
            "total_gb": round(disk.total / (1024**3), 1),
            "used_gb": round(disk.used / (1024**3), 1),
            "free_gb": round(disk.free / (1024**3), 1),
            "percent": disk.percent
        }

        memory = psutil.virtual_memory()

        return SystemInfo(
            hostname=platform.node(),
            os=f"{platform.system()} {platform.release()}",
            kernel=platform.release(),
            cpu_count=psutil.cpu_count(),
            cpu_percent=psutil.cpu_percent(interval=1),
            memory_total_gb=round(memory.total / (1024**3), 1),
            memory_used_percent=memory.percent,
            disk_usage=disk_info,
            uptime=uptime,
            load_average=list(os.getloadavg())
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"获取系统信息失败: {str(e)}")

# --- 服务管理 ---
@app.get("/services/{service_name}/status")
def get_service_status(service_name: str, user=Depends(get_current_user)):
    """
    查看服务状态
    curl -H "X-API-Key: ops-admin-key-001" http://localhost:8000/services/nginx/status
    """
    try:
        result = subprocess.run(
            ["systemctl", "status", service_name],
            capture_output=True, text=True, timeout=10
        )
        # 解析 systemctl 输出
        is_active = "active (running)" in result.stdout
        lines = result.stdout.strip().split("\n")

        return {
            "service_name": service_name,
            "active": is_active,
            "status": "running" if is_active else "stopped",
            "raw_output": result.stdout[:500]  # 限制输出长度
        }
    except FileNotFoundError:
        # 非 systemd 系统的兼容处理
        return {
            "service_name": service_name,
            "active": False,
            "status": "unknown",
            "raw_output": "systemctl 不可用（非 systemd 系统）"
        }
    except subprocess.TimeoutExpired:
        raise HTTPException(status_code=504, detail="查询超时")

@app.post("/services/operate")
def operate_service(
    req: ServiceRequest,
    user=Depends(require_admin)  # 需要管理员权限
):
    """
    操作服务（启动/停止/重启/重载）
    curl -X POST http://localhost:8000/services/operate \
      -H "X-API-Key: ops-admin-key-001" \
      -H "Content-Type: application/json" \
      -d '{"service_name":"nginx","action":"restart","reason":"更新配置"}'
    """
    try:
        # 记录审计日志
        write_audit_log(
            operator=user["name"],
            action=req.action.value,
            target=req.service_name,
            result="executing"
        )

        result = subprocess.run(
            ["systemctl", req.action.value, req.service_name],
            capture_output=True, text=True, timeout=30
        )

        success = result.returncode == 0
        write_audit_log(
            operator=user["name"],
            action=req.action.value,
            target=req.service_name,
            result="success" if success else f"failed: {result.stderr[:200]}"
        )

        return {
            "success": success,
            "service_name": req.service_name,
            "action": req.action.value,
            "message": "操作成功" if success else result.stderr[:200],
            "operator": user["name"],
            "timestamp": datetime.now().isoformat()
        }
    except subprocess.TimeoutExpired:
        raise HTTPException(status_code=504, detail="操作超时")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# --- 命令执行 ---
@app.post("/ops/exec")
def execute_command(
    req: CommandRequest,
    user=Depends(require_admin)
):
    """
    执行系统命令（仅管理员）
    curl -X POST http://localhost:8000/ops/exec \
      -H "X-API-Key: ops-admin-key-001" \
      -H "Content-Type: application/json" \
      -d '{"command":"df -h","timeout":10}'
    """
    write_audit_log(user["name"], "exec", req.command, "executing")

    try:
        result = subprocess.run(
            req.command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=req.timeout
        )

        output = {
            "command": req.command,
            "returncode": result.returncode,
            "stdout": result.stdout[:10000],   # 限制输出大小
            "stderr": result.stderr[:5000],
            "operator": user["name"],
            "timestamp": datetime.now().isoformat()
        }

        write_audit_log(user["name"], "exec", req.command,
                        "success" if result.returncode == 0 else "failed")

        return output
    except subprocess.TimeoutExpired:
        write_audit_log(user["name"], "exec", req.command, "timeout")
        raise HTTPException(status_code=504, detail=f"命令执行超时（{req.timeout}秒）")

# --- 审计日志查询 ---
@app.get("/audit/logs")
def get_audit_logs(
    limit: int = 50,
    user=Depends(get_current_user)
):
    """
    查看审计日志
    curl -H "X-API-Key: ops-admin-key-001" \
      "http://localhost:8000/audit/logs?limit=20"
    """
    logs = []
    try:
        with open(AUDIT_LOG_FILE, "r") as f:
            for line in f:
                line = line.strip()
                if line:
                    logs.append(json.loads(line))
    except FileNotFoundError:
        pass

    # 返回最新的 N 条
    return {
        "total": len(logs),
        "limit": limit,
        "logs": logs[-limit:]
    }

# --- 进程信息 ---
@app.get("/system/processes")
def get_top_processes(
    sort_by: str = "cpu",  # cpu 或 memory
    limit: int = 10,
    user=Depends(get_current_user)
):
    """
    获取资源占用 Top N 进程
    等价于: ps aux --sort=-%cpu | head -10
    """
    try:
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent']):
            try:
                info = proc.info
                processes.append({
                    "pid": info["pid"],
                    "name": info["name"],
                    "cpu_percent": info["cpu_percent"] or 0,
                    "memory_percent": round(info["memory_percent"] or 0, 1),
                })
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                continue

        # 排序
        key = "cpu_percent" if sort_by == "cpu" else "memory_percent"
        processes.sort(key=lambda x: x[key], reverse=True)

        return {"processes": processes[:limit]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# ========== 启动事件 ==========
@app.on_event("startup")
async def startup_event():
    logger.info("SRE 运维 API 服务启动")

@app.on_event("shutdown")
async def shutdown_event():
    logger.info("SRE 运维 API 服务关闭")

# ========== 直接运行 ==========
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：同步函数阻塞事件循环

```python
# 错误：在 async 路由中执行同步阻塞操作
@app.get("/bad")
async def bad_handler():
    time.sleep(10)  # 阻塞整个事件循环！
    return {"result": "done"}

# 正确方案 1：使用同步函数（FastAPI 自动放入线程池）
@app.get("/good1")
def good_handler():  # 注意没有 async
    time.sleep(10)  # 在线程池中执行，不阻塞
    return {"result": "done"}

# 正确方案 2：使用异步操作
@app.get("/good2")
async def good_handler():
    await asyncio.sleep(10)  # 异步等待
    return {"result": "done"}
```

### 坑点 2：未做输入校验导致命令注入

```python
# 错误：直接拼接用户输入到命令
@app.get("/exec")
def bad_exec(cmd: str):
    os.system(cmd)  # 极度危险！

# 正确：白名单 + 参数化
ALLOWED_COMMANDS = {"df", "free", "uptime", "ps"}

@app.get("/exec")
def safe_exec(cmd: str):
    if cmd not in ALLOWED_COMMANDS:
        raise HTTPException(400, "不允许的命令")
    result = subprocess.run([cmd, "-h"], capture_output=True, text=True)
    return {"output": result.stdout}
```

### 坑点 3：没有设置 CORS

```python
# 如果前端需要调用 API，必须配置 CORS
from starlette.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://ops.company.com"],  # 指定具体域名
    allow_methods=["GET", "POST"],
    allow_headers=["X-API-Key"],
)
```

### 最佳实践

1. **始终使用 Pydantic 模型**校验输入
2. **所有写操作需要认证**，读操作按需
3. **操作审计**：谁在什么时间做了什么
4. **限制命令执行范围**：白名单优于黑名单
5. **设置超时**：防止请求/命令卡住
6. **使用 HTTPS**：生产环境必须
7. **限流**：防止 API 被滥用

---

## 8. 速查表

### 路由装饰器

```python
@app.get("/path")            # GET 请求
@app.post("/path")           # POST 请求
@app.put("/path")            # PUT 请求
@app.delete("/path")         # DELETE 请求
@app.patch("/path")          # PATCH 请求
@app.options("/path")        # OPTIONS 请求
```

### 参数类型

```python
# 路径参数
@app.get("/items/{item_id}")
def read(item_id: int): ...

# 查询参数
@app.get("/items")
def read(skip: int = 0, limit: int = 10): ...

# 请求体
@app.post("/items")
def create(item: ItemModel): ...

# 请求头
from fastapi import Header
def read(x_token: str = Header(...)): ...

# Cookie
from fastapi import Cookie
def read(session_id: str = Cookie(...)): ...

# 文件上传
from fastapi import File, UploadFile
@app.post("/upload")
async def upload(file: UploadFile = File(...)): ...
```

### 响应类型

```python
# JSON 响应（默认）
return {"key": "value"}

# 指定响应模型
@app.get("/items", response_model=ItemModel)

# 自定义状态码
@app.post("/items", status_code=201)

# 直接返回 Response
from fastapi.responses import JSONResponse, PlainTextResponse
return JSONResponse(content=data, status_code=201)
return PlainTextResponse("ok")
```

### uvicorn 启动参数

```bash
uvicorn app:app \
  --host 0.0.0.0 \        # 监听地址
  --port 8000 \            # 端口
  --workers 4 \            # Worker 进程数（生产用）
  --reload \               # 代码变更自动重载（开发用）
  --ssl-keyfile key.pem \  # HTTPS 私钥
  --ssl-certfile cert.pem \ # HTTPS 证书
  --log-level info          # 日志级别
```

### curl 测试速查

```bash
# 健康检查
curl http://localhost:8000/health

# GET 带认证
curl -H "X-API-Key: ops-admin-key-001" http://localhost:8000/system/info

# POST JSON
curl -X POST http://localhost:8000/services/operate \
  -H "X-API-Key: ops-admin-key-001" \
  -H "Content-Type: application/json" \
  -d '{"service_name":"nginx","action":"restart"}'

# 查看自动生成的 API 文档
curl http://localhost:8000/openapi.json

# 浏览器打开 Swagger UI
# http://localhost:8000/docs
```
