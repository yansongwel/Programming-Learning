# Jinja2 模板渲染 — 用 Python 批量生成运维配置文件

## 1. 是什么 & 为什么用它

Jinja2 是 Python 最流行的模板引擎，Ansible 的配置模板底层就是 Jinja2。
对 SRE 来说，它可以替代 shell 中的 `sed`/`envsubst`/`awk` 进行批量配置生成：

- **变量替换**：比 `envsubst` 更灵活
- **条件/循环**：比 `sed` 更强大
- **模板继承**：比手工拼接更可维护
- **过滤器/宏**：比 `awk` 更简洁

典型使用场景：
- 批量生成 Nginx/HAProxy 配置
- 生成 Prometheus/Alertmanager 规则
- 生成 Kubernetes YAML 清单
- 生成 Terraform 配置

## 2. 安装与环境

```bash
# 安装 Jinja2
pip install Jinja2

# 验证安装
python3 -c "import jinja2; print(jinja2.__version__)"

# 对比 shell 工具
# echo "Hello $NAME" | envsubst              # 简单变量替换
# sed 's/{{port}}/8080/g' template.conf      # 简单替换
# Jinja2 优势：循环、条件、过滤器、宏、继承
```

## 3. 核心概念与原理

### 3.1 Jinja2 模板语法

```
{{ ... }}    变量输出（表达式）
{% ... %}    控制语句（if/for/macro 等）
{# ... #}    注释（不会出现在输出中）
```

### 3.2 渲染流程

```
模板文件(.j2) + 上下文数据(dict) → Jinja2 引擎 → 渲染结果(字符串) → 输出文件
```

### 3.3 核心组件

| 组件 | 说明 |
|------|------|
| Environment | 模板环境，管理配置和加载器 |
| Loader | 模板加载器（文件系统/字符串/包等） |
| Template | 模板对象，调用 render() 生成结果 |
| Filter | 过滤器，对变量做转换 |
| Test | 测试函数，在条件中使用 |
| Extension | 扩展，增加自定义功能 |

## 4. 基础用法

### 4.1 字符串模板

```python
#!/usr/bin/env python3
"""Jinja2 基础用法：字符串模板"""

from jinja2 import Template

# === 最简单的变量替换 ===
template = Template("服务器: {{ host }}:{{ port }}")
result = template.render(host="10.0.0.1", port=8080)
print(result)
# 输出: 服务器: 10.0.0.1:8080

# === 循环 ===
tpl_loop = Template("""
{% for server in servers %}
upstream_server {{ server.host }}:{{ server.port }} weight={{ server.weight }};
{% endfor %}
""".strip())

servers = [
    {"host": "10.0.0.1", "port": 8080, "weight": 5},
    {"host": "10.0.0.2", "port": 8080, "weight": 3},
    {"host": "10.0.0.3", "port": 8080, "weight": 2},
]
print(tpl_loop.render(servers=servers))

# === 条件判断 ===
tpl_cond = Template("""
server {
    listen {{ port }};
    server_name {{ domain }};
    {% if ssl_enabled %}
    ssl_certificate {{ ssl_cert }};
    ssl_certificate_key {{ ssl_key }};
    {% endif %}
    {% if env == 'production' %}
    access_log /var/log/nginx/{{ domain }}.access.log;
    {% else %}
    access_log off;
    {% endif %}
}
""".strip())

result = tpl_cond.render(
    port=443,
    domain="api.example.com",
    ssl_enabled=True,
    ssl_cert="/etc/ssl/cert.pem",
    ssl_key="/etc/ssl/key.pem",
    env="production"
)
print(result)
```

### 4.2 过滤器

```python
#!/usr/bin/env python3
"""Jinja2 过滤器用法"""

from jinja2 import Template

# === 内置过滤器 ===
tpl = Template("""
主机名: {{ hostname | upper }}
描述: {{ description | default('无描述', true) }}
标签: {{ tags | join(', ') }}
IP 数量: {{ ips | length }}
第一个 IP: {{ ips | first }}
JSON 输出: {{ config | tojson(indent=2) }}
""".strip())

result = tpl.render(
    hostname="web-server-01",
    description="",  # 空字符串，default(true) 会替换
    tags=["web", "production", "nginx"],
    ips=["10.0.0.1", "10.0.0.2", "10.0.0.3"],
    config={"port": 8080, "workers": 4}
)
print(result)

# === 自定义过滤器 ===
from jinja2 import Environment

env = Environment()

def to_cidr(ip, prefix=24):
    """将 IP 转为 CIDR 表示"""
    return f"{ip}/{prefix}"

def bytes_to_human(size):
    """字节数转可读格式"""
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if size < 1024.0:
            return f"{size:.1f}{unit}"
        size /= 1024.0

# 注册自定义过滤器
env.filters['to_cidr'] = to_cidr
env.filters['bytes_to_human'] = bytes_to_human

tpl2 = env.from_string("""
防火墙规则: allow {{ ip | to_cidr(16) }}
磁盘使用: {{ disk_bytes | bytes_to_human }}
""".strip())

print(tpl2.render(ip="10.0.0.0", disk_bytes=5368709120))
# 输出:
# 防火墙规则: allow 10.0.0.0/16
# 磁盘使用: 5.0GB
```

### 4.3 宏（可复用的模板片段）

```python
#!/usr/bin/env python3
"""Jinja2 宏的使用"""

from jinja2 import Template

# 定义宏 — 类似于函数
tpl_macro = Template("""
{# 定义 upstream 块的宏 #}
{% macro upstream_block(name, servers, method='round-robin') %}
upstream {{ name }} {
    {% if method == 'ip_hash' %}
    ip_hash;
    {% elif method == 'least_conn' %}
    least_conn;
    {% endif %}
    {% for s in servers %}
    server {{ s.host }}:{{ s.port }} weight={{ s.get('weight', 1) }}{% if s.get('backup') %} backup{% endif %};
    {% endfor %}
}
{% endmacro %}

{# 调用宏生成多个 upstream #}
{{ upstream_block('web_backend', web_servers) }}
{{ upstream_block('api_backend', api_servers, 'ip_hash') }}
""".strip())

result = tpl_macro.render(
    web_servers=[
        {"host": "10.0.1.1", "port": 80, "weight": 5},
        {"host": "10.0.1.2", "port": 80, "weight": 3},
        {"host": "10.0.1.3", "port": 80, "backup": True},
    ],
    api_servers=[
        {"host": "10.0.2.1", "port": 8080},
        {"host": "10.0.2.2", "port": 8080},
    ]
)
print(result)
```

## 5. 进阶用法

### 5.1 文件模板加载

```python
#!/usr/bin/env python3
"""从文件系统加载 Jinja2 模板"""

from jinja2 import Environment, FileSystemLoader, select_autoescape
from pathlib import Path

# 创建模板目录
template_dir = Path("/tmp/jinja2-templates")
template_dir.mkdir(exist_ok=True)

# 创建基础模板
base_tpl = """
{# base.conf.j2 — 所有配置的基础模板 #}
# 自动生成，请勿手动修改
# 生成时间: {{ generated_at }}
# 环境: {{ env }}

{% block content %}{% endblock %}
""".strip()

# 创建 Nginx 模板（继承基础模板）
nginx_tpl = """
{% extends "base.conf.j2" %}

{% block content %}
{% for site in sites %}
server {
    listen {{ site.port | default(80) }};
    server_name {{ site.domain }};
    root {{ site.root | default('/var/www/' + site.domain) }};

    {% if site.ssl %}
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/{{ site.domain }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ site.domain }}/privkey.pem;
    {% endif %}

    {% for location in site.get('locations', [{'path': '/'}]) %}
    location {{ location.path }} {
        {% if location.get('proxy_pass') %}
        proxy_pass {{ location.proxy_pass }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        {% else %}
        try_files $uri $uri/ =404;
        {% endif %}
    }
    {% endfor %}
}

{% endfor %}
{% endblock %}
""".strip()

# 写入模板文件
(template_dir / "base.conf.j2").write_text(base_tpl)
(template_dir / "nginx.conf.j2").write_text(nginx_tpl)

# 创建 Jinja2 环境
env = Environment(
    loader=FileSystemLoader(str(template_dir)),
    # 去除多余空行
    trim_blocks=True,
    lstrip_blocks=True,
    # 未定义变量报错
    undefined=jinja2.StrictUndefined if False else jinja2.Undefined
)

# 加载并渲染模板
template = env.get_template("nginx.conf.j2")

from datetime import datetime
result = template.render(
    generated_at=datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    env="production",
    sites=[
        {
            "domain": "www.example.com",
            "port": 80,
            "ssl": True,
            "locations": [
                {"path": "/", "proxy_pass": "http://127.0.0.1:8080"},
                {"path": "/static"},
            ]
        },
        {
            "domain": "api.example.com",
            "port": 80,
            "ssl": True,
            "locations": [
                {"path": "/", "proxy_pass": "http://127.0.0.1:9090"},
            ]
        },
    ]
)

print(result)
```

### 5.2 空白控制

```python
#!/usr/bin/env python3
"""Jinja2 空白控制技巧"""

from jinja2 import Environment

# trim_blocks: 删除块标签后的第一个换行
# lstrip_blocks: 删除块标签前的空白
env = Environment(trim_blocks=True, lstrip_blocks=True)

# 使用 - 号控制空白
tpl = env.from_string("""
servers:
{%- for s in servers %}
  - host: {{ s.host }}
    port: {{ s.port }}
{%- endfor %}
""")

result = tpl.render(servers=[
    {"host": "10.0.0.1", "port": 80},
    {"host": "10.0.0.2", "port": 80},
])
print(result)
```

## 6. SRE 实战案例：批量生成 Nginx/Prometheus 配置

```python
#!/usr/bin/env python3
"""
SRE 实战：批量配置文件生成器
功能：
1. 从 YAML 数据源读取服务清单
2. 使用 Jinja2 模板生成 Nginx 配置
3. 使用 Jinja2 模板生成 Prometheus 抓取配置
4. 配置校验与 diff
5. 批量部署
"""

import yaml
import os
import hashlib
from pathlib import Path
from datetime import datetime
from jinja2 import Environment, FileSystemLoader, StrictUndefined

# ============================================================
# 模板与数据准备
# ============================================================

TEMPLATE_DIR = Path("/tmp/sre-templates")
OUTPUT_DIR = Path("/tmp/sre-output")
TEMPLATE_DIR.mkdir(exist_ok=True)
OUTPUT_DIR.mkdir(exist_ok=True)

# ---- Nginx 虚拟主机模板 ----
NGINX_TEMPLATE = """
{# nginx-vhost.conf.j2 #}
# ========================================
# 自动生成 - 请勿手动修改
# 生成时间: {{ generated_at }}
# 服务: {{ service.name }}
# ========================================

{% if service.upstream_servers | length > 0 %}
upstream {{ service.name }}_backend {
    {% if service.lb_method == 'ip_hash' %}
    ip_hash;
    {% elif service.lb_method == 'least_conn' %}
    least_conn;
    {% endif %}
    {% for server in service.upstream_servers %}
    server {{ server.host }}:{{ server.port }} weight={{ server.weight | default(1) }}{% if server.backup | default(false) %} backup{% endif %};
    {% endfor %}
}
{% endif %}

server {
    listen {{ service.listen_port | default(80) }};
    server_name {{ service.domains | join(' ') }};

    {% if service.ssl | default(false) %}
    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/{{ service.domains[0] }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ service.domains[0] }}/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # HTTP 重定向到 HTTPS
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    }
    {% endif %}

    # 访问日志
    access_log /var/log/nginx/{{ service.name }}.access.log main;
    error_log /var/log/nginx/{{ service.name }}.error.log warn;

    # 通用安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    {% if service.rate_limit | default(false) %}
    # 限流配置
    limit_req_zone $binary_remote_addr zone={{ service.name }}_limit:10m rate={{ service.rate_limit }}r/s;
    {% endif %}

    {% for location in service.locations %}
    location {{ location.path }} {
        {% if location.proxy_pass is defined %}
        proxy_pass {{ location.proxy_pass }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout {{ location.timeout | default(30) }}s;
        proxy_read_timeout {{ location.timeout | default(60) }}s;
        {% elif location.static | default(false) %}
        root {{ location.root }};
        expires {{ location.expires | default('7d') }};
        add_header Cache-Control "public, immutable";
        {% endif %}
        {% if location.auth | default(false) %}
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        {% endif %}
    }
    {% endfor %}

    # 健康检查端点
    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
"""

# ---- Prometheus 抓取配置模板 ----
PROMETHEUS_TEMPLATE = """
{# prometheus-scrape.yml.j2 #}
# 自动生成 - 请勿手动修改
# 生成时间: {{ generated_at }}

scrape_configs:
{% for service in services %}
  - job_name: '{{ service.name }}'
    {% if service.metrics_path | default(false) %}
    metrics_path: '{{ service.metrics_path }}'
    {% endif %}
    scrape_interval: {{ service.scrape_interval | default('15s') }}
    scrape_timeout: {{ service.scrape_timeout | default('10s') }}
    {% if service.scheme | default('http') == 'https' %}
    scheme: https
    tls_config:
      insecure_skip_verify: false
    {% endif %}
    static_configs:
      - targets:
        {% for target in service.targets %}
          - '{{ target.host }}:{{ target.metrics_port | default(9090) }}'
        {% endfor %}
        labels:
          env: '{{ env }}'
          service: '{{ service.name }}'
          {% for key, value in (service.labels | default({})).items() %}
          {{ key }}: '{{ value }}'
          {% endfor %}
    {% if service.relabel | default(false) %}
    relabel_configs:
      {% for rule in service.relabel %}
      - source_labels: [{{ rule.source | join(', ') }}]
        regex: '{{ rule.regex }}'
        target_label: '{{ rule.target }}'
        replacement: '{{ rule.replacement | default("$1") }}'
      {% endfor %}
    {% endif %}
{% endfor %}

{% if alertmanager | default(false) %}
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          {% for am in alertmanager.targets %}
            - '{{ am }}'
          {% endfor %}
{% endif %}
"""

# ---- 服务清单数据 ----
SERVICES_YAML = """
services:
  - name: web-frontend
    domains:
      - www.example.com
      - example.com
    ssl: true
    lb_method: least_conn
    rate_limit: 100
    upstream_servers:
      - host: 10.0.1.1
        port: 3000
        weight: 5
      - host: 10.0.1.2
        port: 3000
        weight: 3
      - host: 10.0.1.3
        port: 3000
        weight: 2
        backup: true
    locations:
      - path: /
        proxy_pass: http://web-frontend_backend
        timeout: 30
      - path: /static
        static: true
        root: /var/www/static
        expires: 30d
    metrics_path: /metrics
    targets:
      - host: 10.0.1.1
        metrics_port: 9100
      - host: 10.0.1.2
        metrics_port: 9100
      - host: 10.0.1.3
        metrics_port: 9100
    labels:
      team: frontend
      tier: web

  - name: api-gateway
    domains:
      - api.example.com
    ssl: true
    lb_method: ip_hash
    upstream_servers:
      - host: 10.0.2.1
        port: 8080
      - host: 10.0.2.2
        port: 8080
    locations:
      - path: /
        proxy_pass: http://api-gateway_backend
        timeout: 60
      - path: /admin
        proxy_pass: http://api-gateway_backend
        auth: true
    metrics_path: /actuator/prometheus
    scrape_interval: 10s
    targets:
      - host: 10.0.2.1
        metrics_port: 8080
      - host: 10.0.2.2
        metrics_port: 8080
    labels:
      team: backend
      tier: api

  - name: monitoring
    domains:
      - grafana.example.com
    ssl: true
    upstream_servers:
      - host: 10.0.3.1
        port: 3000
    locations:
      - path: /
        proxy_pass: http://monitoring_backend
    targets:
      - host: 10.0.3.1
        metrics_port: 9090
    labels:
      team: sre
      tier: monitoring
"""

# ============================================================
# 配置生成器
# ============================================================
class ConfigGenerator:
    """批量配置文件生成器"""

    def __init__(self, template_dir: str, output_dir: str):
        self.template_dir = Path(template_dir)
        self.output_dir = Path(output_dir)
        self.env = Environment(
            loader=FileSystemLoader(str(self.template_dir)),
            trim_blocks=True,
            lstrip_blocks=True,
            undefined=StrictUndefined,
            keep_trailing_newline=True,
        )
        # 注册自定义过滤器
        self.env.filters['to_yaml'] = lambda d: yaml.dump(d, default_flow_style=False)
        self.generated_files = []

    def _file_hash(self, filepath: Path) -> str:
        """计算文件 MD5"""
        if filepath.exists():
            return hashlib.md5(filepath.read_bytes()).hexdigest()
        return ""

    def generate_nginx_configs(self, services: list, env_name: str):
        """生成 Nginx 虚拟主机配置"""
        nginx_dir = self.output_dir / "nginx" / "conf.d"
        nginx_dir.mkdir(parents=True, exist_ok=True)

        template = self.env.get_template("nginx-vhost.conf.j2")
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        for service in services:
            output_file = nginx_dir / f"{service['name']}.conf"
            old_hash = self._file_hash(output_file)

            # 渲染模板
            content = template.render(
                service=service,
                env=env_name,
                generated_at=now,
            )

            # 写入文件
            output_file.write_text(content)
            new_hash = self._file_hash(output_file)

            status = "已更新" if old_hash != new_hash else "未变化"
            print(f"  [Nginx] {output_file.name}: {status}")
            self.generated_files.append(str(output_file))

    def generate_prometheus_config(self, services: list, env_name: str):
        """生成 Prometheus 抓取配置"""
        prom_dir = self.output_dir / "prometheus"
        prom_dir.mkdir(parents=True, exist_ok=True)

        template = self.env.get_template("prometheus-scrape.yml.j2")
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        output_file = prom_dir / "scrape_configs.yml"
        content = template.render(
            services=services,
            env=env_name,
            generated_at=now,
            alertmanager={"targets": ["alertmanager:9093"]},
        )
        output_file.write_text(content)
        print(f"  [Prometheus] {output_file.name}: 已生成")
        self.generated_files.append(str(output_file))

    def generate_all(self, services_data: dict, env_name: str = "production"):
        """生成所有配置"""
        services = services_data['services']
        print(f"\n{'='*60}")
        print(f"开始生成 {env_name} 环境配置...")
        print(f"{'='*60}")

        self.generate_nginx_configs(services, env_name)
        self.generate_prometheus_config(services, env_name)

        print(f"\n生成完成！共 {len(self.generated_files)} 个文件")
        print(f"输出目录: {self.output_dir}")

    def show_generated_files(self):
        """展示所有生成的文件"""
        print(f"\n生成的文件列表:")
        for f in self.generated_files:
            size = Path(f).stat().st_size
            print(f"  {f} ({size} bytes)")


# ============================================================
# 主程序
# ============================================================
def main():
    # 写入模板文件
    (TEMPLATE_DIR / "nginx-vhost.conf.j2").write_text(NGINX_TEMPLATE)
    (TEMPLATE_DIR / "prometheus-scrape.yml.j2").write_text(PROMETHEUS_TEMPLATE)

    # 解析服务清单
    services_data = yaml.safe_load(SERVICES_YAML)

    # 初始化生成器
    generator = ConfigGenerator(str(TEMPLATE_DIR), str(OUTPUT_DIR))

    # 生成所有配置
    generator.generate_all(services_data, env_name="production")

    # 展示生成结果
    generator.show_generated_files()

    # 打印一个生成的配置文件内容作为示例
    nginx_file = OUTPUT_DIR / "nginx" / "conf.d" / "web-frontend.conf"
    if nginx_file.exists():
        print(f"\n{'='*60}")
        print(f"示例: {nginx_file}")
        print(f"{'='*60}")
        print(nginx_file.read_text())

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **未定义变量默认静默**：默认不报错，使用 `StrictUndefined` 强制报错
2. **空白控制**：不设置 `trim_blocks`/`lstrip_blocks` 会有大量空行
3. **YAML 模板中的缩进**：Jinja2 标签会影响 YAML 缩进，善用 `indent` 过滤器
4. **安全问题**：不要用 `Template(用户输入)` 渲染用户提供的模板字符串
5. **模板中的 dict.items()**：Jinja2 中用 `dict.items()` 而非 Python3 的语法

### 最佳实践

```python
# 1. 使用 StrictUndefined 捕获未定义变量
env = Environment(undefined=StrictUndefined)

# 2. 设置空白控制
env = Environment(trim_blocks=True, lstrip_blocks=True)

# 3. 模板文件统一用 .j2 后缀
# 4. 生成的文件头部加注释说明是自动生成的
# 5. 生成前后做 diff，只在有变化时才写入
# 6. 使用宏封装可复用的配置片段
```

## 8. 速查表

```python
# === 基础 ===
from jinja2 import Template, Environment, FileSystemLoader

# 字符串模板
Template("{{ var }}").render(var="value")

# 文件模板
env = Environment(loader=FileSystemLoader("./templates"))
tpl = env.get_template("config.j2")
tpl.render(data=data)

# === 语法 ===
# {{ var }}                     变量输出
# {{ var | filter }}            过滤器
# {% if cond %}...{% endif %}   条件
# {% for i in list %}...{% endfor %}  循环
# {% macro name() %}...{% endmacro %} 宏定义
# {% extends "base.j2" %}       模板继承
# {% block name %}...{% endblock %}  块定义
# {# 注释 #}                    注释

# === 常用过滤器 ===
# {{ var | default('fallback') }}   默认值
# {{ var | upper / lower }}         大小写
# {{ list | join(', ') }}           列表拼接
# {{ list | length }}               长度
# {{ list | first / last }}         首/尾
# {{ dict | tojson(indent=2) }}     JSON 输出
# {{ var | trim }}                  去空白
# {{ var | replace('a', 'b') }}     替换
# {{ list | sort }}                 排序
# {{ list | unique }}               去重
# {{ list | map(attribute='name') | list }}  映射

# === 对比 shell 工具 ===
# envsubst < template > output     → Template().render()
# sed 's/OLD/NEW/g' file           → {{ var | replace() }}
# for 循环 + sed                   → {% for %} 循环
```
