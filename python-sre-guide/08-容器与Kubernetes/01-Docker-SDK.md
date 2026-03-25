# Docker SDK for Python — 用代码管理容器生命周期与自动化运维

## 1. 是什么 & 为什么用它

`docker-py`（包名 `docker`）是 Docker 官方提供的 Python SDK，可以通过代码完成所有 Docker CLI 能做的操作：
容器创建/启停/删除、镜像构建/拉取/推送、网络和卷管理等。

**对 SRE 的价值：**
- 将容器运维操作编程化，嵌入自动化流水线
- 编写自定义的容器健康检查和自愈脚本
- 批量管理多台 Docker Host 上的容器
- 构建自定义的部署工具和 CI/CD 组件

## 2. 安装与环境

```bash
# 安装 docker SDK
pip install docker

# 验证安装
python -c "import docker; client = docker.from_env(); print(client.version())"
```

**环境要求：**
- Python >= 3.7
- Docker Engine 已安装并运行
- 当前用户有 Docker Socket 访问权限（在 `docker` 用户组中或使用 sudo）

```bash
# 确认 Docker 运行
docker info

# 如果权限不足，将用户加入 docker 组
sudo usermod -aG docker $USER
```

## 3. 核心概念与原理

### 3.1 docker-py 架构

```
+-------------------+          +-------------------+          +-------------------+
|  Python 脚本      |   HTTP   |  Docker Daemon    |  系统    |  容器 / 镜像      |
|  (docker-py)      |--------->|  (dockerd)        |--------->|  (containerd)     |
+-------------------+  REST    +-------------------+  调用    +-------------------+
        |              API             |
        |                              |
        v                              v
  Unix Socket                   /var/run/docker.sock
  (默认) 或 TCP                 或 tcp://host:2375
```

### 3.2 核心对象模型

```
DockerClient
  ├── containers      容器管理
  │     ├── run()           运行容器
  │     ├── list()          列出容器
  │     ├── get()           获取容器
  │     └── prune()         清理停止的容器
  ├── images          镜像管理
  │     ├── build()         构建镜像
  │     ├── pull()          拉取镜像
  │     ├── push()          推送镜像
  │     └── prune()         清理无用镜像
  ├── networks        网络管理
  │     ├── create()        创建网络
  │     └── list()          列出网络
  ├── volumes         卷管理
  │     ├── create()        创建卷
  │     └── list()          列出卷
  └── swarm           Swarm 管理
```

### 3.3 连接方式

```
方式                       URL                          适用场景
───────────────────────────────────────────────────────────────────
本地 Socket（默认）        unix:///var/run/docker.sock   本地 Docker
TCP 不加密                 tcp://host:2375               内网测试
TCP + TLS                  tcp://host:2376               生产远程管理
SSH                        ssh://user@host               安全远程管理
```

## 4. 基础用法

### 4.1 连接 Docker

```python
"""
Docker 客户端连接方式
"""
import docker

# 方式一：从环境变量自动检测（推荐）
# 自动读取 DOCKER_HOST、DOCKER_TLS_VERIFY、DOCKER_CERT_PATH
client = docker.from_env()

# 方式二：连接本地 Docker Socket
client = docker.DockerClient(base_url='unix:///var/run/docker.sock')

# 方式三：连接远程 Docker（TCP）
client = docker.DockerClient(base_url='tcp://192.168.1.100:2375')

# 方式四：连接远程 Docker（TLS）
tls_config = docker.tls.TLSConfig(
    client_cert=('/path/to/cert.pem', '/path/to/key.pem'),
    ca_cert='/path/to/ca.pem',
    verify=True
)
client = docker.DockerClient(base_url='tcp://192.168.1.100:2376', tls=tls_config)

# 方式五：SSH 连接
client = docker.DockerClient(base_url='ssh://user@remote-host')

# 验证连接
info = client.info()
print(f"Docker 版本：{client.version()['Version']}")
print(f"容器数量：{info['Containers']}")
print(f"镜像数量：{info['Images']}")
print(f"系统：{info['OperatingSystem']}")
```

### 4.2 容器管理

```python
"""
容器生命周期管理：创建、启动、停止、删除、日志
"""
import docker
import time

client = docker.from_env()

# === 运行容器（相当于 docker run）===
container = client.containers.run(
    image='nginx:alpine',               # 镜像名
    name='my-nginx',                     # 容器名
    detach=True,                         # 后台运行
    ports={'80/tcp': 8080},              # 端口映射 容器80 -> 宿主机8080
    environment={                        # 环境变量
        'NGINX_HOST': 'example.com',
        'NGINX_PORT': '80'
    },
    volumes={                            # 卷挂载
        '/tmp/nginx-html': {
            'bind': '/usr/share/nginx/html',
            'mode': 'rw'
        }
    },
    labels={                             # 标签
        'app': 'nginx',
        'env': 'development',
        'team': 'sre'
    },
    restart_policy={                     # 重启策略
        'Name': 'unless-stopped',
        'MaximumRetryCount': 3
    },
    mem_limit='256m',                    # 内存限制
    cpu_period=100000,                   # CPU 限制
    cpu_quota=50000,                     # 使用 50% CPU
    healthcheck={                        # 健康检查
        'test': ['CMD', 'curl', '-f', 'http://localhost/'],
        'interval': 10 * 1000000000,     # 10 秒（纳秒）
        'timeout': 5 * 1000000000,
        'retries': 3,
        'start_period': 5 * 1000000000
    }
)
print(f"容器已创建：{container.id[:12]} ({container.name})")

# === 列出容器 ===
# 所有运行中的容器
running = client.containers.list()
for c in running:
    print(f"  运行中：{c.name} ({c.short_id}) - {c.status}")

# 包含已停止的容器
all_containers = client.containers.list(all=True)

# 按标签过滤
sre_containers = client.containers.list(filters={'label': 'team=sre'})

# === 获取容器详情 ===
container = client.containers.get('my-nginx')
print(f"容器名：{container.name}")
print(f"状态：{container.status}")
print(f"IP 地址：{container.attrs['NetworkSettings']['IPAddress']}")
print(f"创建时间：{container.attrs['Created']}")

# === 容器操作 ===
container.stop(timeout=10)    # 停止（等待 10 秒后强制 kill）
container.start()             # 启动
container.restart(timeout=10) # 重启
container.pause()             # 暂停
container.unpause()           # 恢复
container.kill()              # 强制终止

# === 获取日志 ===
# 获取全部日志
logs = container.logs().decode('utf-8')
print(logs)

# 流式获取日志（实时跟踪）
for line in container.logs(stream=True, follow=True, tail=10):
    print(line.decode('utf-8').strip())

# 获取指定时间范围的日志
import datetime
since = datetime.datetime.now() - datetime.timedelta(hours=1)
logs = container.logs(since=since).decode('utf-8')

# === 在容器内执行命令 ===
exit_code, output = container.exec_run('nginx -t')
print(f"退出码：{exit_code}")
print(f"输出：{output.decode('utf-8')}")

# 交互式执行
exit_code, output = container.exec_run(
    'ls -la /etc/nginx/',
    user='root',
    environment={'MY_VAR': 'hello'}
)

# === 容器资源统计 ===
stats = container.stats(stream=False)
cpu_delta = stats['cpu_stats']['cpu_usage']['total_usage'] - \
            stats['precpu_stats']['cpu_usage']['total_usage']
system_delta = stats['cpu_stats']['system_cpu_usage'] - \
               stats['precpu_stats']['system_cpu_usage']
cpu_percent = (cpu_delta / system_delta) * 100.0 if system_delta > 0 else 0
mem_usage = stats['memory_stats']['usage']
mem_limit = stats['memory_stats']['limit']
mem_percent = (mem_usage / mem_limit) * 100.0

print(f"CPU 使用率：{cpu_percent:.2f}%")
print(f"内存使用：{mem_usage / 1024 / 1024:.1f}MB / {mem_limit / 1024 / 1024:.1f}MB ({mem_percent:.1f}%)")

# === 删除容器 ===
container.stop()
container.remove(force=True)  # force=True 可以删除运行中的容器

# === 清理停止的容器 ===
pruned = client.containers.prune()
print(f"已清理：{pruned}")
```

### 4.3 镜像管理

```python
"""
镜像管理：构建、拉取、推送、清理
"""
import docker
import json

client = docker.from_env()

# === 拉取镜像 ===
image = client.images.pull('python', tag='3.11-slim')
print(f"已拉取：{image.tags}")

# 拉取并显示进度
for line in client.api.pull('nginx', tag='alpine', stream=True, decode=True):
    if 'status' in line:
        print(f"  {line['status']} {line.get('progress', '')}")

# === 列出本地镜像 ===
images = client.images.list()
for img in images:
    tags = img.tags if img.tags else ['<none>']
    size_mb = img.attrs['Size'] / 1024 / 1024
    print(f"  {tags[0]:40s} {size_mb:.1f}MB")

# === 构建镜像 ===
# 从 Dockerfile 构建
image, build_logs = client.images.build(
    path='/path/to/context',            # 构建上下文路径
    dockerfile='Dockerfile',            # Dockerfile 路径（相对于 context）
    tag='myapp:latest',                 # 镜像标签
    buildargs={                         # 构建参数
        'PIP_INDEX_URL': 'https://mirrors.aliyun.com/pypi/simple/',
    },
    rm=True,                            # 构建后删除中间容器
    nocache=False,                      # 是否使用缓存
    labels={
        'maintainer': 'sre@example.com',
        'version': '1.0.0'
    }
)

# 打印构建日志
for chunk in build_logs:
    if 'stream' in chunk:
        print(chunk['stream'], end='')

# 从字符串构建（fileobj）
import io
dockerfile_content = """
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
"""
f = io.BytesIO(dockerfile_content.encode('utf-8'))
image, logs = client.images.build(fileobj=f, tag='myapp:dev')

# === 标记镜像 ===
image = client.images.get('myapp:latest')
image.tag('registry.example.com/myapp', tag='v1.0.0')

# === 推送镜像 ===
# 先登录
client.login(
    username='user',
    password='password',
    registry='registry.example.com'
)

# 推送
for line in client.images.push('registry.example.com/myapp', tag='v1.0.0',
                                stream=True, decode=True):
    print(line)

# === 删除镜像 ===
client.images.remove('myapp:dev', force=True)

# === 清理无用镜像 ===
pruned = client.images.prune(filters={'dangling': True})
print(f"已清理：{pruned['ImagesDeleted']}")
print(f"释放空间：{pruned['SpaceReclaimed'] / 1024 / 1024:.1f}MB")
```

### 4.4 网络和卷管理

```python
"""
Docker 网络和卷管理
"""
import docker

client = docker.from_env()

# === 网络管理 ===

# 创建自定义网络
network = client.networks.create(
    name='myapp-network',
    driver='bridge',
    ipam=docker.types.IPAMConfig(
        pool_configs=[
            docker.types.IPAMPool(
                subnet='172.20.0.0/16',
                gateway='172.20.0.1'
            )
        ]
    ),
    labels={'app': 'myapp', 'env': 'dev'}
)
print(f"网络已创建：{network.name} ({network.short_id})")

# 列出网络
for net in client.networks.list():
    print(f"  {net.name:30s} {net.attrs['Driver']:10s} {net.short_id}")

# 将容器连接到网络
network.connect('my-container')  # 通过容器名或 ID
network.disconnect('my-container')

# 删除网络
network.remove()

# 清理未使用的网络
client.networks.prune()

# === 卷管理 ===

# 创建卷
volume = client.volumes.create(
    name='myapp-data',
    driver='local',
    labels={'app': 'myapp', 'purpose': 'persistent-data'}
)
print(f"卷已创建：{volume.name}")

# 列出卷
for vol in client.volumes.list():
    print(f"  {vol.name:30s} {vol.attrs['Driver']}")

# 使用卷运行容器
container = client.containers.run(
    'mysql:8.0',
    name='my-mysql',
    detach=True,
    volumes={'myapp-data': {'bind': '/var/lib/mysql', 'mode': 'rw'}},
    environment={'MYSQL_ROOT_PASSWORD': 'secret'}
)

# 删除卷
volume.remove(force=True)

# 清理未使用的卷
client.volumes.prune()
```

## 5. 进阶用法

### 5.1 Docker Compose 操作

```python
"""
通过 docker-py 实现类似 Docker Compose 的编排操作
"""
import docker
import yaml
import time

client = docker.from_env()

class SimpleCompose:
    """
    简化版 Compose：从 YAML 定义创建和管理多容器应用
    """

    def __init__(self, project_name, compose_config):
        """
        参数：
            project_name: 项目名称（用于容器名前缀）
            compose_config: compose 配置字典
        """
        self.project = project_name
        self.config = compose_config
        self.client = docker.from_env()
        self.containers = {}
        self.networks = {}

    def up(self):
        """启动所有服务（类似 docker-compose up -d）"""
        print(f"启动项目：{self.project}")

        # 创建网络
        network_name = f"{self.project}_default"
        try:
            network = self.client.networks.get(network_name)
        except docker.errors.NotFound:
            network = self.client.networks.create(network_name, driver='bridge')
        self.networks['default'] = network

        # 按依赖顺序创建服务
        for svc_name, svc_config in self.config['services'].items():
            self._start_service(svc_name, svc_config, network_name)

        print(f"所有服务已启动")

    def _start_service(self, name, config, network_name):
        """启动单个服务"""
        container_name = f"{self.project}_{name}"

        # 检查是否已存在
        try:
            existing = self.client.containers.get(container_name)
            if existing.status == 'running':
                print(f"  {name}: 已在运行")
                self.containers[name] = existing
                return
            existing.remove(force=True)
        except docker.errors.NotFound:
            pass

        # 拉取镜像
        image = config.get('image', '')
        if image:
            try:
                self.client.images.get(image)
            except docker.errors.ImageNotFound:
                print(f"  {name}: 拉取镜像 {image}...")
                self.client.images.pull(image)

        # 准备端口映射
        ports = {}
        for port_mapping in config.get('ports', []):
            host_port, container_port = str(port_mapping).split(':')
            ports[f'{container_port}/tcp'] = int(host_port)

        # 准备卷挂载
        volumes = {}
        for vol in config.get('volumes', []):
            parts = vol.split(':')
            if len(parts) >= 2:
                volumes[parts[0]] = {'bind': parts[1], 'mode': parts[2] if len(parts) > 2 else 'rw'}

        # 创建并启动容器
        container = self.client.containers.run(
            image=image,
            name=container_name,
            detach=True,
            network=network_name,
            ports=ports,
            volumes=volumes,
            environment=config.get('environment', {}),
            command=config.get('command'),
            restart_policy={'Name': config.get('restart', 'unless-stopped')},
            labels={
                'com.docker.compose.project': self.project,
                'com.docker.compose.service': name,
            }
        )

        self.containers[name] = container
        print(f"  {name}: 已启动 ({container.short_id})")

    def down(self):
        """停止并删除所有服务（类似 docker-compose down）"""
        print(f"停止项目：{self.project}")

        for name, container in self.containers.items():
            try:
                container.stop(timeout=10)
                container.remove()
                print(f"  {name}: 已停止并删除")
            except Exception as e:
                print(f"  {name}: 停止失败 - {e}")

        # 删除网络
        for net_name, network in self.networks.items():
            try:
                network.remove()
                print(f"  网络 {net_name}: 已删除")
            except Exception as e:
                print(f"  网络 {net_name}: 删除失败 - {e}")

    def ps(self):
        """查看服务状态（类似 docker-compose ps）"""
        print(f"\n{'服务':<20s} {'状态':<15s} {'容器 ID':<15s} {'端口'}")
        print("-" * 70)
        for name, container in self.containers.items():
            container.reload()
            ports_info = container.attrs['NetworkSettings']['Ports']
            port_str = ', '.join(
                f"{p}->{binds[0]['HostPort']}" if binds else p
                for p, binds in (ports_info or {}).items()
            )
            print(f"{name:<20s} {container.status:<15s} {container.short_id:<15s} {port_str}")

    def logs(self, service_name, tail=50):
        """查看服务日志"""
        if service_name in self.containers:
            logs = self.containers[service_name].logs(tail=tail).decode('utf-8')
            print(f"--- {service_name} 日志 ---")
            print(logs)

# === 使用示例 ===
compose_config = {
    'services': {
        'web': {
            'image': 'nginx:alpine',
            'ports': ['8080:80'],
            'restart': 'unless-stopped'
        },
        'redis': {
            'image': 'redis:alpine',
            'ports': ['6379:6379'],
            'restart': 'unless-stopped'
        }
    }
}

if __name__ == '__main__':
    compose = SimpleCompose('myapp', compose_config)
    compose.up()
    compose.ps()
    time.sleep(5)
    compose.down()
```

### 5.2 事件监听

```python
"""
Docker 事件实时监听
用于：自动化响应、审计日志、容器编排
"""
import docker
import json
from datetime import datetime

client = docker.from_env()

def monitor_events(filters=None, timeout=None):
    """
    实时监听 Docker 事件

    参数：
        filters: 事件过滤器
        timeout: 超时时间（秒）
    """
    event_filters = filters or {
        'type': ['container', 'image', 'network', 'volume'],
    }

    print("开始监听 Docker 事件...")
    print(f"过滤器：{event_filters}")
    print("-" * 80)

    for event in client.events(decode=True, filters=event_filters):
        timestamp = datetime.fromtimestamp(event['time']).strftime('%Y-%m-%d %H:%M:%S')
        event_type = event.get('Type', 'unknown')
        action = event.get('Action', 'unknown')
        actor = event.get('Actor', {})
        name = actor.get('Attributes', {}).get('name', actor.get('ID', 'unknown')[:12])

        print(f"[{timestamp}] {event_type:10s} {action:15s} {name}")

        # 自动响应逻辑
        if event_type == 'container' and action == 'die':
            exit_code = actor.get('Attributes', {}).get('exitCode', 'unknown')
            print(f"  ⚠ 容器退出，退出码：{exit_code}")
            if exit_code != '0':
                print(f"  ⚠ 异常退出！可能需要自动重启")

# 只监听容器事件
# monitor_events(filters={'type': ['container']})
```

## 6. SRE 实战案例：自动化容器健康检查与自动重启

```python
"""
SRE 实战：容器健康检查与自动重启系统
功能：
  1. 定期检查所有容器的健康状态
  2. 对不健康的容器执行自动重启
  3. 资源使用率监控与告警
  4. 自动清理退出的容器和无用镜像
  5. 生成巡检报告
"""
import docker
import time
import logging
import json
from datetime import datetime, timedelta
from collections import defaultdict

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger('container-watchdog')

class ContainerWatchdog:
    """
    容器看门狗：自动健康检查与自愈
    """

    def __init__(self, docker_url=None):
        """
        初始化看门狗

        参数：
            docker_url: Docker 连接地址（None 使用默认）
        """
        if docker_url:
            self.client = docker.DockerClient(base_url=docker_url)
        else:
            self.client = docker.from_env()

        # 配置参数
        self.config = {
            'check_interval': 30,           # 检查间隔（秒）
            'max_restart_count': 3,         # 最大自动重启次数
            'restart_window': 3600,         # 重启窗口期（秒）
            'cpu_threshold': 90.0,          # CPU 告警阈值（%）
            'memory_threshold': 85.0,       # 内存告警阈值（%）
            'cleanup_interval': 3600,       # 清理间隔（秒）
            'managed_label': 'watchdog.managed',  # 受管理的容器标签
        }

        # 重启历史记录
        self.restart_history = defaultdict(list)

        # 统计数据
        self.stats = {
            'checks_total': 0,
            'restarts_total': 0,
            'alerts_total': 0,
            'start_time': datetime.now()
        }

        logger.info("容器看门狗已初始化")
        self._log_docker_info()

    def _log_docker_info(self):
        """记录 Docker 环境信息"""
        info = self.client.info()
        logger.info(f"Docker 版本：{self.client.version()['Version']}")
        logger.info(f"容器数量：{info['ContainersRunning']} 运行 / "
                    f"{info['ContainersStopped']} 停止 / "
                    f"{info['ContainersPaused']} 暂停")
        logger.info(f"镜像数量：{info['Images']}")

    # ================================================================
    # 健康检查
    # ================================================================

    def check_all_containers(self):
        """检查所有受管理容器的健康状态"""
        self.stats['checks_total'] += 1
        containers = self.client.containers.list(
            filters={'label': self.config['managed_label']}
        )

        if not containers:
            # 如果没有带标签的容器，检查所有运行中的容器
            containers = self.client.containers.list()

        results = []
        for container in containers:
            result = self._check_container(container)
            results.append(result)

        return results

    def _check_container(self, container):
        """
        检查单个容器的健康状态

        返回：检查结果字典
        """
        container.reload()  # 刷新容器状态

        result = {
            'name': container.name,
            'id': container.short_id,
            'status': container.status,
            'image': container.image.tags[0] if container.image.tags else 'unknown',
            'health': 'unknown',
            'cpu_percent': 0.0,
            'memory_percent': 0.0,
            'memory_usage_mb': 0.0,
            'uptime': '',
            'restart_count': 0,
            'issues': [],
            'timestamp': datetime.now().isoformat()
        }

        # 检查运行状态
        if container.status != 'running':
            result['issues'].append(f'容器未运行（状态：{container.status}）')
            return result

        # 检查 Docker 内置健康检查
        health_status = container.attrs.get('State', {}).get('Health', {})
        if health_status:
            result['health'] = health_status.get('Status', 'unknown')
            if result['health'] == 'unhealthy':
                # 获取最后一次健康检查的输出
                logs = health_status.get('Log', [])
                if logs:
                    last_check = logs[-1]
                    result['issues'].append(
                        f'健康检查失败：{last_check.get("Output", "")[:200]}'
                    )
        else:
            result['health'] = 'no_healthcheck'

        # 获取资源使用情况
        try:
            stats = container.stats(stream=False)
            result['cpu_percent'] = self._calculate_cpu_percent(stats)
            result['memory_percent'] = self._calculate_memory_percent(stats)
            result['memory_usage_mb'] = stats['memory_stats'].get('usage', 0) / 1024 / 1024

            # CPU 告警
            if result['cpu_percent'] > self.config['cpu_threshold']:
                result['issues'].append(
                    f'CPU 使用率过高：{result["cpu_percent"]:.1f}%'
                )

            # 内存告警
            if result['memory_percent'] > self.config['memory_threshold']:
                result['issues'].append(
                    f'内存使用率过高：{result["memory_percent"]:.1f}%'
                )
        except Exception as e:
            result['issues'].append(f'获取资源统计失败：{e}')

        # 获取重启次数
        result['restart_count'] = container.attrs.get('RestartCount', 0)

        # 计算运行时间
        started_at = container.attrs.get('State', {}).get('StartedAt', '')
        if started_at:
            try:
                start_time = datetime.fromisoformat(started_at.replace('Z', '+00:00'))
                uptime = datetime.now(start_time.tzinfo) - start_time
                result['uptime'] = str(uptime).split('.')[0]
            except Exception:
                pass

        return result

    def _calculate_cpu_percent(self, stats):
        """计算 CPU 使用率百分比"""
        try:
            cpu_delta = (stats['cpu_stats']['cpu_usage']['total_usage'] -
                        stats['precpu_stats']['cpu_usage']['total_usage'])
            system_delta = (stats['cpu_stats']['system_cpu_usage'] -
                          stats['precpu_stats']['system_cpu_usage'])
            num_cpus = len(stats['cpu_stats']['cpu_usage'].get('percpu_usage', [1]))
            if system_delta > 0:
                return (cpu_delta / system_delta) * num_cpus * 100.0
        except (KeyError, ZeroDivisionError):
            pass
        return 0.0

    def _calculate_memory_percent(self, stats):
        """计算内存使用率百分比"""
        try:
            usage = stats['memory_stats']['usage']
            limit = stats['memory_stats']['limit']
            if limit > 0:
                return (usage / limit) * 100.0
        except KeyError:
            pass
        return 0.0

    # ================================================================
    # 自动重启
    # ================================================================

    def auto_restart(self, container_name, reason=''):
        """
        自动重启不健康的容器

        参数：
            container_name: 容器名称
            reason: 重启原因

        返回：是否执行了重启
        """
        # 检查重启频率限制
        now = time.time()
        window = self.config['restart_window']
        max_restarts = self.config['max_restart_count']

        # 清理过期的重启记录
        self.restart_history[container_name] = [
            t for t in self.restart_history[container_name]
            if now - t < window
        ]

        # 检查是否超出限制
        recent_restarts = len(self.restart_history[container_name])
        if recent_restarts >= max_restarts:
            logger.warning(
                f"[{container_name}] 在 {window}s 内已重启 {recent_restarts} 次，"
                f"跳过自动重启（需要人工介入）"
            )
            self.stats['alerts_total'] += 1
            return False

        try:
            container = self.client.containers.get(container_name)

            # 记录重启前的日志（最后 50 行）
            pre_logs = container.logs(tail=50).decode('utf-8', errors='replace')
            logger.info(f"[{container_name}] 重启前日志：\n{pre_logs[-500:]}")

            # 执行重启
            logger.info(f"[{container_name}] 开始重启，原因：{reason}")
            container.restart(timeout=30)

            # 记录重启
            self.restart_history[container_name].append(now)
            self.stats['restarts_total'] += 1

            # 等待容器启动
            time.sleep(5)
            container.reload()

            if container.status == 'running':
                logger.info(f"[{container_name}] 重启成功，当前状态：{container.status}")
                return True
            else:
                logger.error(f"[{container_name}] 重启后状态异常：{container.status}")
                return False

        except docker.errors.NotFound:
            logger.error(f"[{container_name}] 容器不存在")
            return False
        except Exception as e:
            logger.error(f"[{container_name}] 重启失败：{e}")
            return False

    # ================================================================
    # 自动清理
    # ================================================================

    def cleanup(self):
        """清理停止的容器和无用镜像"""
        logger.info("开始自动清理...")

        # 清理退出的容器（保留最近 1 小时的）
        try:
            pruned = self.client.containers.prune(
                filters={'until': '1h'}
            )
            deleted = pruned.get('ContainersDeleted', []) or []
            logger.info(f"已清理 {len(deleted)} 个停止的容器")
        except Exception as e:
            logger.error(f"容器清理失败：{e}")

        # 清理悬空镜像
        try:
            pruned = self.client.images.prune(filters={'dangling': True})
            deleted = pruned.get('ImagesDeleted', []) or []
            space = pruned.get('SpaceReclaimed', 0)
            logger.info(
                f"已清理 {len(deleted)} 个无用镜像，"
                f"释放 {space / 1024 / 1024:.1f}MB"
            )
        except Exception as e:
            logger.error(f"镜像清理失败：{e}")

        # 清理未使用的网络
        try:
            pruned = self.client.networks.prune()
            deleted = pruned.get('NetworksDeleted', []) or []
            if deleted:
                logger.info(f"已清理 {len(deleted)} 个未使用网络")
        except Exception as e:
            logger.error(f"网络清理失败：{e}")

    # ================================================================
    # 巡检报告
    # ================================================================

    def generate_report(self, results):
        """生成巡检报告"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'hostname': self.client.info()['Name'],
            'docker_version': self.client.version()['Version'],
            'summary': {
                'total': len(results),
                'healthy': sum(1 for r in results if not r['issues']),
                'unhealthy': sum(1 for r in results if r['issues']),
                'not_running': sum(1 for r in results if r['status'] != 'running'),
            },
            'containers': results,
            'watchdog_stats': {
                'uptime': str(datetime.now() - self.stats['start_time']).split('.')[0],
                'total_checks': self.stats['checks_total'],
                'total_restarts': self.stats['restarts_total'],
                'total_alerts': self.stats['alerts_total'],
            }
        }

        # 打印摘要
        s = report['summary']
        logger.info(
            f"巡检报告 - 总计: {s['total']}, "
            f"健康: {s['healthy']}, "
            f"异常: {s['unhealthy']}, "
            f"未运行: {s['not_running']}"
        )

        # 打印异常容器详情
        for r in results:
            if r['issues']:
                logger.warning(
                    f"  [{r['name']}] {r['status']} | "
                    f"CPU: {r['cpu_percent']:.1f}% | "
                    f"内存: {r['memory_percent']:.1f}% | "
                    f"问题: {'; '.join(r['issues'])}"
                )

        return report

    # ================================================================
    # 主循环
    # ================================================================

    def run(self):
        """启动看门狗主循环"""
        logger.info("容器看门狗启动...")
        logger.info(f"检查间隔：{self.config['check_interval']}s")
        logger.info(f"CPU 告警阈值：{self.config['cpu_threshold']}%")
        logger.info(f"内存告警阈值：{self.config['memory_threshold']}%")

        last_cleanup = time.time()

        try:
            while True:
                # 执行健康检查
                results = self.check_all_containers()

                # 生成报告
                report = self.generate_report(results)

                # 对异常容器执行自动重启
                for result in results:
                    if result['health'] == 'unhealthy' or (
                        result['status'] == 'running' and result['issues']
                    ):
                        # 只对标记了自动重启的容器执行
                        try:
                            container = self.client.containers.get(result['name'])
                            auto_restart_label = container.labels.get(
                                'watchdog.auto-restart', 'true'
                            )
                            if auto_restart_label.lower() == 'true':
                                reason = '; '.join(result['issues'])
                                self.auto_restart(result['name'], reason=reason)
                        except docker.errors.NotFound:
                            pass

                # 定期清理
                if time.time() - last_cleanup > self.config['cleanup_interval']:
                    self.cleanup()
                    last_cleanup = time.time()

                time.sleep(self.config['check_interval'])

        except KeyboardInterrupt:
            logger.info("看门狗已停止")


# ============================================================
# 主入口
# ============================================================

def main():
    """启动容器看门狗"""
    watchdog = ContainerWatchdog()

    # 可以自定义配置
    watchdog.config.update({
        'check_interval': 30,
        'max_restart_count': 3,
        'cpu_threshold': 90.0,
        'memory_threshold': 85.0,
    })

    # 执行一次检查（用于测试）
    results = watchdog.check_all_containers()
    report = watchdog.generate_report(results)

    # 如果需要持续运行，取消下面的注释
    # watchdog.run()

    print(f"\n巡检完成，共检查 {len(results)} 个容器")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Permission denied | 无 Docker Socket 权限 | 加入 docker 组或使用 sudo |
| 连接超时 | Docker Daemon 未启动 | `systemctl start docker` |
| 镜像拉取慢 | 网络问题 | 配置国内镜像源 |
| 容器名冲突 | 同名容器已存在 | 先检查并删除旧容器 |
| stats 接口慢 | 默认 stream=True | 使用 `stream=False` |
| 内存泄漏 | 未关闭 stream | 使用 `with` 或手动关闭 |
| API 版本不匹配 | SDK 和 Daemon 版本差异 | 指定 `api_version` |

### 7.2 最佳实践

```
[✓] 使用 docker.from_env() 自动检测连接方式
[✓] 操作前检查容器/镜像是否存在（try/except NotFound）
[✓] 使用标签（labels）标记和过滤容器
[✓] 获取 stats 时使用 stream=False 避免阻塞
[✓] 设置容器资源限制（mem_limit / cpu_quota）
[✓] 为自动化脚本设置合理的超时时间
[✓] 批量操作前先做 dry-run 检查
[✓] 日志操作使用 tail 参数限制行数
[✓] 定期执行 prune 清理无用资源
[✓] 远程连接务必使用 TLS 加密
```

## 8. 速查表

```python
# === 安装 ===
# pip install docker

# === 连接 ===
import docker
client = docker.from_env()                              # 自动检测
client = docker.DockerClient(base_url='unix:///var/run/docker.sock')
client = docker.DockerClient(base_url='tcp://host:2375')

# === 容器 ===
c = client.containers.run('nginx', detach=True, name='web', ports={'80/tcp': 8080})
c = client.containers.get('container_name_or_id')
client.containers.list()                                 # 运行中的
client.containers.list(all=True)                         # 所有
client.containers.list(filters={'label': 'app=web'})     # 按标签

c.start() / c.stop() / c.restart() / c.kill() / c.pause() / c.unpause()
c.remove(force=True)
c.logs(tail=100)                                         # 获取日志
c.exec_run('cmd')                                        # 执行命令
c.stats(stream=False)                                    # 资源统计
c.reload()                                               # 刷新状态
client.containers.prune()                                # 清理停止的

# === 镜像 ===
client.images.pull('nginx', tag='alpine')
client.images.build(path='.', tag='app:v1')
client.images.get('nginx:alpine')
client.images.list()
client.images.remove('image:tag')
client.images.prune(filters={'dangling': True})

# === 网络 ===
net = client.networks.create('mynet', driver='bridge')
net.connect('container')
net.disconnect('container')
net.remove()

# === 卷 ===
vol = client.volumes.create('mydata')
vol.remove()
client.volumes.prune()

# === 事件 ===
for event in client.events(decode=True):
    print(event)

# === 系统 ===
client.info()                # Docker 系统信息
client.version()             # Docker 版本
client.df()                  # 磁盘使用
client.ping()                # 连通性测试
```
