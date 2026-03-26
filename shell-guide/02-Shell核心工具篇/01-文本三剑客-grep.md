# 文本三剑客 — grep

> 一句话说明：grep 是 Linux 下最常用的文本搜索工具，掌握它就掌握了日志排查的第一把利器。

---

## 1. 是什么 & 为什么用它

grep = **G**lobal **R**egular **E**xpression **P**rint，在文件中搜索匹配正则表达式的行。

```bash
# 最基本的用法：在文件中搜索关键词
grep "error" /var/log/syslog
```

---

## 2. 基本用法

```bash
# 基本搜索
grep "pattern" file.txt

# 多文件搜索
grep "pattern" file1.txt file2.txt
grep "pattern" /var/log/*.log

# 递归搜索目录
grep -r "pattern" /path/to/dir/
grep -rn "TODO" ./src/          # 递归 + 显示行号
```

---

## 3. 常用选项

```bash
# -i  忽略大小写
grep -i "error" app.log

# -n  显示行号
grep -n "error" app.log

# -c  只显示匹配行数
grep -c "error" app.log

# -l  只显示包含匹配的文件名
grep -rl "TODO" ./src/

# -L  只显示不包含匹配的文件名
grep -rL "TODO" ./src/

# -v  反向匹配（排除）
grep -v "^#" /etc/ssh/sshd_config     # 去掉注释行
grep -v "^$" config.txt                 # 去掉空行

# -w  全词匹配
grep -w "port" config.txt    # 匹配 "port"，不匹配 "export"

# -x  整行匹配
grep -x "exact line" file.txt

# -o  只输出匹配的部分
echo "IP is 192.168.1.100" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
# 输出: 192.168.1.100

# -A/-B/-C  显示上下文
grep -A 3 "error" app.log     # 匹配行 + 后3行（After）
grep -B 3 "error" app.log     # 前3行 + 匹配行（Before）
grep -C 3 "error" app.log     # 前3行 + 匹配行 + 后3行（Context）

# -E  扩展正则（等同于 egrep）
grep -E "error|warn|fatal" app.log

# -P  Perl 正则（支持更强大的语法）
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" access.log

# --include/--exclude  过滤文件
grep -r "function" --include="*.sh" ./
grep -r "TODO" --exclude="*.min.js" --exclude-dir=node_modules ./

# --color  高亮匹配（通常已默认开启）
grep --color=auto "error" app.log
```

---

## 4. 正则表达式基础

### 4.1 基本正则（BRE，grep 默认）

```bash
.           # 匹配任意单个字符
*           # 前一个字符重复 0 次或多次
^           # 行首
$           # 行尾
[]          # 字符集合
[^]         # 否定字符集合
\           # 转义
\{n\}       # 精确重复 n 次
\{n,m\}     # 重复 n 到 m 次
```

### 4.2 扩展正则（ERE，grep -E）

```bash
+           # 前一个字符重复 1 次或多次
?           # 前一个字符 0 次或 1 次
|           # 或
()          # 分组
{n}         # 精确重复 n 次（不需要转义）
{n,m}       # 重复 n 到 m 次
```

### 4.3 实用正则示例

```bash
# 匹配 IP 地址
grep -E '\b[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\b' access.log

# 匹配邮箱
grep -E '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' contacts.txt

# 匹配日期 (YYYY-MM-DD)
grep -E '20[0-9]{2}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])' log.txt

# 匹配有效配置行（去掉注释和空行）
grep -E '^[^#;]' /etc/ssh/sshd_config | grep -v '^$'
```

---

## 5. 实战场景

### 5.1 日志分析

```bash
# 统计各级别日志数量
for level in ERROR WARN INFO DEBUG; do
    count=$(grep -c "$level" app.log 2>/dev/null || echo 0)
    echo "$level: $count"
done

# 查看最近的错误及上下文
grep -B 2 -A 5 "Exception" app.log | tail -50

# 按时间范围过滤
grep "^2024-01-15 1[4-6]:" app.log     # 14:00 ~ 16:59 的日志

# 统计每小时错误数
grep "ERROR" app.log | grep -oE '\d{2}:\d{2}' | cut -d: -f1 | sort | uniq -c
```

### 5.2 系统管理

```bash
# 查找监听端口
ss -tlnp | grep ":80\b"

# 查找正在运行的进程
ps aux | grep "[n]ginx"     # [] 技巧：避免 grep 自身出现在结果中

# 检查配置文件语法
grep -n "server_name" /etc/nginx/conf.d/*.conf

# 查找大文件属主
find /home -size +100M -exec ls -lh {} \; 2>/dev/null | grep -v root
```

### 5.3 代码审查

```bash
# 查找 TODO/FIXME
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.sh" ./

# 查找硬编码密码
grep -rn "password\s*=" --include="*.py" --include="*.sh" ./

# 查找未使用的函数
grep -roh '\b[a-z_]\+()' *.sh | sort -u | while read -r func; do
    name="${func%()*}"
    count=$(grep -c "\b${name}\b" *.sh)
    (( count <= 1 )) && echo "可能未使用: $name"
done
```

---

## 6. grep 家族

```bash
# egrep = grep -E（扩展正则）
egrep "error|warn" app.log

# fgrep = grep -F（固定字符串，不解释正则，最快）
fgrep "192.168.1.100" access.log

# zgrep — 搜索压缩文件
zgrep "error" /var/log/syslog.*.gz

# rg (ripgrep) — 现代替代品，速度极快
# 安装后可替代 grep -r
rg "pattern" ./src/
rg -t sh "TODO" .        # 只搜索 .sh 文件
```

---

## 7. 小结

| 选项 | 作用 |
|------|------|
| `-i` | 忽略大小写 |
| `-n` | 显示行号 |
| `-r` | 递归搜索 |
| `-v` | 反向匹配 |
| `-c` | 统计匹配行数 |
| `-o` | 只输出匹配部分 |
| `-E` | 扩展正则 |
| `-A/-B/-C` | 显示上下文 |
| `-w` | 全词匹配 |
| `-l` | 只显示文件名 |

> **下一篇**：[文本三剑客-sed](02-文本三剑客-sed.md) — 流编辑器，批量文本替换的利器。
