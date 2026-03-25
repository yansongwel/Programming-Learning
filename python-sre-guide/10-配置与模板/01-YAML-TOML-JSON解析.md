# YAML/TOML/JSON 配置文件解析 — 用 Python 统一管理多格式运维配置

## 1. 是什么 & 为什么用它

在 SRE 日常工作中，配置文件无处不在：Kubernetes 用 YAML、应用程序用 TOML/JSON、Prometheus 用 YAML。
Python 提供了强大的多格式配置解析能力：

- **json**：标准库自带，零依赖，适合 API 响应和简单配置
- **PyYAML**：解析 YAML 的事实标准，对标 shell 下的 `yq`
- **tomli/tomllib**：解析 TOML 格式（Python 3.11+ 内置 `tomllib`）

统一用 Python 管理配置的好处：
- 可编程的配置校验与合并
- 多环境（dev/staging/prod）配置管理
- 比 shell 下用 `jq`/`yq` 更灵活、更易维护

## 2. 安装与环境

```bash
# Python 3.8+
python3 --version

# 安装依赖
pip install pyyaml toml jsonschema

# Python 3.11+ 已内置 tomllib，无需额外安装
# 对比 shell 工具
# sudo apt install jq  # JSON
# pip install yq       # YAML (基于 jq)
```

## 3. 核心概念与原理

### 3.1 三种格式对比

| 特性       | JSON           | YAML           | TOML           |
|------------|----------------|----------------|----------------|
| 注释支持   | 不支持         | 支持 `#`       | 支持 `#`       |
| 数据类型   | 基础类型       | 丰富（日期等） | 丰富（日期等） |
| 可读性     | 一般           | 好             | 好             |
| 使用场景   | API/Web        | K8s/Ansible    | Python项目配置 |
| 标准库支持 | 是             | 否             | 3.11+          |

### 3.2 解析流程

```
配置文件(文本) → 解析器(Parser) → Python 字典/对象 → 业务逻辑处理
                                                      ↓
Python 字典/对象 ← 序列化器(Dumper) ← 修改后的配置 ← 配置合并/校验
```

## 4. 基础用法

### 4.1 JSON 解析（标准库）

```python
#!/usr/bin/env python3
"""JSON 配置文件解析基础示例"""

import json
from pathlib import Path

# === 从字符串解析 JSON ===
json_str = '''
{
    "server": {
        "host": "0.0.0.0",
        "port": 8080,
        "debug": false
    },
    "database": {
        "url": "postgresql://localhost:5432/mydb",
        "pool_size": 10
    }
}
'''

# 解析 JSON 字符串为 Python 字典
config = json.loads(json_str)
print(f"服务器地址: {config['server']['host']}:{config['server']['port']}")
# 输出: 服务器地址: 0.0.0.0:8080

# === 写入 JSON 文件 ===
config_path = Path("/tmp/app_config.json")
with open(config_path, 'w', encoding='utf-8') as f:
    # ensure_ascii=False 保留中文
    # indent=2 格式化缩进
    json.dump(config, f, ensure_ascii=False, indent=2)

# === 从文件读取 JSON ===
with open(config_path, 'r', encoding='utf-8') as f:
    loaded_config = json.load(f)

print(f"数据库连接池: {loaded_config['database']['pool_size']}")

# === 处理特殊类型 ===
import datetime

class SREEncoder(json.JSONEncoder):
    """自定义编码器：处理日期等特殊类型"""
    def default(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)

# 使用自定义编码器
data = {
    "timestamp": datetime.datetime.now(),
    "tags": {"web", "production"}
}
result = json.dumps(data, cls=SREEncoder, indent=2)
print(result)
```

### 4.2 YAML 解析（PyYAML）

```python
#!/usr/bin/env python3
"""YAML 配置文件解析基础示例"""

import yaml
from pathlib import Path

# === YAML 字符串解析 ===
yaml_str = """
# Kubernetes Deployment 配置示例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
    env: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    spec:
      containers:
        - name: web
          image: nginx:1.24
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
            requests:
              cpu: "200m"
              memory: "128Mi"
"""

# 安全加载（推荐，避免代码注入）
config = yaml.safe_load(yaml_str)
print(f"应用名称: {config['metadata']['name']}")
print(f"副本数: {config['spec']['replicas']}")
# 输出: 应用名称: web-app
# 输出: 副本数: 3

# === 多文档 YAML 解析 ===
multi_doc_yaml = """
---
name: service-a
port: 8080
---
name: service-b
port: 8081
---
name: service-c
port: 8082
"""

# safe_load_all 处理多文档
services = list(yaml.safe_load_all(multi_doc_yaml))
for svc in services:
    print(f"服务: {svc['name']} -> 端口: {svc['port']}")

# === 写入 YAML 文件 ===
output_config = {
    'server': {
        'host': '0.0.0.0',
        'port': 8080,
        'workers': 4
    },
    'logging': {
        'level': 'INFO',
        'file': '/var/log/app.log'
    }
}

config_path = Path("/tmp/app_config.yaml")
with open(config_path, 'w', encoding='utf-8') as f:
    # allow_unicode=True 保留中文
    # default_flow_style=False 使用块格式（更可读）
    yaml.dump(output_config, f, allow_unicode=True, default_flow_style=False)

print(f"\n生成的 YAML 文件内容:")
print(config_path.read_text())
```

### 4.3 TOML 解析

```python
#!/usr/bin/env python3
"""TOML 配置文件解析基础示例"""

import sys

# Python 3.11+ 使用内置 tomllib
if sys.version_info >= (3, 11):
    import tomllib
else:
    # Python 3.10 及以下使用第三方库
    try:
        import tomllib
    except ImportError:
        import tomli as tomllib

import toml  # 用于写入 TOML

# === TOML 字符串解析 ===
toml_str = """
# 项目配置文件 pyproject.toml 风格
[project]
name = "sre-toolkit"
version = "1.0.0"
description = "SRE 运维工具箱"

[project.dependencies]
python = ">=3.8"
requests = ">=2.28"

[server]
host = "0.0.0.0"
port = 8080
workers = 4

[server.ssl]
enabled = true
cert_file = "/etc/ssl/cert.pem"
key_file = "/etc/ssl/key.pem"

[database]
url = "postgresql://localhost:5432/mydb"
pool_size = 10
timeout = 30

[[services]]
name = "web"
port = 80

[[services]]
name = "api"
port = 8080
"""

# 解析 TOML（tomllib 需要 bytes）
config = tomllib.loads(toml_str)
print(f"项目名: {config['project']['name']}")
print(f"服务器端口: {config['server']['port']}")
print(f"SSL 启用: {config['server']['ssl']['enabled']}")

# 遍历服务列表
for svc in config['services']:
    print(f"服务: {svc['name']} -> 端口: {svc['port']}")

# === 写入 TOML 文件 ===
output = {
    'server': {'host': '0.0.0.0', 'port': 9090},
    'alerting': {'enabled': True, 'channels': ['email', 'slack']}
}

with open('/tmp/config.toml', 'w') as f:
    toml.dump(output, f)
```

## 5. 进阶用法

### 5.1 配置文件校验（jsonschema）

```python
#!/usr/bin/env python3
"""使用 jsonschema 校验配置文件"""

import json
import yaml
from jsonschema import validate, ValidationError

# 定义配置文件的 JSON Schema
config_schema = {
    "type": "object",
    "required": ["server", "database"],
    "properties": {
        "server": {
            "type": "object",
            "required": ["host", "port"],
            "properties": {
                "host": {"type": "string", "format": "ipv4"},
                "port": {"type": "integer", "minimum": 1, "maximum": 65535},
                "workers": {"type": "integer", "minimum": 1, "maximum": 32},
            }
        },
        "database": {
            "type": "object",
            "required": ["url"],
            "properties": {
                "url": {"type": "string", "pattern": "^postgresql://"},
                "pool_size": {"type": "integer", "minimum": 1, "maximum": 100},
            }
        }
    }
}

# 正确的配置
good_config = {
    "server": {"host": "10.0.0.1", "port": 8080, "workers": 4},
    "database": {"url": "postgresql://localhost/db", "pool_size": 10}
}

# 错误的配置（端口超范围）
bad_config = {
    "server": {"host": "10.0.0.1", "port": 99999},
    "database": {"url": "mysql://localhost/db"}  # 不匹配 postgresql 模式
}

# 校验正确配置
try:
    validate(instance=good_config, schema=config_schema)
    print("配置校验通过")
except ValidationError as e:
    print(f"配置校验失败: {e.message}")

# 校验错误配置
try:
    validate(instance=bad_config, schema=config_schema)
    print("配置校验通过")
except ValidationError as e:
    print(f"配置校验失败: {e.message}")
    # 输出: 配置校验失败: 99999 is greater than the maximum of 65535
```

### 5.2 多环境配置合并

```python
#!/usr/bin/env python3
"""多环境配置合并：base + 环境覆盖"""

import copy
import yaml

def deep_merge(base: dict, override: dict) -> dict:
    """深度合并两个字典，override 覆盖 base 中的同名键"""
    result = copy.deepcopy(base)
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            # 递归合并嵌套字典
            result[key] = deep_merge(result[key], value)
        else:
            # 直接覆盖
            result[key] = copy.deepcopy(value)
    return result

# 基础配置
base_yaml = """
server:
  host: 0.0.0.0
  port: 8080
  workers: 4
  log_level: INFO

database:
  host: localhost
  port: 5432
  name: myapp
  pool_size: 5

redis:
  host: localhost
  port: 6379
  db: 0

monitoring:
  enabled: true
  interval: 60
"""

# 生产环境覆盖
prod_yaml = """
server:
  workers: 16
  log_level: WARNING

database:
  host: db-master.prod.internal
  pool_size: 50

redis:
  host: redis-cluster.prod.internal

monitoring:
  interval: 30
"""

# 开发环境覆盖
dev_yaml = """
server:
  workers: 1
  log_level: DEBUG

database:
  name: myapp_dev

monitoring:
  enabled: false
"""

# 解析配置
base_config = yaml.safe_load(base_yaml)
prod_config = yaml.safe_load(prod_yaml)
dev_config = yaml.safe_load(dev_yaml)

# 合并配置
final_prod = deep_merge(base_config, prod_config)
final_dev = deep_merge(base_config, dev_config)

print("=== 生产环境最终配置 ===")
print(yaml.dump(final_prod, allow_unicode=True, default_flow_style=False))

print("=== 开发环境最终配置 ===")
print(yaml.dump(final_dev, allow_unicode=True, default_flow_style=False))
```

### 5.3 环境变量替换

```python
#!/usr/bin/env python3
"""配置文件中的环境变量替换"""

import os
import re
import yaml

def resolve_env_vars(config, default_marker=":-"):
    """递归替换配置中的环境变量引用
    支持格式: ${VAR_NAME} 和 ${VAR_NAME:-default_value}
    """
    if isinstance(config, str):
        # 匹配 ${VAR_NAME} 或 ${VAR_NAME:-default}
        pattern = r'\$\{([^}]+)\}'
        def replacer(match):
            expr = match.group(1)
            if default_marker in expr:
                var_name, default_val = expr.split(default_marker, 1)
                return os.environ.get(var_name.strip(), default_val.strip())
            else:
                return os.environ.get(expr.strip(), match.group(0))
        return re.sub(pattern, replacer, config)
    elif isinstance(config, dict):
        return {k: resolve_env_vars(v) for k, v in config.items()}
    elif isinstance(config, list):
        return [resolve_env_vars(item) for item in config]
    return config

# 设置测试环境变量
os.environ['DB_HOST'] = 'prod-db.internal'
os.environ['DB_PASSWORD'] = 'supersecret'

yaml_with_env = """
database:
  host: ${DB_HOST}
  port: 5432
  password: ${DB_PASSWORD}
  name: ${DB_NAME:-myapp}

server:
  secret_key: ${SECRET_KEY:-default-dev-key}
"""

config = yaml.safe_load(yaml_with_env)
resolved = resolve_env_vars(config)

print("解析后的配置:")
print(yaml.dump(resolved, allow_unicode=True, default_flow_style=False))
# database.host = prod-db.internal (来自环境变量)
# database.name = myapp (使用默认值)
```

## 6. SRE 实战案例：多环境配置管理工具

```python
#!/usr/bin/env python3
"""
SRE 实战：多环境配置管理工具
功能：
1. 支持 JSON/YAML/TOML 多格式读写
2. 多环境配置合并（base → env → local）
3. 配置校验（JSON Schema）
4. 环境变量替换
5. 配置 diff 比较
6. 加密敏感字段
"""

import json
import yaml
import copy
import os
import sys
import hashlib
import base64
from pathlib import Path
from datetime import datetime
from typing import Any, Dict, Optional

# ============================================================
# 配置加载器：支持多格式
# ============================================================
class ConfigLoader:
    """多格式配置文件加载器"""

    @staticmethod
    def load(filepath: str) -> dict:
        """根据文件扩展名自动选择解析器"""
        path = Path(filepath)
        suffix = path.suffix.lower()

        with open(path, 'r', encoding='utf-8') as f:
            content = f.read()

        if suffix in ('.yaml', '.yml'):
            return yaml.safe_load(content) or {}
        elif suffix == '.json':
            return json.loads(content)
        elif suffix == '.toml':
            if sys.version_info >= (3, 11):
                import tomllib
            else:
                import tomli as tomllib
            return tomllib.loads(content)
        else:
            raise ValueError(f"不支持的配置格式: {suffix}")

    @staticmethod
    def save(data: dict, filepath: str):
        """根据扩展名自动选择序列化器"""
        path = Path(filepath)
        suffix = path.suffix.lower()
        path.parent.mkdir(parents=True, exist_ok=True)

        with open(path, 'w', encoding='utf-8') as f:
            if suffix in ('.yaml', '.yml'):
                yaml.dump(data, f, allow_unicode=True, default_flow_style=False)
            elif suffix == '.json':
                json.dump(data, f, ensure_ascii=False, indent=2)
            elif suffix == '.toml':
                import toml
                toml.dump(data, f)

# ============================================================
# 配置管理器
# ============================================================
class ConfigManager:
    """多环境配置管理器"""

    def __init__(self, config_dir: str):
        self.config_dir = Path(config_dir)
        self.loader = ConfigLoader()
        self._cache = {}

    def deep_merge(self, base: dict, override: dict) -> dict:
        """深度合并字典"""
        result = copy.deepcopy(base)
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = self.deep_merge(result[key], value)
            else:
                result[key] = copy.deepcopy(value)
        return result

    def resolve_env_vars(self, config: Any) -> Any:
        """递归替换环境变量"""
        import re
        if isinstance(config, str):
            def replacer(match):
                expr = match.group(1)
                if ':-' in expr:
                    var, default = expr.split(':-', 1)
                    return os.environ.get(var.strip(), default.strip())
                return os.environ.get(expr.strip(), match.group(0))
            return re.sub(r'\$\{([^}]+)\}', replacer, config)
        elif isinstance(config, dict):
            return {k: self.resolve_env_vars(v) for k, v in config.items()}
        elif isinstance(config, list):
            return [self.resolve_env_vars(i) for i in config]
        return config

    def load_env_config(self, env: str, resolve_env: bool = True) -> dict:
        """加载指定环境的配置（base + env 覆盖 + local 覆盖）"""
        # 1. 加载基础配置
        base_file = self.config_dir / "base.yaml"
        if not base_file.exists():
            raise FileNotFoundError(f"基础配置文件不存在: {base_file}")
        config = self.loader.load(str(base_file))

        # 2. 加载环境覆盖
        env_file = self.config_dir / f"{env}.yaml"
        if env_file.exists():
            env_config = self.loader.load(str(env_file))
            config = self.deep_merge(config, env_config)

        # 3. 加载本地覆盖（不提交到 Git）
        local_file = self.config_dir / f"{env}.local.yaml"
        if local_file.exists():
            local_config = self.loader.load(str(local_file))
            config = self.deep_merge(config, local_config)

        # 4. 替换环境变量
        if resolve_env:
            config = self.resolve_env_vars(config)

        self._cache[env] = config
        return config

    def diff_configs(self, env1: str, env2: str) -> dict:
        """比较两个环境的配置差异"""
        config1 = self.load_env_config(env1, resolve_env=False)
        config2 = self.load_env_config(env2, resolve_env=False)

        def find_diffs(d1, d2, path=""):
            diffs = {}
            all_keys = set(list(d1.keys()) + list(d2.keys()))
            for key in sorted(all_keys):
                current_path = f"{path}.{key}" if path else key
                v1 = d1.get(key, "<不存在>")
                v2 = d2.get(key, "<不存在>")
                if isinstance(v1, dict) and isinstance(v2, dict):
                    nested = find_diffs(v1, v2, current_path)
                    diffs.update(nested)
                elif v1 != v2:
                    diffs[current_path] = {"env1": v1, "env2": v2}
            return diffs

        return find_diffs(config1, config2)

    def validate_config(self, config: dict, schema: dict) -> list:
        """校验配置是否符合 Schema"""
        try:
            from jsonschema import validate, ValidationError
            validate(instance=config, schema=schema)
            return []
        except ValidationError as e:
            return [f"{'.'.join(str(p) for p in e.absolute_path)}: {e.message}"]
        except ImportError:
            print("警告: jsonschema 未安装，跳过校验")
            return []

    def export_config(self, env: str, output_path: str):
        """导出最终合并后的配置到文件"""
        config = self.load_env_config(env)
        self.loader.save(config, output_path)
        print(f"配置已导出: {output_path}")

# ============================================================
# 演示运行
# ============================================================
def demo():
    """演示多环境配置管理"""
    # 创建示例配置目录
    config_dir = Path("/tmp/sre-configs")
    config_dir.mkdir(exist_ok=True)

    # 创建基础配置
    base_config = {
        'app': {'name': 'sre-platform', 'version': '2.0'},
        'server': {'host': '0.0.0.0', 'port': 8080, 'workers': 4},
        'database': {'host': 'localhost', 'port': 5432, 'name': 'app_db', 'pool_size': 5},
        'redis': {'host': 'localhost', 'port': 6379},
        'monitoring': {'enabled': True, 'interval': 60},
    }
    ConfigLoader.save(base_config, str(config_dir / "base.yaml"))

    # 创建生产环境配置
    prod_override = {
        'server': {'workers': 16, 'port': 80},
        'database': {'host': '${DB_HOST:-db.prod.internal}', 'pool_size': 50},
        'redis': {'host': 'redis.prod.internal'},
        'monitoring': {'interval': 15},
    }
    ConfigLoader.save(prod_override, str(config_dir / "prod.yaml"))

    # 创建开发环境配置
    dev_override = {
        'server': {'workers': 1},
        'database': {'name': 'app_dev'},
        'monitoring': {'enabled': False},
    }
    ConfigLoader.save(dev_override, str(config_dir / "dev.yaml"))

    # 初始化管理器
    mgr = ConfigManager(str(config_dir))

    # 加载各环境配置
    print("=" * 60)
    print("生产环境配置:")
    print("=" * 60)
    prod = mgr.load_env_config('prod')
    print(yaml.dump(prod, allow_unicode=True, default_flow_style=False))

    print("=" * 60)
    print("开发环境配置:")
    print("=" * 60)
    dev = mgr.load_env_config('dev')
    print(yaml.dump(dev, allow_unicode=True, default_flow_style=False))

    # 配置 diff
    print("=" * 60)
    print("prod vs dev 差异:")
    print("=" * 60)
    diffs = mgr.diff_configs('prod', 'dev')
    for path, diff in diffs.items():
        print(f"  {path}:")
        print(f"    prod = {diff['env1']}")
        print(f"    dev  = {diff['env2']}")

    # 导出配置
    mgr.export_config('prod', '/tmp/sre-configs/output/prod-final.yaml')

if __name__ == '__main__':
    demo()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **YAML 的 `yaml.load()` 不安全**：务必使用 `yaml.safe_load()`，避免任意代码执行
2. **YAML 的布尔值陷阱**：`yes`/`no`/`on`/`off` 会被解析为布尔值，需加引号
3. **JSON 不支持注释**：如需注释可用 `_comment` 字段或改用 YAML
4. **TOML 写入需要额外库**：`tomllib` 只读不写，写入需用 `toml` 库
5. **编码问题**：始终指定 `encoding='utf-8'`

### 最佳实践

```python
# 1. 始终使用 safe_load
config = yaml.safe_load(content)  # 正确
# config = yaml.load(content)     # 危险！

# 2. YAML 中布尔值字符串加引号
# version: "yes"   # 字符串
# enabled: true     # 布尔值

# 3. 配置文件加 Schema 校验
# 4. 敏感信息用环境变量，不要硬编码
# 5. 配置合并时注意深拷贝，避免引用污染
```

## 8. 速查表

```python
# === JSON ===
import json
json.loads(s)                # 字符串 → 字典
json.dumps(d, indent=2)      # 字典 → 字符串
json.load(f)                 # 文件 → 字典
json.dump(d, f)              # 字典 → 文件

# === YAML ===
import yaml
yaml.safe_load(s)            # 字符串 → 字典
yaml.safe_load_all(s)        # 多文档 → 生成器
yaml.dump(d)                 # 字典 → 字符串
yaml.dump(d, f)              # 字典 → 文件

# === TOML ===
import tomllib               # 3.11+（只读）
tomllib.loads(s)             # 字符串 → 字典
tomllib.load(f)              # 文件(bytes) → 字典

import toml                  # 读写
toml.loads(s)                # 字符串 → 字典
toml.dumps(d)                # 字典 → 字符串
toml.dump(d, f)              # 字典 → 文件

# === 对比 shell 工具 ===
# cat config.json | jq '.server.port'    →  config['server']['port']
# cat config.yaml | yq '.server.port'    →  config['server']['port']
# Python 优势：可编程、可校验、可合并
```
