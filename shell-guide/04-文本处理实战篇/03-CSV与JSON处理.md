# CSV 与 JSON 处理

> 一句话说明：掌握用 Shell 工具处理 CSV 和 JSON 结构化数据的实用技巧。

---

## 1. CSV 处理

### 1.1 基本读取

```bash
# 简单 CSV（无引号、无逗号字段）
# name,age,city
# 张三,30,北京
# 李四,25,上海

# 按列提取
awk -F, '{print $1, $3}' data.csv
cut -d, -f1,3 data.csv

# 跳过表头
awk -F, 'NR>1{print $1, $3}' data.csv
tail -n +2 data.csv | cut -d, -f1,3

# 过滤
awk -F, 'NR>1 && $2>25{print $0}' data.csv

# 排序
sort -t, -k2 -n data.csv         # 按第 2 列数字排序
```

### 1.2 处理带引号的 CSV

```bash
# 带引号: "name","address"
# "张三","北京,海淀区"

# awk FPAT 方式
awk -v FPAT='[^,]*|"[^"]*"' '{print $1, $2}' data.csv

# 用 csvtool（需安装）
csvtool col 1,3 data.csv
csvtool readable data.csv
```

### 1.3 CSV 转换

```bash
# CSV → TSV
sed 's/,/\t/g' data.csv

# CSV → JSON（简易版）
awk -F, 'NR==1{split($0,h); next}
{
    printf "  {"
    for(i=1;i<=NF;i++) {
        gsub(/"/, "", $i)
        printf "\"%s\":\"%s\"", h[i], $i
        if(i<NF) printf ","
    }
    print "}"
}' data.csv
```

---

## 2. JSON 处理（jq）

### 2.1 基础操作

```bash
# 格式化
echo '{"a":1,"b":2}' | jq .

# 提取字段
echo '{"name":"test","port":8080}' | jq '.name'
echo '{"name":"test","port":8080}' | jq -r '.name'    # 无引号

# 嵌套访问
echo '{"db":{"host":"localhost","port":3306}}' | jq '.db.host'

# 数组操作
echo '[1,2,3,4,5]' | jq '.[2]'         # 3
echo '[1,2,3,4,5]' | jq '.[-1]'        # 5
echo '[1,2,3,4,5]' | jq '.[1:3]'       # [2,3]
echo '[1,2,3,4,5]' | jq 'length'       # 5
echo '[1,2,3,4,5]' | jq 'map(. * 2)'   # [2,4,6,8,10]
```

### 2.2 过滤与选择

```bash
# select 过滤
cat << 'EOF' | jq '.[] | select(.status == "active")'
[
  {"name": "web01", "status": "active", "cpu": 45},
  {"name": "web02", "status": "down", "cpu": 0},
  {"name": "db01", "status": "active", "cpu": 78}
]
EOF

# 多条件
jq '.[] | select(.cpu > 50 and .status == "active")' servers.json

# 提取特定字段
jq '.[] | {name, cpu}' servers.json
jq '[.[] | {name, cpu}]' servers.json    # 包在数组中
```

### 2.3 修改与构造

```bash
# 修改字段
echo '{"name":"test","port":8080}' | jq '.port = 9090'

# 添加字段
echo '{"name":"test"}' | jq '. + {"port": 8080}'

# 删除字段
echo '{"name":"test","tmp":"del"}' | jq 'del(.tmp)'

# 从变量构造
jq -n \
    --arg name "myapp" \
    --argjson port 8080 \
    --arg host "localhost" \
    '{name: $name, port: $port, host: $host}'
```

### 2.4 高级用法

```bash
# 分组统计
jq 'group_by(.status) | map({status: .[0].status, count: length})' servers.json

# 排序
jq 'sort_by(.cpu) | reverse' servers.json

# 转换为 CSV
jq -r '.[] | [.name, .status, .cpu] | @csv' servers.json

# 转换为 TSV
jq -r '.[] | [.name, .status, .cpu] | @tsv' servers.json
```

---

## 3. 实战：API 数据处理

```bash
#!/usr/bin/env bash
# api_report.sh — 从 API 获取数据并生成报告

set -euo pipefail

API_URL="${1:?用法: $0 <API_URL>}"

# 获取数据
DATA=$(curl -sf "$API_URL") || { echo "API 请求失败"; exit 1; }

# 解析
TOTAL=$(echo "$DATA" | jq 'length')
ACTIVE=$(echo "$DATA" | jq '[.[] | select(.status=="active")] | length')
DOWN=$(echo "$DATA" | jq '[.[] | select(.status=="down")] | length')

echo "服务器状态报告"
echo "==============="
echo "总数: $TOTAL"
echo "活跃: $ACTIVE"
echo "宕机: $DOWN"
echo ""

echo "宕机服务器列表:"
echo "$DATA" | jq -r '.[] | select(.status=="down") | "  \(.name) (\(.ip))"'

echo ""
echo "CPU 使用率 Top 5:"
echo "$DATA" | jq -r '.[] | select(.status=="active") | "\(.cpu)% \(.name)"' | sort -rn | head -5
```

---

## 4. 小结

| 工具 | 适用场景 |
|------|----------|
| `awk -F,` | 简单 CSV 处理 |
| `jq` | JSON 解析首选 |
| `csvtool` | 复杂 CSV（带引号） |
| `jq -r @csv` | JSON → CSV 转换 |

> **下一篇**：[数据统计与报表](04-数据统计与报表.md) — 用 Shell 生成数据报表。
