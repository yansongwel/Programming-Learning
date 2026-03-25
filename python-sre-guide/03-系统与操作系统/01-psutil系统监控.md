# psutil 系统监控 — 用 Python 替代 top/free/df/netstat 实现全方位系统监控

## 1. 是什么 & 为什么用它

### 什么是 psutil

psutil（process and system utilities）是一个跨平台的 Python 库，用于获取系统运行时的进程和系统利用率信息（CPU、内存、磁盘、网络、传感器）。它实现了许多 UNIX 命令行工具提供的功能，如 `top`、`free`、`df`、`netstat`、`ifconfig`、`who`、`ps`、`kill` 等。

### 为什么 SRE 需要它

| 传统 Shell 方式 | psutil 方式 | 优势 |
|---|---|---|
| `top -bn1 \| grep Cpu` 然后用 awk 解析 | `psutil.cpu_percent()` | 直接返回数值，无需文本解析 |
| `free -m \| awk '/Mem/{print $3}'` | `psutil.virtual_memory().used` | 结构化数据，跨平台 |
| `df -h \| awk '{print $5}'` | `psutil.disk_usage('/').percent` | 类型安全，可编程 |
| `netstat -tlnp \| grep LISTEN` | `psutil.net_connections()` | 返回对象列表，易于过滤 |

**核心价值：**
- 告别 Shell 中的文本解析（awk/sed/grep），直接获取结构化数据
- 跨平台（Linux/macOS/Windows），同一套代码适配多环境
- 与 Python 生态无缝集成（Prometheus、Grafana、数据库）

## 2. 安装与环境

```bash
# 安装 psutil
pip install psutil

# 验证安装
python3 -c "import psutil; print(psutil.__version__)"

# 如果在生产环境，建议使用虚拟环境
python3 -m venv ~/sre-env
source ~/sre-env/bin/activate
pip install psutil
```

**系统依赖（部分 Linux 发行版需要）：**
```bash
# CentOS/RHEL
sudo yum install python3-devel gcc

# Ubuntu/Debian
sudo apt-get install python3-dev gcc
```

## 3. 核心概念与原理

### psutil 架构总览

```
+-------------------------------------------------------------------+
|                        你的 Python SRE 脚本                         |
+-------------------------------------------------------------------+
|                         psutil Python API                          |
|  cpu_percent()  virtual_memory()  disk_usage()  net_connections()  |
+-------------------------------------------------------------------+
|                      psutil C 扩展层 (_psutil)                      |
+-------------------------------------------------------------------+
|                        操作系统内核接口                               |
|  /proc 文件系统    sysctl    Windows API    macOS IOKit             |
+-------------------------------------------------------------------+
|                           操作系统内核                               |
|              CPU调度器  内存管理  文件系统  网络栈                     |
+-------------------------------------------------------------------+
```

### 数据来源对照

```
psutil 函数                 Linux 数据源          等效 Shell 命令
─────────────────────────  ──────────────────    ────────────────
cpu_percent()              /proc/stat            top, mpstat
cpu_freq()                 /proc/cpuinfo         lscpu
virtual_memory()           /proc/meminfo         free
swap_memory()              /proc/swaps           swapon -s
disk_usage()               statvfs() 系统调用     df
disk_io_counters()         /proc/diskstats       iostat
net_io_counters()          /proc/net/dev         ifconfig, ip -s
net_connections()          /proc/net/tcp(6)      netstat, ss
process_iter()             /proc/[pid]/          ps aux
sensors_temperatures()     /sys/class/hwmon      sensors
boot_time()                /proc/stat            uptime, who -b
```

## 4. 基础用法

### 4.1 CPU 信息采集

```python
#!/usr/bin/env python3
"""CPU 信息采集 —— 对比 top/mpstat 命令"""

import psutil
import time

# ============================================================
# Shell 对比: top -bn1 | head -5
# 用 psutil 我们不需要解析文本输出
# ============================================================

# --- CPU 使用率 ---
# interval=1 表示采样 1 秒（阻塞式），更准确
# 不传 interval 则返回自上次调用以来的使用率
cpu_percent = psutil.cpu_percent(interval=1)
print(f"CPU 总使用率: {cpu_percent}%")

# 每个核心的使用率（对比: mpstat -P ALL 1 1）
per_cpu = psutil.cpu_percent(interval=1, percpu=True)
for i, percent in enumerate(per_cpu):
    print(f"  核心 {i}: {percent}%")

# --- CPU 核心数 ---
# logical=True 包含超线程（对比: nproc）
# logical=False 只返回物理核心（对比: lscpu | grep "Core(s)"）
print(f"逻辑核心数: {psutil.cpu_count(logical=True)}")
print(f"物理核心数: {psutil.cpu_count(logical=False)}")

# --- CPU 频率 ---
# 对比: lscpu | grep MHz
freq = psutil.cpu_freq()
if freq:
    print(f"CPU 频率: 当前 {freq.current:.0f}MHz, "
          f"最小 {freq.min:.0f}MHz, 最大 {freq.max:.0f}MHz")

# --- CPU 时间 ---
# 对比: cat /proc/stat | head -1
cpu_times = psutil.cpu_times()
print(f"CPU 时间: user={cpu_times.user:.1f}s, "
      f"system={cpu_times.system:.1f}s, "
      f"idle={cpu_times.idle:.1f}s, "
      f"iowait={getattr(cpu_times, 'iowait', 0):.1f}s")

# --- 系统负载 ---
# 对比: uptime 或 cat /proc/loadavg
load1, load5, load15 = psutil.getloadavg()
cpu_count = psutil.cpu_count()
print(f"系统负载: 1分钟={load1:.2f}, 5分钟={load5:.2f}, 15分钟={load15:.2f}")
print(f"每核负载: 1分钟={load1/cpu_count:.2f}")
```

### 4.2 内存信息采集

```python
#!/usr/bin/env python3
"""内存信息采集 —— 对比 free -m 命令"""

import psutil

# ============================================================
# Shell 对比: free -m
#               total   used   free   shared  buff/cache  available
# Mem:          15884   4523   2341      532       9020      10545
# Swap:          2048    128   1920
# ============================================================

# --- 物理内存 ---
mem = psutil.virtual_memory()

# 自动转换单位的辅助函数
def bytes_to_human(n):
    """将字节转换为人类可读格式（对比 free -h 的输出）"""
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}PB"

print("=== 物理内存（对比 free -m）===")
print(f"总量:     {bytes_to_human(mem.total)}")
print(f"已用:     {bytes_to_human(mem.used)}")
print(f"空闲:     {bytes_to_human(mem.free)}")
print(f"可用:     {bytes_to_human(mem.available)}")  # 最重要的指标
print(f"缓存:     {bytes_to_human(getattr(mem, 'cached', 0))}")
print(f"缓冲:     {bytes_to_human(getattr(mem, 'buffers', 0))}")
print(f"使用率:   {mem.percent}%")

# --- Swap 内存 ---
# 对比: swapon -s 或 free -m 的 Swap 行
swap = psutil.swap_memory()
print(f"\n=== Swap 内存（对比 swapon -s）===")
print(f"总量: {bytes_to_human(swap.total)}")
print(f"已用: {bytes_to_human(swap.used)} ({swap.percent}%)")
print(f"空闲: {bytes_to_human(swap.free)}")

# --- 判断内存是否紧张 ---
if mem.percent > 90:
    print("\n[警告] 内存使用超过 90%!")
if swap.percent > 50:
    print("[警告] Swap 使用超过 50%，可能存在内存不足！")
```

### 4.3 磁盘信息采集

```python
#!/usr/bin/env python3
"""磁盘信息采集 —— 对比 df -h / iostat 命令"""

import psutil

# ============================================================
# Shell 对比: df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   20G   28G  42% /
# ============================================================

def bytes_to_human(n):
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}PB"

# --- 磁盘分区信息 ---
# 对比: lsblk 或 fdisk -l
print("=== 磁盘分区（对比 df -h）===")
print(f"{'设备':<20} {'挂载点':<15} {'文件系统':<10} {'总量':>8} {'已用':>8} {'可用':>8} {'使用率':>6}")
print("-" * 85)

for part in psutil.disk_partitions():
    try:
        usage = psutil.disk_usage(part.mountpoint)
        print(f"{part.device:<20} {part.mountpoint:<15} {part.fstype:<10} "
              f"{bytes_to_human(usage.total):>8} {bytes_to_human(usage.used):>8} "
              f"{bytes_to_human(usage.free):>8} {usage.percent:>5.1f}%")
    except PermissionError:
        # 某些挂载点可能无权限访问
        print(f"{part.device:<20} {part.mountpoint:<15} {part.fstype:<10} {'无权限':>8}")

# --- 磁盘 IO 统计 ---
# 对比: iostat -d
print(f"\n=== 磁盘 IO（对比 iostat）===")
disk_io = psutil.disk_io_counters(perdisk=True)
for disk_name, io in disk_io.items():
    print(f"磁盘 {disk_name}: "
          f"读={bytes_to_human(io.read_bytes)}, "
          f"写={bytes_to_human(io.write_bytes)}, "
          f"读次数={io.read_count}, 写次数={io.write_count}")
```

### 4.4 网络信息采集

```python
#!/usr/bin/env python3
"""网络信息采集 —— 对比 ifconfig/netstat/ss 命令"""

import psutil

def bytes_to_human(n):
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}PB"

# --- 网络接口信息 ---
# 对比: ifconfig 或 ip addr
print("=== 网络接口（对比 ifconfig）===")
addrs = psutil.net_if_addrs()
for iface, addr_list in addrs.items():
    print(f"\n接口: {iface}")
    for addr in addr_list:
        if addr.family.name == 'AF_INET':
            print(f"  IPv4: {addr.address}, 掩码: {addr.netmask}")
        elif addr.family.name == 'AF_INET6':
            print(f"  IPv6: {addr.address}")

# --- 网络 IO 统计 ---
# 对比: ip -s link 或 ifconfig
print("\n=== 网络流量（对比 ip -s link）===")
net_io = psutil.net_io_counters(pernic=True)
for iface, io in net_io.items():
    print(f"{iface}: 发送={bytes_to_human(io.bytes_sent)}, "
          f"接收={bytes_to_human(io.bytes_recv)}, "
          f"发送包={io.packets_sent}, 接收包={io.packets_recv}")

# --- 网络连接 ---
# 对比: netstat -tlnp 或 ss -tlnp
print("\n=== 监听端口（对比 netstat -tlnp）===")
connections = psutil.net_connections(kind='inet')
listening = [c for c in connections if c.status == 'LISTEN']
for conn in sorted(listening, key=lambda c: c.laddr.port):
    pid = conn.pid or '-'
    try:
        proc_name = psutil.Process(conn.pid).name() if conn.pid else '-'
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        proc_name = '-'
    print(f"  {conn.laddr.ip}:{conn.laddr.port} "
          f"(PID={pid}, 进程={proc_name})")
```

### 4.5 进程管理

```python
#!/usr/bin/env python3
"""进程管理 —— 对比 ps/kill/pgrep 命令"""

import psutil

# ============================================================
# Shell 对比: ps aux --sort=-%mem | head -10
# ============================================================

def bytes_to_human(n):
    for unit in ['B', 'KB', 'MB', 'GB']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}TB"

# --- 按内存使用排序的进程列表（对比 ps aux --sort=-%mem）---
print("=== Top 10 内存占用进程 ===")
print(f"{'PID':>7} {'进程名':<20} {'CPU%':>6} {'内存%':>6} {'内存':>10} {'状态':<10}")
print("-" * 65)

# 收集进程信息（使用 process_iter 避免竞态条件）
procs = []
for proc in psutil.process_iter(['pid', 'name', 'memory_percent',
                                  'cpu_percent', 'memory_info', 'status']):
    try:
        info = proc.info
        procs.append(info)
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        pass

# 按内存使用率排序
procs.sort(key=lambda p: p.get('memory_percent', 0) or 0, reverse=True)

for p in procs[:10]:
    mem_bytes = p['memory_info'].rss if p.get('memory_info') else 0
    print(f"{p['pid']:>7} {(p['name'] or '-'):<20} "
          f"{(p.get('cpu_percent') or 0):>5.1f}% "
          f"{(p.get('memory_percent') or 0):>5.1f}% "
          f"{bytes_to_human(mem_bytes):>10} "
          f"{p.get('status', '-'):<10}")

# --- 查找特定进程（对比 pgrep / ps aux | grep xxx）---
print("\n=== 查找 python 进程（对比 pgrep python）===")
for proc in psutil.process_iter(['pid', 'name', 'cmdline']):
    try:
        if 'python' in (proc.info['name'] or '').lower():
            cmdline = ' '.join(proc.info['cmdline'] or [])
            print(f"  PID={proc.info['pid']}, 命令: {cmdline[:80]}")
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        pass
```

## 5. 进阶用法

### 5.1 持续监控与趋势分析

```python
#!/usr/bin/env python3
"""持续监控系统指标 —— 带历史趋势"""

import psutil
import time
from collections import deque
from datetime import datetime

class SystemMonitor:
    """系统监控器：采集并存储历史数据"""

    def __init__(self, history_size=60):
        """
        初始化监控器
        :param history_size: 保留最近多少个数据点（默认 60 个，即 1 分钟）
        """
        self.history = {
            'cpu': deque(maxlen=history_size),
            'memory': deque(maxlen=history_size),
            'disk_io_read': deque(maxlen=history_size),
            'disk_io_write': deque(maxlen=history_size),
            'net_sent': deque(maxlen=history_size),
            'net_recv': deque(maxlen=history_size),
        }
        # 初始化上次 IO 计数（用于计算速率）
        self._last_disk_io = psutil.disk_io_counters()
        self._last_net_io = psutil.net_io_counters()
        self._last_time = time.time()

    def collect(self):
        """采集一次系统数据"""
        now = time.time()
        elapsed = now - self._last_time

        # CPU 和内存
        self.history['cpu'].append(psutil.cpu_percent())
        self.history['memory'].append(psutil.virtual_memory().percent)

        # 磁盘 IO 速率（字节/秒）
        disk_io = psutil.disk_io_counters()
        if disk_io and self._last_disk_io and elapsed > 0:
            read_speed = (disk_io.read_bytes - self._last_disk_io.read_bytes) / elapsed
            write_speed = (disk_io.write_bytes - self._last_disk_io.write_bytes) / elapsed
            self.history['disk_io_read'].append(read_speed)
            self.history['disk_io_write'].append(write_speed)
            self._last_disk_io = disk_io

        # 网络 IO 速率（字节/秒）
        net_io = psutil.net_io_counters()
        if elapsed > 0:
            sent_speed = (net_io.bytes_sent - self._last_net_io.bytes_sent) / elapsed
            recv_speed = (net_io.bytes_recv - self._last_net_io.bytes_recv) / elapsed
            self.history['net_sent'].append(sent_speed)
            self.history['net_recv'].append(recv_speed)
            self._last_net_io = net_io

        self._last_time = now

    def get_summary(self):
        """获取统计摘要"""
        summary = {}
        for key, values in self.history.items():
            if values:
                summary[key] = {
                    'current': values[-1],
                    'avg': sum(values) / len(values),
                    'max': max(values),
                    'min': min(values),
                }
        return summary

    def check_alerts(self, thresholds=None):
        """检查是否触发告警阈值"""
        if thresholds is None:
            thresholds = {
                'cpu': 80.0,        # CPU 超过 80%
                'memory': 85.0,     # 内存超过 85%
            }

        alerts = []
        summary = self.get_summary()
        for key, threshold in thresholds.items():
            if key in summary and summary[key]['current'] > threshold:
                alerts.append(
                    f"[告警] {key} 当前值 {summary[key]['current']:.1f}% "
                    f"超过阈值 {threshold}%"
                )
        return alerts


def bytes_to_human(n):
    for unit in ['B/s', 'KB/s', 'MB/s', 'GB/s']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}TB/s"


# --- 主循环 ---
if __name__ == '__main__':
    monitor = SystemMonitor(history_size=60)

    print("系统监控已启动（Ctrl+C 退出）")
    print("每秒采集一次，每 10 秒输出摘要...\n")

    try:
        count = 0
        while True:
            monitor.collect()
            count += 1

            # 每 10 秒输出一次摘要
            if count % 10 == 0:
                summary = monitor.get_summary()
                ts = datetime.now().strftime('%H:%M:%S')
                print(f"[{ts}] CPU: {summary['cpu']['current']:.1f}% "
                      f"(avg:{summary['cpu']['avg']:.1f}%) | "
                      f"内存: {summary['memory']['current']:.1f}% | "
                      f"磁盘读: {bytes_to_human(summary.get('disk_io_read', {}).get('current', 0))} | "
                      f"网络发: {bytes_to_human(summary.get('net_sent', {}).get('current', 0))}")

                # 检查告警
                for alert in monitor.check_alerts():
                    print(f"  {alert}")

            time.sleep(1)
    except KeyboardInterrupt:
        print("\n监控已停止。")
```

### 5.2 进程资源跟踪

```python
#!/usr/bin/env python3
"""跟踪指定进程的资源使用情况 —— 对比 pidstat"""

import psutil
import time
import sys

def track_process(pid_or_name, duration=30, interval=2):
    """
    跟踪进程资源使用
    :param pid_or_name: 进程 PID（int）或进程名（str）
    :param duration: 监控持续时间（秒）
    :param interval: 采样间隔（秒）
    """
    # 查找进程
    target_proc = None
    if isinstance(pid_or_name, int):
        try:
            target_proc = psutil.Process(pid_or_name)
        except psutil.NoSuchProcess:
            print(f"进程 PID={pid_or_name} 不存在")
            return
    else:
        for proc in psutil.process_iter(['name']):
            if pid_or_name.lower() in (proc.info['name'] or '').lower():
                target_proc = proc
                break
        if not target_proc:
            print(f"未找到名为 '{pid_or_name}' 的进程")
            return

    print(f"开始跟踪进程: {target_proc.name()} (PID={target_proc.pid})")
    print(f"{'时间':>10} {'CPU%':>6} {'内存%':>6} {'RSS':>10} "
          f"{'线程数':>6} {'FD数':>5} {'状态':<10}")
    print("-" * 60)

    def bytes_to_human(n):
        for unit in ['B', 'KB', 'MB', 'GB']:
            if n < 1024:
                return f"{n:.1f}{unit}"
            n /= 1024
        return f"{n:.1f}TB"

    end_time = time.time() + duration
    while time.time() < end_time:
        try:
            with target_proc.oneshot():
                # oneshot() 优化：一次系统调用获取多个属性
                cpu = target_proc.cpu_percent(interval=0.1)
                mem = target_proc.memory_percent()
                rss = target_proc.memory_info().rss
                threads = target_proc.num_threads()
                try:
                    fds = target_proc.num_fds()
                except AttributeError:
                    fds = 0  # Windows 不支持
                status = target_proc.status()

            ts = time.strftime('%H:%M:%S')
            print(f"{ts:>10} {cpu:>5.1f}% {mem:>5.1f}% "
                  f"{bytes_to_human(rss):>10} {threads:>6} {fds:>5} {status:<10}")

        except psutil.NoSuchProcess:
            print("进程已退出！")
            break

        time.sleep(interval)


if __name__ == '__main__':
    # 用法: python script.py <pid 或进程名>
    if len(sys.argv) > 1:
        target = sys.argv[1]
        try:
            target = int(target)
        except ValueError:
            pass
        track_process(target)
    else:
        print("用法: python script.py <PID 或进程名>")
        print("示例: python script.py 1234")
        print("      python script.py nginx")
```

## 6. SRE 实战案例：类 glances 系统监控脚本

```python
#!/usr/bin/env python3
"""
SRE 实战：类 glances 系统监控面板
功能：
  - 实时显示 CPU/内存/磁盘/网络/进程信息
  - 自动刷新（终端清屏）
  - 告警高亮
  - 支持导出 JSON

对比 Shell: 要实现同样功能需要组合 top + free + df + netstat + awk + sed，
            代码量是 Python 的 3-5 倍，且难以维护
"""

import psutil
import time
import os
import json
from datetime import datetime, timedelta

def bytes_to_human(n):
    """字节转人类可读"""
    for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}PB"

def speed_to_human(n):
    """速率转人类可读"""
    for unit in ['B/s', 'KB/s', 'MB/s', 'GB/s']:
        if n < 1024:
            return f"{n:.1f}{unit}"
        n /= 1024
    return f"{n:.1f}TB/s"

def progress_bar(percent, width=30):
    """生成进度条（文本方式）"""
    filled = int(width * percent / 100)
    bar = '#' * filled + '-' * (width - filled)
    # 超过阈值用不同标记
    if percent > 90:
        marker = '!'  # 危险
    elif percent > 70:
        marker = '*'  # 警告
    else:
        marker = '#'  # 正常
    bar = marker * filled + '-' * (width - filled)
    return f"[{bar}] {percent:.1f}%"

class SREMonitor:
    """SRE 系统监控面板"""

    def __init__(self):
        self._last_net = psutil.net_io_counters()
        self._last_disk = psutil.disk_io_counters()
        self._last_time = time.time()
        self.alerts = []

    def get_system_info(self):
        """获取系统基本信息"""
        boot_time = datetime.fromtimestamp(psutil.boot_time())
        uptime = datetime.now() - boot_time
        uptime_str = str(timedelta(seconds=int(uptime.total_seconds())))

        return {
            'hostname': os.uname().nodename,
            'uptime': uptime_str,
            'boot_time': boot_time.strftime('%Y-%m-%d %H:%M:%S'),
            'users': len(psutil.users()),
        }

    def get_cpu_info(self):
        """采集 CPU 信息"""
        cpu_percent = psutil.cpu_percent(interval=0)
        per_cpu = psutil.cpu_percent(percpu=True)
        load1, load5, load15 = psutil.getloadavg()
        cpu_count = psutil.cpu_count()

        # 告警判断
        if cpu_percent > 90:
            self.alerts.append(f"CPU 使用率 {cpu_percent}% 超过 90%")
        if load1 / cpu_count > 2:
            self.alerts.append(f"系统负载 {load1:.1f} 过高（每核 > 2）")

        return {
            'total_percent': cpu_percent,
            'per_cpu': per_cpu,
            'count': cpu_count,
            'load': (load1, load5, load15),
        }

    def get_memory_info(self):
        """采集内存信息"""
        mem = psutil.virtual_memory()
        swap = psutil.swap_memory()

        if mem.percent > 85:
            self.alerts.append(f"内存使用率 {mem.percent}% 超过 85%")
        if swap.percent > 50:
            self.alerts.append(f"Swap 使用率 {swap.percent}% 超过 50%")

        return {
            'total': mem.total,
            'used': mem.used,
            'available': mem.available,
            'percent': mem.percent,
            'swap_total': swap.total,
            'swap_used': swap.used,
            'swap_percent': swap.percent,
        }

    def get_disk_info(self):
        """采集磁盘信息"""
        disks = []
        for part in psutil.disk_partitions():
            try:
                usage = psutil.disk_usage(part.mountpoint)
                disks.append({
                    'device': part.device,
                    'mountpoint': part.mountpoint,
                    'fstype': part.fstype,
                    'total': usage.total,
                    'used': usage.used,
                    'free': usage.free,
                    'percent': usage.percent,
                })
                if usage.percent > 90:
                    self.alerts.append(
                        f"磁盘 {part.mountpoint} 使用率 {usage.percent}% 超过 90%"
                    )
            except (PermissionError, OSError):
                pass
        return disks

    def get_network_info(self):
        """采集网络信息（含速率计算）"""
        now = time.time()
        elapsed = now - self._last_time
        net = psutil.net_io_counters()

        sent_speed = (net.bytes_sent - self._last_net.bytes_sent) / max(elapsed, 0.1)
        recv_speed = (net.bytes_recv - self._last_net.bytes_recv) / max(elapsed, 0.1)

        self._last_net = net
        self._last_time = now

        return {
            'bytes_sent': net.bytes_sent,
            'bytes_recv': net.bytes_recv,
            'sent_speed': sent_speed,
            'recv_speed': recv_speed,
            'packets_sent': net.packets_sent,
            'packets_recv': net.packets_recv,
            'errin': net.errin,
            'errout': net.errout,
        }

    def get_top_processes(self, n=10):
        """获取资源占用 Top N 进程"""
        procs = []
        for proc in psutil.process_iter(
            ['pid', 'name', 'cpu_percent', 'memory_percent', 'memory_info', 'status']
        ):
            try:
                info = proc.info
                procs.append(info)
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                pass

        # 按 CPU 使用率排序
        procs.sort(key=lambda p: p.get('cpu_percent', 0) or 0, reverse=True)
        return procs[:n]

    def display(self):
        """渲染监控面板到终端"""
        self.alerts = []  # 重置告警

        sys_info = self.get_system_info()
        cpu = self.get_cpu_info()
        mem = self.get_memory_info()
        disks = self.get_disk_info()
        net = self.get_network_info()
        procs = self.get_top_processes()

        # 清屏
        os.system('clear' if os.name != 'nt' else 'cls')

        now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print(f"{'=' * 70}")
        print(f"  SRE 系统监控面板  |  {sys_info['hostname']}  |  {now}")
        print(f"  运行时间: {sys_info['uptime']}  |  在线用户: {sys_info['users']}")
        print(f"{'=' * 70}")

        # CPU
        print(f"\n--- CPU ---")
        print(f"  总使用率: {progress_bar(cpu['total_percent'])}")
        load1, load5, load15 = cpu['load']
        print(f"  负载均值: {load1:.2f} / {load5:.2f} / {load15:.2f} "
              f"（{cpu['count']} 核）")

        # 每核 CPU（紧凑显示）
        cores_per_line = 4
        for i in range(0, len(cpu['per_cpu']), cores_per_line):
            chunk = cpu['per_cpu'][i:i + cores_per_line]
            line = "  " + "  ".join(
                f"核{i+j}:{p:5.1f}%" for j, p in enumerate(chunk)
            )
            print(line)

        # 内存
        print(f"\n--- 内存 ---")
        print(f"  物理内存: {progress_bar(mem['percent'])}  "
              f"({bytes_to_human(mem['used'])} / {bytes_to_human(mem['total'])})")
        print(f"  Swap:     {progress_bar(mem['swap_percent'])}  "
              f"({bytes_to_human(mem['swap_used'])} / {bytes_to_human(mem['swap_total'])})")

        # 磁盘
        print(f"\n--- 磁盘 ---")
        for d in disks:
            print(f"  {d['mountpoint']:<15} {progress_bar(d['percent'])}  "
                  f"({bytes_to_human(d['used'])} / {bytes_to_human(d['total'])})")

        # 网络
        print(f"\n--- 网络 ---")
        print(f"  上传: {speed_to_human(net['sent_speed'])} "
              f"(总计: {bytes_to_human(net['bytes_sent'])})")
        print(f"  下载: {speed_to_human(net['recv_speed'])} "
              f"(总计: {bytes_to_human(net['bytes_recv'])})")
        if net['errin'] > 0 or net['errout'] > 0:
            print(f"  错误: 入={net['errin']}, 出={net['errout']}")

        # Top 进程
        print(f"\n--- Top 进程（按 CPU）---")
        print(f"  {'PID':>7} {'名称':<18} {'CPU%':>6} {'内存%':>6} {'RSS':>10}")
        for p in procs:
            rss = p['memory_info'].rss if p.get('memory_info') else 0
            print(f"  {p['pid']:>7} {(p['name'] or '-')[:18]:<18} "
                  f"{(p.get('cpu_percent') or 0):>5.1f}% "
                  f"{(p.get('memory_percent') or 0):>5.1f}% "
                  f"{bytes_to_human(rss):>10}")

        # 告警
        if self.alerts:
            print(f"\n{'!' * 70}")
            print("  告警信息:")
            for alert in self.alerts:
                print(f"  >>> {alert}")
            print(f"{'!' * 70}")

    def export_json(self, filepath='/tmp/sre_monitor.json'):
        """导出监控数据为 JSON（可供 Prometheus/Grafana 使用）"""
        data = {
            'timestamp': datetime.now().isoformat(),
            'system': self.get_system_info(),
            'cpu': self.get_cpu_info(),
            'memory': self.get_memory_info(),
            'disks': self.get_disk_info(),
            'network': self.get_network_info(),
        }
        # 处理不可序列化的类型
        data['cpu']['load'] = list(data['cpu']['load'])
        with open(filepath, 'w') as f:
            json.dump(data, f, indent=2, default=str)
        return filepath


if __name__ == '__main__':
    monitor = SREMonitor()
    print("SRE 监控面板启动中...（Ctrl+C 退出）")
    time.sleep(1)

    # 先调用一次 cpu_percent 初始化
    psutil.cpu_percent()

    try:
        while True:
            monitor.display()
            # 每 5 次循环导出一次 JSON
            monitor.export_json()
            time.sleep(2)
    except KeyboardInterrupt:
        print("\n\n监控已停止。最后一次数据已导出到 /tmp/sre_monitor.json")
```

## 7. 常见坑点与最佳实践

### Do's (推荐做法)

```python
# 1. 使用 process_iter() 而非 pids() + Process()
# 好：安全迭代，自动处理进程消失
for proc in psutil.process_iter(['pid', 'name', 'cpu_percent']):
    print(proc.info)

# 2. 使用 oneshot() 上下文管理器批量获取进程属性
proc = psutil.Process(1234)
with proc.oneshot():
    proc.name()        # 只触发一次系统调用
    proc.cpu_percent()
    proc.memory_info()

# 3. cpu_percent() 首次调用需要初始化
psutil.cpu_percent()       # 首次调用，返回无意义值
time.sleep(1)
psutil.cpu_percent()       # 第二次调用返回真实值

# 4. 异常处理：进程可能随时消失
try:
    proc = psutil.Process(pid)
    proc.name()
except psutil.NoSuchProcess:
    print("进程已退出")
except psutil.AccessDenied:
    print("权限不足")
except psutil.ZombieProcess:
    print("僵尸进程")
```

### Don'ts (避免做法)

```python
# 1. 不要用 pids() 遍历再逐个创建 Process（竞态条件）
# 坏：
for pid in psutil.pids():
    p = psutil.Process(pid)  # 进程可能在这时已经退出！
    p.name()                 # NoSuchProcess!

# 2. 不要忽略 interval 参数
# 坏：返回无意义的 0.0
print(psutil.cpu_percent())  # 首次调用

# 3. 不要在循环中频繁调用 net_connections()
# 坏：这个函数开销较大
while True:
    conns = psutil.net_connections()  # 每次都扫描 /proc/net/tcp
    time.sleep(0.1)  # 太频繁了！

# 4. 不要假设所有字段在所有平台都存在
# 坏：
mem = psutil.virtual_memory()
print(mem.cached)  # Windows 上没有此属性！
# 好：
print(getattr(mem, 'cached', 0))
```

## 8. 速查表

```
# ==================== psutil 速查表 ====================

# --- 安装 ---
pip install psutil

# --- CPU ---
psutil.cpu_percent(interval=1)         # CPU使用率(%) —— 对比: top
psutil.cpu_percent(percpu=True)        # 每核使用率   —— 对比: mpstat -P ALL
psutil.cpu_count(logical=True)         # 逻辑核心数   —— 对比: nproc
psutil.cpu_count(logical=False)        # 物理核心数   —— 对比: lscpu
psutil.cpu_freq()                      # CPU频率      —— 对比: lscpu
psutil.cpu_times()                     # CPU时间      —— 对比: /proc/stat
psutil.getloadavg()                    # 系统负载     —— 对比: uptime

# --- 内存 ---
psutil.virtual_memory()                # 物理内存     —— 对比: free -m
psutil.swap_memory()                   # Swap内存     —— 对比: swapon -s

# --- 磁盘 ---
psutil.disk_partitions()               # 分区信息     —— 对比: lsblk
psutil.disk_usage('/')                 # 磁盘使用     —— 对比: df -h
psutil.disk_io_counters()              # 磁盘IO       —— 对比: iostat

# --- 网络 ---
psutil.net_io_counters()               # 网络IO       —— 对比: ifconfig
psutil.net_if_addrs()                  # 网络地址     —— 对比: ip addr
psutil.net_connections()               # 网络连接     —— 对比: netstat/ss
psutil.net_if_stats()                  # 接口状态     —— 对比: ip link

# --- 进程 ---
psutil.process_iter(['pid','name'])    # 遍历进程     —— 对比: ps aux
psutil.Process(pid)                    # 获取进程     —— 对比: /proc/PID/
proc.cpu_percent()                     # 进程CPU      —— 对比: pidstat
proc.memory_info()                     # 进程内存     —— 对比: pmap
proc.num_fds()                         # 进程FD数     —— 对比: ls /proc/PID/fd
proc.connections()                     # 进程连接     —— 对比: ss -p
proc.children()                        # 子进程       —— 对比: pstree

# --- 系统 ---
psutil.boot_time()                     # 启动时间     —— 对比: who -b
psutil.users()                         # 登录用户     —— 对比: who
psutil.sensors_temperatures()          # 温度传感器   —— 对比: sensors
```
