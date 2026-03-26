# 子 Shell 与进程替换

> 一句话说明：理解子 Shell 的创建时机和变量隔离机制，掌握进程替换避免常见陷阱。

---

## 1. 什么是子 Shell

子 Shell 是当前 Shell 的一个子进程副本，继承父 Shell 的环境变量但拥有独立的变量空间。

```bash
# 以下操作会创建子 Shell：
# 1. () 圆括号
(cd /tmp; echo $PWD)     # 在子 Shell 中执行
echo $PWD                 # 当前目录未改变

# 2. 管道
echo "hello" | read -r VAR
echo "$VAR"               # 空！read 在子 Shell 中执行

# 3. 命令替换
RESULT=$(echo "hello")    # $() 内在子 Shell 中执行

# 4. 后台进程
command &                 # 在子 Shell 中执行

# 5. bash -c
bash -c 'echo $BASHPID'  # 新的子 Shell
```

### 子 Shell 变量陷阱

```bash
# ❌ 管道中的变量修改不会传回
COUNT=0
cat /etc/passwd | while read -r line; do
    ((COUNT++))
done
echo "$COUNT"    # 0！

# ✅ 解决方案 1：进程替换
COUNT=0
while read -r line; do
    ((COUNT++))
done < <(cat /etc/passwd)
echo "$COUNT"    # 正确的行数

# ✅ 解决方案 2：Here String
COUNT=0
while read -r line; do
    ((COUNT++))
done <<< "$(cat /etc/passwd)"
echo "$COUNT"

# ✅ 解决方案 3：lastpipe（Bash 4.2+）
shopt -s lastpipe
COUNT=0
cat /etc/passwd | while read -r line; do
    ((COUNT++))
done
echo "$COUNT"    # 正确
```

---

## 2. () 与 {} 的区别

```bash
# () — 在子 Shell 中执行
(VAR="子Shell"; echo $VAR)    # 子Shell
echo $VAR                      # 空（父 Shell 不受影响）

# {} — 在当前 Shell 中执行（命令组）
{ VAR="当前Shell"; echo $VAR; }   # 当前Shell
echo $VAR                          # 当前Shell（变量保留）

# 注意 {} 的语法要求：
# 1. { 后必须有空格
# 2. 最后一条命令必须有分号
# 3. } 前必须有分号或换行
```

---

## 3. 进程替换详解

```bash
# <(command) — 将命令输出作为文件
# >(command) — 将文件写入作为命令输入

# 比较两个命令的输出
diff <(ls /dir1) <(ls /dir2)

# 比较远程和本地文件
diff <(ssh host cat /etc/config) /etc/config

# 同时写入多个目标
tee >(gzip > file.gz) >(wc -l > count.txt) < input.txt > /dev/null

# 避免临时文件
# 传统方式：
sort file1 > /tmp/s1
sort file2 > /tmp/s2
comm /tmp/s1 /tmp/s2
rm /tmp/s1 /tmp/s2

# 进程替换：
comm <(sort file1) <(sort file2)
```

---

## 4. 实战：并行执行

```bash
#!/usr/bin/env bash
# parallel_check.sh — 并行检查多台服务器

set -euo pipefail

SERVERS=("web01" "web02" "db01" "db02" "redis01")
TIMEOUT=5

check_server() {
    local server="$1"
    if ping -c 1 -W "$TIMEOUT" "$server" &>/dev/null; then
        echo "$server: 在线"
    else
        echo "$server: 离线"
    fi
}

# 并行执行
for server in "${SERVERS[@]}"; do
    check_server "$server" &
done

# 等待所有后台任务完成
wait
echo "检查完成"
```

---

## 5. 小结

| 知识点 | 要点 |
|--------|------|
| 子 Shell | `()`, 管道, `$()`, `&` 都会创建 |
| 变量隔离 | 子 Shell 中的修改不影响父 Shell |
| 管道陷阱 | 用 `< <(cmd)` 替代 `cmd \| while` |
| `()` vs `{}` | `()` 子 Shell，`{}` 当前 Shell |
| 进程替换 | `<(cmd)` `>(cmd)` 替代临时文件 |

> **下一篇**：[信号与trap](04-信号与trap.md) — 优雅地处理中断和清理工作。
