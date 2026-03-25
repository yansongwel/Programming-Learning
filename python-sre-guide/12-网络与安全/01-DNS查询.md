# DNS 查询 — 用 dnspython 实现 DNS 监控与管理

## 1. 是什么 & 为什么用它

`dnspython` 是 Python 最完善的 DNS 工具库，可以执行各种 DNS 查询操作。
对 SRE 来说，DNS 是基础设施的关键环节：

- **DNS 解析监控**：检测域名解析是否正常
- **DNS 变更审计**：记录 DNS 记录变化
- **故障排查**：快速定位 DNS 相关问题
- **自动化管理**：批量查询和校验 DNS 配置

对比命令行工具：
- `dig`：功能强大但不可编程
- `nslookup`：交互式，不适合自动化
- `dnspython`：可编程、可集成、可做复杂逻辑

## 2. 安装与环境

```bash
# 安装 dnspython
pip install dnspython

# 验证
python3 -c "import dns.resolver; print('dnspython 可用')"

# 对比 shell 命令
# dig example.com A
# nslookup example.com
# host example.com
```

## 3. 核心概念与原理

### 3.1 DNS 记录类型

| 类型 | 说明 | 示例 |
|------|------|------|
| A | IPv4 地址 | `93.184.216.34` |
| AAAA | IPv6 地址 | `2606:2800:220:1::` |
| CNAME | 别名记录 | `www.example.com → example.com` |
| MX | 邮件服务器 | `mail.example.com priority 10` |
| TXT | 文本记录 | SPF/DKIM/验证信息 |
| NS | 名称服务器 | `ns1.example.com` |
| SOA | 起始授权 | 区域配置信息 |
| SRV | 服务定位 | `_http._tcp.example.com` |
| PTR | 反向解析 | IP → 域名 |

### 3.2 DNS 查询流程

```
客户端 → 本地 DNS 缓存
  ↓ (未命中)
本地 DNS 解析器 → 根域名服务器(.)
  ↓
  → 顶级域服务器(.com)
  ↓
  → 权威 DNS 服务器(example.com)
  ↓
返回结果 → 缓存 → 客户端
```

## 4. 基础用法

### 4.1 常见记录查询

```python
#!/usr/bin/env python3
"""DNS 基础查询示例"""

import dns.resolver
import dns.reversename

# === A 记录查询（IPv4） ===
def query_a(domain: str):
    """查询 A 记录"""
    try:
        answers = dns.resolver.resolve(domain, 'A')
        print(f"\n[A 记录] {domain}:")
        for rdata in answers:
            print(f"  {rdata.address}  TTL={answers.rrset.ttl}")
    except dns.resolver.NXDOMAIN:
        print(f"  域名不存在: {domain}")
    except dns.resolver.NoAnswer:
        print(f"  无 A 记录: {domain}")
    except Exception as e:
        print(f"  查询失败: {e}")

# === AAAA 记录查询（IPv6） ===
def query_aaaa(domain: str):
    """查询 AAAA 记录"""
    try:
        answers = dns.resolver.resolve(domain, 'AAAA')
        print(f"\n[AAAA 记录] {domain}:")
        for rdata in answers:
            print(f"  {rdata.address}  TTL={answers.rrset.ttl}")
    except Exception as e:
        print(f"  无 AAAA 记录或查询失败: {e}")

# === MX 记录查询 ===
def query_mx(domain: str):
    """查询邮件服务器记录"""
    try:
        answers = dns.resolver.resolve(domain, 'MX')
        print(f"\n[MX 记录] {domain}:")
        for rdata in sorted(answers, key=lambda x: x.preference):
            print(f"  优先级 {rdata.preference:>3d}  {rdata.exchange}")
    except Exception as e:
        print(f"  查询失败: {e}")

# === CNAME 记录查询 ===
def query_cname(domain: str):
    """查询别名记录"""
    try:
        answers = dns.resolver.resolve(domain, 'CNAME')
        print(f"\n[CNAME 记录] {domain}:")
        for rdata in answers:
            print(f"  → {rdata.target}")
    except Exception as e:
        print(f"  无 CNAME 记录: {e}")

# === TXT 记录查询 ===
def query_txt(domain: str):
    """查询 TXT 记录（SPF/DKIM 等）"""
    try:
        answers = dns.resolver.resolve(domain, 'TXT')
        print(f"\n[TXT 记录] {domain}:")
        for rdata in answers:
            for txt in rdata.strings:
                print(f"  {txt.decode('utf-8', errors='replace')}")
    except Exception as e:
        print(f"  查询失败: {e}")

# === NS 记录查询 ===
def query_ns(domain: str):
    """查询名称服务器"""
    try:
        answers = dns.resolver.resolve(domain, 'NS')
        print(f"\n[NS 记录] {domain}:")
        for rdata in answers:
            print(f"  {rdata.target}")
    except Exception as e:
        print(f"  查询失败: {e}")

# === 反向解析 ===
def reverse_lookup(ip: str):
    """IP 反向解析为域名"""
    try:
        rev_name = dns.reversename.from_address(ip)
        answers = dns.resolver.resolve(rev_name, 'PTR')
        print(f"\n[反向解析] {ip}:")
        for rdata in answers:
            print(f"  → {rdata.target}")
    except Exception as e:
        print(f"  反向解析失败: {e}")

# === 执行查询 ===
test_domain = "example.com"
print(f"{'='*50}")
print(f"DNS 查询: {test_domain}")
print(f"{'='*50}")

query_a(test_domain)
query_aaaa(test_domain)
query_mx(test_domain)
query_txt(test_domain)
query_ns(test_domain)
reverse_lookup("8.8.8.8")
```

### 4.2 指定 DNS 服务器查询

```python
#!/usr/bin/env python3
"""指定 DNS 服务器查询"""

import dns.resolver

def query_with_nameserver(domain: str, nameserver: str, rdtype: str = 'A'):
    """使用指定的 DNS 服务器查询"""
    resolver = dns.resolver.Resolver()
    resolver.nameservers = [nameserver]
    resolver.timeout = 5
    resolver.lifetime = 10

    try:
        answers = resolver.resolve(domain, rdtype)
        results = []
        for rdata in answers:
            if rdtype == 'A':
                results.append(rdata.address)
            elif rdtype == 'CNAME':
                results.append(str(rdata.target))
            else:
                results.append(str(rdata))
        return results
    except Exception as e:
        return [f"错误: {e}"]

# 对比不同 DNS 服务器的解析结果
domain = "example.com"
nameservers = {
    "Google DNS": "8.8.8.8",
    "Cloudflare DNS": "1.1.1.1",
    "阿里 DNS": "223.5.5.5",
    "腾讯 DNS": "119.29.29.29",
}

print(f"域名: {domain}")
print(f"{'DNS 服务器':<20s} {'IP':>20s} {'结果'}")
print("-" * 60)

for name, ns in nameservers.items():
    results = query_with_nameserver(domain, ns)
    for r in results:
        print(f"{name:<20s} {ns:>20s} {r}")
```

### 4.3 SOA 和区域传输

```python
#!/usr/bin/env python3
"""SOA 查询与区域传输"""

import dns.resolver
import dns.query
import dns.zone

def query_soa(domain: str):
    """查询 SOA 记录（域名权威信息）"""
    try:
        answers = dns.resolver.resolve(domain, 'SOA')
        for rdata in answers:
            print(f"[SOA] {domain}:")
            print(f"  主名称服务器: {rdata.mname}")
            print(f"  管理员邮箱: {rdata.rname}")
            print(f"  序列号: {rdata.serial}")
            print(f"  刷新间隔: {rdata.refresh}s")
            print(f"  重试间隔: {rdata.retry}s")
            print(f"  过期时间: {rdata.expire}s")
            print(f"  最小 TTL: {rdata.minimum}s")
    except Exception as e:
        print(f"SOA 查询失败: {e}")

query_soa("example.com")

# DNS 区域传输（AXFR）— 通常需要授权
def attempt_zone_transfer(domain: str, nameserver: str):
    """尝试 DNS 区域传输"""
    try:
        zone = dns.zone.from_xfr(
            dns.query.xfr(nameserver, domain, timeout=10)
        )
        print(f"\n[区域传输成功] {domain} @ {nameserver}")
        for name, node in zone.nodes.items():
            rdatasets = node.rdatasets
            for rdataset in rdatasets:
                print(f"  {name} {rdataset}")
    except Exception as e:
        print(f"区域传输失败（正常，大多数服务器禁止）: {e}")

# 注意：大多数 DNS 服务器会拒绝区域传输请求
attempt_zone_transfer("example.com", "8.8.8.8")
```

## 5. 进阶用法

### 5.1 批量 DNS 查询

```python
#!/usr/bin/env python3
"""批量 DNS 查询与比较"""

import dns.resolver
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Dict, List, Tuple

def batch_query(domains: List[str], rdtype: str = 'A',
                nameserver: str = None, timeout: float = 5) -> Dict:
    """批量 DNS 查询"""
    resolver = dns.resolver.Resolver()
    if nameserver:
        resolver.nameservers = [nameserver]
    resolver.timeout = timeout
    resolver.lifetime = timeout * 2

    results = {}
    for domain in domains:
        try:
            start = time.time()
            answers = resolver.resolve(domain, rdtype)
            elapsed = (time.time() - start) * 1000  # 毫秒

            records = []
            for rdata in answers:
                if rdtype == 'A':
                    records.append(rdata.address)
                else:
                    records.append(str(rdata))

            results[domain] = {
                'status': 'ok',
                'records': records,
                'ttl': answers.rrset.ttl,
                'latency_ms': round(elapsed, 2),
            }
        except dns.resolver.NXDOMAIN:
            results[domain] = {'status': 'nxdomain', 'records': [], 'latency_ms': 0}
        except dns.resolver.NoAnswer:
            results[domain] = {'status': 'no_answer', 'records': [], 'latency_ms': 0}
        except dns.resolver.Timeout:
            results[domain] = {'status': 'timeout', 'records': [], 'latency_ms': timeout * 1000}
        except Exception as e:
            results[domain] = {'status': f'error: {e}', 'records': [], 'latency_ms': 0}

    return results

# 测试域名列表
domains = [
    "example.com",
    "google.com",
    "github.com",
    "nonexistent-domain-xyz.com",
    "cloudflare.com",
]

print("=== 批量 DNS 查询 ===")
results = batch_query(domains)
for domain, info in results.items():
    records_str = ', '.join(info['records']) if info['records'] else 'N/A'
    print(f"  {domain:40s} [{info['status']:>10s}] {info['latency_ms']:>8.2f}ms  {records_str}")
```

### 5.2 DNS 解析链追踪

```python
#!/usr/bin/env python3
"""DNS 解析链追踪 — 模拟 dig +trace"""

import dns.resolver
import dns.name

def trace_dns(domain: str, rdtype: str = 'A'):
    """追踪 DNS 解析链"""
    print(f"\n=== DNS 解析链追踪: {domain} ({rdtype}) ===\n")

    resolver = dns.resolver.Resolver()

    # 步骤 1: 查询 NS
    try:
        parts = domain.split('.')
        # 逐级查询 NS
        for i in range(len(parts), 0, -1):
            sub_domain = '.'.join(parts[-i:])
            try:
                ns_answers = resolver.resolve(sub_domain, 'NS')
                ns_list = [str(ns.target) for ns in ns_answers]
                print(f"  [{sub_domain}] NS: {', '.join(ns_list[:3])}")
            except Exception:
                pass
    except Exception as e:
        print(f"  NS 查询异常: {e}")

    # 步骤 2: 查 CNAME 链
    try:
        current = domain
        chain = [current]
        for _ in range(10):  # 最多跟踪 10 层
            try:
                cname_answers = resolver.resolve(current, 'CNAME')
                for rdata in cname_answers:
                    current = str(rdata.target).rstrip('.')
                    chain.append(current)
            except (dns.resolver.NoAnswer, dns.resolver.NXDOMAIN):
                break
        if len(chain) > 1:
            print(f"\n  CNAME 链: {' → '.join(chain)}")
    except Exception:
        pass

    # 步骤 3: 最终记录
    try:
        answers = resolver.resolve(domain, rdtype)
        print(f"\n  最终结果 ({rdtype}):")
        for rdata in answers:
            print(f"    {rdata}  TTL={answers.rrset.ttl}")
    except Exception as e:
        print(f"\n  最终查询失败: {e}")

trace_dns("example.com")
```

## 6. SRE 实战案例：DNS 监控告警工具

```python
#!/usr/bin/env python3
"""
SRE 实战：DNS 监控告警工具
功能：
1. 定期检测 DNS 解析是否正常
2. 检测解析结果变化
3. 检测解析延迟
4. 多 DNS 服务器一致性检查
5. 告警通知（控制台输出模拟）
6. 历史记录与报表
"""

import dns.resolver
import time
import json
import hashlib
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional
from dataclasses import dataclass, field, asdict
from collections import defaultdict

# ============================================================
# 数据模型
# ============================================================
@dataclass
class DNSCheckResult:
    """DNS 检查结果"""
    domain: str
    record_type: str
    nameserver: str
    status: str           # ok / nxdomain / timeout / error
    records: List[str]
    ttl: int = 0
    latency_ms: float = 0
    timestamp: str = ""
    records_hash: str = ""

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()
        # 计算记录哈希，用于变更检测
        self.records_hash = hashlib.md5(
            json.dumps(sorted(self.records)).encode()
        ).hexdigest()[:8]

@dataclass
class DNSAlert:
    """DNS 告警"""
    level: str            # warning / critical
    domain: str
    message: str
    timestamp: str = ""
    details: Dict = field(default_factory=dict)

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

# ============================================================
# DNS 检测器
# ============================================================
class DNSChecker:
    """DNS 检测器"""

    def __init__(self, timeout: float = 5.0):
        self.timeout = timeout

    def check(self, domain: str, record_type: str = 'A',
              nameserver: str = None) -> DNSCheckResult:
        """执行单次 DNS 检查"""
        resolver = dns.resolver.Resolver()
        if nameserver:
            resolver.nameservers = [nameserver]
        resolver.timeout = self.timeout
        resolver.lifetime = self.timeout * 2

        ns_name = nameserver or "default"

        try:
            start = time.time()
            answers = resolver.resolve(domain, record_type)
            latency = (time.time() - start) * 1000

            records = []
            for rdata in answers:
                if record_type in ('A', 'AAAA'):
                    records.append(rdata.address)
                elif record_type == 'MX':
                    records.append(f"{rdata.preference} {rdata.exchange}")
                elif record_type == 'CNAME':
                    records.append(str(rdata.target))
                else:
                    records.append(str(rdata))

            return DNSCheckResult(
                domain=domain,
                record_type=record_type,
                nameserver=ns_name,
                status='ok',
                records=sorted(records),
                ttl=answers.rrset.ttl,
                latency_ms=round(latency, 2),
            )

        except dns.resolver.NXDOMAIN:
            return DNSCheckResult(
                domain=domain, record_type=record_type,
                nameserver=ns_name, status='nxdomain', records=[]
            )
        except dns.resolver.Timeout:
            return DNSCheckResult(
                domain=domain, record_type=record_type,
                nameserver=ns_name, status='timeout', records=[],
                latency_ms=self.timeout * 1000
            )
        except Exception as e:
            return DNSCheckResult(
                domain=domain, record_type=record_type,
                nameserver=ns_name, status=f'error:{e}', records=[]
            )

# ============================================================
# DNS 监控器
# ============================================================
class DNSMonitor:
    """DNS 监控告警系统"""

    def __init__(self, config: dict):
        self.config = config
        self.checker = DNSChecker(timeout=config.get('timeout', 5.0))
        self.history: Dict[str, List[DNSCheckResult]] = defaultdict(list)
        self.last_records: Dict[str, str] = {}  # domain → records_hash
        self.alerts: List[DNSAlert] = []

        # 告警阈值
        self.latency_warning = config.get('latency_warning_ms', 200)
        self.latency_critical = config.get('latency_critical_ms', 1000)

    def _generate_alert(self, level: str, domain: str, message: str, **details):
        """生成告警"""
        alert = DNSAlert(
            level=level,
            domain=domain,
            message=message,
            details=details
        )
        self.alerts.append(alert)
        # 模拟告警发送
        icon = "!!" if level == "critical" else "!"
        print(f"  [{icon} {level.upper()}] {domain}: {message}")

    def check_domain(self, domain: str, record_type: str = 'A',
                     expected_records: List[str] = None):
        """检查单个域名"""
        nameservers = self.config.get('nameservers', [None])

        for ns in nameservers:
            result = self.checker.check(domain, record_type, ns)
            key = f"{domain}/{record_type}/{ns or 'default'}"
            self.history[key].append(result)

            # 检查 1: 解析是否成功
            if result.status != 'ok':
                self._generate_alert(
                    'critical', domain,
                    f"DNS 解析失败: {result.status} (NS: {result.nameserver})",
                    record_type=record_type, nameserver=result.nameserver
                )
                continue

            # 检查 2: 延迟是否过高
            if result.latency_ms > self.latency_critical:
                self._generate_alert(
                    'critical', domain,
                    f"DNS 延迟过高: {result.latency_ms:.0f}ms > {self.latency_critical}ms",
                    latency_ms=result.latency_ms
                )
            elif result.latency_ms > self.latency_warning:
                self._generate_alert(
                    'warning', domain,
                    f"DNS 延迟偏高: {result.latency_ms:.0f}ms > {self.latency_warning}ms",
                    latency_ms=result.latency_ms
                )

            # 检查 3: 记录是否变化
            last_hash = self.last_records.get(key)
            if last_hash and last_hash != result.records_hash:
                self._generate_alert(
                    'warning', domain,
                    f"DNS 记录变化: {result.records} (NS: {result.nameserver})",
                    old_hash=last_hash, new_hash=result.records_hash
                )
            self.last_records[key] = result.records_hash

            # 检查 4: 是否匹配预期记录
            if expected_records and set(result.records) != set(expected_records):
                self._generate_alert(
                    'critical', domain,
                    f"DNS 记录与预期不符: 实际={result.records}, 预期={expected_records}",
                    actual=result.records, expected=expected_records
                )

    def check_consistency(self, domain: str, record_type: str = 'A'):
        """检查多 DNS 服务器一致性"""
        nameservers = self.config.get('nameservers', [])
        if len(nameservers) < 2:
            return

        results = {}
        for ns in nameservers:
            result = self.checker.check(domain, record_type, ns)
            if result.status == 'ok':
                results[ns] = result.records_hash

        # 检查是否所有服务器返回一致
        unique_hashes = set(results.values())
        if len(unique_hashes) > 1:
            self._generate_alert(
                'warning', domain,
                f"DNS 记录不一致: 不同 DNS 服务器返回了不同结果",
                server_results={ns: h for ns, h in results.items()}
            )

    def run_checks(self, domains_config: List[dict]):
        """运行所有检查"""
        print(f"\n{'='*60}")
        print(f"DNS 监控检查 - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'='*60}")

        for dc in domains_config:
            domain = dc['domain']
            record_type = dc.get('type', 'A')
            expected = dc.get('expected', None)
            print(f"\n检查: {domain} ({record_type})")

            self.check_domain(domain, record_type, expected)
            self.check_consistency(domain, record_type)

    def report(self) -> str:
        """生成检查报告"""
        lines = [
            f"\n{'='*60}",
            f"DNS 监控报告",
            f"{'='*60}",
            f"检查时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"告警数量: {len(self.alerts)}",
        ]

        # 告警汇总
        if self.alerts:
            lines.append(f"\n--- 告警列表 ---")
            for alert in self.alerts:
                lines.append(f"  [{alert.level.upper()}] {alert.domain}: {alert.message}")

        # 延迟汇总
        lines.append(f"\n--- 延迟汇总 ---")
        for key, results in self.history.items():
            if results:
                latest = results[-1]
                if latest.status == 'ok':
                    lines.append(
                        f"  {key:50s}  {latest.latency_ms:>8.2f}ms  "
                        f"TTL={latest.ttl}  Records={len(latest.records)}"
                    )

        return '\n'.join(lines)

# ============================================================
# 演示
# ============================================================
def main():
    # 监控配置
    config = {
        'nameservers': ['8.8.8.8', '1.1.1.1'],
        'timeout': 5.0,
        'latency_warning_ms': 100,
        'latency_critical_ms': 500,
    }

    # 要监控的域名列表
    domains_config = [
        {'domain': 'example.com', 'type': 'A'},
        {'domain': 'example.com', 'type': 'MX'},
        {'domain': 'example.com', 'type': 'NS'},
        {'domain': 'google.com', 'type': 'A'},
        {'domain': 'nonexistent-domain-test-xyz.com', 'type': 'A'},
    ]

    # 执行监控
    monitor = DNSMonitor(config)
    monitor.run_checks(domains_config)

    # 生成报告
    report = monitor.report()
    print(report)

    # 保存报告
    report_path = Path("/tmp/dns_monitor_report.txt")
    report_path.write_text(report)
    print(f"\n报告已保存: {report_path}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **超时设置**：默认超时可能太长，生产环境建议设置 `timeout=5`
2. **缓存问题**：`dns.resolver` 默认会缓存，多次查询可能得到过期结果
3. **CNAME 追踪**：查询 A 记录时如果遇到 CNAME，dnspython 会自动追踪
4. **IPv6**：某些环境不支持 IPv6，AAAA 查询可能超时
5. **并发安全**：`Resolver` 对象不是线程安全的，每线程创建独立实例

### 最佳实践

```python
# 1. 设置合理超时
resolver = dns.resolver.Resolver()
resolver.timeout = 5
resolver.lifetime = 10

# 2. 异常处理要全面
# NXDOMAIN, NoAnswer, Timeout, NoNameservers

# 3. 使用多个 DNS 服务器做冗余
resolver.nameservers = ['8.8.8.8', '1.1.1.1']

# 4. 预编译查询名称
# 5. 大批量查询用线程池并发
```

## 8. 速查表

```python
import dns.resolver

# === 基础查询 ===
dns.resolver.resolve('example.com', 'A')      # A 记录
dns.resolver.resolve('example.com', 'AAAA')   # IPv6
dns.resolver.resolve('example.com', 'MX')     # 邮件
dns.resolver.resolve('example.com', 'TXT')    # 文本
dns.resolver.resolve('example.com', 'NS')     # 名称服务器
dns.resolver.resolve('example.com', 'SOA')    # SOA
dns.resolver.resolve('example.com', 'CNAME')  # 别名

# === 指定 DNS 服务器 ===
resolver = dns.resolver.Resolver()
resolver.nameservers = ['8.8.8.8']
resolver.resolve('example.com', 'A')

# === 反向解析 ===
import dns.reversename
rev = dns.reversename.from_address('8.8.8.8')
dns.resolver.resolve(rev, 'PTR')

# === 异常处理 ===
# dns.resolver.NXDOMAIN      域名不存在
# dns.resolver.NoAnswer       无对应记录
# dns.resolver.Timeout        查询超时
# dns.resolver.NoNameservers  无可用 DNS 服务器

# === 对比 shell ===
# dig example.com A              → resolve('example.com', 'A')
# dig @8.8.8.8 example.com      → resolver.nameservers = ['8.8.8.8']
# dig -x 8.8.8.8                → reverse_lookup
# nslookup example.com          → resolve()
```
