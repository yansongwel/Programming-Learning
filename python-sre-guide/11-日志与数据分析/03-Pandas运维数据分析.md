# Pandas 运维数据分析 — 用 DataFrame 分析日志与监控数据

## 1. 是什么 & 为什么用它

Pandas 是 Python 的数据分析核心库，提供了 DataFrame（二维表格）数据结构。
对 SRE 来说，Pandas 是分析运维数据的利器：

- 分析 Nginx/应用日志 → 找出异常模式
- 分析监控数据 → 容量规划
- 读取 CSV/JSON/Excel → 快速生成报表
- 时间序列分析 → 趋势预测

对比其他工具：
- **Excel**：数据量受限，无法自动化
- **awk/sort/uniq**：功能单一，链式命令难维护
- **SQL**：需要数据库，小数据集不方便
- **Pandas**：内存中处理，API 丰富，可编程

## 2. 安装与环境

```bash
# 安装 pandas 和可视化库
pip install pandas matplotlib openpyxl

# 验证
python3 -c "import pandas; print(pandas.__version__)"
python3 -c "import matplotlib; print(matplotlib.__version__)"
```

## 3. 核心概念与原理

### 3.1 两大数据结构

```
Series（一维）：带标签的数组
  索引   值
  0      10.5
  1      20.3
  2      15.8

DataFrame（二维）：带行列标签的表格
        host       cpu    memory    disk
  0     web-01     45.2   60.1      70.5
  1     web-02     78.9   55.3      45.2
  2     db-01      30.1   82.7      90.3
```

### 3.2 核心操作

```
读取 → 清洗 → 筛选 → 分组聚合 → 排序 → 可视化/导出
```

## 4. 基础用法

### 4.1 创建 DataFrame

```python
#!/usr/bin/env python3
"""Pandas DataFrame 基础"""

import pandas as pd
from datetime import datetime, timedelta
import random

# === 从字典创建 ===
servers = pd.DataFrame({
    'hostname': ['web-01', 'web-02', 'web-03', 'db-01', 'db-02', 'cache-01'],
    'role': ['web', 'web', 'web', 'database', 'database', 'cache'],
    'cpu_percent': [45.2, 78.9, 30.1, 82.7, 55.3, 25.8],
    'memory_percent': [60.1, 55.3, 82.7, 90.3, 70.2, 45.6],
    'disk_percent': [70.5, 45.2, 90.3, 85.1, 62.8, 30.4],
    'status': ['running', 'running', 'warning', 'critical', 'running', 'running'],
})

print("=== 服务器概况 ===")
print(servers)
print(f"\n形状: {servers.shape}")  # (行数, 列数)
print(f"列名: {list(servers.columns)}")
print(f"数据类型:\n{servers.dtypes}")

# === 基本统计 ===
print(f"\n=== 数值列统计 ===")
print(servers.describe())

# === 选择数据 ===
# 选择列
print(f"\n主机名列表:\n{servers['hostname']}")

# 选择多列
print(f"\n主机和CPU:\n{servers[['hostname', 'cpu_percent']]}")

# 条件筛选
high_cpu = servers[servers['cpu_percent'] > 70]
print(f"\nCPU > 70% 的服务器:\n{high_cpu}")

# 多条件筛选
critical = servers[(servers['cpu_percent'] > 60) & (servers['memory_percent'] > 80)]
print(f"\nCPU>60% 且 内存>80%:\n{critical}")

# loc（按标签）和 iloc（按位置）
print(f"\n第一行: {servers.iloc[0].to_dict()}")
print(f"\nweb 角色服务器:\n{servers.loc[servers['role'] == 'web']}")
```

### 4.2 读取外部数据

```python
#!/usr/bin/env python3
"""Pandas 读取各种格式的数据"""

import pandas as pd
import json
from pathlib import Path

# === 写入测试 CSV ===
csv_content = """timestamp,hostname,cpu,memory,disk,status
2024-01-15 10:00:00,web-01,45.2,60.1,70.5,ok
2024-01-15 10:00:00,web-02,78.9,55.3,45.2,ok
2024-01-15 10:00:00,db-01,92.3,88.5,85.1,critical
2024-01-15 10:05:00,web-01,50.1,62.3,70.5,ok
2024-01-15 10:05:00,web-02,65.4,58.7,45.3,ok
2024-01-15 10:05:00,db-01,85.6,90.2,85.2,warning
2024-01-15 10:10:00,web-01,42.8,59.5,70.6,ok
2024-01-15 10:10:00,web-02,71.2,56.1,45.3,ok
2024-01-15 10:10:00,db-01,78.9,85.3,85.3,warning
"""

Path("/tmp/metrics.csv").write_text(csv_content.strip())

# 读取 CSV
df = pd.read_csv(
    "/tmp/metrics.csv",
    parse_dates=['timestamp'],  # 自动解析日期列
)
print("=== CSV 数据 ===")
print(df.head())
print(f"\n数据类型:\n{df.dtypes}")

# === 读取 JSON ===
json_data = [
    {"host": "web-01", "metrics": {"cpu": 45, "mem": 60}},
    {"host": "web-02", "metrics": {"cpu": 78, "mem": 55}},
]
Path("/tmp/metrics.json").write_text(json.dumps(json_data))

df_json = pd.json_normalize(json_data)
print(f"\n=== JSON 数据（展平） ===")
print(df_json)

# === 导出 ===
# 导出 CSV
df.to_csv("/tmp/output.csv", index=False, encoding='utf-8')

# 导出 Excel
df.to_excel("/tmp/output.xlsx", index=False, sheet_name='监控数据')

# 导出 JSON
df.to_json("/tmp/output.json", orient='records', force_ascii=False, indent=2, date_format='iso')

print("\n文件已导出到 /tmp/")
```

### 4.3 数据清洗与转换

```python
#!/usr/bin/env python3
"""数据清洗与转换"""

import pandas as pd
import numpy as np

# 模拟脏数据
data = pd.DataFrame({
    'hostname': ['web-01', 'web-02', None, 'db-01', 'web-01', 'db-01'],
    'cpu': [45.2, 78.9, 50.0, None, 45.2, 88.5],
    'memory': ['60.1%', '55.3%', '82.7%', '90.3%', '60.1%', '92.1%'],
    'status': ['running', 'RUNNING', 'Running', 'stopped', 'running', 'stopped'],
    'uptime': ['30d 5h', '15d 2h', '7d 12h', '60d 1h', '30d 5h', '60d 1h'],
})

print("=== 原始数据 ===")
print(data)

# 1. 处理缺失值
print(f"\n缺失值统计:\n{data.isnull().sum()}")
data['hostname'] = data['hostname'].fillna('unknown')  # 填充缺失
data['cpu'] = data['cpu'].fillna(data['cpu'].mean())    # 用均值填充

# 2. 数据类型转换
data['memory'] = data['memory'].str.replace('%', '').astype(float)

# 3. 统一大小写
data['status'] = data['status'].str.lower()

# 4. 删除重复行
print(f"\n重复行: {data.duplicated().sum()}")
data = data.drop_duplicates()

# 5. 提取信息
data['uptime_days'] = data['uptime'].str.extract(r'(\d+)d').astype(int)

# 6. 新增计算列
data['health_score'] = 100 - (data['cpu'] * 0.4 + data['memory'] * 0.3)

print(f"\n=== 清洗后的数据 ===")
print(data)
```

## 5. 进阶用法

### 5.1 分组聚合

```python
#!/usr/bin/env python3
"""分组聚合分析"""

import pandas as pd
import numpy as np

# 模拟服务器监控数据
np.random.seed(42)
n = 100
data = pd.DataFrame({
    'timestamp': pd.date_range('2024-01-15', periods=n, freq='5min'),
    'hostname': np.random.choice(['web-01', 'web-02', 'db-01', 'cache-01'], n),
    'role': '',  # 后面根据 hostname 填充
    'cpu': np.random.uniform(20, 95, n),
    'memory': np.random.uniform(30, 90, n),
    'requests': np.random.randint(100, 5000, n),
    'errors': np.random.randint(0, 50, n),
})

# 根据 hostname 设置 role
role_map = {'web-01': 'web', 'web-02': 'web', 'db-01': 'database', 'cache-01': 'cache'}
data['role'] = data['hostname'].map(role_map)

# === 按主机分组统计 ===
print("=== 按主机统计 ===")
host_stats = data.groupby('hostname').agg({
    'cpu': ['mean', 'max', 'min', 'std'],
    'memory': ['mean', 'max'],
    'requests': 'sum',
    'errors': 'sum',
}).round(2)
print(host_stats)

# === 按角色分组 ===
print("\n=== 按角色统计 ===")
role_stats = data.groupby('role').agg({
    'cpu': 'mean',
    'memory': 'mean',
    'requests': 'sum',
    'errors': 'sum',
}).round(2)
print(role_stats)

# === 错误率计算 ===
data['error_rate'] = (data['errors'] / data['requests'] * 100).round(2)
print("\n=== 错误率 Top 10 ===")
print(data.nlargest(10, 'error_rate')[['timestamp', 'hostname', 'requests', 'errors', 'error_rate']])

# === 自定义聚合函数 ===
def p95(series):
    """计算 P95"""
    return np.percentile(series, 95)

def p99(series):
    """计算 P99"""
    return np.percentile(series, 99)

print("\n=== CPU 分位数 ===")
cpu_percentiles = data.groupby('hostname')['cpu'].agg(['mean', 'median', p95, p99]).round(2)
print(cpu_percentiles)
```

### 5.2 时间序列分析

```python
#!/usr/bin/env python3
"""时间序列分析"""

import pandas as pd
import numpy as np

# 生成时间序列数据（模拟一天的请求量）
np.random.seed(42)
timestamps = pd.date_range('2024-01-15', periods=288, freq='5min')  # 一天288个5分钟
# 模拟日间高峰
base_load = 1000
hourly_pattern = np.array([0.3, 0.2, 0.1, 0.1, 0.2, 0.3, 0.6, 0.9, 1.0, 0.95,
                           0.9, 0.85, 0.8, 0.85, 0.9, 0.95, 1.0, 0.9, 0.8, 0.7,
                           0.6, 0.5, 0.4, 0.35])

requests = []
for ts in timestamps:
    hour = ts.hour
    noise = np.random.normal(0, 50)
    requests.append(int(base_load * hourly_pattern[hour] + noise))

df = pd.DataFrame({
    'timestamp': timestamps,
    'requests': requests,
    'latency_ms': np.random.exponential(50, len(timestamps)),
    'errors': np.random.poisson(5, len(timestamps)),
})
df.set_index('timestamp', inplace=True)

# === 重采样（时间聚合） ===
print("=== 按小时聚合 ===")
hourly = df.resample('1h').agg({
    'requests': 'sum',
    'latency_ms': ['mean', 'max'],
    'errors': 'sum',
}).round(2)
print(hourly.head(12))

# === 滑动窗口 ===
print("\n=== 请求量 1小时滑动平均 ===")
df['requests_rolling_mean'] = df['requests'].rolling(window=12).mean()  # 12个5分钟 = 1小时
df['requests_rolling_std'] = df['requests'].rolling(window=12).std()

# 异常检测（超过 2 个标准差）
df['is_anomaly'] = abs(df['requests'] - df['requests_rolling_mean']) > 2 * df['requests_rolling_std']
anomalies = df[df['is_anomaly'] == True]
print(f"检测到 {len(anomalies)} 个异常点")
if len(anomalies) > 0:
    print(anomalies[['requests', 'requests_rolling_mean']].head())

# === 同比/环比 ===
print("\n=== 环比变化（每小时 vs 上一小时） ===")
hourly_requests = df['requests'].resample('1h').sum()
hourly_requests_pct = hourly_requests.pct_change() * 100
print(hourly_requests_pct.round(2).head(12))
```

## 6. SRE 实战案例：Nginx 访问日志分析报表

```python
#!/usr/bin/env python3
"""
SRE 实战：Nginx 访问日志分析报表
功能：
1. 解析 Nginx 日志为 DataFrame
2. 流量分析（PV/UV/带宽）
3. 状态码分析
4. 慢请求分析
5. 热门 URL/IP 分析
6. 时间趋势分析
7. 生成文本报表
"""

import pandas as pd
import numpy as np
import re
from datetime import datetime
from collections import defaultdict
from pathlib import Path

# ============================================================
# Nginx 日志解析
# ============================================================
NGINX_PATTERN = re.compile(
    r'(?P<ip>\S+)\s+'
    r'\S+\s+'
    r'(?P<user>\S+)\s+'
    r'\[(?P<timestamp>[^\]]+)\]\s+'
    r'"(?P<method>\S+)\s+(?P<url>\S+)\s+(?P<protocol>[^"]+)"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<size>\d+)\s+'
    r'"(?P<referer>[^"]*)"\s+'
    r'"(?P<user_agent>[^"]*)"'
    r'(?:\s+(?P<response_time>[\d.]+))?'  # 可选的响应时间
)

def parse_nginx_log(log_lines: list) -> pd.DataFrame:
    """解析 Nginx 日志为 DataFrame"""
    records = []
    for line in log_lines:
        match = NGINX_PATTERN.match(line.strip())
        if match:
            data = match.groupdict()
            data['status'] = int(data['status'])
            data['size'] = int(data['size'])
            data['response_time'] = float(data.get('response_time') or 0)
            try:
                data['datetime'] = datetime.strptime(
                    data['timestamp'], '%d/%b/%Y:%H:%M:%S %z'
                )
            except (ValueError, TypeError):
                data['datetime'] = None
            records.append(data)

    df = pd.DataFrame(records)
    if not df.empty and 'datetime' in df.columns:
        df['datetime'] = pd.to_datetime(df['datetime'], utc=True)
        df['hour'] = df['datetime'].dt.hour
        df['date'] = df['datetime'].dt.date
    return df

# ============================================================
# 报表生成器
# ============================================================
class NginxReportGenerator:
    """Nginx 日志分析报表生成器"""

    def __init__(self, df: pd.DataFrame):
        self.df = df
        self.report_lines = []

    def _add_section(self, title: str, content: str):
        """添加报表章节"""
        self.report_lines.append(f"\n{'='*60}")
        self.report_lines.append(f" {title}")
        self.report_lines.append(f"{'='*60}")
        self.report_lines.append(content)

    def analyze_overview(self):
        """总览分析"""
        total_requests = len(self.df)
        unique_ips = self.df['ip'].nunique()
        total_bandwidth = self.df['size'].sum()
        avg_response_time = self.df['response_time'].mean()
        error_count = len(self.df[self.df['status'] >= 400])
        error_rate = error_count / total_requests * 100 if total_requests else 0

        content = f"""
  总请求数: {total_requests:,}
  独立 IP 数: {unique_ips:,}
  总带宽: {total_bandwidth / 1024 / 1024:.2f} MB
  平均响应时间: {avg_response_time:.2f}s
  错误请求数: {error_count:,} ({error_rate:.2f}%)
"""
        self._add_section("总览", content)

    def analyze_status_codes(self):
        """状态码分析"""
        status_counts = self.df['status'].value_counts().sort_index()
        total = len(self.df)

        lines = []
        for status, count in status_counts.items():
            pct = count / total * 100
            bar = '#' * int(pct / 2)
            lines.append(f"  {status}: {count:>8,} ({pct:>6.2f}%) {bar}")

        # 按状态码大类统计
        self.df['status_group'] = self.df['status'].apply(lambda x: f"{str(x)[0]}xx")
        group_counts = self.df['status_group'].value_counts().sort_index()
        lines.append(f"\n  --- 按大类 ---")
        for group, count in group_counts.items():
            pct = count / total * 100
            lines.append(f"  {group}: {count:>8,} ({pct:>6.2f}%)")

        self._add_section("状态码分布", '\n'.join(lines))

    def analyze_top_urls(self, n: int = 10):
        """热门 URL 分析"""
        top_urls = self.df['url'].value_counts().head(n)
        lines = []
        for url, count in top_urls.items():
            lines.append(f"  {count:>8,}  {url}")
        self._add_section(f"Top {n} URL", '\n'.join(lines))

    def analyze_top_ips(self, n: int = 10):
        """Top IP 分析"""
        top_ips = self.df['ip'].value_counts().head(n)
        lines = []
        for ip, count in top_ips.items():
            # 计算该 IP 的错误率
            ip_data = self.df[self.df['ip'] == ip]
            ip_errors = len(ip_data[ip_data['status'] >= 400])
            ip_error_rate = ip_errors / count * 100
            lines.append(f"  {ip:>18s}  {count:>8,} 次  错误率: {ip_error_rate:.1f}%")
        self._add_section(f"Top {n} IP", '\n'.join(lines))

    def analyze_slow_requests(self, threshold: float = 1.0, n: int = 10):
        """慢请求分析"""
        slow = self.df[self.df['response_time'] > threshold].sort_values(
            'response_time', ascending=False
        ).head(n)

        lines = [f"  响应时间 > {threshold}s 的请求: {len(self.df[self.df['response_time'] > threshold])} 个\n"]
        for _, row in slow.iterrows():
            lines.append(f"  {row['response_time']:>8.3f}s  {row['method']}  {row['url']}  [{row['status']}]")

        # 响应时间分位数
        lines.append(f"\n  --- 响应时间分位数 ---")
        for p in [50, 75, 90, 95, 99]:
            val = np.percentile(self.df['response_time'], p)
            lines.append(f"  P{p}: {val:.3f}s")

        self._add_section("慢请求分析", '\n'.join(lines))

    def analyze_error_details(self, n: int = 10):
        """错误详情分析"""
        errors = self.df[self.df['status'] >= 400]
        if len(errors) == 0:
            self._add_section("错误详情", "  无错误请求")
            return

        # 错误 URL 排行
        error_urls = errors.groupby(['url', 'status']).size().sort_values(ascending=False).head(n)
        lines = ["  --- 错误 URL Top ---"]
        for (url, status), count in error_urls.items():
            lines.append(f"  [{status}] {count:>6,}  {url}")

        self._add_section("错误详情", '\n'.join(lines))

    def analyze_traffic_by_hour(self):
        """按小时流量分析"""
        if 'hour' not in self.df.columns:
            return

        hourly = self.df.groupby('hour').agg({
            'ip': 'count',
            'size': 'sum',
            'response_time': 'mean',
        }).round(2)
        hourly.columns = ['请求数', '带宽(bytes)', '平均响应时间(s)']

        lines = [f"  {'时间':>4s}  {'请求数':>8s}  {'带宽(MB)':>10s}  {'平均响应(s)':>12s}"]
        lines.append(f"  {'-'*40}")
        for hour, row in hourly.iterrows():
            bw_mb = row['带宽(bytes)'] / 1024 / 1024
            lines.append(f"  {hour:>02d}:00  {int(row['请求数']):>8,}  {bw_mb:>10.2f}  {row['平均响应时间(s)']:>12.3f}")

        self._add_section("按小时流量", '\n'.join(lines))

    def analyze_user_agents(self, n: int = 10):
        """User-Agent 分析"""
        # 简单分类
        def classify_ua(ua):
            ua_lower = ua.lower()
            if 'bot' in ua_lower or 'spider' in ua_lower or 'crawler' in ua_lower:
                return '爬虫'
            elif 'python' in ua_lower or 'curl' in ua_lower or 'wget' in ua_lower:
                return '脚本/工具'
            elif 'mobile' in ua_lower or 'android' in ua_lower or 'iphone' in ua_lower:
                return '移动端'
            else:
                return '桌面端'

        self.df['ua_type'] = self.df['user_agent'].apply(classify_ua)
        ua_dist = self.df['ua_type'].value_counts()

        lines = []
        for ua_type, count in ua_dist.items():
            pct = count / len(self.df) * 100
            lines.append(f"  {ua_type}: {count:>8,} ({pct:.1f}%)")

        self._add_section("客户端类型分布", '\n'.join(lines))

    def generate_report(self) -> str:
        """生成完整报表"""
        self.report_lines = [
            f"{'#'*60}",
            f"#  Nginx 访问日志分析报表",
            f"#  生成时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"{'#'*60}",
        ]

        self.analyze_overview()
        self.analyze_status_codes()
        self.analyze_top_urls()
        self.analyze_top_ips()
        self.analyze_slow_requests()
        self.analyze_error_details()
        self.analyze_traffic_by_hour()
        self.analyze_user_agents()

        return '\n'.join(self.report_lines)


# ============================================================
# 演示
# ============================================================
def generate_sample_logs(n: int = 500) -> list:
    """生成模拟 Nginx 日志"""
    import random
    random.seed(42)

    ips = ['192.168.1.100', '10.0.0.55', '172.16.0.1', '10.0.0.33',
           '192.168.1.200', '10.0.0.77', '172.16.0.2', '192.168.2.50']
    urls = ['/api/health', '/api/users', '/api/orders', '/static/app.js',
            '/static/style.css', '/api/login', '/api/servers', '/not-found',
            '/admin/dashboard', '/api/metrics']
    methods = ['GET', 'POST', 'PUT', 'DELETE']
    statuses = [200, 200, 200, 200, 200, 301, 304, 400, 403, 404, 500, 502, 503]
    user_agents = [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
        'Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X)',
        'python-requests/2.28.0',
        'curl/7.68.0',
        'Googlebot/2.1 (+http://www.google.com/bot.html)',
    ]

    logs = []
    base_time = datetime(2024, 1, 15, 0, 0, 0)
    for i in range(n):
        ts = base_time + pd.Timedelta(seconds=i * 10)
        ip = random.choice(ips)
        method = random.choice(methods)
        url = random.choice(urls)
        status = random.choice(statuses)
        size = random.randint(0, 50000)
        response_time = random.expovariate(1/0.5)  # 平均 0.5s
        ua = random.choice(user_agents)

        log_line = (
            f'{ip} - - [{ts.strftime("%d/%b/%Y:%H:%M:%S")} +0800] '
            f'"{method} {url} HTTP/1.1" {status} {size} "-" "{ua}" {response_time:.3f}'
        )
        logs.append(log_line)
    return logs


def main():
    # 生成模拟日志
    print("生成模拟 Nginx 日志...")
    sample_logs = generate_sample_logs(500)

    # 解析为 DataFrame
    print("解析日志...")
    df = parse_nginx_log(sample_logs)
    print(f"成功解析 {len(df)} 条记录")

    # 生成报表
    reporter = NginxReportGenerator(df)
    report = reporter.generate_report()
    print(report)

    # 保存报表
    report_path = Path("/tmp/nginx_report.txt")
    report_path.write_text(report)
    print(f"\n报表已保存: {report_path}")

    # 导出分析数据
    df.to_csv("/tmp/nginx_parsed.csv", index=False)
    print(f"解析数据已导出: /tmp/nginx_parsed.csv")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **内存问题**：大文件一次读入内存，用 `chunksize` 参数分块读取
2. **SettingWithCopyWarning**：链式赋值会触发警告，用 `.loc` 赋值
3. **时区问题**：`pd.to_datetime` 默认无时区，注意指定 `utc=True`
4. **类型推断**：读取 CSV 时数字字符串可能被转为数字，用 `dtype` 指定
5. **索引混淆**：`loc` 用标签，`iloc` 用位置，混用会出错

### 最佳实践

```python
# 1. 大文件分块读取
for chunk in pd.read_csv('huge.csv', chunksize=10000):
    process(chunk)

# 2. 安全赋值
df.loc[df['cpu'] > 90, 'alert'] = True  # 正确
# df[df['cpu'] > 90]['alert'] = True      # 可能失败

# 3. 明确指定数据类型
df = pd.read_csv('data.csv', dtype={'port': int, 'ip': str})

# 4. 选择需要的列（减少内存）
df = pd.read_csv('data.csv', usecols=['timestamp', 'cpu', 'memory'])
```

## 8. 速查表

```python
import pandas as pd

# === 读取 ===
pd.read_csv('file.csv')
pd.read_json('file.json')
pd.read_excel('file.xlsx')

# === 查看 ===
df.head(10)           # 前10行
df.shape              # (行, 列)
df.dtypes             # 列类型
df.describe()         # 统计摘要
df.info()             # 内存信息

# === 选择 ===
df['col']             # 单列
df[['col1','col2']]   # 多列
df[df['col'] > 50]    # 条件筛选
df.loc[0:5, 'col']    # 按标签
df.iloc[0:5, 0:3]     # 按位置

# === 分组聚合 ===
df.groupby('col').agg({'val': ['mean', 'max', 'sum']})
df.groupby('col').size()   # 计数

# === 时间序列 ===
df.resample('1H').mean()         # 按小时聚合
df['col'].rolling(12).mean()     # 滑动平均
df['col'].pct_change()           # 环比变化

# === 排序 ===
df.sort_values('col', ascending=False)
df.nlargest(10, 'col')           # Top N

# === 导出 ===
df.to_csv('out.csv', index=False)
df.to_json('out.json', orient='records')
df.to_excel('out.xlsx', index=False)
```
