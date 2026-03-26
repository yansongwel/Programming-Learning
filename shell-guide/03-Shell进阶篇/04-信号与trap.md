# 信号与 trap

> 一句话说明：掌握 Linux 信号机制和 trap 命令，让脚本能优雅地处理中断、清理临时文件。

---

## 1. 常用信号

| 信号 | 编号 | 说明 | 默认行为 |
|------|------|------|----------|
| SIGHUP | 1 | 终端关闭 | 终止 |
| SIGINT | 2 | Ctrl+C | 终止 |
| SIGQUIT | 3 | Ctrl+\ | 终止+core dump |
| SIGKILL | 9 | 强制杀死 | 终止（不可捕获） |
| SIGTERM | 15 | 优雅终止 | 终止 |
| SIGSTOP | 19 | 暂停 | 暂停（不可捕获） |
| SIGUSR1 | 10 | 用户自定义 1 | 终止 |
| SIGUSR2 | 12 | 用户自定义 2 | 终止 |
| EXIT | - | 脚本退出（伪信号） | - |
| ERR | - | 命令失败（伪信号） | - |

```bash
# 查看所有信号
kill -l
trap -l
```

---

## 2. trap 基本用法

```bash
# trap '命令' 信号列表
trap 'echo "收到 Ctrl+C"' INT
trap 'echo "脚本退出"' EXIT
trap 'echo "命令出错: $BASH_COMMAND"' ERR

# 忽略信号
trap '' INT          # 忽略 Ctrl+C

# 重置为默认行为
trap - INT           # 恢复 Ctrl+C 默认行为

# 查看当前 trap
trap -p
```

---

## 3. 最佳实践

### 3.1 清理临时文件

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPDIR=$(mktemp -d)
TMPFILE=$(mktemp)

# 无论如何退出都会清理
cleanup() {
    rm -rf "$TMPDIR" "$TMPFILE"
    echo "临时文件已清理"
}
trap cleanup EXIT

# 正常的脚本逻辑
echo "使用临时目录: $TMPDIR"
echo "data" > "$TMPFILE"
# ... 其他操作 ...
# 退出时自动调用 cleanup
```

### 3.2 优雅停止服务

```bash
#!/usr/bin/env bash
# graceful_server.sh — 模拟优雅停止

set -euo pipefail

RUNNING=true
PID_FILE="/tmp/myserver.pid"

shutdown() {
    echo "收到停止信号，正在优雅关闭..."
    RUNNING=false
    # 清理资源
    rm -f "$PID_FILE"
    echo "服务已停止"
    exit 0
}

trap shutdown SIGTERM SIGINT

echo $$ > "$PID_FILE"
echo "服务已启动 (PID: $$)"

while $RUNNING; do
    echo "处理请求中... $(date '+%H:%M:%S')"
    sleep 2
done
```

### 3.3 错误追踪

```bash
#!/usr/bin/env bash
set -euo pipefail

on_error() {
    local line=$1
    local cmd=$2
    local code=$3
    echo "错误: 行 ${line}, 命令 '${cmd}', 退出码 ${code}" >&2
}
trap 'on_error ${LINENO} "${BASH_COMMAND}" $?' ERR

# 任何命令失败都会触发 on_error
echo "开始执行..."
ls /nonexistent_file     # 这里会触发 ERR trap
echo "这行不会执行"
```

### 3.4 锁文件防止重复执行

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/tmp/myscript.lock"

cleanup() {
    rm -f "$LOCK_FILE"
}

# 检查锁
if [[ -f "$LOCK_FILE" ]]; then
    PID=$(cat "$LOCK_FILE")
    if kill -0 "$PID" 2>/dev/null; then
        echo "脚本已在运行 (PID: $PID)"
        exit 1
    fi
    echo "发现过期锁文件，清理..."
    rm -f "$LOCK_FILE"
fi

# 设置锁和清理
echo $$ > "$LOCK_FILE"
trap cleanup EXIT

echo "开始执行... (PID: $$)"
sleep 30    # 模拟长时间任务
echo "完成"
```

---

## 4. 小结

| 知识点 | 要点 |
|--------|------|
| 常用信号 | INT(Ctrl+C), TERM(优雅终止), KILL(强制) |
| trap 用法 | `trap '命令' 信号` |
| EXIT trap | 脚本退出时清理资源（最常用） |
| ERR trap | 命令失败时记录错误信息 |
| 锁文件 | trap EXIT + PID 文件防止重复运行 |

> **下一篇**：[调试与错误处理](05-调试与错误处理.md) — 系统化的脚本调试方法。
