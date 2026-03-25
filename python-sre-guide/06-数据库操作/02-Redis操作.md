# Redis 操作 — Python SRE 的缓存与消息队列指南

> 一句话说明：用 `redis-py` 操作 Redis 的所有数据结构（String/Hash/List/Set/ZSet），掌握 Pipeline 批量操作、发布订阅、分布式锁，实战运维任务队列。

---

## 1. 是什么 & 为什么用它

### 对比 Shell 操作 Redis

```bash
# Shell：用 redis-cli 执行命令
redis-cli -h 127.0.0.1 -p 6379 SET mykey "hello"
redis-cli GET mykey

# Shell：批量操作（管道模式）
echo -e "SET k1 v1\nSET k2 v2\nSET k3 v3" | redis-cli --pipe

# Shell：监控 Redis 信息
redis-cli INFO memory | grep used_memory_human
redis-cli INFO clients | grep connected_clients

# Shell：批量删除 Key
redis-cli KEYS "cache:*" | xargs redis-cli DEL

# 缺点：
# 1. 无法进行复杂逻辑处理
# 2. 无连接池，每次命令都新建连接
# 3. 数据序列化/反序列化困难
# 4. 不支持发布订阅的复杂消费
# 5. 分布式锁等高级功能无法实现
```

### Python redis-py 的优势

| 特性 | redis-cli/Shell | redis-py |
|------|----------------|----------|
| 连接池 | 无 | 内置 |
| Pipeline 批量 | 有限 | 完整支持 |
| 事务 | MULTI/EXEC 手动 | 封装好的 pipeline |
| 发布订阅 | 只能阻塞监听 | 异步/线程消费 |
| 分布式锁 | 手动 SETNX | Lock 对象封装 |
| 数据序列化 | 纯文本 | JSON/Pickle/自定义 |
| 异步支持 | 无 | aioredis 集成 |

---

## 2. 安装与环境

```bash
# 安装 redis-py
pip install redis

# 如果需要异步支持
pip install redis[hiredis]  # hiredis C 解析器，性能提升 10x

# 验证安装
python3 -c "import redis; print(f'redis-py 版本: {redis.__version__}')"

# 如果本地没有 Redis，用 Docker 快速启动
# docker run -d --name redis-test -p 6379:6379 redis:7-alpine

# 验证 Redis 连接
# redis-cli ping  # 应返回 PONG
```

---

## 3. 核心概念与原理

### Redis 数据结构一览

```
String  — 字符串/数字/二进制
  SET key value / GET key / INCR key
  SRE 用途：计数器、配置缓存、限流

Hash    — 哈希表（字段-值对）
  HSET key field value / HGET key field / HGETALL key
  SRE 用途：存储服务器信息、配置项

List    — 双向链表
  LPUSH key value / RPOP key / LRANGE key 0 -1
  SRE 用途：任务队列、日志缓冲

Set     — 无序集合（去重）
  SADD key member / SMEMBERS key / SINTER key1 key2
  SRE 用途：在线主机列表、标签管理

ZSet    — 有序集合（带分数排序）
  ZADD key score member / ZRANGE key 0 -1 / ZRANGEBYSCORE
  SRE 用途：延迟队列、排行榜、时间线
```

### 连接池原理

```
应用代码
    |
    v
ConnectionPool（连接池）
    +-- 空闲连接队列（复用已建立的 TCP 连接）
    +-- 活跃连接计数
    +-- max_connections 上限
    |
    v
TCP 连接到 Redis Server
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 连接与 String 操作

```python
#!/usr/bin/env python3
"""Redis 连接与 String 基础操作"""

import redis
import json
from datetime import datetime

# ========== 连接 Redis ==========
# 等价于: redis-cli -h 127.0.0.1 -p 6379

# 方式 1：直接连接
r = redis.Redis(
    host="127.0.0.1",
    port=6379,
    db=0,                    # 数据库编号（0-15）
    password=None,           # 密码（如果有）
    decode_responses=True,   # 自动将 bytes 解码为 str
    socket_timeout=5,        # 超时
    socket_connect_timeout=3,
    retry_on_timeout=True,   # 超时自动重试
)

# 方式 2：URL 连接
# r = redis.from_url("redis://:password@127.0.0.1:6379/0")

# 测试连接
print(f"PING: {r.ping()}")  # True

# ========== String 操作 ==========

# SET / GET
# 等价于: redis-cli SET server:web-01:status online
r.set("server:web-01:status", "online")
status = r.get("server:web-01:status")
print(f"服务器状态: {status}")  # online

# 带过期时间的 SET
# 等价于: redis-cli SET alert:cpu:web-01 "CPU 95%" EX 300
r.set("alert:cpu:web-01", "CPU 使用率 95%", ex=300)  # 300 秒后过期
ttl = r.ttl("alert:cpu:web-01")
print(f"告警 TTL: {ttl}s")

# SETNX — 仅在 Key 不存在时设置（分布式锁基础）
# 等价于: redis-cli SET lock:deploy myhost NX EX 60
result = r.set("lock:deploy", "web-01", nx=True, ex=60)
print(f"获取锁: {result}")  # True 或 None

# INCR / DECR — 原子计数器
# 等价于: redis-cli INCR counter:requests
r.set("counter:requests", 0)
r.incr("counter:requests")       # +1
r.incr("counter:requests")       # +1
r.incrby("counter:requests", 10) # +10
count = r.get("counter:requests")
print(f"请求计数: {count}")  # 12

# MSET / MGET — 批量操作
# 等价于: redis-cli MSET k1 v1 k2 v2 k3 v3
r.mset({
    "config:db_host": "10.0.2.1",
    "config:db_port": "3306",
    "config:db_name": "sre_db"
})
values = r.mget("config:db_host", "config:db_port", "config:db_name")
print(f"配置: {values}")

# 存储 JSON 数据
server_info = {
    "hostname": "web-01",
    "ip": "10.0.1.1",
    "cpu_cores": 8,
    "status": "online",
    "updated": datetime.now().isoformat()
}
r.set("server:web-01:info", json.dumps(server_info, ensure_ascii=False))
info = json.loads(r.get("server:web-01:info"))
print(f"服务器信息: {info['hostname']} ({info['ip']})")

# 清理测试数据
r.delete("server:web-01:status", "alert:cpu:web-01", "lock:deploy",
         "counter:requests", "server:web-01:info")
```

### 4.2 Hash 操作

```python
#!/usr/bin/env python3
"""Redis Hash 操作 — 存储结构化数据"""

import redis

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== Hash 操作 ==========
# Hash 适合存储对象（类似 Python dict）

# HSET — 设置字段
# 等价于: redis-cli HSET server:web-01 hostname web-01 ip 10.0.1.1
r.hset("server:web-01", mapping={
    "hostname": "web-01",
    "ip": "10.0.1.1",
    "cpu_cores": "8",
    "memory_gb": "16",
    "status": "online",
    "region": "cn-east"
})

# HGET — 获取单个字段
ip = r.hget("server:web-01", "ip")
print(f"IP: {ip}")

# HGETALL — 获取所有字段
info = r.hgetall("server:web-01")
print(f"全部信息: {info}")

# HMGET — 获取多个字段
values = r.hmget("server:web-01", "hostname", "status", "cpu_cores")
print(f"部分信息: {values}")

# HINCRBY — 原子递增
r.hset("server:web-01", "request_count", "0")
r.hincrby("server:web-01", "request_count", 1)
r.hincrby("server:web-01", "request_count", 1)
print(f"请求数: {r.hget('server:web-01', 'request_count')}")

# HEXISTS — 检查字段是否存在
print(f"有 ip 字段: {r.hexists('server:web-01', 'ip')}")
print(f"有 gpu 字段: {r.hexists('server:web-01', 'gpu')}")

# HDEL — 删除字段
r.hdel("server:web-01", "request_count")

# HLEN — 字段数量
print(f"字段数: {r.hlen('server:web-01')}")

# 批量存储多台服务器
servers = [
    {"hostname": "web-02", "ip": "10.0.1.2", "status": "online"},
    {"hostname": "db-01", "ip": "10.0.2.1", "status": "online"},
    {"hostname": "cache-01", "ip": "10.0.3.1", "status": "offline"},
]
for s in servers:
    r.hset(f"server:{s['hostname']}", mapping=s)

print("批量存储完成")

# 清理
for key in r.scan_iter("server:*"):
    r.delete(key)
```

### 4.3 List 操作

```python
#!/usr/bin/env python3
"""Redis List 操作 — 队列与栈"""

import redis
import json
from datetime import datetime

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== List 作为任务队列 ==========

# LPUSH — 从左端推入（等价于入队）
# 等价于: redis-cli LPUSH task:queue '{"task":"restart","target":"nginx"}'
tasks = [
    {"task": "restart", "target": "nginx", "time": datetime.now().isoformat()},
    {"task": "backup", "target": "mysql", "time": datetime.now().isoformat()},
    {"task": "deploy", "target": "web-app", "time": datetime.now().isoformat()},
]
for task in tasks:
    r.lpush("task:queue", json.dumps(task, ensure_ascii=False))

print(f"队列长度: {r.llen('task:queue')}")

# RPOP — 从右端弹出（等价于出队，FIFO）
while r.llen("task:queue") > 0:
    task_json = r.rpop("task:queue")
    task = json.loads(task_json)
    print(f"处理任务: {task['task']} -> {task['target']}")

# BRPOP — 阻塞弹出（队列为空时等待，适合 Worker 模式）
# result = r.brpop("task:queue", timeout=5)  # 等待 5 秒

# LRANGE — 查看队列内容（不弹出）
# 等价于: redis-cli LRANGE task:queue 0 -1
r.lpush("task:queue", "task-1", "task-2", "task-3")
items = r.lrange("task:queue", 0, -1)  # 获取所有元素
print(f"队列内容: {items}")

# LTRIM — 保留指定范围（可用于限制队列长度）
r.ltrim("task:queue", 0, 99)  # 只保留前 100 个元素

# 清理
r.delete("task:queue")
```

### 4.4 Set 和 ZSet 操作

```python
#!/usr/bin/env python3
"""Redis Set & ZSet 操作"""

import redis
import time

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== Set — 无序集合 ==========

# SADD — 添加成员
# 等价于: redis-cli SADD online:servers web-01 web-02 db-01
r.sadd("online:servers", "web-01", "web-02", "db-01", "cache-01")
r.sadd("region:cn-east", "web-01", "web-02", "cache-01")
r.sadd("region:cn-north", "db-01", "db-02")

# SMEMBERS — 获取所有成员
online = r.smembers("online:servers")
print(f"在线服务器: {online}")

# SCARD — 成员数量
print(f"在线数量: {r.scard('online:servers')}")

# SISMEMBER — 是否是成员
print(f"web-01 在线: {r.sismember('online:servers', 'web-01')}")

# SINTER — 交集（在线 且 cn-east 区域）
cn_east_online = r.sinter("online:servers", "region:cn-east")
print(f"华东在线: {cn_east_online}")

# SDIFF — 差集
not_online = r.sdiff("region:cn-north", "online:servers")
print(f"华北离线: {not_online}")

# SREM — 移除成员
r.srem("online:servers", "cache-01")  # cache-01 下线

# ========== ZSet — 有序集合 ==========

# ZADD — 添加带分数的成员
# 分数可以是时间戳、优先级、得分等
# 等价于: redis-cli ZADD alert:timeline 1700000001 "alert-001"
now = time.time()
r.zadd("alert:timeline", {
    "alert-001: CPU 过高": now - 3600,      # 1 小时前
    "alert-002: 内存不足": now - 1800,      # 30 分钟前
    "alert-003: 磁盘空间": now - 600,       # 10 分钟前
    "alert-004: 服务宕机": now,              # 刚才
})

# ZRANGE — 按分数升序获取
alerts = r.zrange("alert:timeline", 0, -1, withscores=True)
print("告警时间线（从旧到新）:")
for alert, score in alerts:
    print(f"  {alert} (时间戳: {score:.0f})")

# ZRANGEBYSCORE — 按分数范围查询
recent = r.zrangebyscore("alert:timeline", now - 1800, now)
print(f"最近 30 分钟告警: {recent}")

# ZCARD — 成员数量
print(f"总告警数: {r.zcard('alert:timeline')}")

# ZREM — 移除成员
r.zrem("alert:timeline", "alert-001: CPU 过高")

# ZREVRANGE — 按分数降序（最新的排前面）
latest = r.zrevrange("alert:timeline", 0, 2, withscores=True)
print(f"最新 3 条告警: {latest}")

# 清理
for key in r.scan_iter("online:*"):
    r.delete(key)
for key in r.scan_iter("region:*"):
    r.delete(key)
r.delete("alert:timeline")
```

---

## 5. 进阶用法

### 5.1 Pipeline 批量操作

```python
#!/usr/bin/env python3
"""Pipeline 批量操作 — 减少网络往返"""

import redis
import time

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== 无 Pipeline（每条命令一次网络往返） ==========
start = time.time()
for i in range(1000):
    r.set(f"bench:single:{i}", f"value-{i}")
single_time = time.time() - start
print(f"逐条执行 1000 次 SET: {single_time:.3f}s")

# ========== 有 Pipeline（批量发送，一次网络往返） ==========
start = time.time()
pipe = r.pipeline()  # 创建 Pipeline
for i in range(1000):
    pipe.set(f"bench:pipeline:{i}", f"value-{i}")
pipe.execute()  # 批量执行
pipeline_time = time.time() - start
print(f"Pipeline 执行 1000 次 SET: {pipeline_time:.3f}s")
print(f"性能提升: {single_time / pipeline_time:.1f}x")

# ========== Pipeline 读取结果 ==========
pipe = r.pipeline()
pipe.get("bench:pipeline:0")
pipe.get("bench:pipeline:1")
pipe.get("bench:pipeline:2")
pipe.hgetall("server:web-01")  # 可以混合不同命令
results = pipe.execute()
print(f"Pipeline 结果: {results}")

# ========== Pipeline 事务模式 ==========
# pipeline(transaction=True) 使用 MULTI/EXEC 保证原子性
with r.pipeline(transaction=True) as pipe:
    pipe.set("account:A:balance", 100)
    pipe.set("account:B:balance", 200)
    pipe.incrby("account:A:balance", -50)  # A 扣 50
    pipe.incrby("account:B:balance", 50)   # B 加 50
    pipe.execute()

# 清理
pipe = r.pipeline()
for i in range(1000):
    pipe.delete(f"bench:single:{i}")
    pipe.delete(f"bench:pipeline:{i}")
pipe.execute()
r.delete("account:A:balance", "account:B:balance")
```

### 5.2 发布订阅

```python
#!/usr/bin/env python3
"""Redis 发布订阅 — 实时事件通知"""

import redis
import json
import threading
import time
from datetime import datetime

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== 订阅者（消费端） ==========

def subscriber_worker():
    """订阅者线程 — 监听运维事件"""
    pubsub = r.pubsub()

    # 订阅频道
    pubsub.subscribe("ops:alerts", "ops:events")

    # 订阅模式（通配符）
    pubsub.psubscribe("ops:*")

    print("[订阅者] 开始监听...")

    for message in pubsub.listen():
        if message["type"] in ("message", "pmessage"):
            channel = message.get("channel", message.get("pattern"))
            data = message["data"]
            print(f"[订阅者] 频道={channel}, 数据={data}")

            # 实际场景：根据消息类型做不同处理
            try:
                event = json.loads(data)
                if event.get("level") == "critical":
                    print(f"  [严重告警] {event['message']}")
            except (json.JSONDecodeError, TypeError):
                pass

# ========== 发布者（生产端） ==========

def publisher_worker():
    """发布者 — 发送运维事件"""
    time.sleep(1)  # 等待订阅者就绪

    events = [
        {"level": "info", "message": "nginx 重启完成", "server": "web-01"},
        {"level": "warning", "message": "磁盘使用率 85%", "server": "db-01"},
        {"level": "critical", "message": "MySQL 主从延迟 > 60s", "server": "db-01"},
    ]

    for event in events:
        event["timestamp"] = datetime.now().isoformat()
        message = json.dumps(event, ensure_ascii=False)

        # 发布到不同频道
        channel = "ops:alerts" if event["level"] != "info" else "ops:events"
        listeners = r.publish(channel, message)
        print(f"[发布者] -> {channel} (收听者: {listeners})")
        time.sleep(0.5)

# 运行（使用线程演示）
# 实际场景中，订阅者和发布者通常在不同进程/服务中
if __name__ == "__main__":
    # 启动订阅者线程
    sub_thread = threading.Thread(target=subscriber_worker, daemon=True)
    sub_thread.start()

    # 发布消息
    publisher_worker()

    time.sleep(1)
    print("演示完成")
```

### 5.3 分布式锁

```python
#!/usr/bin/env python3
"""Redis 分布式锁 — 防止重复操作"""

import redis
import time
import uuid
import threading

r = redis.Redis(host="127.0.0.1", decode_responses=True)

# ========== 方式 1：使用 redis-py 内置 Lock ==========

def deploy_with_builtin_lock(server_name: str):
    """使用内置锁进行部署（推荐）"""

    # 创建分布式锁
    lock = r.lock(
        name=f"lock:deploy:{server_name}",  # 锁名称
        timeout=120,                         # 锁最长持有时间（秒）
        blocking_timeout=10,                 # 等待获取锁的超时
        blocking=True,                       # 是否阻塞等待
    )

    # 尝试获取锁
    acquired = lock.acquire()
    if not acquired:
        print(f"[{server_name}] 获取锁失败，其他人正在部署")
        return False

    try:
        print(f"[{server_name}] 获取锁成功，开始部署...")
        time.sleep(3)  # 模拟部署过程
        print(f"[{server_name}] 部署完成")
        return True
    finally:
        # 释放锁（只有持有者能释放）
        try:
            lock.release()
            print(f"[{server_name}] 锁已释放")
        except redis.exceptions.LockNotOwnedError:
            print(f"[{server_name}] 锁已过期（被自动释放）")

# ========== 方式 2：使用 with 语句 ==========

def safe_operation():
    """使用 with 语句自动管理锁"""
    lock = r.lock("lock:operation", timeout=60)

    # with 语句自动获取和释放锁
    with lock:
        print("正在执行互斥操作...")
        time.sleep(1)
        print("操作完成")

# ========== 方式 3：手动实现（了解原理） ==========

class DistributedLock:
    """
    手动实现的分布式锁（了解原理用）
    实际项目请用 redis-py 内置的 Lock
    """

    def __init__(self, redis_client, name, timeout=30):
        self.redis = redis_client
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.token = str(uuid.uuid4())  # 唯一标识，防止误释放

    def acquire(self, retry_times=3, retry_delay=1):
        """获取锁"""
        for i in range(retry_times):
            # SET NX EX — 原子操作：不存在则设置 + 设置过期时间
            if self.redis.set(self.name, self.token, nx=True, ex=self.timeout):
                return True
            time.sleep(retry_delay)
        return False

    def release(self):
        """释放锁（Lua 脚本保证原子性）"""
        # 用 Lua 脚本确保只有持有者才能释放
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        return self.redis.eval(lua_script, 1, self.name, self.token)

    def __enter__(self):
        if not self.acquire():
            raise Exception("获取锁失败")
        return self

    def __exit__(self, *args):
        self.release()

# 演示并发锁竞争
def demo_concurrent_lock():
    """多线程竞争锁演示"""
    def worker(name):
        lock = r.lock(f"lock:demo", timeout=10, blocking_timeout=5)
        acquired = lock.acquire()
        if acquired:
            try:
                print(f"  [{name}] 获得锁，执行操作中...")
                time.sleep(2)
            finally:
                lock.release()
                print(f"  [{name}] 释放锁")
        else:
            print(f"  [{name}] 未能获得锁")

    print("并发锁竞争演示:")
    threads = [
        threading.Thread(target=worker, args=(f"Worker-{i}",))
        for i in range(3)
    ]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

if __name__ == "__main__":
    demo_concurrent_lock()
```

### 5.4 连接池管理

```python
#!/usr/bin/env python3
"""Redis 连接池配置与管理"""

import redis

# ========== 标准连接池 ==========
pool = redis.ConnectionPool(
    host="127.0.0.1",
    port=6379,
    db=0,
    max_connections=50,       # 最大连接数
    decode_responses=True,
    socket_timeout=5,
    socket_connect_timeout=3,
    retry_on_timeout=True,
    health_check_interval=30, # 健康检查间隔（秒）
)

# 所有 Redis 客户端共享同一个连接池
r1 = redis.Redis(connection_pool=pool)
r2 = redis.Redis(connection_pool=pool)

r1.set("test", "from r1")
print(r2.get("test"))  # "from r1" — 共享同一个 Redis

# 查看连接池状态
print(f"已创建连接: {pool._created_connections}")
print(f"可用连接: {pool._available_connections}")
print(f"使用中连接: {pool._in_use_connections}")

# ========== Sentinel 连接（高可用） ==========
# from redis.sentinel import Sentinel
#
# sentinel = Sentinel(
#     [("sentinel-1", 26379), ("sentinel-2", 26379), ("sentinel-3", 26379)],
#     socket_timeout=3
# )
# master = sentinel.master_for("mymaster", db=0, decode_responses=True)
# slave = sentinel.slave_for("mymaster", db=0, decode_responses=True)

# ========== Cluster 连接（集群） ==========
# from redis.cluster import RedisCluster
#
# rc = RedisCluster(
#     host="127.0.0.1",
#     port=7000,
#     decode_responses=True
# )

r1.delete("test")
```

---

## 6. SRE 实战案例：运维任务队列 + 分布式锁

```python
#!/usr/bin/env python3
"""
SRE 实战：基于 Redis 的运维任务队列 + 分布式锁
功能：
  1. 任务生产者：提交运维任务到队列
  2. 任务消费者：从队列取任务并执行
  3. 分布式锁：防止同一任务被重复执行
  4. 任务状态跟踪
  5. 延迟任务支持
  6. 死信队列（失败重试）
"""

import redis
import json
import time
import uuid
import threading
import signal
import sys
from datetime import datetime
from typing import Optional, Callable, Dict


# ========== Redis 连接 ==========
pool = redis.ConnectionPool(
    host="127.0.0.1",
    port=6379,
    db=0,
    max_connections=20,
    decode_responses=True,
    socket_timeout=5,
    retry_on_timeout=True,
)
rds = redis.Redis(connection_pool=pool)


# ========== 任务定义 ==========

class Task:
    """运维任务"""

    def __init__(self, task_type: str, target: str,
                 params: dict = None, priority: int = 0):
        self.id = str(uuid.uuid4())[:8]
        self.task_type = task_type   # 任务类型：restart/backup/deploy/check
        self.target = target         # 操作目标
        self.params = params or {}
        self.priority = priority     # 优先级（数字越大越优先）
        self.status = "pending"
        self.created_at = datetime.now().isoformat()
        self.started_at = None
        self.completed_at = None
        self.result = None
        self.retries = 0
        self.max_retries = 3

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "task_type": self.task_type,
            "target": self.target,
            "params": self.params,
            "priority": self.priority,
            "status": self.status,
            "created_at": self.created_at,
            "started_at": self.started_at,
            "completed_at": self.completed_at,
            "result": self.result,
            "retries": self.retries,
            "max_retries": self.max_retries,
        }

    def to_json(self) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False)

    @classmethod
    def from_json(cls, json_str: str) -> 'Task':
        data = json.loads(json_str)
        task = cls(
            task_type=data["task_type"],
            target=data["target"],
            params=data.get("params", {}),
            priority=data.get("priority", 0)
        )
        task.id = data["id"]
        task.status = data.get("status", "pending")
        task.created_at = data.get("created_at")
        task.retries = data.get("retries", 0)
        task.max_retries = data.get("max_retries", 3)
        return task


# ========== 任务队列 ==========

class TaskQueue:
    """基于 Redis 的任务队列"""

    # Redis Key 命名规范
    QUEUE_KEY = "ops:task:queue"           # 任务队列（List）
    PRIORITY_KEY = "ops:task:priority"     # 优先级队列（ZSet）
    STATUS_PREFIX = "ops:task:status:"     # 任务状态（Hash）
    LOCK_PREFIX = "ops:task:lock:"         # 任务锁
    DEAD_LETTER_KEY = "ops:task:dead"      # 死信队列
    STATS_KEY = "ops:task:stats"           # 统计数据

    def __init__(self, redis_client: redis.Redis):
        self.rds = redis_client

    def submit(self, task: Task) -> str:
        """
        提交任务到队列

        参数:
            task: 任务对象

        返回:
            str: 任务 ID
        """
        pipe = self.rds.pipeline()

        # 1. 存储任务状态
        pipe.hset(f"{self.STATUS_PREFIX}{task.id}", mapping=task.to_dict())

        # 2. 如果有优先级，放入优先级队列（ZSet）
        if task.priority > 0:
            pipe.zadd(self.PRIORITY_KEY, {task.to_json(): task.priority})
        else:
            # 普通任务放入 FIFO 队列
            pipe.lpush(self.QUEUE_KEY, task.to_json())

        # 3. 更新统计
        pipe.hincrby(self.STATS_KEY, "total_submitted", 1)

        pipe.execute()
        print(f"[队列] 任务已提交: {task.id} ({task.task_type} -> {task.target})")
        return task.id

    def fetch(self, timeout: int = 5) -> Optional[Task]:
        """
        从队列获取任务（优先级队列优先）

        参数:
            timeout: 等待超时（秒）

        返回:
            Task 或 None
        """
        # 1. 先检查优先级队列
        result = self.rds.zpopmax(self.PRIORITY_KEY)
        if result:
            task_json, score = result[0]
            return Task.from_json(task_json)

        # 2. 再从普通队列取（阻塞等待）
        result = self.rds.brpop(self.QUEUE_KEY, timeout=timeout)
        if result:
            _, task_json = result
            return Task.from_json(task_json)

        return None

    def update_status(self, task: Task):
        """更新任务状态"""
        self.rds.hset(
            f"{self.STATUS_PREFIX}{task.id}",
            mapping=task.to_dict()
        )

    def get_status(self, task_id: str) -> dict:
        """查询任务状态"""
        return self.rds.hgetall(f"{self.STATUS_PREFIX}{task_id}")

    def move_to_dead_letter(self, task: Task):
        """将失败任务移到死信队列"""
        task.status = "dead"
        self.rds.lpush(self.DEAD_LETTER_KEY, task.to_json())
        self.update_status(task)
        self.rds.hincrby(self.STATS_KEY, "total_dead", 1)

    def get_stats(self) -> dict:
        """获取队列统计信息"""
        stats = self.rds.hgetall(self.STATS_KEY) or {}
        stats["queue_length"] = str(self.rds.llen(self.QUEUE_KEY))
        stats["priority_queue_length"] = str(self.rds.zcard(self.PRIORITY_KEY))
        stats["dead_letter_count"] = str(self.rds.llen(self.DEAD_LETTER_KEY))
        return stats


# ========== 任务处理器 ==========

class TaskWorker:
    """任务消费者/Worker"""

    def __init__(self, queue: TaskQueue, worker_id: str = None):
        self.queue = queue
        self.worker_id = worker_id or f"worker-{uuid.uuid4().hex[:6]}"
        self.running = True
        self.handlers: Dict[str, Callable] = {}

        # 注册默认处理器
        self.register("restart", self._handle_restart)
        self.register("backup", self._handle_backup)
        self.register("deploy", self._handle_deploy)
        self.register("check", self._handle_check)

    def register(self, task_type: str, handler: Callable):
        """注册任务处理器"""
        self.handlers[task_type] = handler

    def _handle_restart(self, task: Task) -> str:
        """处理重启任务"""
        print(f"    正在重启 {task.target}...")
        time.sleep(1)  # 模拟重启
        return f"{task.target} 重启成功"

    def _handle_backup(self, task: Task) -> str:
        """处理备份任务"""
        print(f"    正在备份 {task.target}...")
        time.sleep(2)  # 模拟备份
        return f"{task.target} 备份完成, 大小: 1.2GB"

    def _handle_deploy(self, task: Task) -> str:
        """处理部署任务"""
        version = task.params.get("version", "latest")
        print(f"    正在部署 {task.target} (版本: {version})...")
        time.sleep(2)
        return f"{task.target} 部署完成 (版本: {version})"

    def _handle_check(self, task: Task) -> str:
        """处理检查任务"""
        print(f"    正在检查 {task.target}...")
        time.sleep(0.5)
        return f"{task.target} 检查正常"

    def process_task(self, task: Task):
        """处理单个任务"""
        # 1. 获取分布式锁（防止重复执行）
        lock = rds.lock(
            f"{TaskQueue.LOCK_PREFIX}{task.id}",
            timeout=120,
            blocking_timeout=1
        )

        if not lock.acquire():
            print(f"  [{self.worker_id}] 任务 {task.id} 已被其他 Worker 锁定，跳过")
            return

        try:
            # 2. 更新状态为执行中
            task.status = "running"
            task.started_at = datetime.now().isoformat()
            self.queue.update_status(task)

            # 3. 查找对应的处理器
            handler = self.handlers.get(task.task_type)
            if not handler:
                raise ValueError(f"未知任务类型: {task.task_type}")

            # 4. 执行任务
            print(f"  [{self.worker_id}] 执行任务: {task.id} ({task.task_type} -> {task.target})")
            result = handler(task)

            # 5. 更新状态为完成
            task.status = "completed"
            task.completed_at = datetime.now().isoformat()
            task.result = result
            self.queue.update_status(task)
            rds.hincrby(TaskQueue.STATS_KEY, "total_completed", 1)

            print(f"  [{self.worker_id}] 任务完成: {task.id} -> {result}")

        except Exception as e:
            # 6. 处理失败
            task.retries += 1
            task.result = f"错误: {str(e)}"

            if task.retries < task.max_retries:
                # 重新入队重试
                task.status = "retry"
                self.queue.update_status(task)
                rds.lpush(TaskQueue.QUEUE_KEY, task.to_json())
                print(f"  [{self.worker_id}] 任务失败，重试 {task.retries}/{task.max_retries}: {e}")
            else:
                # 超过最大重试次数，移入死信队列
                self.queue.move_to_dead_letter(task)
                print(f"  [{self.worker_id}] 任务彻底失败，移入死信队列: {task.id}")

        finally:
            try:
                lock.release()
            except Exception:
                pass

    def start(self, max_tasks: int = None):
        """
        启动 Worker 循环

        参数:
            max_tasks: 最多处理任务数（None=无限）
        """
        print(f"[{self.worker_id}] Worker 启动，等待任务...")
        processed = 0

        while self.running:
            if max_tasks and processed >= max_tasks:
                break

            task = self.queue.fetch(timeout=2)
            if task:
                self.process_task(task)
                processed += 1
            else:
                pass  # 无任务，继续等待

        print(f"[{self.worker_id}] Worker 停止，共处理 {processed} 个任务")

    def stop(self):
        """停止 Worker"""
        self.running = False


# ========== 主程序 ==========

def main():
    """演示完整的任务队列流程"""

    queue = TaskQueue(rds)

    # 1. 提交任务
    print("=" * 50)
    print("  提交任务到队列")
    print("=" * 50)

    tasks = [
        Task("restart", "nginx"),
        Task("backup", "mysql-master", {"compress": True}),
        Task("deploy", "web-app", {"version": "2.1.0"}, priority=10),  # 高优先级
        Task("check", "redis-cluster"),
        Task("restart", "php-fpm"),
    ]

    for task in tasks:
        queue.submit(task)

    # 查看队列状态
    stats = queue.get_stats()
    print(f"\n队列状态: {json.dumps(stats, indent=2)}")

    # 2. 启动 Worker 处理任务
    print("\n" + "=" * 50)
    print("  启动 Worker 处理任务")
    print("=" * 50)

    worker = TaskWorker(queue, "main-worker")
    worker.start(max_tasks=5)  # 最多处理 5 个任务

    # 3. 查看最终统计
    print("\n" + "=" * 50)
    print("  最终统计")
    print("=" * 50)
    final_stats = queue.get_stats()
    print(json.dumps(final_stats, indent=2, ensure_ascii=False))

    # 4. 查看某个任务的状态
    task_id = tasks[0].id
    status = queue.get_status(task_id)
    print(f"\n任务 {task_id} 状态: {json.dumps(status, indent=2, ensure_ascii=False)}")

    # 清理测试数据
    pipe = rds.pipeline()
    pipe.delete(TaskQueue.QUEUE_KEY)
    pipe.delete(TaskQueue.PRIORITY_KEY)
    pipe.delete(TaskQueue.DEAD_LETTER_KEY)
    pipe.delete(TaskQueue.STATS_KEY)
    for task in tasks:
        pipe.delete(f"{TaskQueue.STATUS_PREFIX}{task.id}")
        pipe.delete(f"{TaskQueue.LOCK_PREFIX}{task.id}")
    pipe.execute()
    print("\n测试数据已清理")


if __name__ == "__main__":
    main()
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：KEYS 命令在生产环境

```python
# 错误：KEYS 会遍历整个键空间，阻塞 Redis！
keys = r.keys("user:*")  # 百万 Key 时会卡住

# 正确：使用 SCAN 迭代
for key in r.scan_iter("user:*", count=100):
    print(key)
```

### 坑点 2：大 Key 问题

```python
# 错误：往一个 List 塞百万元素
for i in range(1_000_000):
    r.lpush("big:list", f"item-{i}")  # 大 Key！删除时会阻塞

# 正确：分片或限制大小
r.lpush("queue:shard:1", item)
r.ltrim("queue:shard:1", 0, 9999)  # 限制长度
```

### 坑点 3：连接泄漏

```python
# 错误：每次操作创建新连接
def bad():
    r = redis.Redis()  # 每次新建连接
    r.get("key")
    # 忘记关闭

# 正确：使用连接池
pool = redis.ConnectionPool(max_connections=20)
r = redis.Redis(connection_pool=pool)  # 全局复用
```

### 坑点 4：decode_responses 不一致

```python
# 错误：存入 bytes，读取时期望 str
r1 = redis.Redis()                       # decode_responses=False（默认）
r2 = redis.Redis(decode_responses=True)
r1.set("key", "value")
result = r2.get("key")  # 类型可能混乱

# 正确：统一配置
r = redis.Redis(decode_responses=True)  # 始终解码
```

---

## 8. 速查表

### 数据类型操作速查

```python
# === String ===
r.set(key, val)                  # 设置
r.set(key, val, ex=60)          # 带过期
r.set(key, val, nx=True)        # 不存在才设置
r.get(key)                       # 获取
r.mset({k1: v1, k2: v2})       # 批量设置
r.mget(k1, k2)                  # 批量获取
r.incr(key)                     # 自增
r.expire(key, 60)               # 设置过期

# === Hash ===
r.hset(key, field, val)         # 设置字段
r.hset(key, mapping=dict)       # 批量设置
r.hget(key, field)              # 获取字段
r.hgetall(key)                  # 获取全部
r.hincrby(key, field, 1)       # 字段自增
r.hdel(key, field)              # 删除字段

# === List ===
r.lpush(key, val)               # 左推入
r.rpush(key, val)               # 右推入
r.rpop(key)                     # 右弹出
r.brpop(key, timeout=5)        # 阻塞弹出
r.lrange(key, 0, -1)           # 获取范围
r.llen(key)                     # 长度

# === Set ===
r.sadd(key, member)             # 添加
r.smembers(key)                 # 全部成员
r.sismember(key, member)        # 是否存在
r.sinter(k1, k2)               # 交集
r.sdiff(k1, k2)                # 差集

# === ZSet ===
r.zadd(key, {member: score})    # 添加
r.zrange(key, 0, -1)           # 按分数升序
r.zrevrange(key, 0, -1)        # 按分数降序
r.zrangebyscore(key, min, max)  # 按分数范围
r.zscore(key, member)           # 获取分数
r.zrem(key, member)             # 删除成员
```

### 对比 redis-cli

| redis-cli | redis-py |
|-----------|----------|
| `SET k v` | `r.set("k", "v")` |
| `SET k v EX 60` | `r.set("k", "v", ex=60)` |
| `SET k v NX` | `r.set("k", "v", nx=True)` |
| `GET k` | `r.get("k")` |
| `DEL k` | `r.delete("k")` |
| `TTL k` | `r.ttl("k")` |
| `EXISTS k` | `r.exists("k")` |
| `KEYS pattern` | `r.scan_iter("pattern")` |
| `INFO` | `r.info()` |
| `DBSIZE` | `r.dbsize()` |
| `FLUSHDB` | `r.flushdb()` |
| `SUBSCRIBE ch` | `pubsub.subscribe("ch")` |
| `PUBLISH ch msg` | `r.publish("ch", "msg")` |
