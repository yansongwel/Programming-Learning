# Python SRE 运维实战学习指南

> 面向有运维经验的 SRE 工程师，从 Python 基础到高级运维实战的完整学习路径。
> 所有代码示例均经过 Python 3.13 验证，可直接复制运行。

---

## 学习路线图

```
第一阶段：Python 基础          第二阶段：Python 进阶          第三阶段：SRE 运维库
┌─────────────────┐        ┌─────────────────┐        ┌──────────────────────┐
│ 01 环境搭建      │        │ 01 面向对象      │        │ 系统监控 (psutil)     │
│ 02 基础语法      │───────>│ 02 装饰器       │───────>│ 远程操作 (paramiko)   │
│ 03 函数与模块    │        │ 03 并发编程      │        │ HTTP/API (requests)   │
│ 04 文件与IO      │        │ 04 subprocess   │        │ 数据库 (Redis/MySQL)  │
│ 05 异常与日志    │        │ 05 CLI开发       │        │ 监控 (Prometheus)     │
│ 06 数据结构      │        └─────────────────┘        │ 容器 (Docker/K8s)     │
└─────────────────┘                                    │ 云平台 (AWS/GCP/Azure)│
                                                       └──────────────────────┘
                                                                 │
                                                                 v
                                                       ┌──────────────────────┐
                                                       │ 第四阶段：实战项目     │
                                                       │ 资产采集 / 日志分析   │
                                                       │ 自动部署 / 监控大盘   │
                                                       └──────────────────────┘
```

---

## 目录

### 01-Python基础篇

从零开始，每个概念对比 Shell/Bash 写法，让运维工程师快速上手。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [环境搭建](01-Python基础篇/01-环境搭建.md) | Python 安装、pyenv、venv、pip、国内镜像 |
| 02 | [基础语法](01-Python基础篇/02-基础语法.md) | 变量、字符串、列表、字典、条件、循环（对比 Shell） |
| 03 | [函数与模块](01-Python基础篇/03-函数与模块.md) | 函数定义、参数、模块导入、标准库概览 |
| 04 | [文件与IO操作](01-Python基础篇/04-文件与IO操作.md) | 文件读写、pathlib、shutil（对比 cat/cp/mv） |
| 05 | [异常处理与日志](01-Python基础篇/05-异常处理与日志.md) | try/except、logging、loguru（对比 set -e/trap） |
| 06 | [数据结构进阶](01-Python基础篇/06-数据结构进阶.md) | collections、Counter 分析日志、数据结构选择指南 |

### 02-Python进阶篇

掌握 Python 高级特性，为编写专业运维工具打基础。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [面向对象编程](02-Python进阶篇/01-面向对象编程.md) | 类、继承、魔术方法、dataclass |
| 02 | [装饰器与上下文管理器](02-Python进阶篇/02-装饰器与上下文管理器.md) | 装饰器原理、计时/重试/限流实现 |
| 03 | [并发编程](02-Python进阶篇/03-并发编程.md) | 多线程、多进程、asyncio、批量端口扫描 |
| 04 | [subprocess与Shell交互](02-Python进阶篇/04-subprocess与Shell交互.md) | 替代 Shell 脚本的核心章节、安全实践 |
| 05 | [CLI工具开发与打包](02-Python进阶篇/05-CLI工具开发与打包.md) | argparse、click、typer、rich、打包分发 |

### 03-系统与操作系统

用 Python 替代日常系统运维命令。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [psutil系统监控](03-系统与操作系统/01-psutil系统监控.md) | CPU/内存/磁盘/进程监控（对比 top/free/df） |
| 02 | [文件系统操作](03-系统与操作系统/02-文件系统操作.md) | 目录遍历、文件管理、watchdog（对比 find/cp/rm） |
| 03 | [Socket网络编程](03-系统与操作系统/03-Socket网络编程.md) | TCP/UDP、端口检测、健康检查（对比 nc/telnet） |

### 04-远程操作与自动化

批量管理服务器，告别 for+ssh 循环。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [paramiko远程连接](04-远程操作与自动化/01-paramiko远程连接.md) | SSH、SFTP、跳板机、批量执行 |
| 02 | [fabric批量部署](04-远程操作与自动化/02-fabric批量部署.md) | 批量部署、任务编排、滚动发布 |
| 03 | [Ansible的Python-API](04-远程操作与自动化/03-Ansible的Python-API.md) | Python 调用 Ansible、动态 Inventory |

### 05-HTTP与API开发

与各类 API 交互，构建运维接口。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [requests与httpx](05-HTTP与API开发/01-requests与httpx.md) | HTTP 请求、健康检查（对比 curl） |
| 02 | [FastAPI运维接口](05-HTTP与API开发/02-FastAPI运维接口.md) | 运维 API、Webhook、自动文档 |
| 03 | [aiohttp异步HTTP](05-HTTP与API开发/03-aiohttp异步HTTP.md) | 异步并发探测、WebSocket |

### 06-数据库操作

运维场景下的数据库编程。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [MySQL与SQLAlchemy](06-数据库操作/01-MySQL与SQLAlchemy.md) | pymysql、ORM、数据库巡检 |
| 02 | [Redis操作](06-数据库操作/02-Redis操作.md) | 数据结构、分布式锁、任务队列 |
| 03 | [Elasticsearch日志查询](06-数据库操作/03-Elasticsearch日志查询.md) | DSL 查询、聚合分析、日志搜索 |
| 04 | [MongoDB操作](06-数据库操作/04-MongoDB操作.md) | CRUD、聚合管道、变更记录存储 |

### 07-监控与告警

构建可观测性体系。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [Prometheus客户端](07-监控与告警/01-Prometheus客户端.md) | 四种指标类型、自定义 Exporter |
| 02 | [Datadog与StatsD](07-监控与告警/02-Datadog与StatsD.md) | 指标上报、APM、事件告警 |
| 03 | [Sentry错误追踪](07-监控与告警/03-Sentry错误追踪.md) | 异常捕获、性能监控、Release 追踪 |

### 08-容器与Kubernetes

容器化时代的运维必备。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [Docker-SDK](08-容器与Kubernetes/01-Docker-SDK.md) | 容器/镜像管理、自动健康检查 |
| 02 | [Kubernetes客户端](08-容器与Kubernetes/02-Kubernetes客户端.md) | K8s API、Watch 事件、Pod 诊断 |

### 09-云平台SDK

多云资源管理。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [boto3-AWS](09-云平台SDK/01-boto3-AWS.md) | EC2/S3/Lambda/CloudWatch |
| 02 | [GCP-Python-SDK](09-云平台SDK/02-GCP-Python-SDK.md) | Compute/Storage/Logging/BigQuery |
| 03 | [Azure-SDK](09-云平台SDK/03-Azure-SDK.md) | VM/Blob/Monitor/Network |

### 10-配置与模板

配置管理自动化。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [YAML-TOML-JSON解析](10-配置与模板/01-YAML-TOML-JSON解析.md) | 多格式解析、多环境配置合并 |
| 02 | [Jinja2模板渲染](10-配置与模板/02-Jinja2模板渲染.md) | 批量生成 Nginx/Prometheus 配置 |

### 11-日志与数据分析

从日志中挖掘价值。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [日志体系详解](11-日志与数据分析/01-日志体系详解.md) | logging 深入、结构化日志、ELK 对接 |
| 02 | [正则表达式](11-日志与数据分析/02-正则表达式.md) | re 模块、解析 Nginx/syslog 日志 |
| 03 | [Pandas运维数据分析](11-日志与数据分析/03-Pandas运维数据分析.md) | DataFrame、日志分析报表 |

### 12-网络与安全

网络诊断与安全运维。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [DNS查询](12-网络与安全/01-DNS查询.md) | dnspython、DNS 监控告警 |
| 02 | [Scapy网络分析](12-网络与安全/02-Scapy网络分析.md) | 抓包、协议分析（对比 tcpdump） |
| 03 | [IP地址与子网计算](12-网络与安全/03-IP地址与子网计算.md) | CIDR、IP 白名单管理 |
| 04 | [加密与证书管理](12-网络与安全/04-加密与证书管理.md) | SSL 证书巡检、加解密 |

### 13-CICD与Git

CI/CD 流水线自动化。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [GitPython仓库操作](13-CICD与Git/01-GitPython仓库操作.md) | 代码变更审计 |
| 02 | [Jenkins-API](13-CICD与Git/02-Jenkins-API.md) | Job 管理、Pipeline 自动化 |
| 03 | [GitHub与GitLab-API](13-CICD与Git/03-GitHub与GitLab-API.md) | PR/MR 自动化、Webhook |

### 14-任务调度

定时任务与分布式执行。

| 序号 | 文档 | 内容概要 |
|------|------|----------|
| 01 | [Celery分布式任务](14-任务调度/01-Celery分布式任务.md) | Worker、Beat、任务链、Flower |
| 02 | [APScheduler定时任务](14-任务调度/02-APScheduler定时任务.md) | cron/interval 触发器（对比 crontab） |

### 15-实战项目篇

综合运用所学知识的完整项目。

| 序号 | 文档 | 内容概要 | 涉及库 |
|------|------|----------|--------|
| 01 | [服务器资产采集工具](15-实战项目篇/01-服务器资产采集工具.md) | 批量采集服务器信息入库 | paramiko, psutil, SQLAlchemy, rich, typer |
| 02 | [日志分析平台](15-实战项目篇/02-日志分析平台.md) | 日志解析、分析、查询、报告 | re, pandas, elasticsearch, FastAPI, jinja2 |
| 03 | [自动化部署机器人](15-实战项目篇/03-自动化部署机器人.md) | 一键部署、回滚、进度追踪 | fabric, gitpython, docker, redis, FastAPI |
| 04 | [自定义监控大盘](15-实战项目篇/04-自定义监控大盘.md) | 多源指标采集与展示 | psutil, prometheus_client, kubernetes, boto3, rich |

---

## 推荐学习顺序

### 快速路线（2 周）
适合急需上手的运维工程师：
1. 01 基础篇 → 02 进阶篇（重点：04-subprocess）
2. 03-01 psutil → 04-01 paramiko → 05-01 requests
3. 15-01 服务器资产采集（实战巩固）

### 标准路线（4-6 周）
系统学习：
1. 01 基础篇全部 → 02 进阶篇全部
2. 03-04 系统与远程 → 05-06 HTTP与数据库
3. 10-11 配置与日志 → 15 实战项目

### 完整路线（8-12 周）
全面掌握：按目录顺序 01 → 15 完整学习

---

## 环境要求

- **Python**: 3.10+（推荐 3.13，本文档基于 3.13.12 验证）
- **操作系统**: Linux（Ubuntu/CentOS）、macOS
- **包管理**: pip + venv
- **版本管理**: pyenv（推荐）

## 文档约定

- `Shell 对比` — 每个概念附 Shell/Bash 等价写法
- `SRE 实战` — 完整可运行的运维场景代码
- `常见坑点` — 踩过的坑和 Do's/Don'ts
- `速查表` — 常用方法速查

---

> 共 15 个章节、50 篇文档、56,000+ 行内容。所有代码示例均经过 Python 3.13.12 验证。
