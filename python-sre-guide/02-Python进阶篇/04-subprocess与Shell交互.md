# subprocess 详解 — 用 Python 替代复杂 Shell 脚本的核心武器

## 1. 是什么 & 为什么用它

### 为什么不直接写 Shell？

Shell 脚本在以下场景容易出问题：

```bash
#!/bin/bash
# Shell 的痛点示例

# 1. 错误处理困难
result=$(ssh user@host "df -h" 2>/dev/null)
if [ $? -ne 0 ]; then
    # 无法区分是 SSH 失败还是 df 命令失败
    echo "出错了"
fi

# 2. 字符串处理笨拙
log_line="2024-01-01 12:00:00 ERROR [app] disk full on /data"
# 提取字段需要 awk/sed/grep 组合拳
timestamp=$(echo "$log_line" | awk '{print $1" "$2}')
level=$(echo "$log_line" | awk '{print $3}')

# 3. 复杂逻辑难以维护
# 嵌套 if/for/case + 管道 + 子shell 变量丢失...
```

### Python subprocess 的优势

```python
import subprocess

# 1. 精确的错误处理
try:
    result = subprocess.run(
        ["ssh", "user@host", "df -h"],
        capture_output=True, text=True,
        timeout=30, check=True
    )
    print(result.stdout)
except subprocess.TimeoutExpired:
    print("SSH连接超时")
except subprocess.CalledProcessError as e:
    print(f"命令失败，退出码: {e.returncode}")
    print(f"错误输出: {e.stderr}")

# 2. 字符串处理强大
log_line = "2024-01-01 12:00:00 ERROR [app] disk full on /data"
parts = log_line.split()
timestamp = f"{parts[0]} {parts[1]}"
level = parts[2]

# 3. 完整的编程语言特性：类、函数、异常、数据结构...
```

---

## 2. 核心概念与原理

### 2.1 subprocess 模块层次

```
subprocess.run()        ← 推荐！覆盖 90% 的场景
    │
    ├── 内部使用 ──→ subprocess.Popen()  ← 需要精细控制时使用
    │
    └── 替代了 ──→ os.system()    ← 已废弃，不要用
                   os.popen()     ← 已废弃，不要用
                   commands.*     ← 已废弃，不要用
```

### 2.2 关键参数速查

| 参数 | 含义 | Shell 类比 |
|------|------|-----------|
| `capture_output=True` | 捕获标准输出和错误 | `$(command)` 或 `command 2>&1` |
| `text=True` | 输出为字符串（不是bytes） | 默认行为 |
| `check=True` | 命令失败时抛异常 | `set -e` |
| `timeout=30` | 超时（秒） | `timeout 30 command` |
| `shell=True` | 通过shell执行（有安全风险） | 直接在shell中执行 |
| `cwd="/path"` | 工作目录 | `cd /path && command` |
| `env={...}` | 环境变量 | `VAR=value command` |
| `stdin=PIPE` | 向命令输入数据 | `echo "data" \| command` |

---

## 3. 基础用法

### 3.1 subprocess.run() — 最常用

```python
#!/usr/bin/env python3
"""
subprocess.run() 基础用法
"""
import subprocess


# === 1. 最简单的调用 ===
# 对比 Shell: ls -la /tmp
result = subprocess.run(["ls", "-la", "/tmp"])
print(f"退出码: {result.returncode}")


# === 2. 捕获输出 ===
# 对比 Shell: output=$(df -h)
result = subprocess.run(
    ["df", "-h"],
    capture_output=True,  # 捕获 stdout 和 stderr
    text=True             # 输出为字符串（默认是 bytes）
)
print("标准输出:")
print(result.stdout)
print(f"退出码: {result.returncode}")


# === 3. 检查命令是否成功 ===
# 对比 Shell: set -e（遇到错误就退出）
try:
    result = subprocess.run(
        ["ls", "/不存在的目录"],
        capture_output=True, text=True,
        check=True  # 退出码非0时抛出 CalledProcessError
    )
except subprocess.CalledProcessError as e:
    print(f"命令失败！退出码: {e.returncode}")
    print(f"错误输出: {e.stderr}")


# === 4. 超时控制 ===
# 对比 Shell: timeout 5 ping -c 100 127.0.0.1
try:
    result = subprocess.run(
        ["ping", "-c", "100", "127.0.0.1"],
        capture_output=True, text=True,
        timeout=3  # 3秒超时
    )
except subprocess.TimeoutExpired as e:
    print(f"命令超时！已运行 {e.timeout} 秒")


# === 5. 指定工作目录 ===
# 对比 Shell: cd /tmp && ls
result = subprocess.run(
    ["ls", "-la"],
    capture_output=True, text=True,
    cwd="/tmp"  # 在 /tmp 目录下执行
)
print(f"/tmp 目录内容:\n{result.stdout[:200]}")


# === 6. 自定义环境变量 ===
# 对比 Shell: MY_VAR=hello env | grep MY_VAR
import os
my_env = os.environ.copy()  # 复制当前环境
my_env["MY_VAR"] = "hello"
my_env["LANG"] = "en_US.UTF-8"

result = subprocess.run(
    ["env"],
    capture_output=True, text=True,
    env=my_env
)
# 在输出中查找我们设置的变量
for line in result.stdout.splitlines():
    if "MY_VAR" in line:
        print(f"找到: {line}")
```

### 3.2 向命令输入数据

```python
#!/usr/bin/env python3
"""
向子进程输入数据
"""
import subprocess


# === 方式1: input 参数 ===
# 对比 Shell: echo "hello world" | wc -w
result = subprocess.run(
    ["wc", "-w"],
    input="hello world\n",   # 向 stdin 输入数据
    capture_output=True, text=True
)
print(f"单词数: {result.stdout.strip()}")  # 2


# === 方式2: 多行输入 ===
# 对比 Shell: cat <<EOF | sort
#   banana
#   apple
#   cherry
# EOF
data = """banana
apple
cherry
date
elderberry"""

result = subprocess.run(
    ["sort"],
    input=data,
    capture_output=True, text=True
)
print(f"排序结果:\n{result.stdout}")


# === 方式3: 传入二进制数据 ===
# 对比 Shell: gzip -c < file
binary_data = b"Hello, this is binary data for compression test\n" * 100
result = subprocess.run(
    ["gzip", "-c"],
    input=binary_data,
    capture_output=True
    # 注意：不加 text=True，输入输出都是 bytes
)
print(f"原始大小: {len(binary_data)}")
print(f"压缩后大小: {len(result.stdout)}")
```

### 3.3 管道链

```python
#!/usr/bin/env python3
"""
管道链：多个命令串联
对比 Shell: ps aux | grep python | grep -v grep | awk '{print $2}'
"""
import subprocess


# === 方式1: Python 中间处理（推荐） ===
# Shell: ps aux | grep python | awk '{print $2, $11}'
result = subprocess.run(
    ["ps", "aux"],
    capture_output=True, text=True
)
# 用 Python 过滤和处理（比 grep + awk 更灵活）
for line in result.stdout.splitlines():
    if "python" in line.lower() and "grep" not in line:
        parts = line.split()
        pid = parts[1]
        cmd = " ".join(parts[10:])
        print(f"PID: {pid}, CMD: {cmd}")


# === 方式2: Popen 管道链（模拟 Shell 管道） ===
# Shell: cat /etc/passwd | grep -v nologin | sort | head -5

# 第一个命令
p1 = subprocess.Popen(
    ["cat", "/etc/passwd"],
    stdout=subprocess.PIPE
)
# 第二个命令，stdin 接第一个的 stdout
p2 = subprocess.Popen(
    ["grep", "-v", "nologin"],
    stdin=p1.stdout,
    stdout=subprocess.PIPE
)
p1.stdout.close()  # 允许 p1 收到 SIGPIPE

# 第三个命令
p3 = subprocess.Popen(
    ["sort"],
    stdin=p2.stdout,
    stdout=subprocess.PIPE
)
p2.stdout.close()

# 第四个命令
p4 = subprocess.Popen(
    ["head", "-5"],
    stdin=p3.stdout,
    stdout=subprocess.PIPE,
    text=True
)
p3.stdout.close()

output, _ = p4.communicate()
print(f"\n管道链结果:\n{output}")


# === 方式3: shell=True（简单但有安全风险） ===
# 仅用于交互式脚本，不要用于处理用户输入！
result = subprocess.run(
    "cat /etc/passwd | grep -v nologin | sort | head -5",
    shell=True,
    capture_output=True, text=True
)
print(f"\nshell=True 结果:\n{result.stdout}")
```

---

## 4. shell=True 的安全风险

```python
#!/usr/bin/env python3
"""
shell=True 的安全风险演示
"""
import subprocess


# === 危险！命令注入 ===
# 假设 username 来自用户输入
username = "admin; rm -rf /"  # 恶意输入！

# 错误：shell=True + 字符串拼接 = 命令注入
# subprocess.run(f"id {username}", shell=True)
# 实际执行: id admin; rm -rf /  ← 灾难！
print(f"危险命令: id {username}")
print("如果用 shell=True，分号后面的内容也会执行！")


# === 安全：用列表传参（推荐） ===
# username 作为一个完整参数传给 id 命令
# 即使包含分号也只是作为参数的一部分
result = subprocess.run(
    ["id", username],
    capture_output=True, text=True
)
print(f"\n安全方式（列表传参）:")
print(f"退出码: {result.returncode}")
print(f"stderr: {result.stderr.strip()}")  # id: 'admin; rm -rf /': no such user


# === 什么时候可以用 shell=True ===
# 1. 命令是硬编码的（没有外部输入）
# 2. 需要 Shell 特性（通配符、管道、重定向等）
# 3. 仅用于运维脚本，不对外暴露

# 如果必须使用 shell=True，用 shlex.quote 转义
import shlex
safe_username = shlex.quote(username)
print(f"\nshlex.quote 转义后: {safe_username}")
# 输出: 'admin; rm -rf /'  ← 整个字符串被单引号包裹
```

---

## 5. 对比 os.system 和 os.popen

```python
#!/usr/bin/env python3
"""
不要再用 os.system / os.popen，用 subprocess 替代
"""
import os
import subprocess


# === os.system（已废弃） ===
# 问题：1. 无法捕获输出  2. shell=True 安全风险  3. 返回值是退出码*256
print("=== os.system（不推荐）===")
ret = os.system("echo hello")  # 无法获取输出
print(f"返回值: {ret}")

# 替代：
result = subprocess.run(["echo", "hello"], capture_output=True, text=True)
print(f"输出: {result.stdout.strip()}")


# === os.popen（已废弃） ===
# 问题：1. 无法获取退出码  2. 无法捕获 stderr  3. shell=True
print("\n=== os.popen（不推荐）===")
output = os.popen("echo hello").read()
print(f"输出: {output.strip()}")

# 替代：
result = subprocess.run(["echo", "hello"], capture_output=True, text=True)
print(f"输出: {result.stdout.strip()}")
print(f"退出码: {result.returncode}")


# === 迁移对照表 ===
print("""
迁移对照表:
os.system("cmd")           → subprocess.run(["cmd"])
os.system("cmd > /dev/null") → subprocess.run(["cmd"], capture_output=True)
os.popen("cmd").read()     → subprocess.run(["cmd"], capture_output=True, text=True).stdout
os.popen("cmd").close()    → subprocess.run(["cmd"]).returncode
""")
```

---

## 6. 进阶用法：Popen

```python
#!/usr/bin/env python3
"""
Popen 进阶：需要精细控制子进程时使用
"""
import subprocess
import sys
import time


# === 1. 实时读取输出（不缓冲） ===
# 对比 Shell: tail -f /var/log/syslog
print("=== 实时读取子进程输出 ===")

# 用 ping 演示实时输出
proc = subprocess.Popen(
    ["ping", "-c", "3", "127.0.0.1"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    bufsize=1  # 行缓冲
)

# 实时逐行读取
while True:
    line = proc.stdout.readline()
    if not line and proc.poll() is not None:
        break
    if line:
        print(f"  [实时] {line.strip()}")

print(f"  退出码: {proc.returncode}")


# === 2. 同时读取 stdout 和 stderr ===
print("\n=== communicate() 方式 ===")
proc = subprocess.Popen(
    ["ls", "-la", "/tmp", "/不存在"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

# communicate() 一次性读取所有输出（不会死锁）
stdout, stderr = proc.communicate(timeout=10)
print(f"  标准输出: {stdout[:100]}...")
print(f"  标准错误: {stderr.strip()}")
print(f"  退出码: {proc.returncode}")


# === 3. 发送信号 ===
print("\n=== 发送信号控制子进程 ===")
import signal

proc = subprocess.Popen(
    ["sleep", "100"],
    stdout=subprocess.PIPE
)
print(f"  子进程PID: {proc.pid}")

# 检查是否在运行
print(f"  运行中: {proc.poll() is None}")

# 发送 SIGTERM（优雅终止）
proc.terminate()  # 等价于 kill -15 pid
proc.wait(timeout=5)
print(f"  终止后退出码: {proc.returncode}")

# 也可以用:
# proc.kill()   → SIGKILL（强制杀死）
# proc.send_signal(signal.SIGUSR1)  → 自定义信号
```

---

## 7. SRE 实战案例：Python 替代复杂 Shell 脚本（批量日志清理）

```python
#!/usr/bin/env python3
"""
SRE 实战：用 Python 替代复杂的日志清理 Shell 脚本

原始 Shell 脚本可能长这样：
    #!/bin/bash
    LOG_DIRS="/var/log/app /var/log/nginx /data/logs"
    MAX_AGE=7
    MAX_SIZE_MB=100

    for dir in $LOG_DIRS; do
        if [ ! -d "$dir" ]; then
            continue
        fi
        # 清理超过7天的日志
        find "$dir" -name "*.log" -mtime +$MAX_AGE -exec rm -f {} \;
        find "$dir" -name "*.log.gz" -mtime +$MAX_AGE -exec rm -f {} \;

        # 压缩超过100MB的日志
        find "$dir" -name "*.log" -size +${MAX_SIZE_MB}M -exec gzip {} \;

        # 清空正在使用的大日志文件
        for f in $(find "$dir" -name "*.log" -size +${MAX_SIZE_MB}M); do
            # 检查文件是否被进程占用
            if lsof "$f" 2>/dev/null | grep -q .; then
                > "$f"  # 清空但不删除
            fi
        done
    done

Python 版本优势：
    - 更好的错误处理
    - 详细的清理报告
    - 可配置的清理策略
    - 支持 dry-run 模式
"""
import subprocess
import os
import time
import shutil
import logging
from pathlib import Path
from dataclasses import dataclass, field
from typing import List, Optional

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)


# ============================================================
# 配置
# ============================================================

@dataclass
class CleanupConfig:
    """清理配置"""
    log_dirs: List[str] = field(default_factory=lambda: [
        "/var/log",
        "/tmp/test_logs",  # 演示用目录
    ])
    max_age_days: int = 7          # 超过多少天的日志要删除
    compress_above_mb: float = 50  # 超过多少MB的日志要压缩
    truncate_above_mb: float = 100 # 超过多少MB且被占用的要清空
    file_patterns: List[str] = field(default_factory=lambda: [
        "*.log", "*.log.*", "*.out", "*.err"
    ])
    dry_run: bool = True           # 试运行模式（不真正删除）


# ============================================================
# 清理报告
# ============================================================

@dataclass
class CleanupReport:
    """清理报告"""
    deleted_files: List[str] = field(default_factory=list)
    compressed_files: List[str] = field(default_factory=list)
    truncated_files: List[str] = field(default_factory=list)
    errors: List[str] = field(default_factory=list)
    space_freed_bytes: int = 0
    start_time: float = 0
    end_time: float = 0

    def print_summary(self):
        """打印报告摘要"""
        elapsed = self.end_time - self.start_time
        freed_mb = self.space_freed_bytes / (1024 * 1024)

        separator = "=" * 60
        print(f"\n{separator}")
        print("  日志清理报告")
        print(separator)
        print(f"  执行时间:   {elapsed:.2f}秒")
        print(f"  删除文件:   {len(self.deleted_files)} 个")
        print(f"  压缩文件:   {len(self.compressed_files)} 个")
        print(f"  清空文件:   {len(self.truncated_files)} 个")
        print(f"  释放空间:   {freed_mb:.1f} MB")
        print(f"  错误数:     {len(self.errors)}")

        if self.deleted_files:
            print(f"\n  --- 删除的文件 ---")
            for f in self.deleted_files[:10]:
                print(f"    {f}")
            if len(self.deleted_files) > 10:
                print(f"    ... 共 {len(self.deleted_files)} 个")

        if self.compressed_files:
            print(f"\n  --- 压缩的文件 ---")
            for f in self.compressed_files[:10]:
                print(f"    {f}")

        if self.truncated_files:
            print(f"\n  --- 清空的文件 ---")
            for f in self.truncated_files:
                print(f"    {f}")

        if self.errors:
            print(f"\n  --- 错误 ---")
            for e in self.errors:
                print(f"    {e}")

        print(separator)


# ============================================================
# 日志清理器
# ============================================================

class LogCleaner:
    """
    日志清理器
    用 Python + subprocess 替代复杂的 shell find/gzip/lsof 组合
    """

    def __init__(self, config: CleanupConfig):
        self.config = config
        self.report = CleanupReport()

    def run(self) -> CleanupReport:
        """执行清理"""
        self.report.start_time = time.time()
        mode = "试运行" if self.config.dry_run else "正式执行"
        logger.info(f"开始日志清理（{mode}模式）")

        for log_dir in self.config.log_dirs:
            if not os.path.isdir(log_dir):
                logger.warning(f"目录不存在，跳过: {log_dir}")
                continue
            logger.info(f"处理目录: {log_dir}")
            self._process_directory(log_dir)

        self.report.end_time = time.time()
        return self.report

    def _process_directory(self, directory: str):
        """处理单个目录"""
        # 使用 find 命令查找日志文件（比 Python glob 更高效）
        # 对比 Shell: find /var/log -name "*.log" -type f
        log_files = self._find_log_files(directory)
        logger.info(f"  找到 {len(log_files)} 个日志文件")

        for filepath in log_files:
            try:
                self._process_file(filepath)
            except Exception as e:
                self.report.errors.append(f"{filepath}: {e}")
                logger.error(f"  处理失败 {filepath}: {e}")

    def _find_log_files(self, directory: str) -> List[str]:
        """
        使用 find 命令查找日志文件
        对比 Shell: find $dir -type f \( -name "*.log" -o -name "*.log.*" \)
        """
        # 构建 find 命令的 -name 参数
        name_args = []
        for i, pattern in enumerate(self.config.file_patterns):
            if i > 0:
                name_args.append("-o")
            name_args.extend(["-name", pattern])

        cmd = ["find", directory, "-type", "f", "("] + name_args + [")"]

        try:
            result = subprocess.run(
                cmd,
                capture_output=True, text=True,
                timeout=60
            )
            files = [f for f in result.stdout.strip().splitlines() if f]
            return files
        except subprocess.TimeoutExpired:
            logger.error(f"find 命令超时: {directory}")
            return []
        except Exception as e:
            logger.error(f"find 命令失败: {e}")
            return []

    def _process_file(self, filepath: str):
        """处理单个文件"""
        try:
            stat = os.stat(filepath)
        except OSError:
            return

        file_size = stat.st_size
        file_age_days = (time.time() - stat.st_mtime) / 86400
        file_size_mb = file_size / (1024 * 1024)

        # 1. 删除超龄文件
        if file_age_days > self.config.max_age_days:
            self._delete_file(filepath, file_size)
            return

        # 2. 压缩大文件（未被占用的）
        if (file_size_mb > self.config.compress_above_mb
                and not filepath.endswith(".gz")):
            if not self._is_file_in_use(filepath):
                self._compress_file(filepath, file_size)
                return

        # 3. 清空被占用的超大文件
        if file_size_mb > self.config.truncate_above_mb:
            if self._is_file_in_use(filepath):
                self._truncate_file(filepath, file_size)

    def _is_file_in_use(self, filepath: str) -> bool:
        """
        检查文件是否被进程占用
        对比 Shell: lsof "$file" 2>/dev/null | grep -q .
        """
        try:
            result = subprocess.run(
                ["lsof", filepath],
                capture_output=True, text=True,
                timeout=10
            )
            return result.returncode == 0 and bool(result.stdout.strip())
        except (subprocess.TimeoutExpired, FileNotFoundError):
            # lsof 不可用时，假设未被占用
            return False

    def _delete_file(self, filepath: str, size: int):
        """删除文件"""
        logger.info(f"  删除: {filepath} ({size / 1024 / 1024:.1f}MB)")
        if not self.config.dry_run:
            os.remove(filepath)
        self.report.deleted_files.append(filepath)
        self.report.space_freed_bytes += size

    def _compress_file(self, filepath: str, size: int):
        """
        压缩文件
        对比 Shell: gzip "$file"
        """
        logger.info(f"  压缩: {filepath} ({size / 1024 / 1024:.1f}MB)")
        if not self.config.dry_run:
            try:
                result = subprocess.run(
                    ["gzip", filepath],
                    capture_output=True, text=True,
                    timeout=300  # 大文件压缩可能需要5分钟
                )
                if result.returncode != 0:
                    raise RuntimeError(f"gzip 失败: {result.stderr}")
            except subprocess.TimeoutExpired:
                raise RuntimeError("gzip 超时")
        self.report.compressed_files.append(filepath)

    def _truncate_file(self, filepath: str, size: int):
        """
        清空文件（不删除，保持文件描述符有效）
        对比 Shell: > "$file"  或  truncate -s 0 "$file"
        """
        logger.info(f"  清空: {filepath} ({size / 1024 / 1024:.1f}MB，被进程占用）")
        if not self.config.dry_run:
            # 方式1: Python 原生
            with open(filepath, "w") as f:
                pass  # 清空文件

            # 方式2: 用 truncate 命令（等效）
            # subprocess.run(["truncate", "-s", "0", filepath], check=True)
        self.report.truncated_files.append(filepath)
        self.report.space_freed_bytes += size


# ============================================================
# 辅助工具：获取磁盘使用情况
# ============================================================

def get_disk_usage(path: str = "/") -> dict:
    """
    获取磁盘使用情况
    对比 Shell: df -h /
    """
    result = subprocess.run(
        ["df", "-BM", path],
        capture_output=True, text=True
    )
    lines = result.stdout.strip().splitlines()
    if len(lines) >= 2:
        parts = lines[1].split()
        return {
            "filesystem": parts[0],
            "total_mb": int(parts[1].rstrip("M")),
            "used_mb": int(parts[2].rstrip("M")),
            "free_mb": int(parts[3].rstrip("M")),
            "usage_pct": parts[4],
        }
    return {}


# ============================================================
# 主程序
# ============================================================

if __name__ == "__main__":
    # 创建演示用的日志目录和文件
    test_dir = "/tmp/test_logs"
    os.makedirs(test_dir, exist_ok=True)

    # 创建一些测试日志文件
    for i in range(5):
        filepath = os.path.join(test_dir, f"app-{i}.log")
        with open(filepath, "w") as f:
            f.write(f"测试日志 {i}\n" * 100)

    # 创建一个"旧"日志文件（修改时间设为8天前）
    old_log = os.path.join(test_dir, "old-app.log")
    with open(old_log, "w") as f:
        f.write("旧日志\n" * 100)
    old_time = time.time() - (8 * 86400)  # 8天前
    os.utime(old_log, (old_time, old_time))

    # 查看清理前的磁盘使用情况
    print("=== 清理前磁盘使用 ===")
    usage = get_disk_usage("/")
    print(f"  使用率: {usage.get('usage_pct', 'N/A')}")

    # 配置清理参数
    config = CleanupConfig(
        log_dirs=[test_dir],
        max_age_days=7,
        compress_above_mb=50,
        truncate_above_mb=100,
        dry_run=True  # 试运行模式
    )

    # 执行清理
    cleaner = LogCleaner(config)
    report = cleaner.run()

    # 打印报告
    report.print_summary()

    # 清理测试目录
    shutil.rmtree(test_dir, ignore_errors=True)
    print("\n测试目录已清理")

    # 提示
    print("\n提示：将 dry_run 改为 False 即可真正执行清理")
    print("建议先用 dry_run=True 确认清理范围，再正式执行")
```

---

## 8. 常见坑点与最佳实践

### 坑点 1: shell=True 与列表参数

```python
import subprocess

# 错误：shell=True 时用列表，只有第一个元素被执行
subprocess.run(["ls", "-la", "/tmp"], shell=True)
# 实际执行的是 "ls"，"-la" 和 "/tmp" 被忽略了

# 正确用法：
# 方式1: 不用 shell=True（推荐）
subprocess.run(["ls", "-la", "/tmp"])

# 方式2: shell=True 时用字符串
subprocess.run("ls -la /tmp", shell=True)
```

### 坑点 2: 死锁问题

```python
# 错误：stdout 和 stderr 都用 PIPE，不用 communicate() 可能死锁
proc = subprocess.Popen(
    ["cat", "/dev/urandom"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)
# proc.stdout.read()  ← 如果数据量大，可能死锁！

# 正确：用 communicate()
stdout, stderr = proc.communicate(timeout=10)
```

### 坑点 3: 忘记处理编码

```python
# 当命令输出包含非 ASCII 字符时
result = subprocess.run(
    ["ls", "/tmp"],
    capture_output=True,
    text=True,
    encoding="utf-8",   # 明确指定编码
    errors="replace"     # 遇到无法解码的字符用 ? 替代
)
```

### 坑点 4: 子进程残留

```python
# 长时间运行的子进程，如果父进程退出，子进程可能变成僵尸进程
proc = subprocess.Popen(["sleep", "3600"])
# 如果不 wait/kill，进程会残留

# 最佳实践：用 try/finally 或 with 确保清理
try:
    proc = subprocess.Popen(["long_running_command"])
    stdout, stderr = proc.communicate(timeout=60)
except subprocess.TimeoutExpired:
    proc.kill()              # 强制杀死
    proc.communicate()       # 读取剩余输出，回收资源
```

### 最佳实践

| 实践 | 说明 |
|------|------|
| 优先用 `subprocess.run()` | 覆盖 90% 的场景 |
| 避免 `shell=True` | 防止命令注入 |
| 总是设置 `timeout` | 防止命令卡住 |
| 用 `check=True` 检查返回值 | 类似 Shell 的 `set -e` |
| 用列表传参 | 避免空格和特殊字符问题 |
| 用 `communicate()` 读取输出 | 避免死锁 |
| 处理好子进程清理 | 避免僵尸进程 |

---

## 9. 速查表

```python
import subprocess

# === 基础调用 ===
subprocess.run(["ls", "-la"])                    # 简单执行
subprocess.run(["cmd"], check=True)              # 失败时抛异常
subprocess.run(["cmd"], timeout=30)              # 30秒超时
subprocess.run(["cmd"], cwd="/tmp")              # 指定工作目录

# === 捕获输出 ===
r = subprocess.run(["cmd"], capture_output=True, text=True)
r.stdout                                          # 标准输出
r.stderr                                          # 标准错误
r.returncode                                      # 退出码

# === 输入数据 ===
subprocess.run(["cmd"], input="data\n", text=True)

# === 环境变量 ===
import os
env = os.environ.copy()
env["MY_VAR"] = "value"
subprocess.run(["cmd"], env=env)

# === Popen 精细控制 ===
proc = subprocess.Popen(["cmd"], stdout=subprocess.PIPE, text=True)
for line in proc.stdout:                          # 实时读取
    print(line, end="")
proc.wait()

stdout, stderr = proc.communicate(timeout=30)     # 安全读取
proc.terminate()                                   # SIGTERM
proc.kill()                                        # SIGKILL
proc.pid                                           # 进程号

# === 管道（Python 处理，推荐） ===
r = subprocess.run(["ps", "aux"], capture_output=True, text=True)
for line in r.stdout.splitlines():
    if "python" in line:
        print(line)

# === Shell 命令对照 ===
# Shell: output=$(command)
# Python: r = subprocess.run(["command"], capture_output=True, text=True)
#         output = r.stdout

# Shell: command1 | command2
# Python: r1 = subprocess.run(["cmd1"], capture_output=True)
#         r2 = subprocess.run(["cmd2"], input=r1.stdout, capture_output=True)

# Shell: command > /dev/null 2>&1
# Python: subprocess.run(["cmd"], capture_output=True)

# Shell: command &
# Python: proc = subprocess.Popen(["cmd"])

# Shell: set -e
# Python: subprocess.run(["cmd"], check=True)

# Shell: timeout 30 command
# Python: subprocess.run(["cmd"], timeout=30)

# Shell: cd /tmp && command
# Python: subprocess.run(["cmd"], cwd="/tmp")

# Shell: VAR=value command
# Python: subprocess.run(["cmd"], env={**os.environ, "VAR": "value"})
```
