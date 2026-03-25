# MongoDB 操作 — Python SRE 的文档数据库指南

> 一句话说明：用 `pymongo` 操作 MongoDB，掌握 CRUD、查询语法、索引、聚合管道、GridFS 大文件存储，实战用 MongoDB 存储运维事件与变更记录。

---

## 1. 是什么 & 为什么用它

### 对比 Shell 操作 MongoDB

```bash
# Shell：用 mongosh/mongo 命令行
mongosh --eval 'db.servers.find({status: "online"})'

# Shell：插入数据
mongosh --eval 'db.events.insertOne({type: "deploy", service: "nginx", time: new Date()})'

# Shell：导出数据
mongoexport --db=sre --collection=events --out=/tmp/events.json

# Shell：查询并格式化
mongosh --eval 'db.events.find({level: "error"}).sort({time: -1}).limit(10).pretty()'

# 缺点：
# 1. JS 语法不熟悉（对运维人员）
# 2. 复杂查询/聚合难以构建
# 3. 结果处理不方便
# 4. 无连接池
# 5. 批量操作效率低
```

### 为什么 SRE 用 MongoDB

| 场景 | MySQL | MongoDB | 推荐 |
|------|-------|---------|------|
| 运维事件日志 | 需预定义 Schema | 灵活 Schema | MongoDB |
| 配置管理 (CMDB) | 字段变化需 DDL | 动态字段 | MongoDB |
| 变更记录 | 固定结构 | 嵌套文档 | MongoDB |
| 监控时序数据 | 不擅长 | TTL + 时序 | MongoDB |
| 运维审计 | 可以 | 更灵活 | 都可以 |

**MongoDB 核心优势：Schema-less（无模式）** — 不同事件可以有不同字段，不需要 ALTER TABLE。

---

## 2. 安装与环境

```bash
# 安装 pymongo
pip install pymongo

# 如果需要 BSON 扩展（性能）
pip install pymongo[encryption]

# 验证安装
python3 -c "import pymongo; print(f'pymongo: {pymongo.version}')"

# Docker 快速启动 MongoDB
# docker run -d --name mongo-test -p 27017:27017 \
#   -e MONGO_INITDB_ROOT_USERNAME=admin \
#   -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
#   mongo:7

# 无认证版（测试用）
# docker run -d --name mongo-test -p 27017:27017 mongo:7
```

---

## 3. 核心概念与原理

### 概念对照

```
关系数据库           MongoDB
──────────          ──────────
Database            Database（数据库）
Table               Collection（集合）
Row                 Document（文档，BSON/JSON）
Column              Field（字段）
Primary Key         _id（自动生成 ObjectId）
Index               Index（索引）
JOIN                $lookup（聚合管道）
SQL                 查询表达式（字典）
```

### 文档结构示例

```json
{
  "_id": "ObjectId('...')",        // 自动生成的唯一 ID
  "event_type": "deploy",          // 字符串
  "service": "nginx",
  "operator": "admin",
  "timestamp": "ISODate('...')",   // 日期
  "details": {                     // 嵌套文档
    "version": "1.2.3",
    "env": "production"
  },
  "affected_hosts": ["web-01", "web-02"],  // 数组
  "tags": ["production", "nginx"],
  "duration_seconds": 120          // 数字
}
```

### pymongo 架构

```
pymongo.MongoClient（客户端，内置连接池）
    |
    +-- client["database_name"]（数据库）
    |       |
    |       +-- db["collection_name"]（集合）
    |               |
    |               +-- insert_one / find / update_one / delete_one
    |               +-- aggregate（聚合管道）
    |               +-- create_index（索引管理）
    |
    +-- 连接池管理
    +-- 自动重连
    +-- 读写偏好
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 连接与基本操作

```python
#!/usr/bin/env python3
"""MongoDB 连接与基础操作"""

from pymongo import MongoClient
from datetime import datetime

# ========== 连接 MongoDB ==========

# 方式 1：直接连接
client = MongoClient(
    host="127.0.0.1",
    port=27017,
    # username="admin",        # 如果有认证
    # password="admin123",
    maxPoolSize=50,            # 连接池最大连接数
    minPoolSize=5,             # 最小连接数
    serverSelectionTimeoutMS=5000,  # 服务发现超时
    connectTimeoutMS=5000,     # 连接超时
    socketTimeoutMS=30000,     # Socket 超时
)

# 方式 2：URI 连接
# client = MongoClient("mongodb://admin:admin123@127.0.0.1:27017/")

# 方式 3：副本集连接
# client = MongoClient("mongodb://host1:27017,host2:27017/?replicaSet=rs0")

# 测试连接
try:
    client.admin.command("ping")
    print("MongoDB 连接成功")
except Exception as e:
    print(f"连接失败: {e}")

# 查看服务器信息
server_info = client.server_info()
print(f"MongoDB 版本: {server_info['version']}")

# ========== 选择数据库和集合 ==========
db = client["sre_db"]              # 选择/创建数据库
collection = db["servers"]          # 选择/创建集合

# 查看已有数据库
print(f"数据库列表: {client.list_database_names()}")
# 查看已有集合
print(f"集合列表: {db.list_collection_names()}")
```

### 4.2 CRUD 操作

```python
#!/usr/bin/env python3
"""MongoDB CRUD 操作"""

from pymongo import MongoClient
from datetime import datetime
from bson import ObjectId

client = MongoClient("127.0.0.1", 27017)
db = client["sre_db"]
servers = db["servers"]

# 先清空集合（测试用）
servers.drop()

# ========== CREATE 插入 ==========

# 插入单条
# 等价于: db.servers.insertOne({...})
doc = {
    "hostname": "web-01",
    "ip": "10.0.1.1",
    "cpu_cores": 8,
    "memory_gb": 16,
    "status": "online",
    "region": "cn-east",
    "tags": ["production", "web"],
    "os_info": {
        "name": "Ubuntu",
        "version": "22.04",
        "kernel": "5.15.0"
    },
    "created_at": datetime.now(),
    "updated_at": datetime.now()
}
result = servers.insert_one(doc)
print(f"插入 ID: {result.inserted_id}")

# 批量插入
docs = [
    {
        "hostname": "web-02", "ip": "10.0.1.2",
        "cpu_cores": 8, "memory_gb": 16,
        "status": "online", "region": "cn-east",
        "tags": ["production", "web"],
        "created_at": datetime.now()
    },
    {
        "hostname": "db-01", "ip": "10.0.2.1",
        "cpu_cores": 16, "memory_gb": 64,
        "status": "online", "region": "cn-north",
        "tags": ["production", "database"],
        "created_at": datetime.now()
    },
    {
        "hostname": "cache-01", "ip": "10.0.3.1",
        "cpu_cores": 4, "memory_gb": 32,
        "status": "offline", "region": "cn-east",
        "tags": ["production", "cache"],
        "created_at": datetime.now()
    },
    {
        "hostname": "monitor-01", "ip": "10.0.4.1",
        "cpu_cores": 4, "memory_gb": 8,
        "status": "online", "region": "cn-east",
        "tags": ["infra", "monitoring"],
        "created_at": datetime.now()
    },
]
result = servers.insert_many(docs)
print(f"批量插入 {len(result.inserted_ids)} 条")

# ========== READ 查询 ==========

# 查询单条
server = servers.find_one({"hostname": "web-01"})
print(f"\n查询 web-01: {server['ip']} ({server['status']})")

# 查询多条（条件查询）
# 等价于: db.servers.find({status: "online"})
print("\n在线服务器:")
for s in servers.find({"status": "online"}):
    print(f"  {s['hostname']} ({s['ip']}) - {s.get('region')}")

# 条件查询 — 比较操作符
# $gt $gte $lt $lte $ne $in $nin
high_mem = servers.find({"memory_gb": {"$gte": 32}})
print(f"\n内存 >= 32GB: {[s['hostname'] for s in high_mem]}")

# 条件查询 — 逻辑操作符
# $and $or $not
result = servers.find({
    "$or": [
        {"region": "cn-north"},
        {"memory_gb": {"$gte": 32}}
    ]
})
print(f"华北 或 内存>=32G: {[s['hostname'] for s in result]}")

# 数组查询
# $all（包含所有）$in（包含任一）$size（数组长度）
web_servers = servers.find({"tags": "web"})  # 包含 "web" 标签
print(f"Web 服务器: {[s['hostname'] for s in web_servers]}")

# 嵌套文档查询
ubuntu = servers.find({"os_info.name": "Ubuntu"})
print(f"Ubuntu 服务器: {[s['hostname'] for s in ubuntu]}")

# 投影（只返回指定字段）
# 等价于: db.servers.find({}, {hostname: 1, ip: 1, _id: 0})
for s in servers.find({}, {"hostname": 1, "ip": 1, "_id": 0}):
    print(f"  {s}")

# 排序 + 限制 + 跳过（分页）
# 等价于: db.servers.find().sort({memory_gb: -1}).skip(0).limit(3)
page = servers.find().sort("memory_gb", -1).skip(0).limit(3)
print(f"\n内存 Top 3: {[(s['hostname'], s['memory_gb']) for s in page]}")

# 计数
online_count = servers.count_documents({"status": "online"})
print(f"在线数量: {online_count}")

# ========== UPDATE 更新 ==========

# 更新单条
# 等价于: db.servers.updateOne({hostname: "web-01"}, {$set: {status: "maintenance"}})
result = servers.update_one(
    {"hostname": "web-01"},
    {
        "$set": {"status": "maintenance", "updated_at": datetime.now()},
        "$inc": {"restart_count": 1}  # 原子递增
    }
)
print(f"\n更新 {result.modified_count} 条")

# 更新多条
result = servers.update_many(
    {"region": "cn-east"},
    {"$addToSet": {"tags": "cn-east-zone"}}  # 添加标签（不重复）
)
print(f"批量更新 {result.modified_count} 条")

# Upsert（不存在则插入）
result = servers.update_one(
    {"hostname": "new-server-01"},
    {
        "$set": {
            "ip": "10.0.5.1",
            "status": "online",
            "created_at": datetime.now()
        }
    },
    upsert=True
)
print(f"Upsert: {'插入' if result.upserted_id else '更新'}")

# ========== DELETE 删除 ==========

# 删除单条
result = servers.delete_one({"hostname": "new-server-01"})
print(f"\n删除 {result.deleted_count} 条")

# 删除多条
# result = servers.delete_many({"status": "offline"})

# 清理
client.close()
```

### 4.3 索引管理

```python
#!/usr/bin/env python3
"""MongoDB 索引管理"""

from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT

client = MongoClient("127.0.0.1", 27017)
db = client["sre_db"]
events = db["ops_events"]

# ========== 创建索引 ==========

# 单字段索引
events.create_index("timestamp", name="idx_timestamp")

# 复合索引
events.create_index(
    [("service", ASCENDING), ("timestamp", DESCENDING)],
    name="idx_service_time"
)

# 唯一索引
events.create_index("event_id", unique=True, name="idx_event_id", sparse=True)

# TTL 索引（自动过期删除，适合日志/临时数据）
# 30 天后自动删除
events.create_index(
    "created_at",
    expireAfterSeconds=30 * 24 * 3600,  # 30 天
    name="idx_ttl_30d"
)

# 文本索引（全文搜索）
events.create_index(
    [("description", TEXT), ("details", TEXT)],
    name="idx_fulltext",
    default_language="none"  # 禁用语言分析器（中文场景）
)

# ========== 查看索引 ==========
print("当前索引:")
for idx_name, idx_info in events.index_information().items():
    print(f"  {idx_name}: {idx_info['key']}")

# ========== 查询计划（explain） ==========
# 等价于: db.events.find({service: "nginx"}).explain()
explain = events.find({"service": "nginx"}).explain()
# print(f"查询计划: {explain['queryPlanner']['winningPlan']}")

# ========== 删除索引 ==========
# events.drop_index("idx_timestamp")
# events.drop_indexes()  # 删除所有索引（除 _id）

client.close()
```

---

## 5. 进阶用法

### 5.1 聚合管道

```python
#!/usr/bin/env python3
"""MongoDB 聚合管道 — 数据分析利器"""

from pymongo import MongoClient
from datetime import datetime, timedelta
import random

client = MongoClient("127.0.0.1", 27017)
db = client["sre_db"]
events = db["ops_events"]

# 先生成测试数据
events.drop()
event_types = ["deploy", "restart", "scale", "rollback", "config_change", "incident"]
services = ["nginx", "app-server", "mysql", "redis", "kafka"]
operators = ["admin", "sre-bot", "developer-a", "developer-b"]
statuses = ["success", "failed", "partial"]

test_events = []
for i in range(500):
    test_events.append({
        "event_type": random.choice(event_types),
        "service": random.choice(services),
        "operator": random.choice(operators),
        "status": random.choices(statuses, weights=[80, 15, 5])[0],
        "duration_seconds": random.randint(5, 600),
        "affected_hosts": random.sample(
            ["web-01", "web-02", "db-01", "cache-01", "app-01"],
            random.randint(1, 3)
        ),
        "timestamp": datetime.now() - timedelta(
            days=random.randint(0, 30),
            hours=random.randint(0, 23)
        ),
        "details": {
            "env": random.choice(["production", "staging"]),
            "region": random.choice(["cn-east", "cn-north"])
        }
    })
events.insert_many(test_events)
print(f"插入 {len(test_events)} 条测试数据")

# ========== 聚合查询 ==========

# 1. 按操作类型统计
# 等价于: SELECT event_type, COUNT(*) FROM events GROUP BY event_type
pipeline = [
    {"$group": {
        "_id": "$event_type",
        "count": {"$sum": 1},
        "avg_duration": {"$avg": "$duration_seconds"},
        "max_duration": {"$max": "$duration_seconds"}
    }},
    {"$sort": {"count": -1}}
]
print("\n操作类型统计:")
for doc in events.aggregate(pipeline):
    print(f"  {doc['_id']:<15} | 次数: {doc['count']:>3} | "
          f"平均耗时: {doc['avg_duration']:.0f}s")

# 2. 按服务 + 状态交叉统计
pipeline = [
    {"$group": {
        "_id": {"service": "$service", "status": "$status"},
        "count": {"$sum": 1}
    }},
    {"$group": {
        "_id": "$_id.service",
        "statuses": {
            "$push": {
                "status": "$_id.status",
                "count": "$count"
            }
        },
        "total": {"$sum": "$count"}
    }},
    {"$sort": {"total": -1}}
]
print("\n服务状态分布:")
for doc in events.aggregate(pipeline):
    status_str = ", ".join(
        f"{s['status']}={s['count']}" for s in doc["statuses"]
    )
    print(f"  {doc['_id']:<15} | 总计: {doc['total']:>3} | {status_str}")

# 3. 每日操作趋势
pipeline = [
    {"$group": {
        "_id": {
            "$dateToString": {
                "format": "%Y-%m-%d",
                "date": "$timestamp"
            }
        },
        "count": {"$sum": 1},
        "failed": {
            "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
        }
    }},
    {"$sort": {"_id": -1}},
    {"$limit": 7}
]
print("\n每日操作趋势（最近 7 天）:")
for doc in events.aggregate(pipeline):
    bar = "#" * (doc["count"] // 2)
    print(f"  {doc['_id']} | 总: {doc['count']:>3} | 失败: {doc['failed']:>2} | {bar}")

# 4. 操作者排行
pipeline = [
    {"$group": {
        "_id": "$operator",
        "total_ops": {"$sum": 1},
        "services": {"$addToSet": "$service"},
        "failed": {
            "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
        }
    }},
    {"$addFields": {
        "failure_rate": {
            "$round": [
                {"$multiply": [
                    {"$divide": ["$failed", "$total_ops"]}, 100
                ]},
                1
            ]
        }
    }},
    {"$sort": {"total_ops": -1}}
]
print("\n操作者排行:")
for doc in events.aggregate(pipeline):
    print(f"  {doc['_id']:<15} | 操作: {doc['total_ops']:>3} | "
          f"失败率: {doc['failure_rate']:.1f}% | "
          f"涉及服务: {', '.join(doc['services'])}")

# 5. 用 $match 过滤 + $project 投影
pipeline = [
    {"$match": {
        "status": "failed",
        "timestamp": {"$gte": datetime.now() - timedelta(days=7)}
    }},
    {"$project": {
        "event_type": 1,
        "service": 1,
        "operator": 1,
        "duration_seconds": 1,
        "timestamp": 1,
        "env": "$details.env",
        "_id": 0
    }},
    {"$sort": {"timestamp": -1}},
    {"$limit": 10}
]
print("\n最近 7 天失败操作:")
for doc in events.aggregate(pipeline):
    print(f"  {doc['timestamp'].strftime('%m-%d %H:%M')} | "
          f"{doc['event_type']:<15} | {doc['service']} | "
          f"{doc['operator']}")

# 6. $unwind 展开数组
pipeline = [
    {"$unwind": "$affected_hosts"},  # 展开数组，每个元素变一行
    {"$group": {
        "_id": "$affected_hosts",
        "event_count": {"$sum": 1},
        "failed_count": {
            "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
        }
    }},
    {"$sort": {"event_count": -1}}
]
print("\n受影响主机统计:")
for doc in events.aggregate(pipeline):
    print(f"  {doc['_id']:<12} | 事件: {doc['event_count']:>3} | "
          f"失败: {doc['failed_count']:>2}")

client.close()
```

### 5.2 GridFS 大文件存储

```python
#!/usr/bin/env python3
"""GridFS — 存储大文件（日志文件、备份、配置快照）"""

from pymongo import MongoClient
import gridfs
from datetime import datetime

client = MongoClient("127.0.0.1", 27017)
db = client["sre_db"]

# 创建 GridFS 实例
fs = gridfs.GridFS(db, collection="ops_files")

# ========== 存储文件 ==========

# 1. 存储文本内容（如配置文件快照）
config_content = """
# nginx.conf 配置快照
worker_processes auto;
events {
    worker_connections 1024;
}
http {
    upstream backend {
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
}
"""

file_id = fs.put(
    config_content.encode("utf-8"),
    filename="nginx.conf",
    content_type="text/plain",
    metadata={
        "type": "config_snapshot",
        "service": "nginx",
        "server": "web-01",
        "operator": "admin",
        "version": "v1.2.3",
        "snapshot_time": datetime.now()
    }
)
print(f"文件已存储, ID: {file_id}")

# 2. 存储二进制文件
test_data = b"\x00" * 1024 * 100  # 100KB 测试数据
file_id2 = fs.put(
    test_data,
    filename="backup_2024.tar.gz",
    content_type="application/gzip",
    metadata={"type": "backup", "size_bytes": len(test_data)}
)

# ========== 读取文件 ==========

# 根据文件名查找
grid_file = fs.find_one({"filename": "nginx.conf"})
if grid_file:
    content = grid_file.read().decode("utf-8")
    print(f"\n读取文件: {grid_file.filename}")
    print(f"大小: {grid_file.length} bytes")
    print(f"元数据: {grid_file.metadata}")
    print(f"内容前 100 字符: {content[:100]}...")
    grid_file.close()

# ========== 列出文件 ==========
print("\n文件列表:")
for f in fs.find().sort("uploadDate", -1):
    print(f"  {f.filename:<30} | {f.length:>8} bytes | "
          f"{f.upload_date.strftime('%Y-%m-%d %H:%M')}")
    f.close()

# ========== 删除文件 ==========
# fs.delete(file_id)

# ========== 检查文件是否存在 ==========
exists = fs.exists({"filename": "nginx.conf"})
print(f"\nnginx.conf 存在: {exists}")

client.close()
```

### 5.3 Change Stream（变更流）

```python
#!/usr/bin/env python3
"""Change Stream — 实时监听数据变更"""

from pymongo import MongoClient
from datetime import datetime
import threading
import time

client = MongoClient("127.0.0.1", 27017)
db = client["sre_db"]
events = db["ops_events_stream"]

# 注意：Change Stream 需要 MongoDB 副本集或分片集群
# 单机模式不支持。可以用以下命令启动单节点副本集：
# mongosh --eval "rs.initiate()"

def watch_changes():
    """监听集合变更"""
    try:
        # 监听指定集合的变更
        with events.watch(
            pipeline=[
                # 只监听 insert 和 update 操作
                {"$match": {
                    "operationType": {"$in": ["insert", "update", "delete"]}
                }}
            ],
            full_document="updateLookup"  # 更新时返回完整文档
        ) as stream:
            print("[监听器] 开始监听变更...")
            for change in stream:
                op = change["operationType"]
                if op == "insert":
                    doc = change["fullDocument"]
                    print(f"[新增] {doc.get('event_type')}: {doc.get('service')}")
                elif op == "update":
                    doc = change.get("fullDocument", {})
                    print(f"[更新] {doc.get('event_type')}: {change['updateDescription']}")
                elif op == "delete":
                    print(f"[删除] ID: {change['documentKey']['_id']}")
    except Exception as e:
        print(f"[监听器] 错误（可能不支持 Change Stream）: {e}")

# 实际使用方式：
# 1. 在一个进程/线程中运行 watch_changes()
# 2. 在另一个进程中执行数据写入
# watch_thread = threading.Thread(target=watch_changes, daemon=True)
# watch_thread.start()
# time.sleep(2)
# events.insert_one({"event_type": "test", "service": "nginx", "timestamp": datetime.now()})
# time.sleep(2)

print("Change Stream 示例（需要副本集环境）")
client.close()
```

---

## 6. SRE 实战案例：运维事件与变更记录系统

```python
#!/usr/bin/env python3
"""
SRE 实战：基于 MongoDB 的运维事件与变更记录系统
功能：
  1. 记录运维事件（部署、重启、扩容、故障等）
  2. 变更记录查询（按时间、服务、操作人）
  3. 变更统计分析
  4. 配置快照（GridFS）
  5. 事件搜索与报告生成
"""

from pymongo import MongoClient, ASCENDING, DESCENDING
from datetime import datetime, timedelta
from typing import Optional, List, Dict
import json
import uuid


class OpsEventStore:
    """运维事件存储系统"""

    def __init__(self, mongo_uri: str = "mongodb://127.0.0.1:27017/",
                 db_name: str = "sre_ops"):
        """
        初始化事件存储

        参数:
            mongo_uri: MongoDB 连接 URI
            db_name: 数据库名
        """
        self.client = MongoClient(
            mongo_uri,
            maxPoolSize=20,
            serverSelectionTimeoutMS=5000,
        )
        self.db = self.client[db_name]

        # 集合
        self.events = self.db["events"]       # 运维事件
        self.changes = self.db["changes"]     # 变更记录
        self.snapshots = self.db["snapshots"] # 配置快照

        # 初始化索引
        self._ensure_indexes()

    def _ensure_indexes(self):
        """确保索引存在"""
        # 事件表索引
        self.events.create_index(
            [("timestamp", DESCENDING)],
            name="idx_timestamp"
        )
        self.events.create_index(
            [("service", ASCENDING), ("timestamp", DESCENDING)],
            name="idx_service_time"
        )
        self.events.create_index(
            [("event_type", ASCENDING)],
            name="idx_event_type"
        )
        self.events.create_index(
            "timestamp",
            expireAfterSeconds=90 * 24 * 3600,  # 90 天自动过期
            name="idx_ttl_90d"
        )

        # 变更记录索引
        self.changes.create_index(
            [("change_id", ASCENDING)],
            unique=True,
            name="idx_change_id"
        )

    # ========== 事件记录 ==========

    def record_event(
        self,
        event_type: str,
        service: str,
        operator: str,
        description: str,
        status: str = "success",
        details: dict = None,
        affected_hosts: list = None,
        duration_seconds: int = 0,
        tags: list = None,
    ) -> str:
        """
        记录运维事件

        参数:
            event_type: 事件类型（deploy/restart/scale/incident/config_change）
            service: 服务名
            operator: 操作人
            description: 事件描述
            status: 状态（success/failed/partial/in_progress）
            details: 详细信息
            affected_hosts: 影响的主机列表
            duration_seconds: 持续时间
            tags: 标签

        返回:
            str: 事件 ID
        """
        event = {
            "event_id": f"evt-{uuid.uuid4().hex[:8]}",
            "event_type": event_type,
            "service": service,
            "operator": operator,
            "description": description,
            "status": status,
            "details": details or {},
            "affected_hosts": affected_hosts or [],
            "duration_seconds": duration_seconds,
            "tags": tags or [],
            "timestamp": datetime.now(),
            "environment": details.get("env", "production") if details else "production",
        }

        self.events.insert_one(event)
        print(f"[事件] 已记录: {event['event_id']} - {event_type} {service}")
        return event["event_id"]

    # ========== 变更记录 ==========

    def record_change(
        self,
        service: str,
        change_type: str,
        operator: str,
        before: dict,
        after: dict,
        reason: str = "",
    ) -> str:
        """
        记录变更（包含变更前后对比）

        参数:
            service: 服务名
            change_type: 变更类型
            operator: 操作人
            before: 变更前配置/状态
            after: 变更后配置/状态
            reason: 变更原因

        返回:
            str: 变更 ID
        """
        change_id = f"chg-{uuid.uuid4().hex[:8]}"

        # 计算差异
        diff = {}
        all_keys = set(list(before.keys()) + list(after.keys()))
        for key in all_keys:
            old_val = before.get(key)
            new_val = after.get(key)
            if old_val != new_val:
                diff[key] = {"before": old_val, "after": new_val}

        change = {
            "change_id": change_id,
            "service": service,
            "change_type": change_type,
            "operator": operator,
            "before": before,
            "after": after,
            "diff": diff,
            "reason": reason,
            "timestamp": datetime.now(),
            "approved": True,  # 简化：实际需要审批流程
        }

        self.changes.insert_one(change)
        print(f"[变更] 已记录: {change_id} - {change_type} {service}")
        return change_id

    # ========== 查询 ==========

    def search_events(
        self,
        event_type: str = None,
        service: str = None,
        operator: str = None,
        status: str = None,
        days: int = 7,
        limit: int = 50,
    ) -> List[Dict]:
        """
        搜索事件

        参数:
            event_type: 按事件类型过滤
            service: 按服务过滤
            operator: 按操作人过滤
            status: 按状态过滤
            days: 时间范围（天）
            limit: 返回条数

        返回:
            list: 事件列表
        """
        query = {
            "timestamp": {"$gte": datetime.now() - timedelta(days=days)}
        }

        if event_type:
            query["event_type"] = event_type
        if service:
            query["service"] = service
        if operator:
            query["operator"] = operator
        if status:
            query["status"] = status

        cursor = self.events.find(
            query,
            {"_id": 0}  # 不返回 _id
        ).sort("timestamp", DESCENDING).limit(limit)

        return list(cursor)

    def get_change_history(self, service: str, limit: int = 20) -> List[Dict]:
        """获取服务的变更历史"""
        return list(
            self.changes.find(
                {"service": service},
                {"_id": 0}
            ).sort("timestamp", DESCENDING).limit(limit)
        )

    # ========== 统计分析 ==========

    def get_statistics(self, days: int = 30) -> Dict:
        """
        获取运维统计数据

        参数:
            days: 统计时间范围

        返回:
            dict: 统计结果
        """
        since = datetime.now() - timedelta(days=days)

        # 总体统计
        pipeline_summary = [
            {"$match": {"timestamp": {"$gte": since}}},
            {"$group": {
                "_id": None,
                "total_events": {"$sum": 1},
                "success_count": {
                    "$sum": {"$cond": [{"$eq": ["$status", "success"]}, 1, 0]}
                },
                "failed_count": {
                    "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
                },
                "avg_duration": {"$avg": "$duration_seconds"},
                "unique_services": {"$addToSet": "$service"},
                "unique_operators": {"$addToSet": "$operator"},
            }}
        ]
        summary = list(self.events.aggregate(pipeline_summary))
        summary = summary[0] if summary else {}

        # 按服务统计
        pipeline_by_service = [
            {"$match": {"timestamp": {"$gte": since}}},
            {"$group": {
                "_id": "$service",
                "total": {"$sum": 1},
                "failed": {
                    "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
                },
                "avg_duration": {"$avg": "$duration_seconds"}
            }},
            {"$sort": {"total": -1}}
        ]
        by_service = list(self.events.aggregate(pipeline_by_service))

        # 按天趋势
        pipeline_trend = [
            {"$match": {"timestamp": {"$gte": since}}},
            {"$group": {
                "_id": {
                    "$dateToString": {"format": "%Y-%m-%d", "date": "$timestamp"}
                },
                "count": {"$sum": 1},
                "failed": {
                    "$sum": {"$cond": [{"$eq": ["$status", "failed"]}, 1, 0]}
                }
            }},
            {"$sort": {"_id": -1}},
            {"$limit": 14}
        ]
        trend = list(self.events.aggregate(pipeline_trend))

        return {
            "period_days": days,
            "total_events": summary.get("total_events", 0),
            "success_count": summary.get("success_count", 0),
            "failed_count": summary.get("failed_count", 0),
            "success_rate": round(
                summary.get("success_count", 0) /
                max(summary.get("total_events", 1), 1) * 100, 1
            ),
            "avg_duration_seconds": round(summary.get("avg_duration", 0), 1),
            "unique_services": len(summary.get("unique_services", [])),
            "unique_operators": len(summary.get("unique_operators", [])),
            "by_service": by_service,
            "daily_trend": trend
        }

    # ========== 报告生成 ==========

    def generate_report(self, days: int = 7) -> str:
        """生成运维报告"""
        stats = self.get_statistics(days)

        report = []
        report.append("=" * 65)
        report.append("        运维事件与变更报告")
        report.append("=" * 65)
        report.append(f"报告时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append(f"统计范围: 最近 {days} 天")

        report.append(f"\n--- 总体概览 ---")
        report.append(f"总事件数:     {stats['total_events']}")
        report.append(f"成功:         {stats['success_count']}")
        report.append(f"失败:         {stats['failed_count']}")
        report.append(f"成功率:       {stats['success_rate']}%")
        report.append(f"平均耗时:     {stats['avg_duration_seconds']}s")
        report.append(f"涉及服务数:   {stats['unique_services']}")
        report.append(f"操作人数:     {stats['unique_operators']}")

        report.append(f"\n--- 按服务统计 ---")
        for svc in stats["by_service"]:
            fail_rate = round(svc["failed"] / max(svc["total"], 1) * 100, 1)
            report.append(
                f"  {svc['_id']:<15} | 事件: {svc['total']:>4} | "
                f"失败: {svc['failed']:>3} ({fail_rate}%) | "
                f"平均耗时: {svc.get('avg_duration', 0):.0f}s"
            )

        report.append(f"\n--- 每日趋势 ---")
        for day in stats["daily_trend"]:
            bar = "#" * min(day["count"] // 2, 30)
            report.append(
                f"  {day['_id']} | 总: {day['count']:>4} | "
                f"失败: {day['failed']:>2} | {bar}"
            )

        # 最近失败事件
        failed_events = self.search_events(status="failed", days=days, limit=5)
        if failed_events:
            report.append(f"\n--- 最近失败事件 ---")
            for evt in failed_events:
                report.append(
                    f"  {evt['timestamp'].strftime('%m-%d %H:%M')} | "
                    f"{evt['event_type']:<15} | {evt['service']:<10} | "
                    f"{evt['operator']:<10} | {evt['description'][:30]}"
                )

        report.append("\n" + "=" * 65)
        return "\n".join(report)

    def close(self):
        """关闭连接"""
        self.client.close()


# ========== 主函数 ==========

def main():
    """演示完整流程"""

    store = OpsEventStore()

    print("=" * 50)
    print("  初始化：录入测试数据")
    print("=" * 50)

    # 清理旧数据
    store.events.drop()
    store.changes.drop()
    store._ensure_indexes()

    # 1. 记录各种运维事件
    import random
    event_configs = [
        ("deploy", "nginx", "admin", "部署 nginx 1.24.0", "success",
         {"version": "1.24.0", "env": "production"}, ["web-01", "web-02"], 120),
        ("restart", "redis", "sre-bot", "Redis 内存过高自动重启", "success",
         {"reason": "memory > 90%"}, ["cache-01"], 30),
        ("scale", "app-server", "admin", "扩容至 4 实例", "success",
         {"from": 2, "to": 4}, ["app-01", "app-02", "app-03", "app-04"], 180),
        ("incident", "mysql", "dba", "MySQL 主从延迟告警", "failed",
         {"delay_seconds": 120}, ["db-01", "db-02"], 600),
        ("config_change", "nginx", "developer-a", "更新 upstream 配置", "success",
         {"config_file": "/etc/nginx/nginx.conf"}, ["web-01"], 15),
        ("deploy", "app-server", "admin", "部署 v2.1.0 版本", "success",
         {"version": "2.1.0"}, ["app-01", "app-02"], 300),
        ("rollback", "app-server", "admin", "回滚至 v2.0.9（发现 Bug）", "success",
         {"from_version": "2.1.0", "to_version": "2.0.9"}, ["app-01", "app-02"], 90),
        ("restart", "kafka", "sre-bot", "Kafka Broker 重启", "failed",
         {"error": "端口被占用"}, ["kafka-01"], 0),
    ]

    for ec in event_configs:
        store.record_event(
            event_type=ec[0], service=ec[1], operator=ec[2],
            description=ec[3], status=ec[4], details=ec[5],
            affected_hosts=ec[6], duration_seconds=ec[7]
        )

    # 补充更多随机事件
    for i in range(50):
        store.record_event(
            event_type=random.choice(["deploy", "restart", "scale", "config_change"]),
            service=random.choice(["nginx", "app-server", "mysql", "redis"]),
            operator=random.choice(["admin", "sre-bot", "developer-a"]),
            description=f"自动生成的测试事件 #{i}",
            status=random.choices(["success", "failed"], weights=[85, 15])[0],
            duration_seconds=random.randint(10, 300)
        )

    # 2. 记录变更
    store.record_change(
        service="nginx",
        change_type="config_update",
        operator="admin",
        before={"worker_connections": 1024, "keepalive_timeout": 65},
        after={"worker_connections": 2048, "keepalive_timeout": 120},
        reason="优化高并发性能"
    )

    store.record_change(
        service="app-server",
        change_type="version_update",
        operator="admin",
        before={"version": "2.0.9", "replicas": 2},
        after={"version": "2.1.0", "replicas": 4},
        reason="新功能上线 + 扩容"
    )

    # 3. 查询事件
    print("\n" + "=" * 50)
    print("  查询：最近的部署事件")
    print("=" * 50)
    deploy_events = store.search_events(event_type="deploy", days=30, limit=5)
    for evt in deploy_events:
        print(f"  {evt['timestamp'].strftime('%m-%d %H:%M')} | "
              f"{evt['service']:<12} | {evt['operator']:<10} | "
              f"{evt['status']:<8} | {evt['description'][:40]}")

    # 4. 查看变更历史
    print("\n" + "=" * 50)
    print("  查询：nginx 变更历史")
    print("=" * 50)
    changes = store.get_change_history("nginx", limit=5)
    for chg in changes:
        print(f"  {chg['change_id']} | {chg['change_type']} | {chg['operator']}")
        if chg.get("diff"):
            for field, vals in chg["diff"].items():
                print(f"    {field}: {vals['before']} -> {vals['after']}")

    # 5. 生成报告
    print("\n")
    report = store.generate_report(days=30)
    print(report)

    # 6. 导出统计数据
    stats = store.get_statistics(30)
    output_path = "/tmp/ops_event_stats.json"
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(stats, f, ensure_ascii=False, indent=2, default=str)
    print(f"\n统计数据已导出: {output_path}")

    store.close()


if __name__ == "__main__":
    main()
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：没有创建索引

```python
# 错误：无索引的查询在数据量大时极慢
events.find({"service": "nginx"})  # 全集合扫描

# 正确：为常用查询字段创建索引
events.create_index([("service", 1), ("timestamp", -1)])
```

### 坑点 2：无限增长的集合

```python
# 错误：日志/事件集合不断增长，磁盘爆满

# 正确方案 1：TTL 索引自动过期
events.create_index("timestamp", expireAfterSeconds=90*86400)

# 正确方案 2：Capped Collection（固定大小集合）
# db.create_collection("logs", capped=True, size=1024*1024*500, max=1000000)
```

### 坑点 3：大文档性能问题

```python
# 错误：文档中嵌入大数组（数组无限增长）
events.update_one(
    {"_id": id},
    {"$push": {"logs": log_entry}}  # logs 数组可能增长到几 MB
)

# 正确：大数组拆分为独立集合，用引用关联
event_logs.insert_one({
    "event_id": event_id,
    "log_entry": log_entry,
    "timestamp": datetime.now()
})
```

### 坑点 4：不处理连接异常

```python
# 错误：不处理连接断开
result = collection.find_one({"key": "value"})  # 可能抛异常

# 正确：异常处理 + 重试
from pymongo.errors import ConnectionFailure, ServerSelectionTimeoutError

try:
    result = collection.find_one({"key": "value"})
except (ConnectionFailure, ServerSelectionTimeoutError) as e:
    print(f"MongoDB 连接失败: {e}")
    # 重试逻辑
```

### 最佳实践

1. **建立合适的索引**：覆盖常用查询条件
2. **使用 TTL 索引**：自动清理过期数据
3. **文档大小控制**：不超过 16MB，避免大数组
4. **使用连接池**：MongoClient 内置，全局复用
5. **投影查询**：只返回需要的字段
6. **批量操作**：insert_many 代替循环 insert_one

---

## 8. 速查表

### pymongo 操作速查

```python
from pymongo import MongoClient

client = MongoClient("mongodb://127.0.0.1:27017/")
db = client["mydb"]
col = db["mycol"]

# --- 插入 ---
col.insert_one(doc)
col.insert_many([doc1, doc2])

# --- 查询 ---
col.find_one({"key": "val"})           # 单条
col.find({"key": "val"})               # 多条（游标）
col.find({}, {"field1": 1, "_id": 0})  # 投影
col.find().sort("field", -1).limit(10) # 排序+限制
col.count_documents({"key": "val"})    # 计数

# --- 更新 ---
col.update_one(filter, {"$set": {...}})
col.update_many(filter, {"$set": {...}})
col.update_one(filter, update, upsert=True)

# --- 删除 ---
col.delete_one(filter)
col.delete_many(filter)

# --- 聚合 ---
col.aggregate([
    {"$match": {...}},
    {"$group": {"_id": "$field", "count": {"$sum": 1}}},
    {"$sort": {"count": -1}}
])

# --- 索引 ---
col.create_index("field")
col.create_index([("f1", 1), ("f2", -1)])
col.index_information()
col.drop_index("index_name")
```

### 查询操作符速查

```python
# 比较
{"age": {"$gt": 25}}      # 大于
{"age": {"$gte": 25}}     # 大于等于
{"age": {"$lt": 25}}      # 小于
{"age": {"$ne": "admin"}} # 不等于
{"status": {"$in": ["A", "B"]}}  # 在列表中

# 逻辑
{"$and": [cond1, cond2]}
{"$or": [cond1, cond2]}
{"$not": {"field": cond}}

# 数组
{"tags": "web"}                 # 包含某元素
{"tags": {"$all": ["a", "b"]}} # 包含所有
{"tags": {"$size": 3}}         # 数组长度

# 更新操作符
{"$set": {"key": "val"}}       # 设置字段
{"$unset": {"key": ""}}        # 删除字段
{"$inc": {"count": 1}}         # 递增
{"$push": {"arr": "item"}}     # 数组追加
{"$pull": {"arr": "item"}}     # 数组删除
{"$addToSet": {"arr": "item"}} # 数组追加（不重复）
```

### 对比 mongosh 命令

| mongosh | pymongo |
|---------|---------|
| `db.col.insertOne(doc)` | `col.insert_one(doc)` |
| `db.col.find(query)` | `col.find(query)` |
| `db.col.updateOne(f, u)` | `col.update_one(f, u)` |
| `db.col.deleteOne(f)` | `col.delete_one(f)` |
| `db.col.aggregate(p)` | `col.aggregate(p)` |
| `db.col.createIndex(k)` | `col.create_index(k)` |
| `db.col.countDocuments(q)` | `col.count_documents(q)` |
| `db.col.drop()` | `col.drop()` |
| `show dbs` | `client.list_database_names()` |
| `show collections` | `db.list_collection_names()` |
