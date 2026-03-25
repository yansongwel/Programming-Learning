# paramiko SSH 远程连接 — 用 Python 替代 ssh/scp 实现自动化远程操作

## 1. 是什么 & 为什么用它

### 什么是 paramiko

paramiko 是 Python 实现的 SSHv2 协议库，提供 SSH 客户端和服务端功能。它支持密码认证、密钥认证、SFTP 文件传输、端口转发等完整的 SSH 功能。

### 为什么 SRE 需要它

| 运维场景 | Shell 方式 | paramiko 方式 |
|---|---|---|
| 单机执行命令 | `ssh user@host "cmd"` | 同等便利，但可编程 |
| 批量执行 | `for h in ...; do ssh $h "cmd"; done` | 多线程并发，速度快 N 倍 |
| 结果收集 | `ssh $h "cmd" > /tmp/$h.log` | 结构化存储，支持数据库 |
| 文件传输 | `scp file user@host:/path` | SFTP 支持断点续传、进度回调 |
| 跳板机 | `ssh -J jump user@target` | 完全可编程的代理连接 |
| 错误处理 | `if [ $? -ne 0 ]; then...` | try/except 精确捕获 |

```
Shell 批量执行的痛点:
  for host in $(cat hosts.txt); do
    ssh user@$host "df -h" 2>/dev/null
  done
  - 串行执行，100 台机器要等很久
  - 某台超时会卡住整个脚本
  - 结果混在一起难以分辨
  - 密码认证需要 expect/sshpass

paramiko 优势:
  - 并发执行（ThreadPoolExecutor）
  - 超时控制（每个连接独立超时）
  - 结构化结果（{host: {stdout, stderr, returncode}}）
  - 原生支持密码/密钥认证
```

## 2. 安装与环境

```bash
# 安装 paramiko
pip install paramiko

# 验证安装
python3 -c "import paramiko; print(paramiko.__version__)"

# 如果安装失败（缺少依赖）
# CentOS/RHEL:
sudo yum install python3-devel libffi-devel openssl-devel gcc
# Ubuntu/Debian:
sudo apt-get install python3-dev libffi-dev libssl-dev gcc

pip install paramiko
```

## 3. 核心概念与原理

### paramiko 架构

```
+------------------------------------------------------------------+
|                      你的 SRE 自动化脚本                           |
+------------------------------------------------------------------+
|                        paramiko 高级 API                          |
|    SSHClient        SFTPClient        Channel        Transport   |
|  (连接管理)       (文件传输)       (命令执行)       (底层传输)     |
+------------------------------------------------------------------+
|                      paramiko 核心层                               |
|   认证层           加密层            压缩层           消息层       |
|  (password/       (AES/3DES)       (zlib)          (SSH 协议     |
|   publickey/                                        消息封装)     |
|   gssapi)                                                        |
+------------------------------------------------------------------+
|                      Python socket                                |
+------------------------------------------------------------------+
|                      TCP/IP 网络                                   |
+------------------------------------------------------------------+

SSH 连接建立过程:
  1. TCP 连接 (socket.connect)
  2. SSH 版本协商
  3. 密钥交换 (Diffie-Hellman)
  4. 服务端认证 (主机密钥验证)
  5. 用户认证 (密码/密钥/keyboard-interactive)
  6. 通道建立 (Channel)
  7. 执行命令 / 文件传输
```

### 核心类关系

```
Transport (底层连接)
    |
    +-- SSHClient (高级封装，推荐使用)
    |       |
    |       +-- exec_command()     执行命令
    |       +-- open_sftp()        打开 SFTP
    |       +-- invoke_shell()     交互式 Shell
    |
    +-- SFTPClient (文件传输)
    |       |
    |       +-- put()              上传
    |       +-- get()              下载
    |       +-- listdir()          列目录
    |       +-- stat()             文件信息
    |
    +-- Channel (数据通道)
            |
            +-- send() / recv()    原始数据收发
            +-- exec_command()     底层命令执行
```

## 4. 基础用法

### 4.1 密码认证连接

```python
#!/usr/bin/env python3
"""SSH 密码认证 —— 对比 ssh user@host"""

import paramiko

# ============================================================
# Shell 对比: ssh user@192.168.1.100
# 密码方式在 Shell 中需要 sshpass 或 expect
# sshpass -p 'password' ssh user@host "hostname"
# ============================================================

def ssh_with_password(host, username, password, command, port=22, timeout=10):
    """
    使用密码认证执行远程命令
    :param host: 目标主机
    :param username: 用户名
    :param password: 密码
    :param command: 要执行的命令
    :param port: SSH 端口
    :param timeout: 连接超时（秒）
    :return: (stdout, stderr, return_code)
    """
    # 创建 SSH 客户端
    client = paramiko.SSHClient()

    # 自动添加未知主机密钥（对比 ssh -o StrictHostKeyChecking=no）
    # 生产环境应使用 load_system_host_keys() 或自定义策略
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        # 连接（对比 ssh user@host）
        client.connect(
            hostname=host,
            port=port,
            username=username,
            password=password,
            timeout=timeout,
            # 禁用自动查找密钥，纯密码模式
            look_for_keys=False,
            allow_agent=False,
        )
        print(f"已连接: {username}@{host}:{port}")

        # 执行命令（对比 ssh user@host "command"）
        stdin, stdout, stderr = client.exec_command(
            command,
            timeout=30,  # 命令执行超时
        )

        # 获取结果
        out = stdout.read().decode('utf-8')
        err = stderr.read().decode('utf-8')
        return_code = stdout.channel.recv_exit_status()

        return out, err, return_code

    except paramiko.AuthenticationException:
        print(f"认证失败: {username}@{host}")
        return None, "认证失败", -1
    except paramiko.SSHException as e:
        print(f"SSH 错误: {e}")
        return None, str(e), -1
    except Exception as e:
        print(f"连接错误: {e}")
        return None, str(e), -1
    finally:
        client.close()


# 使用示例
# stdout, stderr, rc = ssh_with_password(
#     '192.168.1.100', 'root', 'password123', 'hostname && uptime'
# )
# print(f"输出: {stdout}")
# print(f"错误: {stderr}")
# print(f"返回码: {rc}")
```

### 4.2 密钥认证连接

```python
#!/usr/bin/env python3
"""SSH 密钥认证 —— 对比 ssh -i ~/.ssh/id_rsa user@host"""

import paramiko
from pathlib import Path

# ============================================================
# Shell 对比: ssh -i ~/.ssh/id_rsa user@192.168.1.100 "hostname"
# ============================================================

def ssh_with_key(host, username, key_path=None, command='hostname',
                 port=22, timeout=10, passphrase=None):
    """
    使用密钥认证执行远程命令
    :param key_path: 私钥文件路径（默认 ~/.ssh/id_rsa）
    :param passphrase: 私钥密码（如果有）
    """
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    # 确定密钥路径
    if key_path is None:
        key_path = Path.home() / '.ssh' / 'id_rsa'
    key_path = str(key_path)

    try:
        # 加载私钥
        # paramiko 自动识别密钥类型（RSA/DSA/ECDSA/Ed25519）
        private_key = paramiko.RSAKey.from_private_key_file(
            key_path,
            password=passphrase,  # 如果私钥有密码保护
        )
        print(f"已加载密钥: {key_path}")

    except FileNotFoundError:
        print(f"密钥文件不存在: {key_path}")
        return None, "密钥不存在", -1
    except paramiko.PasswordRequiredException:
        print(f"密钥需要密码: {key_path}")
        return None, "密钥需要密码", -1

    try:
        client.connect(
            hostname=host,
            port=port,
            username=username,
            pkey=private_key,
            timeout=timeout,
        )

        stdin, stdout, stderr = client.exec_command(command, timeout=30)
        out = stdout.read().decode('utf-8')
        err = stderr.read().decode('utf-8')
        rc = stdout.channel.recv_exit_status()

        return out, err, rc

    except paramiko.AuthenticationException:
        print(f"密钥认证失败: {username}@{host}")
        return None, "认证失败", -1
    finally:
        client.close()


# 自动检测密钥类型的通用函数
def load_private_key(key_path, passphrase=None):
    """自动检测并加载私钥（支持 RSA/DSA/ECDSA/Ed25519）"""
    key_classes = [
        paramiko.RSAKey,
        paramiko.DSSKey,
        paramiko.ECDSAKey,
        paramiko.Ed25519Key,
    ]

    for key_class in key_classes:
        try:
            return key_class.from_private_key_file(key_path, password=passphrase)
        except (paramiko.SSHException, ValueError):
            continue

    raise paramiko.SSHException(f"无法识别密钥类型: {key_path}")
```

### 4.3 SFTP 文件传输（对比 scp）

```python
#!/usr/bin/env python3
"""SFTP 文件传输 —— 对比 scp 命令"""

import paramiko
import os
from pathlib import Path

# ============================================================
# Shell 对比:
#   scp local_file user@host:/remote/path      # 上传
#   scp user@host:/remote/file local_path       # 下载
#   scp -r local_dir user@host:/remote/path     # 上传目录
# ============================================================

class SFTPManager:
    """SFTP 文件传输管理器"""

    def __init__(self, host, username, password=None, key_path=None, port=22):
        self.host = host
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

        connect_params = {
            'hostname': host,
            'port': port,
            'username': username,
        }
        if password:
            connect_params['password'] = password
            connect_params['look_for_keys'] = False
        elif key_path:
            connect_params['key_filename'] = str(key_path)

        self.client.connect(**connect_params)
        self.sftp = self.client.open_sftp()
        print(f"SFTP 已连接: {username}@{host}")

    def upload(self, local_path, remote_path, callback=None):
        """
        上传文件（对比 scp local remote）
        :param callback: 进度回调函数 (transferred, total)
        """
        local_path = str(local_path)
        remote_path = str(remote_path)

        # 默认进度回调
        if callback is None:
            file_size = os.path.getsize(local_path)
            def callback(transferred, total):
                percent = transferred / total * 100 if total > 0 else 0
                print(f"\r  上传进度: {percent:.1f}% "
                      f"({transferred}/{total} 字节)", end='', flush=True)

        self.sftp.put(local_path, remote_path, callback=callback)
        print(f"\n  上传完成: {local_path} -> {self.host}:{remote_path}")

    def download(self, remote_path, local_path, callback=None):
        """下载文件（对比 scp remote local）"""
        remote_path = str(remote_path)
        local_path = str(local_path)

        # 确保本地目录存在
        Path(local_path).parent.mkdir(parents=True, exist_ok=True)

        if callback is None:
            remote_stat = self.sftp.stat(remote_path)
            total = remote_stat.st_size
            def callback(transferred, _total):
                percent = transferred / total * 100 if total > 0 else 0
                print(f"\r  下载进度: {percent:.1f}% "
                      f"({transferred}/{total} 字节)", end='', flush=True)

        self.sftp.get(remote_path, local_path, callback=callback)
        print(f"\n  下载完成: {self.host}:{remote_path} -> {local_path}")

    def upload_dir(self, local_dir, remote_dir):
        """
        上传整个目录（对比 scp -r）
        """
        local_dir = Path(local_dir)
        count = 0

        for local_file in local_dir.rglob('*'):
            if local_file.is_file():
                # 计算远程路径
                relative = local_file.relative_to(local_dir)
                remote_file = f"{remote_dir}/{relative}"
                remote_parent = str(Path(remote_file).parent)

                # 确保远程目录存在
                self._makedirs(remote_parent)

                # 上传
                self.sftp.put(str(local_file), remote_file)
                count += 1
                print(f"  [{count}] {local_file.name} -> {remote_file}")

        print(f"目录上传完成: {count} 个文件")

    def _makedirs(self, remote_dir):
        """递归创建远程目录（对比 mkdir -p）"""
        dirs_to_create = []
        current = remote_dir

        while current and current != '/':
            try:
                self.sftp.stat(current)
                break  # 目录存在
            except FileNotFoundError:
                dirs_to_create.append(current)
                current = str(Path(current).parent)

        for d in reversed(dirs_to_create):
            try:
                self.sftp.mkdir(d)
            except OSError:
                pass  # 可能已存在

    def listdir(self, remote_path='/'):
        """列出远程目录内容（对比 ls -la）"""
        print(f"\n=== {self.host}:{remote_path} ===")
        for entry in self.sftp.listdir_attr(remote_path):
            size = entry.st_size or 0
            mtime = entry.st_mtime or 0
            from datetime import datetime
            mtime_str = datetime.fromtimestamp(mtime).strftime('%Y-%m-%d %H:%M')
            print(f"  {entry.longname}")

    def close(self):
        """关闭连接"""
        self.sftp.close()
        self.client.close()
        print(f"SFTP 已断开: {self.host}")

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()


# 使用示例
# with SFTPManager('192.168.1.100', 'root', password='xxx') as sftp:
#     sftp.upload('/tmp/local_file.txt', '/tmp/remote_file.txt')
#     sftp.download('/etc/hosts', '/tmp/remote_hosts')
#     sftp.listdir('/var/log')
```

### 4.4 跳板机连接（ProxyJump）

```python
#!/usr/bin/env python3
"""
通过跳板机连接目标服务器
对比 Shell: ssh -J jump_user@jump_host target_user@target_host
"""

import paramiko

def ssh_via_jumphost(jump_host, jump_user, jump_password,
                     target_host, target_user, target_password,
                     command, jump_port=22, target_port=22):
    """
    通过跳板机连接目标服务器
    对比: ssh -J jump_user@jump_host target_user@target_host "command"

    网络拓扑:
      本机 --> 跳板机(jump_host) --> 目标机(target_host)
    """
    # --- 第一步: 连接跳板机 ---
    jump_client = paramiko.SSHClient()
    jump_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    jump_client.connect(
        hostname=jump_host,
        port=jump_port,
        username=jump_user,
        password=jump_password,
    )
    print(f"已连接跳板机: {jump_user}@{jump_host}")

    # --- 第二步: 通过跳板机建立到目标机的隧道 ---
    # 获取跳板机的 Transport
    jump_transport = jump_client.get_transport()

    # 打开一个通道，从跳板机连到目标机
    # 这等效于在跳板机上执行 nc target_host target_port
    channel = jump_transport.open_channel(
        'direct-tcpip',
        (target_host, target_port),   # 目标地址
        ('127.0.0.1', 0),             # 本地地址（跳板机上的）
    )
    print(f"隧道已建立: {jump_host} -> {target_host}:{target_port}")

    # --- 第三步: 通过隧道连接目标机 ---
    target_client = paramiko.SSHClient()
    target_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    target_client.connect(
        hostname=target_host,
        port=target_port,
        username=target_user,
        password=target_password,
        sock=channel,  # 关键：使用隧道通道作为 socket
    )
    print(f"已连接目标机: {target_user}@{target_host}")

    # --- 第四步: 执行命令 ---
    stdin, stdout, stderr = target_client.exec_command(command, timeout=30)
    out = stdout.read().decode('utf-8')
    err = stderr.read().decode('utf-8')
    rc = stdout.channel.recv_exit_status()

    # --- 清理 ---
    target_client.close()
    channel.close()
    jump_client.close()

    return out, err, rc


# 使用示例
# out, err, rc = ssh_via_jumphost(
#     jump_host='10.0.0.1', jump_user='admin', jump_password='jump_pass',
#     target_host='172.16.0.100', target_user='root', target_password='target_pass',
#     command='hostname && ip addr'
# )
# print(out)
```

## 5. 进阶用法

### 5.1 SSH 连接池

```python
#!/usr/bin/env python3
"""SSH 连接池 —— 复用连接避免重复握手开销"""

import paramiko
import threading
import time
from collections import defaultdict

class SSHConnectionPool:
    """
    SSH 连接池
    - 复用已建立的连接
    - 自动重连断开的连接
    - 线程安全
    - 空闲超时自动清理
    """

    def __init__(self, max_connections=10, idle_timeout=300):
        """
        :param max_connections: 每台主机最大连接数
        :param idle_timeout: 空闲连接超时（秒）
        """
        self.max_connections = max_connections
        self.idle_timeout = idle_timeout
        self._pool = defaultdict(list)  # {host_key: [(client, last_used_time)]}
        self._lock = threading.Lock()

    def _make_key(self, host, port, username):
        """生成连接池键"""
        return f"{username}@{host}:{port}"

    def get_connection(self, host, username, password=None,
                       key_path=None, port=22):
        """获取一个连接（从池中取或新建）"""
        key = self._make_key(host, port, username)

        with self._lock:
            # 尝试从池中获取可用连接
            while self._pool[key]:
                client, last_used = self._pool[key].pop(0)
                # 检查连接是否仍然有效
                if client.get_transport() and client.get_transport().is_active():
                    return client
                else:
                    # 连接已断开，关闭并丢弃
                    try:
                        client.close()
                    except Exception:
                        pass

        # 池中没有可用连接，创建新连接
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

        connect_kwargs = {
            'hostname': host,
            'port': port,
            'username': username,
            'timeout': 10,
        }
        if password:
            connect_kwargs['password'] = password
            connect_kwargs['look_for_keys'] = False
        elif key_path:
            connect_kwargs['key_filename'] = str(key_path)

        client.connect(**connect_kwargs)
        return client

    def return_connection(self, host, username, client, port=22):
        """归还连接到池中"""
        key = self._make_key(host, port, username)

        with self._lock:
            # 检查连接是否有效且池未满
            if (client.get_transport()
                    and client.get_transport().is_active()
                    and len(self._pool[key]) < self.max_connections):
                self._pool[key].append((client, time.time()))
            else:
                try:
                    client.close()
                except Exception:
                    pass

    def execute(self, host, username, command, password=None,
                key_path=None, port=22, timeout=30):
        """执行命令（自动获取和归还连接）"""
        client = self.get_connection(host, username, password, key_path, port)

        try:
            stdin, stdout, stderr = client.exec_command(command, timeout=timeout)
            out = stdout.read().decode('utf-8')
            err = stderr.read().decode('utf-8')
            rc = stdout.channel.recv_exit_status()

            # 执行成功，归还连接
            self.return_connection(host, username, client, port)
            return out, err, rc

        except Exception as e:
            # 执行失败，关闭连接
            try:
                client.close()
            except Exception:
                pass
            raise

    def cleanup(self):
        """清理所有空闲连接"""
        with self._lock:
            now = time.time()
            for key in list(self._pool.keys()):
                active = []
                for client, last_used in self._pool[key]:
                    if now - last_used > self.idle_timeout:
                        try:
                            client.close()
                        except Exception:
                            pass
                    else:
                        active.append((client, last_used))
                self._pool[key] = active

    def close_all(self):
        """关闭所有连接"""
        with self._lock:
            for key in self._pool:
                for client, _ in self._pool[key]:
                    try:
                        client.close()
                    except Exception:
                        pass
            self._pool.clear()


# 使用示例
# pool = SSHConnectionPool(max_connections=5)
# try:
#     out, err, rc = pool.execute('192.168.1.100', 'root', 'hostname',
#                                  password='xxx')
#     print(out)
# finally:
#     pool.close_all()
```

### 5.2 交互式命令执行

```python
#!/usr/bin/env python3
"""交互式 Shell —— 处理需要交互的命令（如 sudo、passwd）"""

import paramiko
import time

def execute_with_sudo(host, username, password, command, port=22):
    """
    使用 sudo 执行命令（需要输入密码）
    对比 Shell: ssh user@host "echo password | sudo -S command"
    """
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(hostname=host, port=port,
                   username=username, password=password)

    # 使用 invoke_shell 获取交互式终端
    channel = client.invoke_shell()
    time.sleep(0.5)  # 等待 shell 初始化

    # 读取初始输出（欢迎信息等）
    if channel.recv_ready():
        channel.recv(4096)

    # 执行 sudo 命令
    channel.send(f'sudo -S {command}\n')
    time.sleep(1)

    # 检查是否需要密码
    output = ''
    if channel.recv_ready():
        output = channel.recv(4096).decode('utf-8')
        if 'password' in output.lower():
            # 输入密码
            channel.send(f'{password}\n')
            time.sleep(2)

    # 读取命令输出
    result = ''
    while channel.recv_ready() or not result:
        if channel.recv_ready():
            chunk = channel.recv(4096).decode('utf-8')
            result += chunk
        else:
            time.sleep(0.5)
            if not channel.recv_ready():
                break

    channel.close()
    client.close()
    return result


# 使用示例
# result = execute_with_sudo('192.168.1.100', 'admin', 'password',
#                             'systemctl status nginx')
# print(result)
```

## 6. SRE 实战案例：批量服务器命令执行

```python
#!/usr/bin/env python3
"""
SRE 实战：批量在多台服务器上执行命令并收集结果
功能：
  - 并发 SSH 连接（可控制并发数）
  - 超时控制
  - 结果收集与报告
  - 失败重试
  - 支持密码和密钥认证

对比 Shell:
  for host in $(cat hosts.txt); do
    ssh user@$host "df -h" >> results.log 2>&1
  done
  # 串行执行，无并发，无超时控制，结果混在一起
"""

import paramiko
import json
import time
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
from dataclasses import dataclass, field
from typing import List, Dict, Optional

@dataclass
class HostConfig:
    """主机配置"""
    hostname: str
    port: int = 22
    username: str = 'root'
    password: Optional[str] = None
    key_path: Optional[str] = None
    tags: Dict = field(default_factory=dict)  # 如 {'env': 'prod', 'role': 'web'}

@dataclass
class ExecutionResult:
    """执行结果"""
    hostname: str
    command: str
    stdout: str = ''
    stderr: str = ''
    return_code: int = -1
    success: bool = False
    duration: float = 0.0
    error: str = ''
    timestamp: str = ''


class BatchExecutor:
    """批量命令执行器"""

    def __init__(self, hosts, max_workers=20, timeout=30, retries=1):
        """
        :param hosts: HostConfig 列表
        :param max_workers: 最大并发线程数
        :param timeout: 每台主机的超时（秒）
        :param retries: 失败重试次数
        """
        self.hosts = hosts
        self.max_workers = max_workers
        self.timeout = timeout
        self.retries = retries
        self.results = []
        self._lock = threading.Lock()

    def _execute_on_host(self, host_config, command):
        """在单台主机上执行命令"""
        result = ExecutionResult(
            hostname=host_config.hostname,
            command=command,
            timestamp=datetime.now().isoformat(),
        )
        start_time = time.time()

        for attempt in range(self.retries + 1):
            try:
                client = paramiko.SSHClient()
                client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

                connect_kwargs = {
                    'hostname': host_config.hostname,
                    'port': host_config.port,
                    'username': host_config.username,
                    'timeout': self.timeout,
                    'banner_timeout': 10,
                    'auth_timeout': 10,
                }

                if host_config.password:
                    connect_kwargs['password'] = host_config.password
                    connect_kwargs['look_for_keys'] = False
                    connect_kwargs['allow_agent'] = False
                elif host_config.key_path:
                    connect_kwargs['key_filename'] = host_config.key_path

                client.connect(**connect_kwargs)

                stdin, stdout, stderr = client.exec_command(
                    command, timeout=self.timeout
                )

                result.stdout = stdout.read().decode('utf-8', errors='replace')
                result.stderr = stderr.read().decode('utf-8', errors='replace')
                result.return_code = stdout.channel.recv_exit_status()
                result.success = (result.return_code == 0)
                result.duration = time.time() - start_time

                client.close()
                break  # 成功则跳出重试循环

            except paramiko.AuthenticationException:
                result.error = "认证失败"
                break  # 认证失败不重试
            except (paramiko.SSHException, OSError) as e:
                result.error = f"连接失败 (尝试 {attempt + 1}): {str(e)}"
                if attempt < self.retries:
                    time.sleep(2)  # 重试前等待
            except Exception as e:
                result.error = f"未知错误: {str(e)}"
                break
            finally:
                try:
                    client.close()
                except Exception:
                    pass

        result.duration = time.time() - start_time
        return result

    def run(self, command):
        """
        在所有主机上并发执行命令
        :param command: 要执行的命令
        :return: ExecutionResult 列表
        """
        self.results = []
        total = len(self.hosts)
        completed = 0

        print(f"{'=' * 60}")
        print(f"  批量执行: {command}")
        print(f"  目标主机: {total} 台")
        print(f"  并发数: {self.max_workers}")
        print(f"{'=' * 60}")

        start_time = time.time()

        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(self._execute_on_host, host, command): host
                for host in self.hosts
            }

            for future in as_completed(futures):
                result = future.result()
                with self._lock:
                    self.results.append(result)
                    completed += 1

                    # 实时输出
                    status = "OK" if result.success else "FAIL"
                    print(f"  [{completed}/{total}] {result.hostname}: "
                          f"{status} ({result.duration:.1f}s)"
                          f"{' - ' + result.error if result.error else ''}")

        elapsed = time.time() - start_time
        print(f"\n完成: {elapsed:.1f}秒, "
              f"成功: {sum(1 for r in self.results if r.success)}, "
              f"失败: {sum(1 for r in self.results if not r.success)}")

        return self.results

    def print_report(self):
        """打印详细报告"""
        if not self.results:
            print("没有执行结果")
            return

        success = [r for r in self.results if r.success]
        failed = [r for r in self.results if not r.success]

        print(f"\n{'=' * 60}")
        print(f"  执行报告")
        print(f"{'=' * 60}")

        # 成功的主机
        if success:
            print(f"\n--- 成功 ({len(success)} 台) ---")
            for r in sorted(success, key=lambda x: x.hostname):
                print(f"\n  [{r.hostname}] (耗时: {r.duration:.1f}s)")
                # 缩进输出
                for line in r.stdout.strip().split('\n')[:10]:
                    print(f"    {line}")
                if r.stdout.count('\n') > 10:
                    print(f"    ... (共 {r.stdout.count(chr(10))} 行)")

        # 失败的主机
        if failed:
            print(f"\n--- 失败 ({len(failed)} 台) ---")
            for r in sorted(failed, key=lambda x: x.hostname):
                print(f"\n  [{r.hostname}] 错误: {r.error}")
                if r.stderr:
                    print(f"    stderr: {r.stderr.strip()[:200]}")

        print(f"\n{'=' * 60}")

    def export_json(self, filepath='/tmp/batch_results.json'):
        """导出结果为 JSON"""
        data = {
            'timestamp': datetime.now().isoformat(),
            'total': len(self.results),
            'success': sum(1 for r in self.results if r.success),
            'failed': sum(1 for r in self.results if not r.success),
            'results': [
                {
                    'hostname': r.hostname,
                    'command': r.command,
                    'stdout': r.stdout,
                    'stderr': r.stderr,
                    'return_code': r.return_code,
                    'success': r.success,
                    'duration': r.duration,
                    'error': r.error,
                    'timestamp': r.timestamp,
                }
                for r in self.results
            ]
        }

        with open(filepath, 'w') as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        print(f"结果已导出: {filepath}")


# ============================================================
# 使用示例
# ============================================================

if __name__ == '__main__':
    # 定义主机列表
    hosts = [
        HostConfig(
            hostname='192.168.1.101',
            username='root',
            password='password123',
            tags={'env': 'prod', 'role': 'web'},
        ),
        HostConfig(
            hostname='192.168.1.102',
            username='root',
            key_path='/root/.ssh/id_rsa',
            tags={'env': 'prod', 'role': 'db'},
        ),
        # 可以从文件加载
    ]

    # 从文件加载主机列表
    def load_hosts_from_file(filepath):
        """从 JSON 文件加载主机配置"""
        with open(filepath) as f:
            data = json.load(f)
        return [HostConfig(**h) for h in data]

    # 创建执行器
    executor = BatchExecutor(
        hosts=hosts,
        max_workers=20,    # 并发 20 个线程
        timeout=30,        # 每台 30 秒超时
        retries=1,         # 失败重试 1 次
    )

    # 执行命令
    # results = executor.run('hostname && uptime && df -h / && free -m')
    # executor.print_report()
    # executor.export_json()

    print("示例代码已准备就绪")
    print("取消注释 executor.run() 行即可执行")
    print("请确保 hosts 列表中的主机可连接")
```

## 7. 常见坑点与最佳实践

### Do's (推荐做法)

```python
# 1. 始终关闭连接（使用 try/finally 或 with）
client = paramiko.SSHClient()
try:
    client.connect(...)
    # 操作
finally:
    client.close()

# 2. 设置合理的超时
client.connect(hostname=host, timeout=10,   # 连接超时
               banner_timeout=10,            # Banner 超时
               auth_timeout=10)              # 认证超时
stdin, stdout, stderr = client.exec_command(cmd, timeout=30)  # 命令超时

# 3. 正确处理主机密钥
# 生产环境：加载已知主机密钥
client.load_system_host_keys()
client.load_host_keys('/path/to/known_hosts')
# 自定义策略（记录并警告）
class WarningPolicy(paramiko.MissingHostKeyPolicy):
    def missing_host_key(self, client, hostname, key):
        import logging
        logging.warning(f"未知主机密钥: {hostname}")
client.set_missing_host_key_policy(WarningPolicy())

# 4. 读取大量输出时使用流式读取
stdin, stdout, stderr = client.exec_command('find / -name "*.log"')
for line in stdout:
    process(line.strip())  # 逐行处理，不会占用过多内存

# 5. 使用 exec_command 的 environment 参数
stdin, stdout, stderr = client.exec_command(
    'echo $MY_VAR',
    environment={'MY_VAR': 'hello'}  # 注意: 服务端需要 AcceptEnv
)
```

### Don'ts (避免做法)

```python
# 1. 不要在生产环境使用 AutoAddPolicy
# 坏（中间人攻击风险）：
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

# 2. 不要忘记读取 stdout/stderr（可能导致死锁）
stdin, stdout, stderr = client.exec_command('some command')
# 坏：直接获取返回码而不读输出（缓冲区可能满导致死锁）
rc = stdout.channel.recv_exit_status()
# 好：先读输出再获取返回码
out = stdout.read()
err = stderr.read()
rc = stdout.channel.recv_exit_status()

# 3. 不要硬编码密码
# 坏：
client.connect(hostname='host', password='P@ssw0rd!')
# 好（从环境变量或密钥管理系统获取）：
import os
password = os.environ.get('SSH_PASSWORD')

# 4. 不要在循环中反复创建和关闭连接
# 坏：
for cmd in commands:
    client = paramiko.SSHClient()
    client.connect(...)
    client.exec_command(cmd)
    client.close()
# 好：复用连接
client.connect(...)
for cmd in commands:
    client.exec_command(cmd)
client.close()
```

## 8. 速查表

```
# ==================== paramiko 速查表 ====================

# --- 安装 ---
pip install paramiko

# --- 连接 ---
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(host, port=22, username='user',
               password='pass')                     # 密码认证
client.connect(host, username='user',
               key_filename='/path/to/key')          # 密钥认证

# --- 执行命令 (对比: ssh user@host "cmd") ---
stdin, stdout, stderr = client.exec_command('cmd')
out = stdout.read().decode()
err = stderr.read().decode()
rc = stdout.channel.recv_exit_status()

# --- SFTP (对比: scp) ---
sftp = client.open_sftp()
sftp.put('local', 'remote')           # 上传    scp local remote
sftp.get('remote', 'local')           # 下载    scp remote local
sftp.listdir('/path')                 # 列目录  ls
sftp.stat('/path/file')               # 文件信息 stat
sftp.mkdir('/path/dir')               # 创建目录 mkdir
sftp.remove('/path/file')             # 删除文件 rm
sftp.rename('old', 'new')             # 重命名   mv
sftp.chmod('/path', 0o755)            # 改权限   chmod
sftp.close()

# --- 跳板机 ---
jump_transport = jump_client.get_transport()
channel = jump_transport.open_channel('direct-tcpip',
    (target_host, 22), ('127.0.0.1', 0))
target_client.connect(target_host, sock=channel)

# --- 交互式 Shell ---
channel = client.invoke_shell()
channel.send('command\n')
output = channel.recv(4096)

# --- 清理 ---
client.close()
```
