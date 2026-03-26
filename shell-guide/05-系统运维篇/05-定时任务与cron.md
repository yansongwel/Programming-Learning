# 定时任务与 cron

> 一句话说明：掌握 cron 和 at 定时任务的配置与管理，实现运维自动化调度。

---

## 1. cron 基础

### 1.1 cron 表达式

```
┌───────────── 分 (0-59)
│ ┌───────────── 时 (0-23)
│ │ ┌───────────── 日 (1-31)
│ │ │ ┌───────────── 月 (1-12)
│ │ │ │ ┌───────────── 星期 (0-7, 0和7都是周日)
│ │ │ │ │
* * * * *  command
```

```bash
# 常见示例
*/5 * * * *     # 每 5 分钟
0 * * * *       # 每小时整点
0 2 * * *       # 每天凌晨 2 点
0 2 * * 1       # 每周一凌晨 2 点
0 0 1 * *       # 每月 1 号零点
0 2 * * 1-5     # 工作日凌晨 2 点
0 */2 * * *     # 每 2 小时
30 8-18 * * 1-5 # 工作日 8:30-18:30 每小时的第 30 分钟
```

### 1.2 crontab 管理

```bash
crontab -e                        # 编辑当前用户的定时任务
crontab -l                        # 查看
crontab -r                        # 删除所有（危险！）
crontab -l -u username            # 查看指定用户

# 系统级 cron
ls /etc/cron.d/                   # 系统定时任务目录
ls /etc/cron.daily/               # 每日任务
ls /etc/cron.weekly/              # 每周任务
ls /etc/cron.monthly/             # 每月任务
```

---

## 2. cron 最佳实践

```bash
# 完整的 cron 任务模板
# 1. 指定 PATH
# 2. 重定向输出
# 3. 使用绝对路径
# 4. 加文件锁防止重叠

PATH=/usr/local/bin:/usr/bin:/bin
SHELL=/bin/bash

# 每天 2 点备份，日志记录，防重叠执行
0 2 * * * /usr/bin/flock -n /tmp/backup.lock /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# 每 5 分钟健康检查
*/5 * * * * /opt/scripts/health_check.sh >> /var/log/health.log 2>&1

# 每周日清理日志
0 3 * * 0 find /var/log/myapp -name "*.log" -mtime +30 -delete
```

### 常见坑

```bash
# 坑 1：环境变量不同
# cron 的环境变量很少，PATH 可能不包含你需要的
# 解决：在脚本中指定完整路径或设置 PATH

# 坑 2：% 有特殊含义
# cron 中 % 会被解释为换行，需要转义
0 1 * * * /usr/bin/date +\%Y\%m\%d > /tmp/date.txt

# 坑 3：不输出导致邮件堆积
# 确保重定向 stdout 和 stderr
0 * * * * command > /dev/null 2>&1

# 坑 4：任务重叠执行
# 使用 flock 防止
*/5 * * * * flock -n /tmp/task.lock command
```

---

## 3. at — 一次性任务

```bash
# 安排一次性任务
at now + 5 minutes << 'EOF'
echo "5 分钟后执行" | mail -s "提醒" admin@example.com
EOF

at 02:00 tomorrow << 'EOF'
/opt/scripts/maintenance.sh
EOF

# 管理
atq                              # 查看待执行任务
atrm 1                           # 删除任务
```

---

## 4. 实战：定时任务管理脚本

```bash
#!/usr/bin/env bash
# cron_manager.sh — 安全管理 cron 任务

set -euo pipefail

ACTION="${1:?用法: $0 <list|add|remove|backup|restore>}"

case "$ACTION" in
    list)
        echo "当前用户 cron 任务:"
        crontab -l 2>/dev/null || echo "  (无)"
        ;;
    add)
        SCHEDULE="${2:?请指定 cron 表达式}"
        COMMAND="${3:?请指定命令}"
        # 备份后追加
        crontab -l 2>/dev/null > /tmp/cron_backup_$$ || true
        echo "$SCHEDULE $COMMAND" >> /tmp/cron_backup_$$
        crontab /tmp/cron_backup_$$
        rm /tmp/cron_backup_$$
        echo "已添加: $SCHEDULE $COMMAND"
        ;;
    backup)
        BACKUP="/tmp/crontab_$(whoami)_$(date +%Y%m%d).bak"
        crontab -l > "$BACKUP" 2>/dev/null
        echo "已备份: $BACKUP"
        ;;
    restore)
        FILE="${2:?请指定备份文件}"
        [[ -f "$FILE" ]] || { echo "文件不存在"; exit 1; }
        crontab "$FILE"
        echo "已恢复"
        ;;
esac
```

---

## 5. 小结

| 功能 | 方式 |
|------|------|
| 重复任务 | cron (`crontab -e`) |
| 一次性任务 | at (`at now + 5min`) |
| 系统级定时 | `/etc/cron.d/` 或 systemd timer |
| 防止重叠 | `flock -n lockfile command` |
| 输出处理 | 重定向到日志文件 `>> log 2>&1` |

> **下一篇**：[系统安全加固](06-系统安全加固.md) — 用 Shell 进行安全基线配置。
