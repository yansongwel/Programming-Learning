# fabric 批量部署 — 用 Python 替代 for+ssh 循环实现自动化部署

## 1. 是什么 & 为什么用它

### 什么是 Fabric

Fabric 是一个基于 SSH 的 Python 远程执行和部署库。它封装了 paramiko，提供了更高级、更简洁的 API 来执行远程命令、传输文件和编排多主机任务。

### 为什么 SRE 需要它

| 运维场景 | Shell 方式 | Fabric 方式 |
|---|---|---|
| 批量部署 | `for h in ...; do ssh $h "deploy.sh"; done` | `Group.run("deploy.sh")` |
| 文件分发 | `for h in ...; do scp file $h:/path; done` | `Group.put(file, remote)` |
| sudo 操作 | `ssh $h "echo pass \| sudo -S cmd"` | `conn.sudo("cmd")` |
| 任务编排 | 写大量 Shell 函数 + 调用顺序 | Python 函数 + 装饰器 |

```
Shell 部署脚本的痛点:
  #!/bin/bash
  HOSTS="web1 web2 web3 web4 web5"
  for host in $HOSTS; do
    echo "部署到 $host..."
    scp app.tar.gz $host:/tmp/
    ssh $host "cd /opt/app && tar xzf /tmp/app.tar.gz && ./restart.sh"
    if [ $? -ne 0 ]; then
      echo "部署失败: $host"
      # 回滚？怎么回滚？
    fi
  done

  问题:
  - 串行执行（5 台机器 × 30 秒 = 2.5 分钟）
  - 错误处理粗糙（部分失败怎么办？）
  - 回滚困难
  - sudo 密码处理麻烦

Fabric 部署:
  - 并行执行（5 台机器同时部署 ≈ 30 秒）
  - 异常处理（try/except 每台机器独立）
  - 任务编排（前置检查 → 部署 → 验证 → 回滚）
  - sudo 原生支持
```

## 2. 安装与环境

```bash
# 安装 Fabric（注意是 fabric 而非 fabric3，后者是旧版社区分支）
pip install fabric

# 验证安装
python3 -c "import fabric; print(fabric.__version__)"
# 应输出 3.x

# Fabric 依赖 paramiko 和 invoke
pip list | grep -E "fabric|paramiko|invoke"
```

**版本说明：**
```
Fabric 1.x: 旧版（已停止维护），用 fab 命令行工具
Fabric 2.x/3.x: 新版（当前推荐），纯 Python API
本教程基于 Fabric 3.x
```

## 3. 核心概念与原理

### Fabric 架构

```
+------------------------------------------------------------------+
|                      你的部署脚本 (fabfile.py)                      |
+------------------------------------------------------------------+
|                        Fabric 高级 API                            |
|                                                                   |
|  Connection        Group           task 装饰器    Config          |
|  (单机连接)       (多机分组)      (任务定义)     (配置管理)       |
+------------------------------------------------------------------+
|                        Invoke                                     |
|              (本地命令执行 + 任务管理框架)                          |
+------------------------------------------------------------------+
|                        Paramiko                                   |
|                    (SSH 协议实现)                                  |
+------------------------------------------------------------------+
|                        SSH / TCP                                  |
+------------------------------------------------------------------+

核心概念:
  Connection  —— 一个 SSH 连接，代表一台远程主机
  Group       —— 一组 Connection，代表多台主机
  task        —— 可复用的操作单元（Python 函数）
  Result      —— 命令执行结果（stdout, stderr, return_code 等）
```

### Connection vs Group

```
Connection (单机):
  conn = Connection('web1')
  conn.run('uptime')
  conn.put('file', '/remote/path')
  conn.sudo('systemctl restart nginx')

Group (多机):
  group = Group('web1', 'web2', 'web3')
  results = group.run('uptime')          # 并行执行
  # results = {conn1: Result, conn2: Result, conn3: Result}

ThreadingGroup (并行):
  group = ThreadingGroup('web1', 'web2', 'web3')
  results = group.run('uptime')          # 真正并行

SerialGroup (串行):
  group = SerialGroup('web1', 'web2', 'web3')
  results = group.run('uptime')          # 逐个执行
```

## 4. 基础用法

### 4.1 Connection —— 单机连接与命令执行

```python
#!/usr/bin/env python3
"""Fabric Connection 基础 —— 对比 ssh user@host "command" """

from fabric import Connection

# ============================================================
# Shell 对比: ssh root@192.168.1.100 "hostname && uptime"
# ============================================================

# --- 基本连接 ---
# 密码认证
conn = Connection(
    host='192.168.1.100',
    user='root',
    port=22,
    connect_kwargs={
        'password': 'your_password',
        # 或使用密钥:
        # 'key_filename': '/path/to/key',
    },
    connect_timeout=10,  # 连接超时
)

# --- 执行远程命令 ---
# 对比: ssh root@host "hostname"
result = conn.run('hostname', hide=True)  # hide=True 不输出到终端
print(f"主机名: {result.stdout.strip()}")
print(f"返回码: {result.return_code}")
print(f"是否成功: {result.ok}")

# --- 执行多条命令 ---
# 对比: ssh root@host "uptime && df -h / && free -m"
result = conn.run('uptime && df -h / && free -m', hide=True)
print(result.stdout)

# --- 带环境变量 ---
# 对比: ssh root@host "APP_ENV=prod ./deploy.sh"
result = conn.run('echo $APP_ENV', env={'APP_ENV': 'production'}, hide=True)
print(f"环境变量: {result.stdout.strip()}")

# --- sudo 执行 ---
# 对比: ssh root@host "sudo systemctl status nginx"
# Fabric 自动处理 sudo 密码提示
result = conn.sudo('whoami', hide=True, password='your_password')
print(f"sudo 身份: {result.stdout.strip()}")

# --- 检查命令是否存在 ---
# 对比: ssh host "which nginx"
result = conn.run('which nginx', warn=True, hide=True)
# warn=True: 命令失败不抛异常
if result.ok:
    print(f"nginx 路径: {result.stdout.strip()}")
else:
    print("nginx 未安装")

# 关闭连接
conn.close()
```

### 4.2 文件传输

```python
#!/usr/bin/env python3
"""Fabric 文件传输 —— 对比 scp 命令"""

from fabric import Connection
from pathlib import Path

# ============================================================
# Shell 对比:
#   scp local_file root@host:/remote/path
#   scp root@host:/remote/file local_path
# ============================================================

conn = Connection('192.168.1.100', user='root',
                  connect_kwargs={'password': 'xxx'})

# --- 上传文件 ---
# 对比: scp /tmp/config.yml root@host:/etc/app/config.yml
conn.put('/tmp/config.yml', '/etc/app/config.yml')
print("文件已上传")

# --- 上传并设置权限 ---
# 对比: scp file host:/path && ssh host "chmod 755 /path/file"
conn.put('/tmp/deploy.sh', '/opt/deploy.sh')
conn.run('chmod 755 /opt/deploy.sh')

# --- 下载文件 ---
# 对比: scp root@host:/var/log/app.log /tmp/app.log
conn.get('/var/log/app.log', '/tmp/app.log')
print("文件已下载")

# --- 使用 StringIO 上传文本内容（无需本地文件）---
from io import StringIO

config_content = """
[app]
name = myservice
port = 8080
debug = false
"""
conn.put(StringIO(config_content), '/etc/app/app.conf')
print("配置已生成并上传")

conn.close()
```

### 4.3 Group —— 多机并行执行

```python
#!/usr/bin/env python3
"""Fabric Group 多机操作 —— 对比 for+ssh 循环"""

from fabric import Connection, SerialGroup, ThreadingGroup

# ============================================================
# Shell 对比:
#   for host in web1 web2 web3; do
#     ssh root@$host "hostname && uptime"
#   done
# ============================================================

# 连接配置
connect_kwargs = {'password': 'your_password'}

# --- SerialGroup: 串行执行（逐台执行）---
# 适用场景: 需要确认一台成功再执行下一台（如数据库升级）
serial_group = SerialGroup(
    '192.168.1.101',
    '192.168.1.102',
    '192.168.1.103',
    user='root',
    connect_kwargs=connect_kwargs,
)

print("=== 串行执行 ===")
results = serial_group.run('hostname && uptime', hide=True)
for conn, result in results.items():
    print(f"  {conn.host}: {result.stdout.strip().split(chr(10))[0]}")

# --- ThreadingGroup: 并行执行 ---
# 适用场景: 批量检查、批量部署（互不依赖的操作）
threading_group = ThreadingGroup(
    '192.168.1.101',
    '192.168.1.102',
    '192.168.1.103',
    user='root',
    connect_kwargs=connect_kwargs,
)

print("\n=== 并行执行 ===")
results = threading_group.run('hostname && df -h /', hide=True)
for conn, result in results.items():
    print(f"\n  [{conn.host}]")
    for line in result.stdout.strip().split('\n'):
        print(f"    {line}")

# --- 自定义 Group（从文件或配置加载）---
def create_group_from_file(hosts_file, group_class=ThreadingGroup, **kwargs):
    """
    从文件加载主机列表创建 Group
    文件格式: 每行一个 host[:port]
    """
    hosts = []
    with open(hosts_file) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#'):
                hosts.append(line)
    return group_class(*hosts, **kwargs)

# 示例: hosts.txt 内容
# 192.168.1.101
# 192.168.1.102
# # 这行是注释
# 192.168.1.103
```

### 4.4 sudo 操作

```python
#!/usr/bin/env python3
"""Fabric sudo 操作 —— 对比 ssh + sudo"""

from fabric import Connection, Config

# ============================================================
# Shell 对比:
#   ssh user@host "echo password | sudo -S systemctl restart nginx"
#   # 或用 expect 脚本处理 sudo 交互
# ============================================================

# --- 方法 1: 每次调用时传密码 ---
conn = Connection('192.168.1.100', user='admin',
                  connect_kwargs={'password': 'user_pass'})
result = conn.sudo('systemctl status nginx', password='sudo_pass', hide=True)
print(result.stdout)

# --- 方法 2: 通过 Config 全局设置 sudo 密码 ---
config = Config(overrides={
    'sudo': {
        'password': 'sudo_pass',
    },
    'connect_kwargs': {
        'password': 'user_pass',
    },
})
conn = Connection('192.168.1.100', user='admin', config=config)

# 现在 sudo 不需要每次传密码
conn.sudo('systemctl restart nginx', hide=True)
conn.sudo('yum update -y', hide=True)

# --- 方法 3: sudo 切换用户 ---
# 对比: sudo -u postgres psql -c "SELECT 1"
conn.sudo('psql -c "SELECT version()"', user='postgres', hide=True)

conn.close()
```

## 5. 进阶用法

### 5.1 任务定义与编排

```python
#!/usr/bin/env python3
"""
Fabric 任务编排 —— 定义可复用的部署步骤
对比 Shell 脚本中的函数定义和调用顺序
"""

from fabric import Connection, ThreadingGroup, Config
from invoke import task
import time

# ============================================================
# Shell 对比:
#   deploy() {
#     check_prerequisites
#     upload_code
#     stop_service
#     install_dependencies
#     start_service
#     verify_health
#   }
# Fabric 的优势: 每步都有错误处理，支持回滚
# ============================================================


class Deployer:
    """应用部署器"""

    def __init__(self, hosts, user='root', password=None, key_path=None):
        connect_kwargs = {}
        if password:
            connect_kwargs['password'] = password
        if key_path:
            connect_kwargs['key_filename'] = key_path

        self.connections = [
            Connection(h, user=user, connect_kwargs=connect_kwargs)
            for h in hosts
        ]
        self.app_dir = '/opt/myapp'
        self.backup_dir = '/opt/myapp_backup'

    def pre_check(self, conn):
        """部署前检查"""
        print(f"  [{conn.host}] 前置检查...")

        # 检查磁盘空间
        result = conn.run(f"df -h {self.app_dir} | tail -1 | awk '{{print $5}}'",
                         hide=True)
        usage = int(result.stdout.strip().replace('%', ''))
        if usage > 90:
            raise Exception(f"磁盘空间不足: {usage}%")

        # 检查端口是否占用
        result = conn.run('ss -tlnp | grep :8080', warn=True, hide=True)
        return True

    def backup(self, conn):
        """备份当前版本"""
        print(f"  [{conn.host}] 备份中...")
        conn.run(f'rm -rf {self.backup_dir}', warn=True, hide=True)
        conn.run(f'cp -r {self.app_dir} {self.backup_dir}', warn=True, hide=True)
        print(f"  [{conn.host}] 备份完成: {self.backup_dir}")

    def upload_code(self, conn, package_path):
        """上传部署包"""
        print(f"  [{conn.host}] 上传部署包...")
        conn.put(package_path, '/tmp/app_package.tar.gz')
        conn.run(f'mkdir -p {self.app_dir}', hide=True)
        conn.run(f'tar xzf /tmp/app_package.tar.gz -C {self.app_dir}', hide=True)
        conn.run('rm -f /tmp/app_package.tar.gz', hide=True)
        print(f"  [{conn.host}] 上传完成")

    def stop_service(self, conn):
        """停止服务"""
        print(f"  [{conn.host}] 停止服务...")
        result = conn.sudo('systemctl stop myapp', warn=True, hide=True)
        if not result.ok:
            print(f"  [{conn.host}] 服务可能未运行，继续...")
        time.sleep(2)

    def start_service(self, conn):
        """启动服务"""
        print(f"  [{conn.host}] 启动服务...")
        conn.sudo('systemctl start myapp', hide=True)
        time.sleep(3)

    def health_check(self, conn, retries=3):
        """健康检查"""
        print(f"  [{conn.host}] 健康检查...")
        for attempt in range(retries):
            result = conn.run(
                'curl -sf http://localhost:8080/health || echo FAIL',
                hide=True, warn=True
            )
            if 'FAIL' not in result.stdout:
                print(f"  [{conn.host}] 健康检查通过")
                return True
            time.sleep(2)

        print(f"  [{conn.host}] 健康检查失败!")
        return False

    def rollback(self, conn):
        """回滚到备份版本"""
        print(f"  [{conn.host}] 回滚中...")
        conn.sudo('systemctl stop myapp', warn=True, hide=True)
        conn.run(f'rm -rf {self.app_dir}', hide=True)
        conn.run(f'mv {self.backup_dir} {self.app_dir}', hide=True)
        conn.sudo('systemctl start myapp', hide=True)
        print(f"  [{conn.host}] 回滚完成")

    def deploy(self, package_path, strategy='rolling'):
        """
        执行部署
        :param package_path: 本地部署包路径
        :param strategy: 部署策略 ('all_at_once' 或 'rolling')
        """
        print(f"{'=' * 60}")
        print(f"  开始部署 ({strategy} 策略)")
        print(f"  目标主机: {len(self.connections)} 台")
        print(f"{'=' * 60}")

        success = []
        failed = []

        for conn in self.connections:
            print(f"\n--- 部署到 {conn.host} ---")
            try:
                self.pre_check(conn)
                self.backup(conn)
                self.stop_service(conn)
                self.upload_code(conn, package_path)
                self.start_service(conn)

                if self.health_check(conn):
                    success.append(conn.host)
                    print(f"  [{conn.host}] 部署成功!")
                else:
                    # 健康检查失败，回滚
                    self.rollback(conn)
                    failed.append((conn.host, '健康检查失败，已回滚'))

            except Exception as e:
                print(f"  [{conn.host}] 部署失败: {e}")
                try:
                    self.rollback(conn)
                    failed.append((conn.host, f'{e}，已回滚'))
                except Exception as re:
                    failed.append((conn.host, f'{e}，回滚也失败: {re}'))

            # rolling 策略: 一台失败则停止
            if strategy == 'rolling' and failed:
                print("\n滚动部署中检测到失败，停止后续部署！")
                break

        # 部署报告
        print(f"\n{'=' * 60}")
        print(f"  部署报告")
        print(f"{'=' * 60}")
        print(f"  成功: {len(success)} 台: {', '.join(success)}")
        if failed:
            print(f"  失败: {len(failed)} 台:")
            for host, reason in failed:
                print(f"    - {host}: {reason}")
        print(f"{'=' * 60}")

    def close(self):
        for conn in self.connections:
            conn.close()


# 使用示例
if __name__ == '__main__':
    # deployer = Deployer(
    #     hosts=['192.168.1.101', '192.168.1.102', '192.168.1.103'],
    #     user='root',
    #     password='xxx',
    # )
    # deployer.deploy('/path/to/app_v2.0.tar.gz', strategy='rolling')
    # deployer.close()
    print("部署器已就绪，取消注释即可使用")
```

### 5.2 配置管理

```python
#!/usr/bin/env python3
"""Fabric 配置管理 —— 管理连接参数和运行时配置"""

from fabric import Connection, Config
from invoke import config as invoke_config

# --- 方法 1: Config 对象 ---
config = Config(overrides={
    # SSH 连接参数
    'connect_kwargs': {
        'password': 'default_password',
    },
    # sudo 设置
    'sudo': {
        'password': 'sudo_password',
        'prompt': '[sudo] password: ',
    },
    # 运行参数
    'run': {
        'hide': True,        # 默认隐藏输出
        'warn': True,        # 默认不因失败抛异常
        'echo': False,       # 不回显命令
    },
    # 超时设置
    'timeouts': {
        'connect': 10,       # 连接超时
        'command': 30,       # 命令超时
    },
})

conn = Connection('192.168.1.100', user='root', config=config)

# --- 方法 2: 从 YAML/JSON 文件加载 ---
# fabric.yml 示例:
# connect_kwargs:
#   password: "default_pass"
# sudo:
#   password: "sudo_pass"
# run:
#   hide: true

# --- 方法 3: 环境变量分组配置 ---
import os
import json

def load_env_config():
    """从环境变量加载配置"""
    env = os.environ.get('DEPLOY_ENV', 'dev')

    configs = {
        'dev': {
            'hosts': ['dev-web1'],
            'user': 'developer',
            'app_dir': '/opt/app-dev',
        },
        'staging': {
            'hosts': ['stg-web1', 'stg-web2'],
            'user': 'deployer',
            'app_dir': '/opt/app-staging',
        },
        'prod': {
            'hosts': ['prod-web1', 'prod-web2', 'prod-web3',
                      'prod-web4', 'prod-web5'],
            'user': 'deployer',
            'app_dir': '/opt/app',
        },
    }

    return configs.get(env, configs['dev'])

env_config = load_env_config()
print(f"当前环境: {os.environ.get('DEPLOY_ENV', 'dev')}")
print(f"目标主机: {env_config['hosts']}")
```

### 5.3 文件模板与配置分发

```python
#!/usr/bin/env python3
"""使用 Fabric 分发配置文件（模板渲染 + 上传）"""

from fabric import Connection
from io import StringIO
from string import Template
import json

# Nginx 配置模板
NGINX_TEMPLATE = """
# 由 Fabric 自动生成 - 请勿手动修改
# 生成时间: $timestamp

upstream backend {
    $upstream_servers
}

server {
    listen $listen_port;
    server_name $server_name;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $$host;
        proxy_set_header X-Real-IP $$remote_addr;
        proxy_connect_timeout ${proxy_timeout}s;
    }

    location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    access_log /var/log/nginx/${app_name}_access.log;
    error_log /var/log/nginx/${app_name}_error.log;
}
"""

def deploy_nginx_config(conn, app_config):
    """
    渲染并部署 Nginx 配置
    :param conn: Fabric Connection
    :param app_config: 应用配置字典
    """
    from datetime import datetime

    # 生成 upstream 配置
    upstream_lines = []
    for backend in app_config['backends']:
        upstream_lines.append(f"    server {backend};")
    upstream_servers = '\n'.join(upstream_lines)

    # 渲染模板
    template = Template(NGINX_TEMPLATE)
    config_content = template.safe_substitute(
        timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        upstream_servers=upstream_servers,
        listen_port=app_config.get('port', 80),
        server_name=app_config.get('server_name', '_'),
        proxy_timeout=app_config.get('proxy_timeout', 30),
        app_name=app_config['name'],
    )

    # 上传配置文件
    remote_path = f"/etc/nginx/conf.d/{app_config['name']}.conf"
    conn.put(StringIO(config_content), f'/tmp/{app_config["name"]}.conf')
    conn.sudo(f'mv /tmp/{app_config["name"]}.conf {remote_path}', hide=True)
    conn.sudo(f'chown root:root {remote_path}', hide=True)
    conn.sudo(f'chmod 644 {remote_path}', hide=True)

    # 测试配置
    result = conn.sudo('nginx -t', hide=True, warn=True)
    if result.ok:
        # 重载 Nginx
        conn.sudo('nginx -s reload', hide=True)
        print(f"  [{conn.host}] Nginx 配置已更新并重载")
    else:
        # 配置有误，回滚
        conn.sudo(f'rm -f {remote_path}', hide=True)
        print(f"  [{conn.host}] Nginx 配置有误，已回滚")
        print(f"  错误: {result.stderr}")

    return result.ok


# 使用示例
app_config = {
    'name': 'myapp',
    'port': 80,
    'server_name': 'app.example.com',
    'proxy_timeout': 30,
    'backends': [
        '10.0.0.1:8080',
        '10.0.0.2:8080',
        '10.0.0.3:8080',
    ],
}

# conn = Connection('192.168.1.100', user='root',
#                   connect_kwargs={'password': 'xxx'})
# deploy_nginx_config(conn, app_config)
```

## 6. SRE 实战案例：一键部署应用到多台服务器

```python
#!/usr/bin/env python3
"""
SRE 实战：一键部署应用到多台服务器
功能：
  - 多环境支持（dev/staging/prod）
  - 滚动部署（逐台部署，失败即停）
  - 自动备份与回滚
  - 健康检查
  - 部署报告与通知
  - 配置文件分发

对比 Shell 部署脚本:
  #!/bin/bash
  HOSTS="web1 web2 web3"
  VERSION=$1
  for host in $HOSTS; do
    scp app-$VERSION.tar.gz $host:/tmp/
    ssh $host "cd /opt && tar xzf /tmp/app-$VERSION.tar.gz && systemctl restart app"
  done
  # 没有备份、没有回滚、没有健康检查、没有并发、没有错误处理
"""

import json
import time
import os
from datetime import datetime
from fabric import Connection, Config
from concurrent.futures import ThreadPoolExecutor, as_completed

class AppDeployer:
    """应用部署管理器"""

    def __init__(self, config_file=None, **kwargs):
        """
        :param config_file: 配置文件路径
        """
        if config_file and os.path.exists(config_file):
            with open(config_file) as f:
                self.config = json.load(f)
        else:
            # 默认配置
            self.config = {
                'app_name': kwargs.get('app_name', 'myapp'),
                'app_dir': kwargs.get('app_dir', '/opt/myapp'),
                'backup_dir': kwargs.get('backup_dir', '/opt/backups'),
                'health_url': kwargs.get('health_url', 'http://localhost:8080/health'),
                'health_retries': kwargs.get('health_retries', 5),
                'health_interval': kwargs.get('health_interval', 3),
                'service_name': kwargs.get('service_name', 'myapp'),
                'max_parallel': kwargs.get('max_parallel', 3),
                'connect_timeout': kwargs.get('connect_timeout', 10),
                'command_timeout': kwargs.get('command_timeout', 60),
            }

        self.deploy_log = []

    def _log(self, host, message, level='INFO'):
        """记录部署日志"""
        entry = {
            'timestamp': datetime.now().isoformat(),
            'host': host,
            'level': level,
            'message': message,
        }
        self.deploy_log.append(entry)
        marker = {'INFO': '[*]', 'OK': '[+]', 'WARN': '[!]', 'ERROR': '[-]'}
        print(f"  {marker.get(level, '[*]')} [{host}] {message}")

    def _create_connection(self, host_config):
        """创建 SSH 连接"""
        conn_kwargs = {}
        if 'password' in host_config:
            conn_kwargs['password'] = host_config['password']
            conn_kwargs['look_for_keys'] = False
        if 'key_file' in host_config:
            conn_kwargs['key_filename'] = host_config['key_file']

        config = Config(overrides={
            'sudo': {'password': host_config.get('sudo_password',
                                                  host_config.get('password', ''))},
        })

        return Connection(
            host=host_config['hostname'],
            user=host_config.get('user', 'root'),
            port=host_config.get('port', 22),
            connect_kwargs=conn_kwargs,
            connect_timeout=self.config['connect_timeout'],
            config=config,
        )

    def _pre_check(self, conn):
        """部署前置检查"""
        host = conn.host
        self._log(host, "执行前置检查...")

        # 检查磁盘空间
        result = conn.run("df / | tail -1 | awk '{print $5}'",
                         hide=True, warn=True)
        if result.ok:
            usage = int(result.stdout.strip().replace('%', ''))
            if usage > 85:
                raise Exception(f"磁盘使用率 {usage}% 超过 85%")

        # 检查应用目录
        app_dir = self.config['app_dir']
        conn.run(f'mkdir -p {app_dir}', hide=True)
        conn.run(f'mkdir -p {self.config["backup_dir"]}', hide=True)

        self._log(host, "前置检查通过", 'OK')

    def _backup(self, conn, version_tag):
        """备份当前版本"""
        host = conn.host
        self._log(host, "备份当前版本...")

        backup_path = f"{self.config['backup_dir']}/{self.config['app_name']}_{version_tag}"
        app_dir = self.config['app_dir']

        # 检查应用目录是否存在内容
        result = conn.run(f'ls {app_dir}/ 2>/dev/null | head -1',
                         hide=True, warn=True)
        if result.stdout.strip():
            conn.run(f'cp -r {app_dir} {backup_path}', hide=True)
            self._log(host, f"备份完成: {backup_path}", 'OK')
        else:
            self._log(host, "应用目录为空，跳过备份", 'WARN')

        return backup_path

    def _deploy_code(self, conn, package_path):
        """部署代码"""
        host = conn.host
        app_dir = self.config['app_dir']

        # 上传部署包
        self._log(host, "上传部署包...")
        remote_pkg = f'/tmp/{self.config["app_name"]}_deploy.tar.gz'
        conn.put(package_path, remote_pkg)

        # 解压到临时目录
        tmp_dir = f'/tmp/{self.config["app_name"]}_new'
        conn.run(f'rm -rf {tmp_dir} && mkdir -p {tmp_dir}', hide=True)
        conn.run(f'tar xzf {remote_pkg} -C {tmp_dir}', hide=True)

        # 替换应用目录
        conn.run(f'rm -rf {app_dir}/*', hide=True)
        conn.run(f'cp -r {tmp_dir}/* {app_dir}/', hide=True)

        # 清理
        conn.run(f'rm -rf {tmp_dir} {remote_pkg}', hide=True)

        self._log(host, "代码部署完成", 'OK')

    def _restart_service(self, conn):
        """重启服务"""
        host = conn.host
        service = self.config['service_name']

        self._log(host, f"重启服务 {service}...")
        conn.sudo(f'systemctl restart {service}', hide=True, warn=True)
        time.sleep(2)

        # 检查服务状态
        result = conn.sudo(f'systemctl is-active {service}',
                          hide=True, warn=True)
        if result.stdout.strip() == 'active':
            self._log(host, f"服务 {service} 已启动", 'OK')
            return True
        else:
            self._log(host, f"服务 {service} 启动失败!", 'ERROR')
            return False

    def _health_check(self, conn):
        """健康检查"""
        host = conn.host
        url = self.config['health_url']
        retries = self.config['health_retries']
        interval = self.config['health_interval']

        self._log(host, f"健康检查: {url} (最多重试 {retries} 次)...")

        for i in range(retries):
            result = conn.run(
                f'curl -sf -o /dev/null -w "%{{http_code}}" {url}',
                hide=True, warn=True
            )
            status_code = result.stdout.strip()
            if status_code == '200':
                self._log(host, "健康检查通过", 'OK')
                return True

            self._log(host, f"健康检查第 {i+1} 次: HTTP {status_code}", 'WARN')
            if i < retries - 1:
                time.sleep(interval)

        self._log(host, "健康检查失败!", 'ERROR')
        return False

    def _rollback(self, conn, backup_path):
        """回滚"""
        host = conn.host
        app_dir = self.config['app_dir']
        service = self.config['service_name']

        self._log(host, "执行回滚...", 'WARN')

        # 停止服务
        conn.sudo(f'systemctl stop {service}', hide=True, warn=True)

        # 恢复备份
        conn.run(f'rm -rf {app_dir}/*', hide=True, warn=True)
        result = conn.run(f'test -d {backup_path}', hide=True, warn=True)
        if result.ok:
            conn.run(f'cp -r {backup_path}/* {app_dir}/', hide=True)

        # 重启
        conn.sudo(f'systemctl start {service}', hide=True, warn=True)

        self._log(host, "回滚完成", 'OK')

    def deploy_to_host(self, host_config, package_path):
        """部署到单台主机"""
        conn = self._create_connection(host_config)
        host = host_config['hostname']
        backup_path = None

        try:
            # 1. 前置检查
            self._pre_check(conn)

            # 2. 备份
            version_tag = datetime.now().strftime('%Y%m%d_%H%M%S')
            backup_path = self._backup(conn, version_tag)

            # 3. 部署代码
            self._deploy_code(conn, package_path)

            # 4. 重启服务
            if not self._restart_service(conn):
                raise Exception("服务启动失败")

            # 5. 健康检查
            if not self._health_check(conn):
                raise Exception("健康检查未通过")

            return {'host': host, 'success': True, 'message': '部署成功'}

        except Exception as e:
            self._log(host, f"部署失败: {e}", 'ERROR')
            # 自动回滚
            if backup_path:
                try:
                    self._rollback(conn, backup_path)
                except Exception as re:
                    self._log(host, f"回滚失败: {re}", 'ERROR')

            return {'host': host, 'success': False, 'message': str(e)}

        finally:
            conn.close()

    def deploy(self, hosts_config, package_path, strategy='rolling'):
        """
        执行部署
        :param hosts_config: 主机配置列表
        :param package_path: 部署包路径
        :param strategy: 'rolling' 逐台 | 'parallel' 并行 | 'canary' 金丝雀
        """
        start_time = datetime.now()
        results = []

        print(f"\n{'=' * 60}")
        print(f"  部署开始")
        print(f"  策略: {strategy}")
        print(f"  目标: {len(hosts_config)} 台主机")
        print(f"  应用: {self.config['app_name']}")
        print(f"  时间: {start_time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'=' * 60}\n")

        if strategy == 'rolling':
            # 逐台部署，一台失败则停止
            for i, host_config in enumerate(hosts_config):
                print(f"\n--- [{i+1}/{len(hosts_config)}] "
                      f"{host_config['hostname']} ---")
                result = self.deploy_to_host(host_config, package_path)
                results.append(result)
                if not result['success']:
                    print(f"\n滚动部署失败，停止后续部署！")
                    break

        elif strategy == 'parallel':
            # 并行部署
            max_workers = min(self.config['max_parallel'], len(hosts_config))
            with ThreadPoolExecutor(max_workers=max_workers) as executor:
                futures = {
                    executor.submit(self.deploy_to_host, hc, package_path): hc
                    for hc in hosts_config
                }
                for future in as_completed(futures):
                    results.append(future.result())

        elif strategy == 'canary':
            # 金丝雀部署: 先部署一台，成功后部署其余
            print("--- 金丝雀阶段 ---")
            canary_result = self.deploy_to_host(hosts_config[0], package_path)
            results.append(canary_result)

            if canary_result['success'] and len(hosts_config) > 1:
                print("\n--- 金丝雀成功，部署其余主机 ---")
                for hc in hosts_config[1:]:
                    result = self.deploy_to_host(hc, package_path)
                    results.append(result)
                    if not result['success']:
                        break
            elif not canary_result['success']:
                print("金丝雀部署失败，停止！")

        # 生成报告
        elapsed = (datetime.now() - start_time).total_seconds()
        self._print_report(results, elapsed)

        return results

    def _print_report(self, results, elapsed):
        """打印部署报告"""
        success = [r for r in results if r['success']]
        failed = [r for r in results if not r['success']]

        print(f"\n{'=' * 60}")
        print(f"  部署报告")
        print(f"{'=' * 60}")
        print(f"  总耗时: {elapsed:.1f} 秒")
        print(f"  成功: {len(success)} 台")
        print(f"  失败: {len(failed)} 台")

        if success:
            print(f"\n  成功主机:")
            for r in success:
                print(f"    + {r['host']}")

        if failed:
            print(f"\n  失败主机:")
            for r in failed:
                print(f"    - {r['host']}: {r['message']}")

        print(f"{'=' * 60}")

        # 保存日志
        log_file = f'/tmp/deploy_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
        with open(log_file, 'w') as f:
            json.dump({
                'config': self.config,
                'results': results,
                'log': self.deploy_log,
                'elapsed': elapsed,
            }, f, indent=2, ensure_ascii=False)
        print(f"  部署日志: {log_file}")


# ============================================================
# 使用示例
# ============================================================

if __name__ == '__main__':
    deployer = AppDeployer(
        app_name='myservice',
        app_dir='/opt/myservice',
        service_name='myservice',
        health_url='http://localhost:8080/health',
    )

    hosts = [
        {'hostname': '192.168.1.101', 'user': 'root', 'password': 'xxx'},
        {'hostname': '192.168.1.102', 'user': 'root', 'password': 'xxx'},
        {'hostname': '192.168.1.103', 'user': 'root', 'password': 'xxx'},
    ]

    # 执行部署（取消注释使用）
    # deployer.deploy(hosts, '/path/to/app-v2.0.tar.gz', strategy='canary')

    print("部署器已就绪")
    print("支持策略: rolling(滚动), parallel(并行), canary(金丝雀)")
```

## 7. 常见坑点与最佳实践

### Do's (推荐做法)

```python
# 1. 使用 Config 统一管理密码（避免明文散落）
config = Config(overrides={
    'sudo': {'password': os.environ['SUDO_PASS']},
    'connect_kwargs': {'password': os.environ['SSH_PASS']},
})

# 2. 使用 warn=True 防止中断
result = conn.run('systemctl status app', warn=True, hide=True)
if not result.ok:
    print("服务未运行")

# 3. 使用 hide=True 控制输出
result = conn.run('cat /etc/passwd', hide=True)  # 不输出到终端

# 4. 文件上传后验证
conn.put('config.yml', '/etc/app/config.yml')
result = conn.run('md5sum /etc/app/config.yml', hide=True)
# 对比本地和远程的 md5

# 5. 部署前备份，失败后回滚
# （参见上面的 SRE 实战案例）
```

### Don'ts (避免做法)

```python
# 1. 不要硬编码密码
# 坏：
conn = Connection('host', connect_kwargs={'password': 'P@ssw0rd!'})
# 好：
conn = Connection('host', connect_kwargs={'password': os.environ['SSH_PASS']})

# 2. 不要忽略命令返回值
# 坏：
conn.run('rm -rf /opt/old_app')  # 可能失败但你不知道
# 好：
result = conn.run('rm -rf /opt/old_app', warn=True, hide=True)
if not result.ok:
    print(f"清理失败: {result.stderr}")

# 3. 不要在循环中创建 Group（重复连接）
# 坏：
for cmd in commands:
    group = ThreadingGroup('h1', 'h2', 'h3')  # 每次都建新连接！
    group.run(cmd)

# 4. 不要用 shell=True 执行不可信输入
# 坏（命令注入风险）：
user_input = "; rm -rf /"
conn.run(f'echo {user_input}')
```

## 8. 速查表

```
# ==================== Fabric 速查表 ====================

# --- 安装 ---
pip install fabric

# --- 连接 ---
conn = Connection('host', user='root',
    connect_kwargs={'password': 'xxx'})           # 密码
conn = Connection('host', user='root',
    connect_kwargs={'key_filename': '/path/key'})  # 密钥

# --- 执行命令 ---
result = conn.run('cmd')                  # 执行    —— 对比: ssh host "cmd"
result = conn.run('cmd', hide=True)       # 静默执行
result = conn.run('cmd', warn=True)       # 失败不抛异常
result = conn.sudo('cmd')                 # sudo   —— 对比: ssh "sudo cmd"
result.stdout                             # 标准输出
result.stderr                             # 标准错误
result.return_code                        # 返回码
result.ok                                 # 是否成功

# --- 文件传输 ---
conn.put('local', 'remote')              # 上传  —— 对比: scp local host:remote
conn.get('remote', 'local')              # 下载  —— 对比: scp host:remote local
conn.put(StringIO('content'), 'remote')  # 上传字符串

# --- 多机执行 ---
from fabric import SerialGroup, ThreadingGroup
sg = SerialGroup('h1', 'h2', user='root')     # 串行
tg = ThreadingGroup('h1', 'h2', user='root')  # 并行
results = tg.run('uptime')                     # {conn: result}

# --- 配置 ---
config = Config(overrides={
    'sudo': {'password': 'xxx'},
    'connect_kwargs': {'password': 'xxx'},
    'run': {'hide': True, 'warn': True},
})
conn = Connection('host', config=config)
```
