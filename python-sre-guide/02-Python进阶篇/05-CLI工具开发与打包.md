# CLI 工具开发与打包 — 把你的运维脚本变成专业命令行工具

## 1. 是什么 & 为什么用它

### Shell 的做法

```bash
#!/bin/bash
# Shell 方式：用 getopts 解析参数
usage() {
    echo "用法: $0 [-h host] [-p port] [-t timeout] [-v]"
    exit 1
}

HOST="localhost"
PORT=22
TIMEOUT=5
VERBOSE=false

while getopts "h:p:t:v" opt; do
    case $opt in
        h) HOST=$OPTARG ;;
        p) PORT=$OPTARG ;;
        t) TIMEOUT=$OPTARG ;;
        v) VERBOSE=true ;;
        *) usage ;;
    esac
done

echo "检测 $HOST:$PORT (超时${TIMEOUT}秒)"
```

**Shell 的问题**：
- `getopts` 不支持长选项（`--host`）
- 没有参数类型验证
- 帮助信息需要手写
- 子命令实现复杂
- 输出美化困难

### Python 的做法

```python
# Python 方式：用 argparse / click / typer
import argparse

parser = argparse.ArgumentParser(description="服务器端口检测工具")
parser.add_argument("-H", "--host", default="localhost", help="目标主机")
parser.add_argument("-p", "--port", type=int, default=22, help="端口号")
parser.add_argument("-t", "--timeout", type=float, default=5.0, help="超时秒数")
parser.add_argument("-v", "--verbose", action="store_true", help="详细输出")

args = parser.parse_args()
print(f"检测 {args.host}:{args.port} (超时{args.timeout}秒)")

# 自动生成帮助信息：python tool.py --help
```

---

## 2. 核心概念与原理

### 2.1 Python CLI 工具框架对比

| 框架 | 难度 | 特点 | 推荐场景 |
|------|------|------|---------|
| `argparse` | 低 | 标准库，无需安装 | 简单工具、无外部依赖要求 |
| `click` | 中 | 装饰器风格，功能丰富 | 中大型CLI工具 |
| `typer` | 低 | 基于类型提示，最现代 | 新项目首选（推荐） |
| `rich` | - | 美化输出（非CLI框架） | 配合以上任意框架使用 |

### 2.2 CLI 工具结构

```
mycli/
├── pyproject.toml          # 项目配置（现代推荐）
├── setup.py                # 传统打包配置
├── mycli/
│   ├── __init__.py
│   ├── __main__.py         # python -m mycli 入口
│   ├── cli.py              # CLI 命令定义
│   ├── commands/
│   │   ├── __init__.py
│   │   ├── check.py        # check 子命令
│   │   ├── deploy.py       # deploy 子命令
│   │   └── report.py       # report 子命令
│   └── utils/
│       ├── __init__.py
│       └── ssh.py
└── tests/
    └── test_cli.py
```

---

## 3. argparse 基础

### 3.1 最小可运行示例

```python
#!/usr/bin/env python3
"""
argparse 基础示例：服务器检测工具
"""
import argparse
import socket
import sys


def check_port(host: str, port: int, timeout: float) -> bool:
    """检测端口是否开放"""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((host, port))
        sock.close()
        return result == 0
    except Exception:
        return False


def main():
    # 创建解析器
    parser = argparse.ArgumentParser(
        prog="portcheck",
        description="服务器端口检测工具",
        epilog="示例: portcheck -H 192.168.1.1 -p 80 443 8080"
    )

    # 添加参数
    parser.add_argument(
        "-H", "--host",
        required=True,
        help="目标主机IP或域名"
    )
    parser.add_argument(
        "-p", "--ports",
        type=int,
        nargs="+",           # 一个或多个参数
        default=[22, 80, 443],
        help="端口列表（默认: 22 80 443）"
    )
    parser.add_argument(
        "-t", "--timeout",
        type=float,
        default=3.0,
        help="连接超时（秒，默认: 3.0）"
    )
    parser.add_argument(
        "-v", "--verbose",
        action="store_true",  # 布尔开关，不需要值
        help="详细输出模式"
    )
    parser.add_argument(
        "-o", "--output",
        choices=["text", "json", "csv"],
        default="text",
        help="输出格式（默认: text）"
    )

    # 解析参数
    args = parser.parse_args()

    if args.verbose:
        print(f"目标: {args.host}")
        print(f"端口: {args.ports}")
        print(f"超时: {args.timeout}秒")
        print(f"格式: {args.output}")
        print("-" * 40)

    # 执行检测
    results = []
    for port in args.ports:
        is_open = check_port(args.host, port, args.timeout)
        results.append({"host": args.host, "port": port, "open": is_open})

    # 输出结果
    if args.output == "json":
        import json
        print(json.dumps(results, indent=2))
    elif args.output == "csv":
        print("host,port,status")
        for r in results:
            status = "open" if r["open"] else "closed"
            print(f"{r['host']},{r['port']},{status}")
    else:
        for r in results:
            status = "开放" if r["open"] else "关闭"
            print(f"  {r['host']}:{r['port']} -> {status}")

    # 返回退出码
    all_open = all(r["open"] for r in results)
    sys.exit(0 if all_open else 1)


if __name__ == "__main__":
    main()
```

### 3.2 argparse 子命令

```python
#!/usr/bin/env python3
"""
argparse 子命令示例
用法:
    mytool server list
    mytool server check --host 192.168.1.1
    mytool log clean --days 7
"""
import argparse
import sys


def cmd_server_list(args):
    """列出服务器"""
    print("服务器列表:")
    servers = ["web-01", "web-02", "db-01"]
    for s in servers:
        print(f"  {s}")


def cmd_server_check(args):
    """检查服务器"""
    print(f"检查服务器: {args.host}")
    if args.port:
        print(f"  端口: {args.port}")


def cmd_log_clean(args):
    """清理日志"""
    print(f"清理 {args.days} 天前的日志")
    if args.dry_run:
        print("  （试运行模式）")


def main():
    # 顶层解析器
    parser = argparse.ArgumentParser(prog="mytool", description="运维工具集")
    subparsers = parser.add_subparsers(dest="command", help="可用命令")

    # === server 命令组 ===
    server_parser = subparsers.add_parser("server", help="服务器管理")
    server_sub = server_parser.add_subparsers(dest="action")

    # server list
    server_list = server_sub.add_parser("list", help="列出服务器")
    server_list.set_defaults(func=cmd_server_list)

    # server check
    server_check = server_sub.add_parser("check", help="检查服务器")
    server_check.add_argument("--host", required=True, help="目标主机")
    server_check.add_argument("--port", type=int, help="端口号")
    server_check.set_defaults(func=cmd_server_check)

    # === log 命令组 ===
    log_parser = subparsers.add_parser("log", help="日志管理")
    log_sub = log_parser.add_subparsers(dest="action")

    # log clean
    log_clean = log_sub.add_parser("clean", help="清理日志")
    log_clean.add_argument("--days", type=int, default=7, help="保留天数")
    log_clean.add_argument("--dry-run", action="store_true", help="试运行")
    log_clean.set_defaults(func=cmd_log_clean)

    # 解析并执行
    args = parser.parse_args()
    if hasattr(args, "func"):
        args.func(args)
    else:
        parser.print_help()


if __name__ == "__main__":
    main()
```

---

## 4. click 框架

```python
#!/usr/bin/env python3
"""
click 框架示例：装饰器风格定义 CLI
pip install click
"""
import click
import socket
import sys


@click.group()
@click.version_option(version="1.0.0")
def cli():
    """运维工具集 - 基于 Click 框架"""
    pass


@cli.group()
def server():
    """服务器管理命令"""
    pass


@server.command("list")
@click.option("--format", "fmt", type=click.Choice(["text", "json"]),
              default="text", help="输出格式")
def server_list(fmt):
    """列出所有服务器"""
    servers = [
        {"name": "web-01", "ip": "192.168.1.1", "role": "web"},
        {"name": "db-01", "ip": "192.168.1.10", "role": "database"},
    ]
    if fmt == "json":
        import json
        click.echo(json.dumps(servers, indent=2, ensure_ascii=False))
    else:
        for s in servers:
            click.echo(f"  {s['name']:<10} {s['ip']:<15} {s['role']}")


@server.command("check")
@click.argument("host")
@click.option("-p", "--port", type=int, multiple=True, default=[22, 80],
              help="端口号（可多次指定）")
@click.option("-t", "--timeout", type=float, default=3.0, help="超时秒数")
@click.option("-v", "--verbose", is_flag=True, help="详细模式")
def server_check(host, port, timeout, verbose):
    """检查服务器端口状态

    HOST: 目标主机IP或域名
    """
    if verbose:
        click.echo(f"检测 {host}, 端口: {port}, 超时: {timeout}秒")

    for p in port:
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, p))
            sock.close()
            if result == 0:
                click.secho(f"  {host}:{p} -> 开放", fg="green")
            else:
                click.secho(f"  {host}:{p} -> 关闭", fg="red")
        except Exception as e:
            click.secho(f"  {host}:{p} -> 错误: {e}", fg="yellow")


@cli.group()
def log():
    """日志管理命令"""
    pass


@log.command("clean")
@click.option("--days", type=int, default=7, help="保留天数")
@click.option("--path", type=click.Path(exists=True), default="/var/log",
              help="日志目录")
@click.option("--dry-run", is_flag=True, help="试运行模式")
@click.confirmation_option(prompt="确认执行日志清理？")
def log_clean(days, path, dry_run):
    """清理过期日志文件"""
    mode = "试运行" if dry_run else "正式执行"
    click.echo(f"清理 {path} 中超过 {days} 天的日志（{mode}）")

    # 带进度条
    import time
    files = [f"file_{i}.log" for i in range(20)]
    with click.progressbar(files, label="清理进度") as bar:
        for f in bar:
            time.sleep(0.1)  # 模拟清理

    click.secho("清理完成！", fg="green", bold=True)


if __name__ == "__main__":
    cli()
```

---

## 5. typer 框架（推荐）

### 5.1 基础用法

```python
#!/usr/bin/env python3
"""
typer 框架示例：基于类型提示的现代 CLI 框架
pip install typer[all]  # [all] 包含 rich 和 shellingham
"""
import typer
from typing import List, Optional
from enum import Enum
import socket


# 创建应用
app = typer.Typer(
    name="sretool",
    help="SRE 运维工具集",
    add_completion=False  # 简化示例，不添加补全
)

# 子命令组
server_app = typer.Typer(help="服务器管理")
log_app = typer.Typer(help="日志管理")
app.add_typer(server_app, name="server")
app.add_typer(log_app, name="log")


# === 输出格式枚举 ===
class OutputFormat(str, Enum):
    text = "text"
    json = "json"
    csv = "csv"


# === server 子命令 ===

@server_app.command("list")
def server_list(
    fmt: OutputFormat = typer.Option(
        OutputFormat.text, "--format", "-f",
        help="输出格式"
    ),
):
    """列出所有管理的服务器"""
    servers = [
        {"name": "web-01", "ip": "192.168.1.1", "role": "web"},
        {"name": "web-02", "ip": "192.168.1.2", "role": "web"},
        {"name": "db-01", "ip": "192.168.1.10", "role": "database"},
    ]

    if fmt == OutputFormat.json:
        import json
        typer.echo(json.dumps(servers, indent=2, ensure_ascii=False))
    else:
        typer.echo(f"{'名称':<10} {'IP':<15} {'角色':<10}")
        typer.echo("-" * 35)
        for s in servers:
            typer.echo(f"{s['name']:<10} {s['ip']:<15} {s['role']:<10}")


@server_app.command("check")
def server_check(
    host: str = typer.Argument(..., help="目标主机"),
    ports: List[int] = typer.Option(
        [22, 80, 443], "--port", "-p",
        help="端口列表"
    ),
    timeout: float = typer.Option(3.0, "--timeout", "-t", help="超时秒数"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="详细模式"),
):
    """检查服务器端口存活状态"""
    if verbose:
        typer.echo(f"目标: {host}")
        typer.echo(f"端口: {ports}")

    has_failure = False
    for port in ports:
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, port))
            sock.close()
            if result == 0:
                typer.secho(f"  {host}:{port} -> 开放", fg=typer.colors.GREEN)
            else:
                typer.secho(f"  {host}:{port} -> 关闭", fg=typer.colors.RED)
                has_failure = True
        except Exception as e:
            typer.secho(f"  {host}:{port} -> 错误: {e}", fg=typer.colors.YELLOW)
            has_failure = True

    if has_failure:
        raise typer.Exit(code=1)


# === log 子命令 ===

@log_app.command("clean")
def log_clean(
    path: str = typer.Option("/var/log", "--path", help="日志目录"),
    days: int = typer.Option(7, "--days", "-d", help="保留天数"),
    dry_run: bool = typer.Option(True, "--dry-run/--execute", help="试运行/正式执行"),
):
    """清理过期日志文件"""
    import os
    mode = "试运行" if dry_run else "正式执行"
    typer.echo(f"清理 {path} 中超过 {days} 天的日志（{mode}）")

    if not os.path.isdir(path):
        typer.secho(f"目录不存在: {path}", fg=typer.colors.RED)
        raise typer.Exit(code=1)

    typer.secho("完成！", fg=typer.colors.GREEN, bold=True)


# === 版本信息 ===
def version_callback(value: bool):
    if value:
        typer.echo("sretool v1.0.0")
        raise typer.Exit()


@app.callback()
def main(
    version: Optional[bool] = typer.Option(
        None, "--version", "-V",
        callback=version_callback,
        is_eager=True,
        help="显示版本信息"
    ),
):
    """SRE 运维工具集 - 让运维更轻松"""
    pass


if __name__ == "__main__":
    app()
```

---

## 6. rich 美化输出

```python
#!/usr/bin/env python3
"""
rich 库：让终端输出更美观
pip install rich
"""
from rich.console import Console
from rich.table import Table
from rich.progress import Progress, SpinnerColumn, TextColumn, BarColumn, TimeElapsedColumn
from rich.panel import Panel
from rich.text import Text
from rich import print as rprint
import time

console = Console()


# === 1. 彩色文本 ===
def demo_colors():
    """彩色输出"""
    console.print("[bold green]操作成功！[/bold green]")
    console.print("[bold red]错误：连接超时[/bold red]")
    console.print("[yellow]警告：磁盘使用率 85%[/yellow]")
    console.print("[blue]信息：服务已启动[/blue]")
    console.print("[bold magenta]提示：[/bold magenta] 使用 --help 查看帮助")


# === 2. 表格 ===
def demo_table():
    """表格输出"""
    table = Table(title="服务器状态", show_header=True, header_style="bold cyan")
    table.add_column("主机名", style="bold")
    table.add_column("IP 地址", style="dim")
    table.add_column("CPU", justify="right")
    table.add_column("内存", justify="right")
    table.add_column("磁盘", justify="right")
    table.add_column("状态", justify="center")

    # 添加数据
    servers = [
        ("web-01", "192.168.1.1", "35%", "52%", "45%", "[green]正常[/green]"),
        ("web-02", "192.168.1.2", "78%", "81%", "62%", "[yellow]警告[/yellow]"),
        ("db-01", "192.168.1.10", "92%", "88%", "95%", "[red]危险[/red]"),
        ("cache-01", "192.168.1.20", "15%", "45%", "30%", "[green]正常[/green]"),
    ]
    for s in servers:
        table.add_row(*s)

    console.print(table)


# === 3. 进度条 ===
def demo_progress():
    """进度条"""
    servers = [f"192.168.1.{i}" for i in range(1, 21)]

    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        BarColumn(),
        TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
        TimeElapsedColumn(),
        console=console,
    ) as progress:
        task = progress.add_task("扫描服务器...", total=len(servers))
        for server in servers:
            time.sleep(0.15)  # 模拟检测
            progress.update(task, advance=1, description=f"检测 {server}...")

    console.print("[bold green]扫描完成！[/bold green]")


# === 4. 面板 ===
def demo_panel():
    """面板输出"""
    report = """[bold]主机名:[/bold] web-01
[bold]IP:[/bold] 192.168.1.1
[bold]操作系统:[/bold] Ubuntu 22.04 LTS
[bold]运行时间:[/bold] 45天 12小时
[bold]CPU使用率:[/bold] [green]35%[/green]
[bold]内存使用率:[/bold] [yellow]72%[/yellow]
[bold]磁盘使用率:[/bold] [green]45%[/green]"""

    console.print(Panel(
        report,
        title="[bold]服务器巡检报告[/bold]",
        border_style="blue",
        expand=False
    ))


# === 5. 状态指示器 ===
def demo_status():
    """状态指示器（适合长时间操作）"""
    with console.status("[bold green]正在部署应用...", spinner="dots"):
        time.sleep(2)  # 模拟部署
    console.print("[bold green]部署完成！[/bold green]")


# --- 运行所有演示 ---
if __name__ == "__main__":
    console.print(Panel("[bold]Rich 输出美化演示[/bold]", style="bold blue"))

    console.print("\n[bold cyan]=== 1. 彩色文本 ===[/bold cyan]")
    demo_colors()

    console.print("\n[bold cyan]=== 2. 表格 ===[/bold cyan]")
    demo_table()

    console.print("\n[bold cyan]=== 3. 面板 ===[/bold cyan]")
    demo_panel()

    console.print("\n[bold cyan]=== 4. 进度条 ===[/bold cyan]")
    demo_progress()

    console.print("\n[bold cyan]=== 5. 状态指示器 ===[/bold cyan]")
    demo_status()
```

---

## 7. 打包分发

### 7.1 pyproject.toml（现代推荐）

```toml
# pyproject.toml - 现代 Python 项目配置（推荐）
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "sretool"
version = "1.0.0"
description = "SRE 运维工具集"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.8"
authors = [
    {name = "SRE Team", email = "sre@example.com"},
]

# 依赖
dependencies = [
    "typer[all]>=0.9.0",
    "rich>=13.0.0",
]

# 可选依赖
[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "black>=23.0",
    "mypy>=1.0",
]

# 命令行入口点 - 安装后可以直接在终端执行 sretool 命令
[project.scripts]
sretool = "sretool.cli:app"

[tool.setuptools.packages.find]
where = ["."]
```

### 7.2 setup.py（传统方式）

```python
#!/usr/bin/env python3
"""
setup.py - 传统打包配置（依然广泛使用）
"""
from setuptools import setup, find_packages

setup(
    name="sretool",
    version="1.0.0",
    description="SRE 运维工具集",
    author="SRE Team",
    author_email="sre@example.com",
    python_requires=">=3.8",
    packages=find_packages(),
    install_requires=[
        "typer[all]>=0.9.0",
        "rich>=13.0.0",
    ],
    extras_require={
        "dev": ["pytest", "black", "mypy"],
    },
    # 关键：入口点配置
    # 安装后会在 PATH 中创建 sretool 命令
    entry_points={
        "console_scripts": [
            "sretool=sretool.cli:app",  # 命令名=包.模块:函数
        ],
    },
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: POSIX :: Linux",
    ],
)
```

### 7.3 打包与安装

```bash
# === 开发模式安装（推荐开发时使用） ===
# 修改代码后不需要重新安装
pip install -e .

# === 构建分发包 ===
pip install build
python -m build
# 生成 dist/sretool-1.0.0.tar.gz 和 dist/sretool-1.0.0-py3-none-any.whl

# === 安装 whl 包 ===
pip install dist/sretool-1.0.0-py3-none-any.whl

# === 使用 ===
sretool --help
sretool server list
sretool server check 127.0.0.1 --port 22 --port 80

# === 上传到私有 PyPI ===
pip install twine
twine upload --repository-url https://pypi.yourcompany.com dist/*
```

---

## 8. SRE 实战案例：服务器巡检 CLI 工具

```python
#!/usr/bin/env python3
"""
SRE 实战：服务器巡检 CLI 工具
功能：
  - 检查本机系统信息
  - 检查端口存活
  - 检查磁盘使用率
  - 检查进程状态
  - 生成巡检报告

用法：
  python 05_cli_tool.py check system
  python 05_cli_tool.py check ports --host 127.0.0.1 --port 22 --port 80
  python 05_cli_tool.py check disk --warning 80 --critical 90
  python 05_cli_tool.py check process --name nginx --name python
  python 05_cli_tool.py report --format text
"""
import os
import sys
import socket
import subprocess
import time
import json
from dataclasses import dataclass, field, asdict
from typing import List, Optional
from pathlib import Path

# 尝试导入 rich，没有的话用简单输出
try:
    from rich.console import Console
    from rich.table import Table
    from rich.panel import Panel
    HAS_RICH = True
except ImportError:
    HAS_RICH = False


# ============================================================
# 简易 CLI 框架（不依赖外部库）
# ============================================================

class SimpleCLI:
    """
    简易 CLI 框架
    不依赖 typer/click，纯 argparse 实现
    适合不想安装额外依赖的场景
    """

    def __init__(self):
        import argparse
        self.parser = argparse.ArgumentParser(
            prog="sre-inspect",
            description="SRE 服务器巡检工具",
        )
        self.subparsers = self.parser.add_subparsers(dest="command")
        self._setup_commands()

    def _setup_commands(self):
        """注册所有命令"""
        # check 命令组
        check_parser = self.subparsers.add_parser("check", help="执行检查")
        check_sub = check_parser.add_subparsers(dest="check_type")

        # check system
        sys_parser = check_sub.add_parser("system", help="检查系统信息")
        sys_parser.set_defaults(func=self.check_system)

        # check ports
        port_parser = check_sub.add_parser("ports", help="检查端口")
        port_parser.add_argument("--host", default="127.0.0.1", help="目标主机")
        port_parser.add_argument("--port", "-p", type=int, action="append",
                                 default=None, help="端口号（可多次指定）")
        port_parser.add_argument("--timeout", "-t", type=float, default=3.0,
                                 help="超时秒数")
        port_parser.set_defaults(func=self.check_ports)

        # check disk
        disk_parser = check_sub.add_parser("disk", help="检查磁盘")
        disk_parser.add_argument("--warning", type=float, default=80.0,
                                 help="警告阈值（%%）")
        disk_parser.add_argument("--critical", type=float, default=90.0,
                                 help="严重阈值（%%）")
        disk_parser.set_defaults(func=self.check_disk)

        # check process
        proc_parser = check_sub.add_parser("process", help="检查进程")
        proc_parser.add_argument("--name", "-n", action="append",
                                 default=None, help="进程名（可多次指定）")
        proc_parser.set_defaults(func=self.check_process)

        # report 命令
        report_parser = self.subparsers.add_parser("report", help="生成巡检报告")
        report_parser.add_argument("--format", "-f",
                                   choices=["text", "json"],
                                   default="text", help="输出格式")
        report_parser.set_defaults(func=self.full_report)

    def run(self):
        """运行 CLI"""
        args = self.parser.parse_args()
        if hasattr(args, "func"):
            args.func(args)
        else:
            self.parser.print_help()

    # ========================================================
    # 检查功能实现
    # ========================================================

    def check_system(self, args):
        """检查系统信息"""
        info = SystemChecker.collect()
        if HAS_RICH:
            console = Console()
            panel_text = "\n".join([
                f"[bold]主机名:[/bold]   {info['hostname']}",
                f"[bold]IP地址:[/bold]   {info['ip']}",
                f"[bold]系统:[/bold]     {info['os']}",
                f"[bold]内核:[/bold]     {info['kernel']}",
                f"[bold]CPU核数:[/bold]  {info['cpu_cores']}",
                f"[bold]运行时间:[/bold] {info['uptime']}",
                f"[bold]负载:[/bold]     {info['load_avg']}",
            ])
            console.print(Panel(panel_text, title="系统信息", border_style="blue"))
        else:
            print("=== 系统信息 ===")
            for key, value in info.items():
                print(f"  {key}: {value}")

    def check_ports(self, args):
        """检查端口"""
        ports = args.port or [22, 80, 443, 3306, 6379]
        results = PortChecker.check_multiple(args.host, ports, args.timeout)

        if HAS_RICH:
            console = Console()
            table = Table(title=f"端口检查 - {args.host}")
            table.add_column("端口", justify="right", style="bold")
            table.add_column("状态", justify="center")
            table.add_column("延迟", justify="right")

            for r in results:
                if r["open"]:
                    status = "[green]开放[/green]"
                    latency = f"{r['latency_ms']:.1f}ms"
                else:
                    status = "[red]关闭[/red]"
                    latency = "-"
                table.add_row(str(r["port"]), status, latency)
            console.print(table)
        else:
            print(f"=== 端口检查 - {args.host} ===")
            for r in results:
                status = "开放" if r["open"] else "关闭"
                print(f"  :{r['port']:<6} {status}")

    def check_disk(self, args):
        """检查磁盘"""
        partitions = DiskChecker.collect()

        if HAS_RICH:
            console = Console()
            table = Table(title="磁盘使用情况")
            table.add_column("挂载点", style="bold")
            table.add_column("总量", justify="right")
            table.add_column("已用", justify="right")
            table.add_column("可用", justify="right")
            table.add_column("使用率", justify="right")
            table.add_column("状态", justify="center")

            for p in partitions:
                pct = p["usage_percent"]
                if pct >= args.critical:
                    status = "[bold red]严重[/bold red]"
                    pct_str = f"[red]{pct:.1f}%[/red]"
                elif pct >= args.warning:
                    status = "[yellow]警告[/yellow]"
                    pct_str = f"[yellow]{pct:.1f}%[/yellow]"
                else:
                    status = "[green]正常[/green]"
                    pct_str = f"[green]{pct:.1f}%[/green]"
                table.add_row(
                    p["mount"],
                    f"{p['total_gb']:.1f}G",
                    f"{p['used_gb']:.1f}G",
                    f"{p['free_gb']:.1f}G",
                    pct_str,
                    status
                )
            console.print(table)
        else:
            print("=== 磁盘使用情况 ===")
            print(f"  {'挂载点':<20} {'总量':>8} {'已用':>8} {'使用率':>6}")
            for p in partitions:
                print(
                    f"  {p['mount']:<20} "
                    f"{p['total_gb']:>6.1f}G "
                    f"{p['used_gb']:>6.1f}G "
                    f"{p['usage_percent']:>5.1f}%"
                )

    def check_process(self, args):
        """检查进程"""
        names = args.name or ["sshd", "cron"]
        results = ProcessChecker.check_multiple(names)

        if HAS_RICH:
            console = Console()
            table = Table(title="进程检查")
            table.add_column("进程名", style="bold")
            table.add_column("状态", justify="center")
            table.add_column("PID", justify="right")
            table.add_column("CPU%", justify="right")
            table.add_column("MEM%", justify="right")

            for r in results:
                if r["running"]:
                    status = "[green]运行中[/green]"
                    pids = ", ".join(str(p) for p in r["pids"][:3])
                else:
                    status = "[red]未运行[/red]"
                    pids = "-"
                table.add_row(
                    r["name"], status, pids,
                    f"{r.get('cpu', 0):.1f}" if r["running"] else "-",
                    f"{r.get('mem', 0):.1f}" if r["running"] else "-",
                )
            console.print(table)
        else:
            print("=== 进程检查 ===")
            for r in results:
                status = "运行中" if r["running"] else "未运行"
                print(f"  {r['name']:<15} {status}")

    def full_report(self, args):
        """生成完整巡检报告"""
        report = {
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "system": SystemChecker.collect(),
            "ports": PortChecker.check_multiple(
                "127.0.0.1", [22, 80, 443, 3306], 3.0
            ),
            "disk": DiskChecker.collect(),
            "process": ProcessChecker.check_multiple(["sshd", "cron"]),
        }

        if args.format == "json":
            print(json.dumps(report, indent=2, ensure_ascii=False))
        else:
            # 文本格式：依次调用各检查
            class FakeArgs:
                pass
            a = FakeArgs()
            a.port = [22, 80, 443]
            a.host = "127.0.0.1"
            a.timeout = 3.0
            a.warning = 80.0
            a.critical = 90.0
            a.name = ["sshd", "cron"]

            self.check_system(a)
            print()
            self.check_ports(a)
            print()
            self.check_disk(a)
            print()
            self.check_process(a)


# ============================================================
# 检查器（核心逻辑，与 CLI 框架解耦）
# ============================================================

class SystemChecker:
    """系统信息采集器"""

    @staticmethod
    def collect() -> dict:
        info = {}
        info["hostname"] = socket.gethostname()
        try:
            info["ip"] = socket.gethostbyname(socket.gethostname())
        except socket.gaierror:
            info["ip"] = "127.0.0.1"

        # 操作系统
        try:
            result = subprocess.run(
                ["uname", "-sr"], capture_output=True, text=True
            )
            info["kernel"] = result.stdout.strip()
        except Exception:
            info["kernel"] = "未知"

        try:
            with open("/etc/os-release") as f:
                for line in f:
                    if line.startswith("PRETTY_NAME"):
                        info["os"] = line.split("=")[1].strip().strip('"')
                        break
        except FileNotFoundError:
            info["os"] = "未知"

        info["cpu_cores"] = os.cpu_count() or 0

        # 运行时间
        try:
            with open("/proc/uptime") as f:
                secs = float(f.read().split()[0])
                days = int(secs // 86400)
                hours = int((secs % 86400) // 3600)
                info["uptime"] = f"{days}天{hours}小时"
        except Exception:
            info["uptime"] = "未知"

        # 负载
        try:
            load1, load5, load15 = os.getloadavg()
            info["load_avg"] = f"{load1:.2f} / {load5:.2f} / {load15:.2f}"
        except Exception:
            info["load_avg"] = "未知"

        return info


class PortChecker:
    """端口检测器"""

    @staticmethod
    def check_multiple(host: str, ports: List[int], timeout: float) -> List[dict]:
        results = []
        for port in ports:
            start = time.time()
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(timeout)
                ret = sock.connect_ex((host, port))
                latency = (time.time() - start) * 1000
                sock.close()
                results.append({
                    "port": port,
                    "open": ret == 0,
                    "latency_ms": round(latency, 1)
                })
            except Exception as e:
                results.append({
                    "port": port,
                    "open": False,
                    "latency_ms": 0,
                    "error": str(e)
                })
        return results


class DiskChecker:
    """磁盘检查器"""

    @staticmethod
    def collect() -> List[dict]:
        partitions = []
        try:
            result = subprocess.run(
                ["df", "-BG", "--output=target,size,used,avail,pcent"],
                capture_output=True, text=True
            )
            for line in result.stdout.strip().splitlines()[1:]:
                parts = line.split()
                if len(parts) >= 5 and parts[0].startswith("/"):
                    partitions.append({
                        "mount": parts[0],
                        "total_gb": float(parts[1].rstrip("G")),
                        "used_gb": float(parts[2].rstrip("G")),
                        "free_gb": float(parts[3].rstrip("G")),
                        "usage_percent": float(parts[4].rstrip("%")),
                    })
        except Exception:
            # 回退到 Python 原生方式
            for mount in ["/", "/tmp", "/var"]:
                if os.path.isdir(mount):
                    try:
                        stat = os.statvfs(mount)
                        total = stat.f_blocks * stat.f_frsize
                        free = stat.f_bfree * stat.f_frsize
                        used = total - free
                        partitions.append({
                            "mount": mount,
                            "total_gb": round(total / (1024**3), 1),
                            "used_gb": round(used / (1024**3), 1),
                            "free_gb": round(free / (1024**3), 1),
                            "usage_percent": round(used / total * 100, 1) if total > 0 else 0,
                        })
                    except OSError:
                        pass
        return partitions


class ProcessChecker:
    """进程检查器"""

    @staticmethod
    def check_multiple(names: List[str]) -> List[dict]:
        results = []
        for name in names:
            try:
                # 对比 Shell: pgrep -x processname
                result = subprocess.run(
                    ["pgrep", "-f", name],
                    capture_output=True, text=True
                )
                pids = [
                    int(pid) for pid in result.stdout.strip().splitlines()
                    if pid.strip()
                ]
                running = len(pids) > 0
                results.append({
                    "name": name,
                    "running": running,
                    "pids": pids,
                    "count": len(pids),
                })
            except Exception as e:
                results.append({
                    "name": name,
                    "running": False,
                    "pids": [],
                    "count": 0,
                    "error": str(e),
                })
        return results


# ============================================================
# 入口
# ============================================================

if __name__ == "__main__":
    cli = SimpleCLI()
    cli.run()
```

---

## 9. 常见坑点与最佳实践

### 坑点 1: argparse 中 `-` 和 `_` 的转换

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--dry-run", action="store_true")
args = parser.parse_args()

# 注意：参数名中的 - 会转换为 _
print(args.dry_run)  # 用下划线访问，不是 args.dry-run
```

### 坑点 2: 退出码

```python
import sys

# SRE 工具应该返回有意义的退出码
# 0 = 成功
# 1 = 一般错误
# 2 = 参数错误（argparse 默认）

def main():
    if check_failed:
        sys.exit(1)   # 检查失败
    sys.exit(0)       # 检查通过

# 配合监控系统：
# nagios/zabbix 通过退出码判断检查结果
# 0=OK, 1=WARNING, 2=CRITICAL, 3=UNKNOWN
```

### 坑点 3: 信号处理

```python
import signal
import sys

def graceful_exit(signum, frame):
    """优雅退出（处理 Ctrl+C）"""
    print("\n收到中断信号，正在清理...")
    # 执行清理操作
    sys.exit(130)  # 128 + 信号编号

signal.signal(signal.SIGINT, graceful_exit)
signal.signal(signal.SIGTERM, graceful_exit)
```

### 最佳实践

| 实践 | 说明 |
|------|------|
| 用有意义的退出码 | 配合监控系统和 Shell 脚本 |
| 支持 `--help` | argparse/click/typer 自动提供 |
| 支持 `--version` | 方便排查问题 |
| 支持 JSON 输出 | 方便程序化处理 |
| 处理 SIGINT/SIGTERM | 优雅退出，清理资源 |
| 核心逻辑与 CLI 解耦 | 方便测试和复用 |
| 用 `entry_points` 打包 | 安装后直接当命令使用 |
| 提供 `--dry-run` | 运维操作先预览再执行 |

---

## 10. 速查表

```python
# === argparse 速查 ===
import argparse
parser = argparse.ArgumentParser(description="工具描述")
parser.add_argument("name")                          # 位置参数（必填）
parser.add_argument("-v", "--verbose", action="store_true")  # 开关参数
parser.add_argument("-n", "--count", type=int, default=1)    # 有默认值
parser.add_argument("-f", "--format", choices=["text","json"])  # 限定选项
parser.add_argument("-p", "--port", type=int, action="append") # 多次指定
parser.add_argument("--host", nargs="+")             # 一个或多个值
args = parser.parse_args()

# 子命令
sub = parser.add_subparsers(dest="command")
cmd1 = sub.add_parser("list", help="列出内容")
cmd1.set_defaults(func=handle_list)

# === click 速查 ===
import click

@click.command()
@click.argument("name")
@click.option("-v", "--verbose", is_flag=True)
@click.option("-n", "--count", type=int, default=1)
def cmd(name, verbose, count):
    pass

# === typer 速查 ===
import typer
app = typer.Typer()

@app.command()
def cmd(
    name: str = typer.Argument(..., help="名称"),
    verbose: bool = typer.Option(False, "--verbose", "-v"),
    count: int = typer.Option(1, "--count", "-n"),
):
    pass

app()

# === rich 速查 ===
from rich.console import Console
from rich.table import Table
console = Console()
console.print("[bold green]成功[/bold green]")
console.print("[red]失败[/red]")
table = Table(title="标题")
table.add_column("列名")
table.add_row("数据")
console.print(table)

# === 打包速查 ===
# pyproject.toml:
#   [project.scripts]
#   mytool = "mypackage.cli:main"
#
# 安装: pip install -e .
# 构建: python -m build
# 上传: twine upload dist/*

# === Shell 对比 ===
# Shell getopts     → Python argparse / click / typer
# Shell echo 颜色   → Python rich / click.secho
# Shell 子命令      → Python subparsers / click.group / typer.Typer
# Shell 退出码 $?   → Python sys.exit(code)
# Shell trap        → Python signal.signal
# Shell 安装脚本到PATH → Python entry_points / project.scripts
```
