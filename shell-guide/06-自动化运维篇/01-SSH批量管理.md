# SSH 批量管理

> 一句话说明：用 Shell 实现 SSH 免密登录配置、批量命令执行和文件分发。

---

## 1. SSH 密钥配置

```bash
# 生成密钥对
ssh-keygen -t ed25519 -C "ops@company.com"
# 或
ssh-keygen -t rsa -b 4096 -C "ops@company.com"

# 分发公钥
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# 批量分发（脚本）
distribute_keys() {
    local KEY_FILE="$HOME/.ssh/id_ed25519.pub"
    local SERVERS=("$@")
    for server in "${SERVERS[@]}"; do
        echo "分发到: $server"
        ssh-copy-id -i "$KEY_FILE" "$server" 2>/dev/null && echo "  成功" || echo "  失败"
    done
}

# SSH 配置 (~/.ssh/config)
cat >> ~/.ssh/config << 'EOF'
Host web-*
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    ConnectTimeout 5

Host web-01
    HostName 192.168.1.10

Host web-02
    HostName 192.168.1.11

Host jump
    HostName 10.0.0.1
    User admin

Host internal-*
    ProxyJump jump
EOF
# 使用：ssh web-01（自动用配置中的参数）
```

---

## 2. 批量命令执行

```bash
#!/usr/bin/env bash
# batch_exec.sh — 批量远程执行命令

set -euo pipefail

SERVERS_FILE="${1:?用法: $0 <服务器列表> <命令>}"
shift
COMMAND="$*"
[[ -z "$COMMAND" ]] && { echo "请指定命令"; exit 1; }

TIMEOUT=10
SUCCESS=0
FAIL=0

while read -r server; do
    [[ -z "$server" || "$server" =~ ^# ]] && continue
    echo ">>> $server"
    if ssh -o ConnectTimeout="$TIMEOUT" -o BatchMode=yes "$server" "$COMMAND" 2>&1; then
        ((SUCCESS++))
    else
        echo "  [失败]"
        ((FAIL++))
    fi
    echo ""
done < "$SERVERS_FILE"

echo "=============================="
echo "成功: $SUCCESS  失败: $FAIL"
```

### 并行执行版本

```bash
#!/usr/bin/env bash
# parallel_exec.sh — 并行远程执行

set -euo pipefail

SERVERS_FILE="$1"; shift
COMMAND="$*"
MAX_PARALLEL=10
RESULT_DIR=$(mktemp -d)

trap 'rm -rf "$RESULT_DIR"' EXIT

run_on_host() {
    local host="$1" cmd="$2" outfile="$3"
    if ssh -o ConnectTimeout=5 -o BatchMode=yes "$host" "$cmd" > "$outfile" 2>&1; then
        echo "PASS" >> "$outfile.status"
    else
        echo "FAIL" >> "$outfile.status"
    fi
}

# 并行执行
PIDS=()
while read -r server; do
    [[ -z "$server" || "$server" =~ ^# ]] && continue

    while (( $(jobs -rp | wc -l) >= MAX_PARALLEL )); do
        wait -n 2>/dev/null || true
    done

    run_on_host "$server" "$COMMAND" "$RESULT_DIR/$server" &
    PIDS+=($!)
done < "$SERVERS_FILE"

wait

# 汇总结果
for server_file in "$RESULT_DIR"/*.status; do
    server=$(basename "${server_file%.status}")
    status=$(cat "$server_file")
    echo "[$status] $server"
    [[ "$status" == "FAIL" ]] && cat "$RESULT_DIR/$server" | sed 's/^/  /'
done
```

---

## 3. 文件分发

```bash
# 单文件
scp local_file user@host:/remote/path/

# 批量分发
batch_scp() {
    local file="$1" dest="$2" servers_file="$3"
    while read -r server; do
        [[ -z "$server" || "$server" =~ ^# ]] && continue
        echo -n "$server: "
        scp -q "$file" "${server}:${dest}" && echo "成功" || echo "失败"
    done < "$servers_file"
}

# rsync 批量同步（增量，更高效）
batch_rsync() {
    local src="$1" dest="$2" servers_file="$3"
    while read -r server; do
        [[ -z "$server" || "$server" =~ ^# ]] && continue
        echo "同步到 $server..."
        rsync -avz --delete "$src" "${server}:${dest}"
    done < "$servers_file"
}
```

---

## 4. 实战：跳板机管理

```bash
#!/usr/bin/env bash
# ssh_manager.sh — SSH 连接管理器

set -euo pipefail

# 服务器清单
declare -A SERVERS=(
    [web01]="deploy@192.168.1.10"
    [web02]="deploy@192.168.1.11"
    [db01]="dba@192.168.1.20"
    [redis01]="admin@192.168.1.30"
)

case "${1:-menu}" in
    list)
        printf "%-10s %s\n" "别名" "地址"
        printf "%-10s %s\n" "----" "----"
        for name in $(echo "${!SERVERS[@]}" | tr ' ' '\n' | sort); do
            printf "%-10s %s\n" "$name" "${SERVERS[$name]}"
        done
        ;;
    connect)
        NAME="${2:?请指定服务器别名}"
        ADDR="${SERVERS[$NAME]:-}"
        [[ -z "$ADDR" ]] && { echo "未知服务器: $NAME"; exit 1; }
        echo "连接到 $NAME ($ADDR)..."
        ssh "$ADDR"
        ;;
    exec)
        NAME="${2:?请指定服务器别名}"
        shift 2
        ADDR="${SERVERS[$NAME]:-}"
        [[ -z "$ADDR" ]] && { echo "未知服务器: $NAME"; exit 1; }
        ssh "$ADDR" "$@"
        ;;
    menu)
        echo "SSH 管理器"
        select name in $(echo "${!SERVERS[@]}" | tr ' ' '\n' | sort) "退出"; do
            [[ "$name" == "退出" ]] && break
            [[ -n "$name" ]] && ssh "${SERVERS[$name]}"
        done
        ;;
esac
```

---

## 5. 小结

| 功能 | 方法 |
|------|------|
| 免密登录 | `ssh-keygen` + `ssh-copy-id` |
| SSH 配置 | `~/.ssh/config` 简化连接 |
| 批量执行 | 循环 + ssh，并行用 `&` + `wait` |
| 文件分发 | `scp` 单次，`rsync` 增量同步 |
| 跳板机 | `ProxyJump` 配置透明跳转 |

> **下一篇**：[自动化部署脚本](02-自动化部署脚本.md) — 用 Shell 实现应用自动部署。
