# 文本三剑客 — awk

> 一句话说明：awk 是一门完整的文本处理语言，擅长按列处理结构化数据，是日志分析和报表生成的终极武器。

---

## 1. 基本语法

```bash
awk '模式 { 动作 }' 文件

# 工作流程：逐行读取 → 按分隔符切分成字段 → 匹配模式 → 执行动作
```

**内置变量：**

| 变量 | 含义 |
|------|------|
| `$0` | 当前整行 |
| `$1, $2...` | 第 1、2... 个字段 |
| `NF` | 当前行的字段数（Number of Fields） |
| `NR` | 当前行号（Number of Records） |
| `FS` | 输入字段分隔符（默认空格/Tab） |
| `OFS` | 输出字段分隔符（默认空格） |
| `RS` | 输入记录分隔符（默认换行） |
| `ORS` | 输出记录分隔符（默认换行） |
| `FILENAME` | 当前文件名 |

---

## 2. 基本用法

```bash
# 打印指定列
awk '{print $1}' file.txt           # 第一列
awk '{print $1, $3}' file.txt       # 第一和第三列
awk '{print $NF}' file.txt          # 最后一列
awk '{print $(NF-1)}' file.txt      # 倒数第二列

# 指定分隔符
awk -F: '{print $1, $3}' /etc/passwd      # 以 : 分隔
awk -F'[,;]' '{print $1}' data.csv        # 多分隔符

# 指定输出分隔符
awk -F: 'BEGIN{OFS="\t"} {print $1, $3, $7}' /etc/passwd

# 格式化输出
awk -F: '{printf "用户: %-15s UID: %5d Shell: %s\n", $1, $3, $7}' /etc/passwd
```

---

## 3. 模式匹配

```bash
# 正则匹配
awk '/error/' app.log                      # 包含 error 的行
awk '!/^#/' config.txt                     # 不以 # 开头的行

# 字段匹配
awk -F: '$3 >= 1000' /etc/passwd           # UID >= 1000
awk -F: '$7 ~ /bash/' /etc/passwd          # Shell 包含 bash
awk -F: '$7 !~ /nologin/' /etc/passwd      # Shell 不含 nologin

# 范围匹配
awk '/START/,/END/' file                   # START 到 END 之间的行

# 行号条件
awk 'NR==5' file                           # 第 5 行
awk 'NR>=5 && NR<=10' file                 # 第 5-10 行
awk 'NR%2==0' file                         # 偶数行

# 组合条件
awk -F: '$3>=1000 && $7~/bash/' /etc/passwd
```

---

## 4. BEGIN 和 END

```bash
awk '
    BEGIN { print "===== 开始 ====="; count=0 }
    /error/ { count++; print NR": "$0 }
    END { print "===== 结束 ====="; print "共 "count" 个错误" }
' app.log
```

---

## 5. 流程控制

```bash
# if-else
awk -F: '{
    if ($3 == 0) print $1, "是 root"
    else if ($3 < 1000) print $1, "是系统用户"
    else print $1, "是普通用户"
}' /etc/passwd

# for 循环
awk '{for(i=1;i<=NF;i++) print $i}' file       # 逐字段打印

# while 循环
awk '{i=1; while(i<=NF) {print $i; i++}}' file

# 数组
awk '{count[$1]++} END {for(k in count) print k, count[k]}' access.log
```

---

## 6. 常用内置函数

```bash
# 字符串函数
awk '{print length($0)}' file                    # 字符串长度
awk '{print substr($0, 1, 10)}' file             # 子串
awk '{print toupper($1)}' file                   # 转大写
awk '{print tolower($1)}' file                   # 转小写
awk '{gsub(/old/, "new"); print}' file           # 全局替换
awk '{sub(/old/, "new"); print}' file            # 替换第一个
awk '{n=split($0, arr, ":"); print arr[1]}' file # 分割

# 数学函数
awk 'BEGIN{print int(3.9)}'        # 3
awk 'BEGIN{print sqrt(144)}'       # 12
awk 'BEGIN{print log(100)}'        # 4.60517
awk 'BEGIN{srand(); print rand()}' # 随机数 0-1

# 时间函数（gawk）
awk 'BEGIN{print strftime("%Y-%m-%d %H:%M:%S", systime())}'
```

---

## 7. 实战场景

### 7.1 日志分析

```bash
# 统计各 IP 访问次数（Top 10）
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 纯 awk 实现（更高效）
awk '{ip[$1]++} END {for(k in ip) print ip[k], k}' access.log | sort -rn | head -10

# 统计各状态码
awk '{code[$9]++} END {for(k in code) printf "%s: %d\n", k, code[k]}' access.log

# 统计各小时请求量
awk '{split($4, t, ":"); hour=t[2]; h[hour]++} END {for(k in h) print k":00", h[k]}' access.log | sort

# 计算平均响应时间（假设最后一列是响应时间）
awk '{sum+=$NF; count++} END {printf "平均响应时间: %.2fms\n", sum/count}' access.log

# 找出响应时间 > 1s 的请求
awk '$NF > 1000 {print $0}' access.log
```

### 7.2 系统监控

```bash
# 列出占用内存最多的进程
ps aux | awk 'NR>1{printf "%-20s %s%%\n", $11, $4}' | sort -t'%' -k1 -rn | head -10

# 磁盘使用率超过 80% 告警
df -h | awk 'NR>1 && +$5 > 80 {printf "警告: %s 使用率 %s\n", $6, $5}'

# 统计网络连接状态
ss -ant | awk 'NR>1{state[$1]++} END {for(k in state) printf "%-15s %d\n", k, state[k]}'
```

### 7.3 CSV 处理

```bash
# 读取 CSV（处理带引号的字段需要用 FPAT）
awk -v FPAT='[^,]*|"[^"]*"' '{print $1, $3}' data.csv

# CSV 转 JSON（简易版）
awk -F, 'NR==1{split($0,h); next}
{
    printf "{"
    for(i=1;i<=NF;i++) {
        printf "\"%s\":\"%s\"", h[i], $i
        if(i<NF) printf ","
    }
    print "}"
}' data.csv
```

---

## 8. awk 脚本文件

当逻辑复杂时，可以把 awk 代码写成文件：

```awk
# analyze.awk
BEGIN {
    FS = " "
    print "====== 日志分析报告 ======"
}

/ERROR/ {
    errors++
    error_detail[$4]++
}

/WARN/ { warnings++ }

END {
    printf "错误总数: %d\n", errors
    printf "警告总数: %d\n", warnings
    print "\n错误分布:"
    for (k in error_detail)
        printf "  %-30s %d\n", k, error_detail[k]
}
```

```bash
awk -f analyze.awk app.log
```

---

## 9. 小结

| 知识点 | 要点 |
|--------|------|
| 基本格式 | `awk '模式{动作}' 文件` |
| 字段引用 | `$1` `$NF` `$0` |
| 分隔符 | `-F:` 或 `FS=","` |
| 模式 | 正则、比较、范围、BEGIN/END |
| 数组 | `arr[key]++` 统计计数 |
| 格式化 | `printf` 精确控制输出 |
| 适用场景 | 列处理、统计汇总、报表生成 |

**三剑客选择指南：**

| 需求 | 推荐工具 |
|------|----------|
| 搜索匹配 | grep |
| 替换/删除 | sed |
| 按列处理/统计 | awk |

> **下一篇**：[文件与目录操作](04-文件与目录操作.md) — 掌握文件管理的核心命令。
