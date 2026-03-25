# socket 网络编程 — 用 Python 替代 nc/telnet/nmap 实现网络检测与服务开发

## 1. 是什么 & 为什么用它

### 什么是 Socket

Socket（套接字）是网络通信的基础抽象层，提供了进程间通过网络进行数据交换的能力。Python 的 `socket` 模块是对操作系统 Berkeley Socket API 的封装。

### 为什么 SRE 需要 Socket 编程

| 运维场景 | Shell 方式 | Python Socket 方式 |
|---|---|---|
| 端口检测 | `nc -z host 80` | `socket.connect_ex()` |
| 批量端口扫描 | `for p in ...; do nc -z host $p; done` | 多线程并发扫描，快 10 倍+ |
| 健康检查 | `curl -s http://host/health` | 自定义协议级检查 |
| 简易 TCP 服务 | `nc -l -p 8080` | 完整的服务端逻辑 |
| 网络诊断 | `telnet host port` | 可编程的连接测试 |

```
Shell 的局限:
  - nc/telnet 只能逐个测试，无法并发
  - 超时控制粗糙
  - 无法实现复杂的协议交互
  - 结果处理需要大量文本解析

Python Socket 的优势:
  - 并发检测（线程/异步）
  - 精确的超时控制
  - 结构化的结果
  - 可实现任意协议
```

## 2. 安装与环境

```bash
# socket 是 Python 标准库，无需安装
python3 -c "import socket; print('socket 模块可用')"

# 如果需要更高级的功能
pip install selectors  # Python 3.4+ 内置，无需安装
```

## 3. 核心概念与原理

### TCP 通信流程

```
服务端 (Server)                     客户端 (Client)
    |                                    |
    | socket()   创建套接字               | socket()   创建套接字
    |                                    |
    | bind()     绑定地址和端口            |
    |                                    |
    | listen()   开始监听                  |
    |                                    |
    | accept()   等待连接 -------- 阻塞    |
    |          <======================== | connect()  发起连接
    | 返回新 socket（conn）               |
    |                                    |
    | recv()   <======================== | send()     发送数据
    |                                    |
    | send()   ========================> | recv()     接收数据
    |                                    |
    | close()    关闭连接                  | close()    关闭连接
```

### UDP 通信流程

```
服务端 (Server)                     客户端 (Client)
    |                                    |
    | socket(SOCK_DGRAM)                 | socket(SOCK_DGRAM)
    |                                    |
    | bind()     绑定地址和端口            |
    |                                    |
    | recvfrom()  <==================== | sendto()   发送数据
    |                                    |
    | sendto()   ====================> | recvfrom()  接收数据
    |                                    |
    无需 listen/accept/connect（无连接协议）
```

### Socket 类型对照

```
类型              协议    特点                  适用场景
────────────────  ──────  ────────────────────  ──────────────
SOCK_STREAM       TCP     可靠、有序、面向连接    HTTP、SSH、数据库
SOCK_DGRAM        UDP     不可靠、无连接、快速    DNS、日志收集、监控
SOCK_RAW          IP      原始套接字              网络诊断、ping
```

## 4. 基础用法

### 4.1 端口检测（对比 nc -z / telnet）

```python
#!/usr/bin/env python3
"""端口检测 —— 对比 nc -z 和 telnet"""

import socket

# ============================================================
# Shell 对比:
#   nc -z -w 3 www.baidu.com 80 && echo "开放" || echo "关闭"
#   echo > /dev/tcp/www.baidu.com/80 && echo "开放"
# ============================================================

def check_port(host, port, timeout=3):
    """
    检测单个端口是否开放
    :param host: 目标主机
    :param port: 目标端口
    :param timeout: 超时时间（秒）
    :return: True 如果端口开放
    """
    # 创建 TCP socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 设置超时（对比 nc -w 3）
    sock.settimeout(timeout)
    try:
        # connect_ex 返回错误码，0 表示成功
        # 比 connect() 更好，不会抛异常
        result = sock.connect_ex((host, port))
        return result == 0
    except socket.gaierror:
        # DNS 解析失败
        print(f"  DNS 解析失败: {host}")
        return False
    except socket.timeout:
        return False
    finally:
        sock.close()


# --- 单端口检测 ---
host = '127.0.0.1'
port = 22
status = "开放" if check_port(host, port) else "关闭"
print(f"{host}:{port} - {status}")

# --- 多端口检测 ---
# 对比 Shell: for port in 22 80 443 3306 6379; do nc -z host $port; done
common_ports = {
    22: 'SSH',
    80: 'HTTP',
    443: 'HTTPS',
    3306: 'MySQL',
    5432: 'PostgreSQL',
    6379: 'Redis',
    8080: 'HTTP-ALT',
    9090: 'Prometheus',
}

print(f"\n=== 端口扫描: {host} ===")
for port, service in common_ports.items():
    is_open = check_port(host, port, timeout=2)
    status = "开放" if is_open else "关闭"
    marker = "[+]" if is_open else "[-]"
    print(f"  {marker} {port:>5}/tcp  {status}  ({service})")
```

### 4.2 TCP 客户端（对比 nc / curl）

```python
#!/usr/bin/env python3
"""TCP 客户端 —— 对比 nc 和 curl"""

import socket

# ============================================================
# Shell 对比:
#   echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80
# ============================================================

def http_get(host, port=80, path='/', timeout=5):
    """
    简易 HTTP GET 请求（展示 TCP 通信原理）
    实际项目请用 requests 库，这里是学习目的
    """
    # 1. 创建 socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(timeout)

    try:
        # 2. 连接服务器
        sock.connect((host, port))
        print(f"已连接到 {host}:{port}")

        # 3. 发送 HTTP 请求
        request = (
            f"GET {path} HTTP/1.1\r\n"
            f"Host: {host}\r\n"
            f"Connection: close\r\n"
            f"\r\n"
        )
        sock.sendall(request.encode('utf-8'))
        print(f"已发送请求: GET {path}")

        # 4. 接收响应
        response = b''
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            response += chunk

        # 5. 解析响应
        response_text = response.decode('utf-8', errors='replace')
        headers, _, body = response_text.partition('\r\n\r\n')

        # 提取状态行
        status_line = headers.split('\r\n')[0]
        print(f"响应状态: {status_line}")
        print(f"响应体长度: {len(body)} 字符")

        return response_text

    except socket.timeout:
        print(f"连接超时: {host}:{port}")
        return None
    except ConnectionRefusedError:
        print(f"连接被拒绝: {host}:{port}")
        return None
    finally:
        sock.close()


# 示例
# http_get('example.com', 80, '/')
```

### 4.3 TCP 服务器

```python
#!/usr/bin/env python3
"""
TCP 服务器 —— 对比 nc -l -p 8080
实现一个简单的回显服务器（Echo Server）
"""

import socket
import threading

def handle_client(conn, addr):
    """处理单个客户端连接"""
    print(f"[+] 新连接: {addr[0]}:{addr[1]}")
    try:
        # 发送欢迎消息
        conn.sendall(b"Welcome to SRE Echo Server!\n")

        while True:
            # 接收数据（最大 1024 字节）
            data = conn.recv(1024)
            if not data:
                break  # 客户端关闭连接

            message = data.decode('utf-8', errors='replace').strip()
            print(f"[{addr[0]}:{addr[1]}] 收到: {message}")

            # 回显数据
            response = f"Echo: {message}\n"
            conn.sendall(response.encode('utf-8'))

            # 退出命令
            if message.lower() in ('quit', 'exit'):
                break

    except (ConnectionResetError, BrokenPipeError):
        print(f"[-] 连接断开: {addr[0]}:{addr[1]}")
    finally:
        conn.close()
        print(f"[-] 连接关闭: {addr[0]}:{addr[1]}")


def start_echo_server(host='0.0.0.0', port=9999, backlog=5):
    """
    启动 TCP Echo 服务器
    :param host: 监听地址
    :param port: 监听端口
    :param backlog: 等待队列长度
    """
    # 创建 TCP socket
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # 设置 SO_REUSEADDR，避免 "Address already in use" 错误
    # 对比: 这是 Shell nc 命令不容易控制的
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    # 绑定地址和端口
    server.bind((host, port))

    # 开始监听
    server.listen(backlog)
    print(f"Echo 服务器启动: {host}:{port}")
    print("等待连接...（Ctrl+C 退出）")

    try:
        while True:
            # 接受连接（阻塞）
            conn, addr = server.accept()

            # 为每个客户端创建线程
            thread = threading.Thread(
                target=handle_client,
                args=(conn, addr),
                daemon=True  # 守护线程，主线程退出时自动结束
            )
            thread.start()
            print(f"活跃线程数: {threading.active_count() - 1}")

    except KeyboardInterrupt:
        print("\n服务器关闭中...")
    finally:
        server.close()


if __name__ == '__main__':
    start_echo_server(port=9999)
    # 测试: nc 127.0.0.1 9999
    # 或:   telnet 127.0.0.1 9999
```

### 4.4 UDP 通信

```python
#!/usr/bin/env python3
"""
UDP 通信 —— 适用于日志收集、监控数据发送
对比: nc -u 127.0.0.1 9999
"""

import socket
import json
from datetime import datetime

# --- UDP 服务端（日志接收器）---
def udp_log_receiver(host='0.0.0.0', port=9998):
    """UDP 日志接收器（对比 nc -lu 9998）"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((host, port))
    print(f"UDP 日志接收器启动: {host}:{port}")

    try:
        while True:
            # recvfrom 返回 (数据, 发送方地址)
            data, addr = sock.recvfrom(65535)
            message = data.decode('utf-8')
            print(f"[{addr[0]}:{addr[1]}] {message}")
    except KeyboardInterrupt:
        print("接收器关闭")
    finally:
        sock.close()


# --- UDP 客户端（日志发送器）---
def udp_log_sender(host='127.0.0.1', port=9998):
    """UDP 日志发送器"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

    # UDP 不需要 connect，直接 sendto
    log_entry = json.dumps({
        'timestamp': datetime.now().isoformat(),
        'level': 'INFO',
        'message': '应用启动完成',
        'host': socket.gethostname(),
    })

    sock.sendto(log_entry.encode('utf-8'), (host, port))
    print(f"已发送日志到 {host}:{port}")
    sock.close()


if __name__ == '__main__':
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == 'server':
        udp_log_receiver()
    else:
        udp_log_sender()
    # 用法: python script.py server  # 启动接收器
    #       python script.py          # 发送一条日志
```

### 4.5 DNS 解析

```python
#!/usr/bin/env python3
"""DNS 解析 —— 对比 nslookup/dig/host 命令"""

import socket

# ============================================================
# Shell 对比:
#   nslookup www.baidu.com
#   dig www.baidu.com +short
#   host www.baidu.com
# ============================================================

def dns_lookup(hostname):
    """全面的 DNS 解析（对比 nslookup）"""
    print(f"=== DNS 查询: {hostname} ===")

    # --- 正向解析（域名 -> IP）---
    try:
        # 简单解析（对比 dig +short）
        ip = socket.gethostbyname(hostname)
        print(f"  IPv4 地址: {ip}")
    except socket.gaierror as e:
        print(f"  解析失败: {e}")
        return

    # 获取所有地址（可能有多个 IP）
    try:
        results = socket.getaddrinfo(hostname, None)
        ips = set()
        for family, socktype, proto, canonname, sockaddr in results:
            ip_addr = sockaddr[0]
            family_name = 'IPv4' if family == socket.AF_INET else 'IPv6'
            ips.add((family_name, ip_addr))
        for family_name, ip_addr in sorted(ips):
            print(f"  {family_name}: {ip_addr}")
    except socket.gaierror:
        pass

    # --- 反向解析（IP -> 域名）---
    try:
        hostname_result, aliases, addresses = socket.gethostbyaddr(ip)
        print(f"  反向解析: {hostname_result}")
        if aliases:
            print(f"  别名: {', '.join(aliases)}")
    except socket.herror:
        print(f"  反向解析: 无记录")

    # --- 获取本机信息 ---
    print(f"\n=== 本机信息 ===")
    print(f"  主机名: {socket.gethostname()}")
    try:
        fqdn = socket.getfqdn()
        print(f"  FQDN: {fqdn}")
    except Exception:
        pass


# 示例
dns_lookup('www.baidu.com')
```

## 5. 进阶用法

### 5.1 非阻塞 Socket 与 select

```python
#!/usr/bin/env python3
"""
非阻塞 Socket + select 多路复用
实现高性能 TCP 服务器（无需多线程）
"""

import socket
import selectors

# selectors 是 select/poll/epoll 的高级封装
# Linux 下自动使用 epoll，macOS 下使用 kqueue
sel = selectors.DefaultSelector()


def accept_connection(server_sock):
    """接受新连接"""
    conn, addr = server_sock.accept()
    print(f"[+] 新连接: {addr}")
    conn.setblocking(False)  # 设为非阻塞
    # 注册读事件
    sel.register(conn, selectors.EVENT_READ, data={'addr': addr})


def handle_data(conn, data):
    """处理客户端数据"""
    try:
        received = conn.recv(1024)
        if received:
            message = received.decode('utf-8', errors='replace').strip()
            print(f"[{data['addr']}] {message}")
            # 回显
            conn.sendall(f"Echo: {message}\n".encode('utf-8'))
        else:
            # 客户端关闭
            print(f"[-] 断开: {data['addr']}")
            sel.unregister(conn)
            conn.close()
    except (ConnectionResetError, BrokenPipeError):
        print(f"[-] 连接重置: {data['addr']}")
        sel.unregister(conn)
        conn.close()


def start_select_server(host='0.0.0.0', port=9999):
    """
    基于 selectors 的事件驱动服务器
    比多线程版本更高效，可处理大量并发连接
    """
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((host, port))
    server.listen(100)
    server.setblocking(False)

    # 注册服务端 socket 的读事件（有新连接时触发）
    sel.register(server, selectors.EVENT_READ, data=None)

    print(f"Select 服务器启动: {host}:{port}")

    try:
        while True:
            # 等待事件（阻塞直到有事件发生）
            events = sel.select(timeout=None)

            for key, mask in events:
                if key.data is None:
                    # 服务端 socket：有新连接
                    accept_connection(key.fileobj)
                else:
                    # 客户端 socket：有数据
                    handle_data(key.fileobj, key.data)

    except KeyboardInterrupt:
        print("\n服务器关闭")
    finally:
        sel.close()
        server.close()


if __name__ == '__main__':
    start_select_server()
```

### 5.2 Socket 选项与性能调优

```python
#!/usr/bin/env python3
"""Socket 高级选项 —— 性能调优与常用配置"""

import socket

def show_socket_options():
    """展示和配置常用 socket 选项"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # --- SO_REUSEADDR ---
    # 允许重用 TIME_WAIT 状态的地址
    # 场景: 服务重启时避免 "Address already in use" 错误
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    print(f"SO_REUSEADDR: {sock.getsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR)}")

    # --- SO_KEEPALIVE ---
    # 启用 TCP 保活机制，检测对端是否存活
    # 场景: 长连接检测（如数据库连接池）
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
    print(f"SO_KEEPALIVE: {sock.getsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE)}")

    # Linux 下可以设置保活参数
    try:
        # 多久没数据开始发送保活包（秒）
        sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)
        # 保活包发送间隔（秒）
        sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
        # 保活包最大重试次数
        sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 5)
        print("TCP Keep-Alive: idle=60s, interval=10s, count=5")
    except AttributeError:
        print("TCP Keep-Alive 详细参数: 当前平台不支持")

    # --- TCP_NODELAY ---
    # 禁用 Nagle 算法，减少小包延迟
    # 场景: 实时性要求高的场景（如监控数据推送）
    sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    print(f"TCP_NODELAY: {sock.getsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY)}")

    # --- SO_SNDBUF / SO_RCVBUF ---
    # 发送/接收缓冲区大小
    sndbuf = sock.getsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF)
    rcvbuf = sock.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
    print(f"发送缓冲区: {sndbuf} 字节")
    print(f"接收缓冲区: {rcvbuf} 字节")

    # 增大缓冲区（高吞吐量场景）
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 262144)  # 256KB
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 262144)

    # --- SO_LINGER ---
    # 控制关闭行为
    # l_onoff=1, l_linger=0: 立即关闭（发送 RST）
    # 场景: 需要快速释放端口时
    import struct
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER,
                    struct.pack('ii', 1, 0))
    print("SO_LINGER: 立即关闭模式")

    sock.close()

show_socket_options()
```

## 6. SRE 实战案例

### 6.1 批量端口扫描工具

```python
#!/usr/bin/env python3
"""
SRE 实战：批量端口扫描工具
功能：
  - 多线程并发扫描
  - 服务识别（通过 Banner 抓取）
  - 结果导出（JSON/表格）
  - 超时控制

对比 Shell:
  for h in host1 host2; do
    for p in 22 80 443; do
      nc -z -w 2 $h $p 2>/dev/null && echo "$h:$p OPEN"
    done
  done
  # Shell 版本是串行的，100 个端口 * 2 秒超时 = 至少 200 秒
  # Python 并发版本可在几秒内完成
"""

import socket
import json
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime

# 常见端口及其服务
WELL_KNOWN_PORTS = {
    21: 'FTP', 22: 'SSH', 23: 'Telnet', 25: 'SMTP',
    53: 'DNS', 80: 'HTTP', 110: 'POP3', 143: 'IMAP',
    443: 'HTTPS', 993: 'IMAPS', 995: 'POP3S',
    3306: 'MySQL', 3389: 'RDP', 5432: 'PostgreSQL',
    6379: 'Redis', 8080: 'HTTP-Alt', 8443: 'HTTPS-Alt',
    9090: 'Prometheus', 9200: 'Elasticsearch', 27017: 'MongoDB',
}


def scan_port(host, port, timeout=2, grab_banner=True):
    """
    扫描单个端口
    :param host: 目标主机
    :param port: 目标端口
    :param timeout: 超时（秒）
    :param grab_banner: 是否尝试抓取 Banner
    :return: 扫描结果字典
    """
    result = {
        'host': host,
        'port': port,
        'state': 'closed',
        'service': WELL_KNOWN_PORTS.get(port, 'unknown'),
        'banner': '',
    }

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(timeout)

    try:
        code = sock.connect_ex((host, port))
        if code == 0:
            result['state'] = 'open'

            # 尝试抓取 Banner
            if grab_banner:
                try:
                    sock.settimeout(1)
                    # 某些服务（如 SSH）会主动发送 Banner
                    banner = sock.recv(1024).decode('utf-8', errors='replace').strip()
                    result['banner'] = banner[:100]  # 截断过长的 Banner
                except (socket.timeout, ConnectionResetError, OSError):
                    pass
        else:
            result['state'] = 'closed'

    except socket.timeout:
        result['state'] = 'filtered'  # 超时可能是被防火墙过滤
    except socket.gaierror:
        result['state'] = 'dns_error'
    except OSError as e:
        result['state'] = f'error:{e.errno}'
    finally:
        sock.close()

    return result


def batch_scan(hosts, ports, max_workers=50, timeout=2):
    """
    批量端口扫描
    :param hosts: 主机列表
    :param ports: 端口列表
    :param max_workers: 最大并发线程数
    :param timeout: 每个端口的超时
    :return: 扫描结果列表
    """
    results = []
    total = len(hosts) * len(ports)
    completed = 0
    lock = threading.Lock()

    print(f"开始扫描: {len(hosts)} 台主机 x {len(ports)} 个端口 = {total} 个目标")
    print(f"并发线程数: {max_workers}")
    start_time = datetime.now()

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        # 提交所有扫描任务
        futures = {}
        for host in hosts:
            for port in ports:
                future = executor.submit(scan_port, host, port, timeout)
                futures[future] = (host, port)

        # 收集结果
        for future in as_completed(futures):
            result = future.result()
            results.append(result)

            with lock:
                completed += 1
                if result['state'] == 'open':
                    banner = f" [{result['banner'][:50]}]" if result['banner'] else ""
                    print(f"  [+] {result['host']}:{result['port']} "
                          f"OPEN ({result['service']}){banner}")

                # 进度显示
                if completed % 100 == 0 or completed == total:
                    elapsed = (datetime.now() - start_time).total_seconds()
                    speed = completed / max(elapsed, 0.1)
                    print(f"  进度: {completed}/{total} "
                          f"({completed/total*100:.1f}%) "
                          f"速度: {speed:.0f} 端口/秒")

    elapsed = (datetime.now() - start_time).total_seconds()
    print(f"\n扫描完成: {elapsed:.1f}秒, 共 {total} 个目标")

    return results


def print_report(results):
    """打印扫描报告"""
    # 只显示开放的端口
    open_ports = [r for r in results if r['state'] == 'open']

    print(f"\n{'=' * 70}")
    print(f"  端口扫描报告 - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'=' * 70}")
    print(f"  总扫描: {len(results)}, 开放: {len(open_ports)}, "
          f"关闭: {len([r for r in results if r['state'] == 'closed'])}, "
          f"过滤: {len([r for r in results if r['state'] == 'filtered'])}")
    print()

    # 按主机分组
    from collections import defaultdict
    by_host = defaultdict(list)
    for r in open_ports:
        by_host[r['host']].append(r)

    for host, ports in sorted(by_host.items()):
        print(f"  主机: {host}")
        for p in sorted(ports, key=lambda x: x['port']):
            banner = f"  |  {p['banner'][:40]}" if p['banner'] else ""
            print(f"    {p['port']:>5}/tcp  {p['state']:<8}  "
                  f"{p['service']:<15}{banner}")
        print()

    print(f"{'=' * 70}")


def export_results(results, filepath='/tmp/scan_results.json'):
    """导出结果到 JSON"""
    with open(filepath, 'w') as f:
        json.dump({
            'scan_time': datetime.now().isoformat(),
            'total': len(results),
            'open': len([r for r in results if r['state'] == 'open']),
            'results': results,
        }, f, indent=2)
    print(f"结果已导出: {filepath}")


if __name__ == '__main__':
    # 扫描目标
    hosts = ['127.0.0.1']
    ports = [22, 80, 443, 3306, 5432, 6379, 8080, 8443, 9090, 9200]

    # 执行扫描
    results = batch_scan(hosts, ports, max_workers=20, timeout=2)

    # 打印报告
    print_report(results)

    # 导出
    export_results(results)
```

### 6.2 简易健康检查服务

```python
#!/usr/bin/env python3
"""
SRE 实战：简易健康检查服务
功能：
  - HTTP 健康检查端点（/health）
  - TCP 端口检查
  - 自定义检查项
  - JSON 响应

对比: 使用 Shell 写健康检查脚本通常需要 curl + jq + cron
      Python 版本可以作为独立服务运行
"""

import socket
import json
import threading
import time
from datetime import datetime

class HealthChecker:
    """健康检查器"""

    def __init__(self):
        self.checks = {}  # 注册的检查项

    def add_tcp_check(self, name, host, port, timeout=3):
        """添加 TCP 端口检查"""
        def check():
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            try:
                result = sock.connect_ex((host, port))
                return {
                    'status': 'healthy' if result == 0 else 'unhealthy',
                    'detail': f'{host}:{port} {"reachable" if result == 0 else "unreachable"}',
                    'latency_ms': 0,
                }
            except Exception as e:
                return {'status': 'unhealthy', 'detail': str(e)}
            finally:
                sock.close()

        # 带延迟测量的包装
        def check_with_latency():
            start = time.time()
            result = check()
            result['latency_ms'] = round((time.time() - start) * 1000, 2)
            return result

        self.checks[name] = check_with_latency

    def add_custom_check(self, name, check_func):
        """添加自定义检查"""
        self.checks[name] = check_func

    def run_all(self):
        """运行所有检查"""
        results = {}
        overall_healthy = True

        for name, check_func in self.checks.items():
            try:
                result = check_func()
                results[name] = result
                if result.get('status') != 'healthy':
                    overall_healthy = False
            except Exception as e:
                results[name] = {
                    'status': 'error',
                    'detail': str(e),
                }
                overall_healthy = False

        return {
            'status': 'healthy' if overall_healthy else 'unhealthy',
            'timestamp': datetime.now().isoformat(),
            'checks': results,
        }


def start_health_server(checker, host='0.0.0.0', port=8080):
    """
    启动健康检查 HTTP 服务
    这是一个极简 HTTP 服务器（不依赖第三方库）
    """
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((host, port))
    server.listen(10)

    print(f"健康检查服务启动: http://{host}:{port}/health")

    def handle_request(conn, addr):
        try:
            request = conn.recv(1024).decode('utf-8', errors='replace')
            if not request:
                conn.close()
                return

            # 解析请求行
            request_line = request.split('\r\n')[0]
            method, path, _ = request_line.split(' ', 2)

            if path == '/health' and method == 'GET':
                # 执行健康检查
                result = checker.run_all()
                body = json.dumps(result, indent=2)
                status_code = 200 if result['status'] == 'healthy' else 503
                status_text = 'OK' if status_code == 200 else 'Service Unavailable'
            elif path == '/ready' and method == 'GET':
                body = json.dumps({'status': 'ready'})
                status_code = 200
                status_text = 'OK'
            else:
                body = json.dumps({'error': 'Not Found'})
                status_code = 404
                status_text = 'Not Found'

            # 构建 HTTP 响应
            response = (
                f"HTTP/1.1 {status_code} {status_text}\r\n"
                f"Content-Type: application/json\r\n"
                f"Content-Length: {len(body.encode())}\r\n"
                f"Connection: close\r\n"
                f"\r\n"
                f"{body}"
            )
            conn.sendall(response.encode('utf-8'))

        except Exception as e:
            print(f"处理请求错误: {e}")
        finally:
            conn.close()

    try:
        while True:
            conn, addr = server.accept()
            threading.Thread(target=handle_request, args=(conn, addr), daemon=True).start()
    except KeyboardInterrupt:
        print("\n健康检查服务关闭")
    finally:
        server.close()


if __name__ == '__main__':
    # 创建健康检查器
    checker = HealthChecker()

    # 添加检查项
    checker.add_tcp_check('ssh', '127.0.0.1', 22)
    checker.add_tcp_check('redis', '127.0.0.1', 6379)

    # 添加自定义检查（磁盘空间）
    import shutil
    def disk_check():
        total, used, free = shutil.disk_usage('/')
        percent = used / total * 100
        return {
            'status': 'healthy' if percent < 90 else 'unhealthy',
            'detail': f'磁盘使用率: {percent:.1f}%',
            'free_gb': round(free / (1024**3), 2),
        }
    checker.add_custom_check('disk_root', disk_check)

    # 先测试一次
    result = checker.run_all()
    print(json.dumps(result, indent=2, ensure_ascii=False))

    # 启动服务
    # start_health_server(checker, port=8080)
    # 测试: curl http://127.0.0.1:8080/health
```

## 7. 常见坑点与最佳实践

### Do's (推荐做法)

```python
# 1. 始终设置超时
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.settimeout(5)  # 5 秒超时，避免永久阻塞

# 2. 使用 SO_REUSEADDR
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))

# 3. 使用 sendall 而非 send
# send() 可能只发送部分数据
sock.sendall(data)  # 确保全部发送

# 4. 正确关闭连接
try:
    sock.shutdown(socket.SHUT_RDWR)  # 优雅关闭
except OSError:
    pass
finally:
    sock.close()

# 5. 使用 with 语句管理 socket（Python 3.2+）
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    sock.connect(('127.0.0.1', 80))
    sock.sendall(b'hello')
    data = sock.recv(1024)
# 自动关闭
```

### Don'ts (避免做法)

```python
# 1. 不要不设超时（可能永远阻塞）
sock.connect(('unreachable.host', 80))  # 可能等好几分钟！

# 2. 不要假设 recv 一次就能收到所有数据
data = sock.recv(4096)  # 可能只收到部分数据！
# 好：循环接收直到预期长度或连接关闭

# 3. 不要在生产环境用 select/poll 直接编程
# 用 selectors 模块（自动选择最佳实现）或异步框架

# 4. 不要忽略 EAGAIN/EWOULDBLOCK
# 非阻塞模式下，这些不是错误而是正常状态
import errno
try:
    data = sock.recv(1024)
except socket.error as e:
    if e.errno in (errno.EAGAIN, errno.EWOULDBLOCK):
        pass  # 暂时没有数据，稍后重试
    else:
        raise
```

## 8. 速查表

```
# ==================== Socket 速查表 ====================

# --- 创建 ---
socket.socket(AF_INET, SOCK_STREAM)     # TCP IPv4
socket.socket(AF_INET, SOCK_DGRAM)      # UDP IPv4
socket.socket(AF_INET6, SOCK_STREAM)    # TCP IPv6

# --- 服务端 ---
sock.bind(('0.0.0.0', 8080))            # 绑定地址      —— 对比: listen 指令
sock.listen(128)                         # 开始监听
conn, addr = sock.accept()              # 接受连接

# --- 客户端 ---
sock.connect(('host', 80))              # 连接          —— 对比: nc host 80
sock.connect_ex(('host', 80))           # 连接(返回码)   —— 对比: nc -z host 80

# --- 数据传输 ---
sock.send(data)                          # 发送(可能部分)
sock.sendall(data)                       # 发送(确保完整)
sock.recv(4096)                          # 接收
sock.sendto(data, addr)                  # UDP 发送
sock.recvfrom(4096)                      # UDP 接收

# --- 选项 ---
sock.settimeout(5)                       # 设置超时
sock.setblocking(False)                  # 非阻塞模式
sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
sock.setsockopt(IPPROTO_TCP, TCP_NODELAY, 1)
sock.setsockopt(SOL_SOCKET, SO_KEEPALIVE, 1)

# --- DNS ---
socket.gethostbyname('host')             # 域名解析      —— 对比: dig +short
socket.getaddrinfo('host', 80)           # 完整解析      —— 对比: getent hosts
socket.gethostbyaddr('1.2.3.4')          # 反向解析      —— 对比: dig -x
socket.gethostname()                     # 本机名        —— 对比: hostname

# --- 关闭 ---
sock.shutdown(SHUT_RDWR)                 # 优雅关闭
sock.close()                             # 释放资源
```
