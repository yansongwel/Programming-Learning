# 服务管理与 systemd

> 一句话说明：掌握 systemd 服务管理、自定义 Unit 文件编写和系统启动管理。

---

## 1. systemctl 基本操作

```bash
# 服务管理
systemctl start nginx             # 启动
systemctl stop nginx              # 停止
systemctl restart nginx           # 重启
systemctl reload nginx            # 重载配置（不中断服务）
systemctl status nginx            # 查看状态

# 开机启动
systemctl enable nginx            # 开机自启
systemctl disable nginx           # 取消自启
systemctl is-enabled nginx        # 查看是否自启
systemctl is-active nginx         # 查看是否运行

# 查看所有服务
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service    # 查看所有启用状态

# 查看失败的服务
systemctl --failed
```

---

## 2. 自定义 Service 文件

```bash
# /etc/systemd/system/myapp.service
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StartLimitBurst=3
StartLimitIntervalSec=60

# 安全加固
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/log/myapp /var/lib/myapp

# 资源限制
LimitNOFILE=65535
MemoryMax=512M

# 环境变量
EnvironmentFile=/etc/myapp/env
Environment="LOG_LEVEL=info"

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
EOF

# 重载 daemon 并启动
systemctl daemon-reload
systemctl enable --now myapp
```

### Service Type 说明

| Type | 说明 | 适用场景 |
|------|------|----------|
| simple | 主进程即服务进程（默认） | 前台运行的程序 |
| forking | 主进程 fork 后退出 | 传统 daemon |
| oneshot | 执行一次就退出 | 初始化脚本 |
| notify | 进程主动通知 systemd 就绪 | 支持 sd_notify 的程序 |

---

## 3. Timer 定时器（替代 cron）

```bash
# /etc/systemd/system/backup.timer
cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
EOF

# /etc/systemd/system/backup.service
cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
EOF

systemctl daemon-reload
systemctl enable --now backup.timer
systemctl list-timers                  # 查看所有定时器
```

---

## 4. 日志管理（journalctl）

```bash
# 查看服务日志
journalctl -u nginx                    # 全部日志
journalctl -u nginx -f                 # 实时跟踪
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since today
journalctl -u nginx -p err             # 只看错误
journalctl -u nginx -n 100             # 最近 100 条

# 系统日志
journalctl -b                          # 本次启动
journalctl -b -1                       # 上次启动
journalctl --disk-usage                # 日志占用空间
journalctl --vacuum-size=200M          # 清理到 200M
```

---

## 5. 实战：服务部署脚本

```bash
#!/usr/bin/env bash
# deploy_service.sh — 部署应用为 systemd 服务

set -euo pipefail

APP_NAME="${1:?用法: $0 <应用名> <可执行文件路径>}"
APP_BIN="${2:?请指定可执行文件路径}"
APP_USER="${APP_NAME}"
SERVICE_FILE="/etc/systemd/system/${APP_NAME}.service"

# 创建用户
if ! id "$APP_USER" &>/dev/null; then
    useradd -r -s /sbin/nologin "$APP_USER"
    echo "创建用户: $APP_USER"
fi

# 创建目录
mkdir -p "/var/log/${APP_NAME}" "/var/lib/${APP_NAME}"
chown "$APP_USER:$APP_USER" "/var/log/${APP_NAME}" "/var/lib/${APP_NAME}"

# 生成 Service 文件
cat > "$SERVICE_FILE" << EOF
[Unit]
Description=${APP_NAME} service
After=network.target

[Service]
Type=simple
User=${APP_USER}
ExecStart=${APP_BIN}
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=${APP_NAME}
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now "$APP_NAME"

echo "服务已部署: $APP_NAME"
systemctl status "$APP_NAME" --no-pager
```

---

## 6. 小结

| 操作 | 命令 |
|------|------|
| 服务控制 | `systemctl start/stop/restart/reload` |
| 开机自启 | `systemctl enable/disable` |
| 自定义服务 | `/etc/systemd/system/*.service` |
| 定时任务 | `.timer` + `.service` |
| 日志查看 | `journalctl -u service -f` |

> **下一篇**：[定时任务与cron](05-定时任务与cron.md) — 掌握任务调度。
