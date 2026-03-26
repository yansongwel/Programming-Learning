# 文本三剑客 — sed

> 一句话说明：sed（Stream Editor）是流编辑器，擅长对文本进行逐行处理——替换、删除、插入，批量修改配置文件的首选工具。

---

## 1. 基本语法

```bash
sed [选项] '命令' 文件
sed [选项] -e '命令1' -e '命令2' 文件
sed [选项] -f 脚本文件 文件
```

**关键选项：**

| 选项 | 作用 |
|------|------|
| `-n` | 静默模式，不自动打印每行 |
| `-i` | 直接修改文件（危险！建议先测试） |
| `-i.bak` | 修改文件前先备份为 `.bak` |
| `-e` | 执行多个命令 |
| `-E` / `-r` | 使用扩展正则 |

---

## 2. 替换（最常用）

```bash
# s/old/new/  替换每行第一个匹配
echo "hello hello hello" | sed 's/hello/hi/'
# hi hello hello

# s/old/new/g  替换所有匹配（global）
echo "hello hello hello" | sed 's/hello/hi/g'
# hi hi hi

# s/old/new/2  替换第 N 个匹配
echo "aaa bbb aaa" | sed 's/aaa/xxx/2'
# aaa bbb xxx

# 不区分大小写
echo "Hello HELLO" | sed 's/hello/hi/gi'
# hi hi

# 使用不同的分隔符（路径中有 / 时很有用）
sed 's|/usr/local/bin|/opt/bin|g' config.txt
sed 's#/var/log#/data/log#g' config.txt
```

### 反向引用

```bash
# () 捕获组 + \1 引用
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# 15/01/2024

# 给数字加引号
echo "port 8080" | sed -E 's/([0-9]+)/"\1"/g'
# port "8080"

# & 代表整个匹配
echo "hello world" | sed 's/[a-z]*/[&]/g'
# [hello] [world]
```

---

## 3. 行定位（地址）

```bash
# 指定行号
sed '3s/old/new/' file          # 只替换第 3 行
sed '3d' file                    # 删除第 3 行

# 行范围
sed '2,5s/old/new/' file        # 第 2-5 行
sed '2,5d' file                  # 删除第 2-5 行

# 最后一行
sed '$d' file                    # 删除最后一行

# 从第 N 行到末尾
sed '10,$s/old/new/' file

# 正则匹配行
sed '/^#/d' file                 # 删除注释行
sed '/error/s/old/new/' file     # 只在包含 error 的行替换

# 两个模式之间
sed '/START/,/END/d' file        # 删除 START 到 END 之间的行

# 取反 !
sed '/^#/!s/foo/bar/' file       # 非注释行才替换
```

---

## 4. 常用命令

```bash
# d 删除
sed '/^$/d' file                 # 删除空行
sed '/^#/d' file                 # 删除注释行
sed '/^#/d;/^$/d' file           # 同时删除注释和空行

# p 打印（配合 -n）
sed -n '5p' file                 # 只打印第 5 行
sed -n '5,10p' file              # 打印第 5-10 行
sed -n '/error/p' file           # 打印包含 error 的行

# i 在行前插入
sed '1i\# This is a header' file

# a 在行后追加
sed '$a\# End of file' file

# c 替换整行
sed '/^HOSTNAME/c\HOSTNAME=newhost' /etc/sysconfig/network

# = 打印行号
sed -n '/error/=' file           # 打印包含 error 的行号

# y 逐字符转换（类似 tr）
echo "hello" | sed 'y/helo/HELO/'   # HELLO
```

---

## 5. 实战场景

### 5.1 配置文件修改

```bash
# 修改 SSH 端口
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
sed -i 's/^Port .*/Port 2222/' /etc/ssh/sshd_config

# 禁用 root 登录
sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# 修改 nginx worker 数
sed -i 's/worker_processes.*/worker_processes auto;/' /etc/nginx/nginx.conf

# 批量修改配置
sed -i.bak \
    -e 's/^listen_port=.*/listen_port=8080/' \
    -e 's/^log_level=.*/log_level=INFO/' \
    -e 's/^max_connections=.*/max_connections=1000/' \
    app.conf
```

### 5.2 日志处理

```bash
# 提取时间范围内的日志
sed -n '/^2024-01-15 14:00/,/^2024-01-15 15:00/p' app.log

# 去除 ANSI 颜色码
sed -E 's/\x1b\[[0-9;]*m//g' colored_output.txt

# 提取 JSON 中的某个字段（简单场景）
sed -n 's/.*"name": "\([^"]*\)".*/\1/p' data.json
```

### 5.3 批量文件处理

```bash
# 批量替换目录下所有文件
find /etc/nginx/conf.d -name "*.conf" -exec \
    sed -i 's/old-domain.com/new-domain.com/g' {} \;

# 给所有 Shell 脚本添加 set -euo pipefail
find . -name "*.sh" -exec \
    sed -i '2a\set -euo pipefail' {} \;

# 批量修改文件头部版权信息
find . -name "*.py" -exec \
    sed -i '1s/Copyright 2023/Copyright 2024/' {} \;
```

---

## 6. 多行处理

```bash
# N 将下一行追加到模式空间
# 合并连续的两行
sed 'N;s/\n/ /' file

# 删除连续空行（只保留一个）
sed '/^$/N;/^\n$/d' file

# 在 pattern 所在行和下一行之间插入空行
sed '/pattern/G' file
```

---

## 7. 小结

| 命令 | 作用 | 示例 |
|------|------|------|
| `s///` | 替换 | `sed 's/old/new/g'` |
| `d` | 删除行 | `sed '/^#/d'` |
| `p` | 打印行 | `sed -n '5,10p'` |
| `i\` | 行前插入 | `sed '1i\header'` |
| `a\` | 行后追加 | `sed '$a\footer'` |
| `c\` | 替换整行 | `sed '/key/c\newline'` |
| `-i` | 原地修改 | `sed -i.bak 's/a/b/g'` |
| `-E` | 扩展正则 | `sed -E 's/([0-9]+)/\1/'` |

> **下一篇**：[文本三剑客-awk](03-文本三剑客-awk.md) — 更强大的文本处理语言。
