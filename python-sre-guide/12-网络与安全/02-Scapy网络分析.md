# Scapy 网络分析 — 用 Python 构造数据包与网络诊断

## 1. 是什么 & 为什么用它

Scapy 是 Python 最强大的网络数据包操纵工具库，可以构造、发送、嗅探和解析网络数据包。
对 SRE 来说，Scapy 是网络故障排查和安全分析的利器：

- **构造自定义数据包**：测试防火墙规则、负载均衡
- **网络嗅探**：替代 tcpdump 进行编程化抓包分析
- **协议分析**：深入理解网络通信细节
- **连通性测试**：TCP/UDP/ICMP 多维度检测

对比命令行工具：
- `tcpdump`：只能抓包和简单过滤，不能构造包
- `wireshark`：GUI 工具，不适合自动化
- `nmap`：扫描功能强但不够灵活
- `scapy`：构造 + 发送 + 嗅探 + 分析，全能且可编程

## 2. 安装与环境

```bash
# 安装 scapy
pip install scapy

# 验证（注意：部分功能需要 root 权限）
python3 -c "from scapy.all import IP, TCP; print('scapy 可用')"

# 可选依赖（增强功能）
# pip install matplotlib  # 可视化
# pip install cryptography  # TLS 支持

# 注意：发送原始数据包需要 root 权限
# sudo python3 script.py
```

## 3. 核心概念与原理

### 3.1 OSI 模型与 Scapy 的层

```
应用层 (L7)  → DNS, HTTP, etc.
传输层 (L4)  → TCP(), UDP()
网络层 (L3)  → IP(), IPv6(), ICMP()
数据链路层 (L2) → Ether()

Scapy 用 / 运算符叠加协议层：
  packet = Ether() / IP() / TCP() / "data"
```

### 3.2 核心操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 发送 L3 包 | `send()` | 发送网络层包（自动填充 L2） |
| 发送 L2 包 | `sendp()` | 发送数据链路层包 |
| 发送并接收 | `sr()` / `sr1()` | 发送并等待响应 |
| 嗅探 | `sniff()` | 抓取网络数据包 |
| 构造包 | `IP()/TCP()` | 用 `/` 叠加协议层 |

## 4. 基础用法

### 4.1 构造数据包

```python
#!/usr/bin/env python3
"""Scapy 数据包构造基础"""

from scapy.all import (
    IP, TCP, UDP, ICMP, DNS, DNSQR, Ether,
    Raw, ls, hexdump
)

# === 构造 IP 包 ===
ip_pkt = IP(dst="8.8.8.8")
print("=== IP 包 ===")
ip_pkt.show()  # 展示包的所有字段

# === 构造 ICMP 包（ping） ===
ping = IP(dst="8.8.8.8") / ICMP()
print("\n=== ICMP Ping 包 ===")
ping.show()

# === 构造 TCP 包 ===
# SYN 包（TCP 三次握手第一步）
syn = IP(dst="93.184.216.34") / TCP(dport=80, flags="S")
print("\n=== TCP SYN 包 ===")
syn.show()

# === 构造 UDP 包 ===
udp_pkt = IP(dst="8.8.8.8") / UDP(dport=53) / b"\x00\x01"
print("\n=== UDP 包 ===")
udp_pkt.show()

# === 构造 DNS 查询包 ===
dns_query = (
    IP(dst="8.8.8.8") /
    UDP(dport=53) /
    DNS(rd=1, qd=DNSQR(qname="example.com"))
)
print("\n=== DNS 查询包 ===")
dns_query.show()

# === 查看字段 ===
print("\n=== TCP 所有字段 ===")
ls(TCP)

# === 构造带 payload 的包 ===
http_req = (
    IP(dst="93.184.216.34") /
    TCP(dport=80, flags="PA") /
    Raw(load="GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
)

# === 查看包的十六进制转储 ===
print("\n=== 十六进制转储 ===")
hexdump(ping)

# === 包的摘要 ===
print(f"\n包摘要: {ping.summary()}")
print(f"包大小: {len(bytes(ping))} bytes")
```

### 4.2 发送与接收

```python
#!/usr/bin/env python3
"""Scapy 发送和接收数据包
注意：需要 root 权限运行
"""

from scapy.all import (
    IP, TCP, UDP, ICMP, DNS, DNSQR,
    send, sr, sr1, RandShort
)
import sys

# 检查权限
import os
if os.geteuid() != 0:
    print("警告：发送原始数据包需要 root 权限")
    print("请使用: sudo python3 script.py")
    print("\n以下为模拟演示（不实际发送）...\n")

# === ICMP Ping（sr1 = 发送一个包并接收一个响应） ===
def ping_host(target: str, timeout: int = 2) -> dict:
    """Ping 一个主机"""
    import time
    pkt = IP(dst=target) / ICMP()
    start = time.time()

    try:
        reply = sr1(pkt, timeout=timeout, verbose=0)
        elapsed = (time.time() - start) * 1000

        if reply:
            return {
                'target': target,
                'status': 'alive',
                'ttl': reply[IP].ttl,
                'latency_ms': round(elapsed, 2),
                'src': reply[IP].src,
            }
        else:
            return {
                'target': target,
                'status': 'timeout',
                'latency_ms': timeout * 1000,
            }
    except Exception as e:
        return {
            'target': target,
            'status': f'error: {e}',
            'latency_ms': 0,
        }

# === TCP SYN 扫描（检测端口是否开放） ===
def tcp_syn_check(target: str, port: int, timeout: int = 2) -> dict:
    """TCP SYN 端口检测"""
    pkt = IP(dst=target) / TCP(sport=RandShort(), dport=port, flags="S")
    try:
        reply = sr1(pkt, timeout=timeout, verbose=0)
        if reply is None:
            return {'port': port, 'status': 'filtered', 'detail': '无响应（可能被防火墙过滤）'}
        elif reply.haslayer(TCP):
            flags = reply[TCP].flags
            if flags == 0x12:  # SYN-ACK
                # 发送 RST 关闭连接
                rst = IP(dst=target) / TCP(sport=pkt[TCP].sport, dport=port, flags="R")
                send(rst, verbose=0)
                return {'port': port, 'status': 'open', 'detail': '端口开放'}
            elif flags == 0x14:  # RST-ACK
                return {'port': port, 'status': 'closed', 'detail': '端口关闭'}
        return {'port': port, 'status': 'unknown', 'detail': str(reply.summary())}
    except Exception as e:
        return {'port': port, 'status': 'error', 'detail': str(e)}

# === DNS 查询 ===
def dns_query(domain: str, nameserver: str = "8.8.8.8") -> dict:
    """DNS 查询"""
    pkt = (
        IP(dst=nameserver) /
        UDP(dport=53) /
        DNS(rd=1, qd=DNSQR(qname=domain))
    )
    try:
        reply = sr1(pkt, timeout=3, verbose=0)
        if reply and reply.haslayer(DNS):
            dns_reply = reply[DNS]
            answers = []
            for i in range(dns_reply.ancount):
                rr = dns_reply.an[i] if dns_reply.ancount > 0 else None
                if rr:
                    answers.append(str(rr.rdata))
            return {'domain': domain, 'answers': answers, 'status': 'ok'}
    except Exception as e:
        return {'domain': domain, 'answers': [], 'status': f'error: {e}'}

# 演示（模拟输出）
print("=== Ping 检测示例 ===")
print("  ping_host('8.8.8.8') → {'target': '8.8.8.8', 'status': 'alive', 'ttl': 117, 'latency_ms': 12.5}")

print("\n=== TCP 端口检测示例 ===")
print("  tcp_syn_check('93.184.216.34', 80) → {'port': 80, 'status': 'open'}")
print("  tcp_syn_check('93.184.216.34', 8080) → {'port': 8080, 'status': 'closed'}")

print("\n=== DNS 查询示例 ===")
print("  dns_query('example.com') → {'domain': 'example.com', 'answers': ['93.184.216.34']}")
```

### 4.3 网络嗅探

```python
#!/usr/bin/env python3
"""Scapy 网络嗅探（需要 root 权限）"""

from scapy.all import sniff, IP, TCP, UDP, DNS, wrpcap, rdpcap
from collections import defaultdict
import json

# ============================================================
# 嗅探回调函数
# ============================================================
class PacketAnalyzer:
    """数据包分析器"""

    def __init__(self):
        self.packet_count = 0
        self.protocol_stats = defaultdict(int)
        self.ip_stats = defaultdict(int)
        self.port_stats = defaultdict(int)
        self.dns_queries = []

    def process_packet(self, packet):
        """处理每个捕获的数据包"""
        self.packet_count += 1

        # 统计协议
        if packet.haslayer(TCP):
            self.protocol_stats['TCP'] += 1
            self.port_stats[packet[TCP].dport] += 1
        elif packet.haslayer(UDP):
            self.protocol_stats['UDP'] += 1
            self.port_stats[packet[UDP].dport] += 1

        # 统计 IP
        if packet.haslayer(IP):
            self.ip_stats[packet[IP].src] += 1
            self.ip_stats[packet[IP].dst] += 1

        # 记录 DNS 查询
        if packet.haslayer(DNS) and packet[DNS].qd:
            qname = packet[DNS].qd.qname.decode('utf-8', errors='replace')
            self.dns_queries.append(qname)

    def report(self):
        """输出分析报告"""
        print(f"\n{'='*50}")
        print(f"抓包分析报告")
        print(f"{'='*50}")
        print(f"总包数: {self.packet_count}")

        print(f"\n协议分布:")
        for proto, count in sorted(self.protocol_stats.items(), key=lambda x: -x[1]):
            print(f"  {proto}: {count}")

        print(f"\nTop IP 地址:")
        for ip, count in sorted(self.ip_stats.items(), key=lambda x: -x[1])[:10]:
            print(f"  {ip}: {count}")

        print(f"\nTop 目标端口:")
        for port, count in sorted(self.port_stats.items(), key=lambda x: -x[1])[:10]:
            service = {80: 'HTTP', 443: 'HTTPS', 53: 'DNS', 22: 'SSH'}.get(port, '')
            print(f"  {port} ({service}): {count}")

        if self.dns_queries:
            print(f"\nDNS 查询:")
            for q in set(self.dns_queries):
                print(f"  {q}")

# === 使用方式示例 ===
print("=== Scapy 嗅探使用方式 ===\n")

print("""
# 基础嗅探（需要 root 权限）
packets = sniff(count=100, timeout=30)  # 抓100个包或30秒

# 带过滤器的嗅探（BPF 语法，同 tcpdump）
packets = sniff(filter="tcp port 80", count=50)   # 只抓 HTTP
packets = sniff(filter="host 10.0.0.1", count=50) # 只抓特定主机
packets = sniff(filter="udp port 53", count=50)    # 只抓 DNS

# 带回调的嗅探
analyzer = PacketAnalyzer()
packets = sniff(
    filter="ip",
    count=100,
    prn=analyzer.process_packet,  # 每个包调用回调
    store=True                     # 同时存储包
)
analyzer.report()

# 保存和加载 pcap 文件
wrpcap('/tmp/capture.pcap', packets)          # 保存
loaded_packets = rdpcap('/tmp/capture.pcap')  # 加载（可用 wireshark 打开）

# 指定网卡嗅探
packets = sniff(iface="eth0", count=100)

# 对比 tcpdump:
# tcpdump -i eth0 -c 100 'tcp port 80'
# → sniff(iface="eth0", count=100, filter="tcp port 80")
""")

# 模拟分析器输出
analyzer = PacketAnalyzer()
analyzer.packet_count = 1000
analyzer.protocol_stats = {'TCP': 750, 'UDP': 200, 'ICMP': 50}
analyzer.ip_stats = {'10.0.0.1': 300, '10.0.0.2': 250, '8.8.8.8': 150}
analyzer.port_stats = {443: 500, 80: 200, 53: 150, 22: 50}
analyzer.dns_queries = ['example.com.', 'google.com.', 'api.github.com.']
analyzer.report()
```

## 5. 进阶用法

### 5.1 Traceroute

```python
#!/usr/bin/env python3
"""Scapy 实现 Traceroute"""

from scapy.all import IP, TCP, ICMP, sr1, RandShort

def traceroute(target: str, max_hops: int = 30, timeout: int = 2):
    """TCP/ICMP Traceroute 实现"""
    print(f"\ntraceroute to {target}, max {max_hops} hops\n")

    for ttl in range(1, max_hops + 1):
        # 使用 ICMP
        pkt = IP(dst=target, ttl=ttl) / ICMP()
        reply = sr1(pkt, timeout=timeout, verbose=0)

        if reply is None:
            print(f"  {ttl:>2d}  * * * (超时)")
        elif reply.type == 11:  # TTL exceeded
            print(f"  {ttl:>2d}  {reply.src}")
        elif reply.type == 0:   # Echo reply（到达目标）
            print(f"  {ttl:>2d}  {reply.src} (到达目标)")
            break
        else:
            print(f"  {ttl:>2d}  {reply.src} (type={reply.type})")

# 模拟输出
print("=== Traceroute 示例输出 ===")
print("""
traceroute to 8.8.8.8, max 30 hops

   1  192.168.1.1
   2  10.0.0.1
   3  172.16.0.1
   4  * * * (超时)
   5  209.85.248.1
   6  8.8.8.8 (到达目标)
""")
```

### 5.2 ARP 扫描

```python
#!/usr/bin/env python3
"""Scapy ARP 扫描 — 发现局域网设备"""

from scapy.all import Ether, ARP, srp
import ipaddress

def arp_scan(network: str, timeout: int = 3):
    """扫描局域网中的活跃主机
    Args:
        network: 网段，如 "192.168.1.0/24"
        timeout: 超时时间（秒）
    """
    print(f"\nARP 扫描: {network}")
    print(f"{'IP 地址':<18s} {'MAC 地址':<20s}")
    print("-" * 40)

    # 构造 ARP 广播包
    arp_request = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=network)

    try:
        # 发送并接收响应
        answered, unanswered = srp(arp_request, timeout=timeout, verbose=0)

        devices = []
        for sent, received in answered:
            devices.append({
                'ip': received.psrc,
                'mac': received.hwsrc,
            })
            print(f"{received.psrc:<18s} {received.hwsrc:<20s}")

        print(f"\n发现 {len(devices)} 台活跃设备")
        return devices
    except Exception as e:
        print(f"扫描失败: {e}")
        return []

# 模拟输出
print("=== ARP 扫描示例输出 ===")
print("""
ARP 扫描: 192.168.1.0/24
IP 地址            MAC 地址
----------------------------------------
192.168.1.1        aa:bb:cc:dd:ee:01
192.168.1.10       aa:bb:cc:dd:ee:0a
192.168.1.20       aa:bb:cc:dd:ee:14
192.168.1.100      aa:bb:cc:dd:ee:64

发现 4 台活跃设备
""")
```

## 6. SRE 实战案例：网络连通性诊断工具

```python
#!/usr/bin/env python3
"""
SRE 实战：网络连通性诊断工具
功能：
1. ICMP Ping 检测
2. TCP 端口检测
3. UDP 端口检测（DNS 为例）
4. MTU 检测
5. 综合网络诊断报告
"""

import time
import json
from datetime import datetime
from typing import Dict, List, Optional
from dataclasses import dataclass, field, asdict

# 注意：实际使用需要 root 权限并导入 scapy
# from scapy.all import IP, TCP, UDP, ICMP, DNS, DNSQR, sr1, send, RandShort

# ============================================================
# 数据模型
# ============================================================
@dataclass
class ProbeResult:
    """探测结果"""
    target: str
    probe_type: str       # icmp / tcp / udp / dns
    status: str           # ok / timeout / error / closed / filtered
    latency_ms: float = 0
    detail: str = ""
    timestamp: str = ""

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

# ============================================================
# 网络诊断器（模拟实现，实际需 root + scapy）
# ============================================================
class NetworkDiagnostic:
    """网络连通性诊断工具"""

    def __init__(self, timeout: float = 3.0):
        self.timeout = timeout
        self.results: List[ProbeResult] = []

    def icmp_ping(self, target: str, count: int = 3) -> List[ProbeResult]:
        """ICMP Ping 检测"""
        results = []
        print(f"  [ICMP] Ping {target} ...")

        for i in range(count):
            # 实际代码:
            # pkt = IP(dst=target) / ICMP()
            # start = time.time()
            # reply = sr1(pkt, timeout=self.timeout, verbose=0)
            # latency = (time.time() - start) * 1000

            # 模拟结果
            import random
            latency = random.uniform(5, 50)
            result = ProbeResult(
                target=target,
                probe_type='icmp',
                status='ok',
                latency_ms=round(latency, 2),
                detail=f"ttl=64 seq={i+1}"
            )
            results.append(result)
            self.results.append(result)

        # 统计
        latencies = [r.latency_ms for r in results if r.status == 'ok']
        if latencies:
            avg = sum(latencies) / len(latencies)
            loss = (count - len(latencies)) / count * 100
            print(f"    → 平均延迟: {avg:.2f}ms, 丢包率: {loss:.1f}%")

        return results

    def tcp_check(self, target: str, ports: List[int]) -> List[ProbeResult]:
        """TCP 端口检测"""
        results = []
        print(f"  [TCP] 检测 {target} 端口 {ports}")

        for port in ports:
            # 实际代码:
            # pkt = IP(dst=target) / TCP(sport=RandShort(), dport=port, flags="S")
            # reply = sr1(pkt, timeout=self.timeout, verbose=0)

            # 模拟结果
            import random
            status = random.choice(['open', 'open', 'open', 'closed', 'filtered'])
            latency = random.uniform(5, 30) if status == 'open' else 0

            service_map = {22: 'SSH', 80: 'HTTP', 443: 'HTTPS', 3306: 'MySQL',
                          5432: 'PostgreSQL', 6379: 'Redis', 8080: 'HTTP-Alt'}
            service = service_map.get(port, 'Unknown')

            result = ProbeResult(
                target=target,
                probe_type='tcp',
                status=status,
                latency_ms=round(latency, 2),
                detail=f"port={port} service={service}"
            )
            results.append(result)
            self.results.append(result)
            icon = "+" if status == 'open' else "-" if status == 'closed' else "?"
            print(f"    [{icon}] {port}/{service}: {status} ({latency:.1f}ms)")

        return results

    def dns_check(self, domain: str, nameservers: List[str] = None) -> List[ProbeResult]:
        """DNS 解析检测"""
        nameservers = nameservers or ['8.8.8.8', '1.1.1.1']
        results = []
        print(f"  [DNS] 查询 {domain}")

        for ns in nameservers:
            # 实际代码:
            # pkt = IP(dst=ns) / UDP(dport=53) / DNS(rd=1, qd=DNSQR(qname=domain))
            # reply = sr1(pkt, timeout=self.timeout, verbose=0)

            # 模拟结果
            import random
            latency = random.uniform(10, 100)
            result = ProbeResult(
                target=ns,
                probe_type='dns',
                status='ok',
                latency_ms=round(latency, 2),
                detail=f"query={domain} answer=93.184.216.34"
            )
            results.append(result)
            self.results.append(result)
            print(f"    NS={ns}: {latency:.1f}ms → 93.184.216.34")

        return results

    def mtu_detect(self, target: str, start_mtu: int = 1500) -> Optional[int]:
        """MTU 检测（Path MTU Discovery）"""
        print(f"  [MTU] 检测到 {target} 的路径 MTU")

        # 二分查找 MTU
        low, high = 68, start_mtu  # 最小 IP 包 68 字节
        result_mtu = low

        while low <= high:
            mid = (low + high) // 2
            # 实际代码:
            # pkt = IP(dst=target, flags="DF") / ICMP() / Raw(load="X" * (mid - 28))
            # reply = sr1(pkt, timeout=2, verbose=0)

            # 模拟：假设 MTU 为 1472
            if mid <= 1472:
                result_mtu = mid
                low = mid + 1
            else:
                high = mid - 1

        print(f"    → 路径 MTU: {result_mtu + 28} bytes（payload: {result_mtu} bytes）")
        return result_mtu + 28

    def full_diagnostic(self, target: str, ports: List[int] = None,
                       domain: str = None) -> str:
        """完整网络诊断"""
        ports = ports or [22, 80, 443]
        report_lines = [
            f"{'='*60}",
            f"网络连通性诊断报告",
            f"{'='*60}",
            f"目标: {target}",
            f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"",
        ]

        # 1. ICMP Ping
        report_lines.append("--- ICMP Ping 检测 ---")
        ping_results = self.icmp_ping(target, count=5)
        ok_pings = [r for r in ping_results if r.status == 'ok']
        if ok_pings:
            latencies = [r.latency_ms for r in ok_pings]
            report_lines.append(f"  状态: 可达")
            report_lines.append(f"  平均延迟: {sum(latencies)/len(latencies):.2f}ms")
            report_lines.append(f"  最小/最大: {min(latencies):.2f}/{max(latencies):.2f}ms")
            report_lines.append(f"  丢包率: {(5-len(ok_pings))/5*100:.1f}%")
        else:
            report_lines.append(f"  状态: 不可达")

        # 2. TCP 端口检测
        report_lines.append(f"\n--- TCP 端口检测 ---")
        tcp_results = self.tcp_check(target, ports)
        for r in tcp_results:
            icon = "OPEN" if r.status == 'open' else r.status.upper()
            report_lines.append(f"  {r.detail}: [{icon}]")

        # 3. DNS 检测
        if domain:
            report_lines.append(f"\n--- DNS 解析检测 ---")
            self.dns_check(domain)

        # 4. MTU 检测
        report_lines.append(f"\n--- MTU 检测 ---")
        mtu = self.mtu_detect(target)
        report_lines.append(f"  Path MTU: {mtu}")

        # 汇总
        report_lines.append(f"\n{'='*60}")
        report_lines.append(f"诊断完成: 共执行 {len(self.results)} 项检测")

        return '\n'.join(report_lines)


# ============================================================
# 演示
# ============================================================
def main():
    diag = NetworkDiagnostic(timeout=3.0)

    # 完整诊断
    report = diag.full_diagnostic(
        target="93.184.216.34",
        ports=[22, 80, 443, 8080, 3306],
        domain="example.com"
    )
    print(f"\n{report}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **权限问题**：发送原始数据包需要 root/管理员权限
2. **防火墙干扰**：本地或网络防火墙可能阻止原始包
3. **性能问题**：大量包发送时注意速率控制，避免被误判为攻击
4. **合法性**：扫描他人网络前务必获得授权
5. **超时设置**：网络延迟高的环境需要适当增大 timeout

### 最佳实践

```python
# 1. 始终设置合理超时
reply = sr1(pkt, timeout=3, verbose=0)

# 2. 使用 verbose=0 减少输出
send(pkt, verbose=0)

# 3. 限制发送速率
send(pkts, inter=0.1)  # 每包间隔 0.1 秒

# 4. 保存抓包结果用于后续分析
wrpcap('/tmp/capture.pcap', packets)

# 5. 使用 BPF 过滤器减少抓包量
sniff(filter="tcp port 80", count=100)
```

## 8. 速查表

```python
from scapy.all import *

# === 构造包 ===
IP(dst="10.0.0.1") / TCP(dport=80, flags="S")
IP(dst="10.0.0.1") / ICMP()
IP(dst="8.8.8.8") / UDP(dport=53) / DNS(qd=DNSQR(qname="example.com"))

# === 发送 ===
send(pkt)                    # 发送 L3（自动填 L2）
sendp(pkt)                   # 发送 L2
sr(pkt)                      # 发送并接收（多包）
sr1(pkt)                     # 发送并接收一个响应

# === 嗅探 ===
sniff(count=100)                          # 抓100个包
sniff(filter="tcp port 80", count=50)    # BPF 过滤
sniff(iface="eth0", prn=callback)        # 回调处理
sniff(timeout=30)                         # 超时

# === pcap ===
wrpcap('file.pcap', packets)             # 保存
packets = rdpcap('file.pcap')            # 加载

# === 包操作 ===
pkt.show()                               # 显示字段
pkt.summary()                            # 摘要
hexdump(pkt)                             # 十六进制
ls(TCP)                                  # 查看协议字段

# === TCP 标志 ===
# S=SYN, A=ACK, F=FIN, R=RST, P=PSH
# SA=SYN-ACK, FA=FIN-ACK, PA=PSH-ACK

# === 对比 shell ===
# tcpdump -i eth0 'tcp port 80'  → sniff(iface="eth0", filter="tcp port 80")
# ping 8.8.8.8                    → sr1(IP(dst="8.8.8.8")/ICMP())
# traceroute 8.8.8.8              → traceroute("8.8.8.8")
```
