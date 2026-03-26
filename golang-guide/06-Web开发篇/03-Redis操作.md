# 03-Redis操作

## 导读

Redis 是 SRE/DevOps 工程师日常工作中不可或缺的基础组件——缓存、会话存储、分布式锁、消息队列、排行榜、限流计数器，无处不在。Go 语言中使用 `go-redis` 客户端库与 Redis 交互，它提供了类型安全的 API 和完整的 Redis 命令覆盖。

本章将系统讲解：

- go-redis 客户端安装与连接
- 基本数据类型操作（String/Hash/List/Set/Sorted Set）
- 过期时间设置（TTL）
- Pipeline 批量操作
- 事务（TxPipeline）
- Pub/Sub 发布订阅
- 分布式锁（SetNX/RedLock）
- 缓存模式（Cache Aside/Read Through/Write Through）
- 连接池配置
- Sentinel 和 Cluster 模式

> 基于 Go 1.24 + go-redis v9，所有代码完整可运行。

---

## 一、go-redis 安装与连接

### 1.1 安装

```bash
go get github.com/redis/go-redis/v9
```

### 1.2 连接单机 Redis

```go
// 文件: redisclient/client.go
package redisclient

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// NewRedisClient 创建 Redis 客户端
func NewRedisClient() *redis.Client {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379", // Redis 地址
		Password: "",              // 密码（无密码留空）
		DB:       0,               // 数据库编号（0-15）

		// 连接池配置
		PoolSize:     10,               // 连接池大小
		MinIdleConns: 5,                // 最小空闲连接数
		MaxIdleConns: 10,               // 最大空闲连接数

		// 超时配置
		DialTimeout:  5 * time.Second,  // 连接超时
		ReadTimeout:  3 * time.Second,  // 读超时
		WriteTimeout: 3 * time.Second,  // 写超时
		PoolTimeout:  4 * time.Second,  // 从连接池获取连接的超时
	})

	return rdb
}

// PingRedis 测试连接
func PingRedis(rdb *redis.Client) error {
	ctx := context.Background()
	pong, err := rdb.Ping(ctx).Result()
	if err != nil {
		return fmt.Errorf("Redis 连接失败: %w", err)
	}
	fmt.Printf("Redis 连接成功: %s\n", pong)
	return nil
}
```

```go
// 文件: main.go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	defer rdb.Close()

	ctx := context.Background()

	// 测试连接
	if err := rdb.Ping(ctx).Err(); err != nil {
		log.Fatalf("Redis 连接失败: %v", err)
	}
	fmt.Println("Redis 连接成功")
}
```

---

## 二、基本数据类型操作

### 2.1 String 字符串

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

func stringOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== SET / GET =====
	// 设置值
	err := rdb.Set(ctx, "name", "张三", 0).Err() // 0 表示不过期
	if err != nil {
		fmt.Printf("SET 失败: %v\n", err)
		return
	}

	// 获取值
	val, err := rdb.Get(ctx, "name").Result()
	if err == redis.Nil {
		fmt.Println("键不存在")
	} else if err != nil {
		fmt.Printf("GET 失败: %v\n", err)
	} else {
		fmt.Printf("name = %s\n", val)
	}

	// 设置值并指定过期时间
	rdb.Set(ctx, "session:abc123", "user_data", 30*time.Minute)

	// ===== SetEX / SetNX =====
	// SetEX：设置值和过期时间
	rdb.SetEx(ctx, "token:xyz", "jwt_data", time.Hour)

	// SetNX：仅当键不存在时设置（常用于分布式锁）
	ok, err := rdb.SetNX(ctx, "lock:resource", "holder_id", 10*time.Second).Result()
	if ok {
		fmt.Println("获取锁成功")
	} else {
		fmt.Println("锁已被占用")
	}

	// ===== INCR / DECR =====
	// 计数器
	rdb.Set(ctx, "counter", 0, 0)
	rdb.Incr(ctx, "counter")           // +1
	rdb.IncrBy(ctx, "counter", 5)      // +5
	rdb.Decr(ctx, "counter")           // -1
	rdb.DecrBy(ctx, "counter", 2)      // -2
	rdb.IncrByFloat(ctx, "counter", 1.5) // +1.5

	count, _ := rdb.Get(ctx, "counter").Int64()
	fmt.Printf("计数器: %d\n", count)

	// ===== MSET / MGET：批量操作 =====
	rdb.MSet(ctx, "k1", "v1", "k2", "v2", "k3", "v3")
	vals, _ := rdb.MGet(ctx, "k1", "k2", "k3").Result()
	fmt.Printf("MGET 结果: %v\n", vals) // [v1 v2 v3]

	// ===== 存储 JSON =====
	type UserSession struct {
		UserID   string `json:"user_id"`
		Username string `json:"username"`
		Role     string `json:"role"`
		LoginAt  string `json:"login_at"`
	}

	session := UserSession{
		UserID:   "U001",
		Username: "zhangsan",
		Role:     "admin",
		LoginAt:  time.Now().Format(time.RFC3339),
	}

	// 序列化为 JSON 存储
	data, _ := json.Marshal(session)
	rdb.Set(ctx, "session:U001", data, 24*time.Hour)

	// 读取并反序列化
	jsonStr, _ := rdb.Get(ctx, "session:U001").Result()
	var loaded UserSession
	json.Unmarshal([]byte(jsonStr), &loaded)
	fmt.Printf("会话: %+v\n", loaded)

	// ===== APPEND / STRLEN =====
	rdb.Set(ctx, "greeting", "Hello", 0)
	rdb.Append(ctx, "greeting", " World")
	length, _ := rdb.StrLen(ctx, "greeting").Result()
	fmt.Printf("字符串长度: %d\n", length)

	// ===== GetSet：设置新值并返回旧值 =====
	oldVal, _ := rdb.GetSet(ctx, "name", "李四").Result()
	fmt.Printf("旧值: %s\n", oldVal) // 张三

	// ===== GetDel：获取并删除 =====
	val, _ = rdb.GetDel(ctx, "name").Result()
	fmt.Printf("获取并删除: %s\n", val) // 李四
}
```

### 2.2 Hash 哈希

```go
func hashOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== HSET / HGET =====
	// 存储用户信息
	rdb.HSet(ctx, "user:1001", map[string]interface{}{
		"name":     "张三",
		"email":    "zhangsan@example.com",
		"age":      25,
		"role":     "admin",
		"login_count": 0,
	})

	// 获取单个字段
	name, _ := rdb.HGet(ctx, "user:1001", "name").Result()
	fmt.Printf("用户名: %s\n", name)

	// 获取所有字段
	all, _ := rdb.HGetAll(ctx, "user:1001").Result()
	fmt.Printf("用户信息: %v\n", all)

	// ===== HMSET / HMGET =====
	rdb.HMSet(ctx, "server:web01", map[string]interface{}{
		"ip":     "10.0.0.1",
		"cpu":    8,
		"memory": 16,
		"status": "running",
	})

	vals, _ := rdb.HMGet(ctx, "server:web01", "ip", "status").Result()
	fmt.Printf("服务器信息: %v\n", vals)

	// ===== HINCRBY：字段递增 =====
	rdb.HIncrBy(ctx, "user:1001", "login_count", 1)
	rdb.HIncrByFloat(ctx, "user:1001", "balance", 99.99)

	// ===== HDEL：删除字段 =====
	rdb.HDel(ctx, "user:1001", "temporary_field")

	// ===== HEXISTS：检查字段是否存在 =====
	exists, _ := rdb.HExists(ctx, "user:1001", "email").Result()
	fmt.Printf("email 字段存在: %v\n", exists)

	// ===== HKEYS / HVALS / HLEN =====
	keys, _ := rdb.HKeys(ctx, "user:1001").Result()
	fmt.Printf("所有字段: %v\n", keys)

	length, _ := rdb.HLen(ctx, "user:1001").Result()
	fmt.Printf("字段数量: %d\n", length)

	// ===== 存储结构体到 Hash =====
	type ServerInfo struct {
		Hostname string `redis:"hostname"`
		IP       string `redis:"ip"`
		CPU      int    `redis:"cpu"`
		Memory   int    `redis:"memory"`
		Status   string `redis:"status"`
	}

	server := ServerInfo{
		Hostname: "web-server-01",
		IP:       "192.168.1.100",
		CPU:      8,
		Memory:   32,
		Status:   "active",
	}

	// 使用 HSet 存储结构体
	rdb.HSet(ctx, "server:web-01", map[string]interface{}{
		"hostname": server.Hostname,
		"ip":       server.IP,
		"cpu":      server.CPU,
		"memory":   server.Memory,
		"status":   server.Status,
	})

	// 读取到结构体
	var loaded ServerInfo
	rdb.HGetAll(ctx, "server:web-01").Scan(&loaded)
	fmt.Printf("服务器: %+v\n", loaded)
}
```

### 2.3 List 列表

```go
func listOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== LPUSH / RPUSH =====
	// 左插入（头部）
	rdb.LPush(ctx, "queue:tasks", "task1", "task2", "task3")
	// 列表：[task3, task2, task1]

	// 右插入（尾部）
	rdb.RPush(ctx, "queue:tasks", "task4", "task5")
	// 列表：[task3, task2, task1, task4, task5]

	// ===== LPOP / RPOP =====
	// 左弹出（从头部取出）—— 实现队列（FIFO）
	val, _ := rdb.LPop(ctx, "queue:tasks").Result()
	fmt.Printf("弹出: %s\n", val) // task3

	// 右弹出（从尾部取出）—— 实现栈（LIFO）
	val, _ = rdb.RPop(ctx, "queue:tasks").Result()
	fmt.Printf("弹出: %s\n", val) // task5

	// ===== LRANGE：获取范围 =====
	items, _ := rdb.LRange(ctx, "queue:tasks", 0, -1).Result() // 获取所有
	fmt.Printf("列表: %v\n", items)

	items, _ = rdb.LRange(ctx, "queue:tasks", 0, 2).Result() // 前3个
	fmt.Printf("前3个: %v\n", items)

	// ===== LLEN：获取长度 =====
	length, _ := rdb.LLen(ctx, "queue:tasks").Result()
	fmt.Printf("列表长度: %d\n", length)

	// ===== LINDEX：按索引获取 =====
	val, _ = rdb.LIndex(ctx, "queue:tasks", 0).Result()
	fmt.Printf("第一个元素: %s\n", val)

	// ===== LREM：删除元素 =====
	// 删除 2 个值为 "task2" 的元素
	rdb.LRem(ctx, "queue:tasks", 2, "task2")

	// ===== LTRIM：修剪列表（保留指定范围）=====
	rdb.LTrim(ctx, "queue:tasks", 0, 99) // 只保留前 100 个

	// ===== BLPOP：阻塞弹出（消息队列场景）=====
	// 阻塞等待，直到有元素可弹出或超时
	result, err := rdb.BLPop(ctx, 5*time.Second, "queue:jobs").Result()
	if err == redis.Nil {
		fmt.Println("等待超时，没有新任务")
	} else if err != nil {
		fmt.Printf("错误: %v\n", err)
	} else {
		fmt.Printf("队列: %s, 值: %s\n", result[0], result[1])
	}

	// ===== 实现简单的消息队列 =====
	// 生产者
	rdb.RPush(ctx, "mq:alerts", `{"level":"critical","message":"CPU 使用率超过 90%"}`)
	rdb.RPush(ctx, "mq:alerts", `{"level":"warning","message":"磁盘使用率超过 80%"}`)

	// 消费者
	for {
		result, err := rdb.BLPop(ctx, 1*time.Second, "mq:alerts").Result()
		if err == redis.Nil {
			break // 没有更多消息
		}
		if err != nil {
			break
		}
		fmt.Printf("收到告警: %s\n", result[1])
	}
}
```

### 2.4 Set 集合

```go
func setOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== SADD：添加成员 =====
	rdb.SAdd(ctx, "tags:server01", "production", "web", "golang", "v2")
	rdb.SAdd(ctx, "tags:server02", "production", "api", "golang", "v3")
	rdb.SAdd(ctx, "tags:server03", "staging", "web", "python", "v1")

	// ===== SMEMBERS：获取所有成员 =====
	members, _ := rdb.SMembers(ctx, "tags:server01").Result()
	fmt.Printf("server01 标签: %v\n", members)

	// ===== SCARD：获取成员数量 =====
	count, _ := rdb.SCard(ctx, "tags:server01").Result()
	fmt.Printf("标签数量: %d\n", count)

	// ===== SISMEMBER：检查成员是否存在 =====
	isMember, _ := rdb.SIsMember(ctx, "tags:server01", "production").Result()
	fmt.Printf("是否是生产环境: %v\n", isMember)

	// ===== SREM：删除成员 =====
	rdb.SRem(ctx, "tags:server01", "v2")

	// ===== 集合运算 =====
	// 交集：两台服务器共同的标签
	inter, _ := rdb.SInter(ctx, "tags:server01", "tags:server02").Result()
	fmt.Printf("共同标签: %v\n", inter) // [production, golang]

	// 并集：所有标签
	union, _ := rdb.SUnion(ctx, "tags:server01", "tags:server02", "tags:server03").Result()
	fmt.Printf("所有标签: %v\n", union)

	// 差集：server01 有但 server02 没有的标签
	diff, _ := rdb.SDiff(ctx, "tags:server01", "tags:server02").Result()
	fmt.Printf("差集: %v\n", diff)

	// ===== SRANDMEMBER / SPOP =====
	// 随机获取成员（不删除）
	random, _ := rdb.SRandMember(ctx, "tags:server01").Result()
	fmt.Printf("随机标签: %s\n", random)

	// 随机弹出成员（删除）
	popped, _ := rdb.SPop(ctx, "tags:server03").Result()
	fmt.Printf("弹出标签: %s\n", popped)

	// ===== 实战：在线用户集合 =====
	rdb.SAdd(ctx, "online:users", "user1", "user2", "user3")
	rdb.SAdd(ctx, "online:users", "user4")
	rdb.SRem(ctx, "online:users", "user2") // user2 下线

	onlineCount, _ := rdb.SCard(ctx, "online:users").Result()
	fmt.Printf("在线用户数: %d\n", onlineCount)
}
```

### 2.5 Sorted Set 有序集合

```go
func sortedSetOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== ZADD：添加带分数的成员 =====
	rdb.ZAdd(ctx, "leaderboard", redis.Z{Score: 100, Member: "player1"})
	rdb.ZAdd(ctx, "leaderboard", redis.Z{Score: 85, Member: "player2"})
	rdb.ZAdd(ctx, "leaderboard",
		redis.Z{Score: 92, Member: "player3"},
		redis.Z{Score: 78, Member: "player4"},
		redis.Z{Score: 95, Member: "player5"},
	)

	// ===== ZRANGE：按分数升序获取 =====
	members, _ := rdb.ZRangeWithScores(ctx, "leaderboard", 0, -1).Result()
	fmt.Println("排行榜（升序）:")
	for _, m := range members {
		fmt.Printf("  %s: %.0f\n", m.Member, m.Score)
	}

	// ===== ZREVRANGE：按分数降序获取（排行榜）=====
	top3, _ := rdb.ZRevRangeWithScores(ctx, "leaderboard", 0, 2).Result()
	fmt.Println("Top 3:")
	for i, m := range top3 {
		fmt.Printf("  第%d名: %s (%.0f分)\n", i+1, m.Member, m.Score)
	}

	// ===== ZSCORE：获取成员分数 =====
	score, _ := rdb.ZScore(ctx, "leaderboard", "player1").Result()
	fmt.Printf("player1 分数: %.0f\n", score)

	// ===== ZRANK / ZREVRANK：获取排名 =====
	rank, _ := rdb.ZRevRank(ctx, "leaderboard", "player1").Result()
	fmt.Printf("player1 排名: %d（从0开始）\n", rank)

	// ===== ZINCRBY：增加分数 =====
	newScore, _ := rdb.ZIncrBy(ctx, "leaderboard", 10, "player2").Result()
	fmt.Printf("player2 新分数: %.0f\n", newScore)

	// ===== ZRANGEBYSCORE：按分数范围查询 =====
	// 分数在 80-100 之间的玩家
	result, _ := rdb.ZRangeByScoreWithScores(ctx, "leaderboard", &redis.ZRangeBy{
		Min: "80",
		Max: "100",
	}).Result()
	fmt.Println("分数 80-100:")
	for _, m := range result {
		fmt.Printf("  %s: %.0f\n", m.Member, m.Score)
	}

	// ===== ZREM：删除成员 =====
	rdb.ZRem(ctx, "leaderboard", "player4")

	// ===== ZCARD：获取成员数量 =====
	count, _ := rdb.ZCard(ctx, "leaderboard").Result()
	fmt.Printf("排行榜人数: %d\n", count)

	// ===== 实战：延迟任务队列 =====
	now := float64(time.Now().Unix())

	// 添加定时任务（分数 = 执行时间戳）
	rdb.ZAdd(ctx, "delayed:tasks",
		redis.Z{Score: now + 60, Member: `{"type":"send_email","to":"user@test.com"}`},
		redis.Z{Score: now + 120, Member: `{"type":"cleanup","target":"temp_files"}`},
		redis.Z{Score: now + 300, Member: `{"type":"report","name":"daily"}`},
	)

	// 消费到期的任务
	currentTime := float64(time.Now().Unix())
	tasks, _ := rdb.ZRangeByScore(ctx, "delayed:tasks", &redis.ZRangeBy{
		Min: "-inf",
		Max: fmt.Sprintf("%f", currentTime),
	}).Result()

	for _, task := range tasks {
		fmt.Printf("执行任务: %s\n", task)
		rdb.ZRem(ctx, "delayed:tasks", task)
	}
}
```

---

## 三、过期时间（TTL）

```go
func ttlOperations(rdb *redis.Client) {
	ctx := context.Background()

	// 设置值时指定过期时间
	rdb.Set(ctx, "cache:user:1001", "user_data", 10*time.Minute)

	// 设置已有键的过期时间
	rdb.Set(ctx, "persistent_key", "value", 0) // 不过期
	rdb.Expire(ctx, "persistent_key", 30*time.Minute) // 设置 30 分钟后过期

	// 设置精确的过期时间点
	rdb.ExpireAt(ctx, "persistent_key", time.Now().Add(24*time.Hour))

	// 查看剩余过期时间
	ttl, _ := rdb.TTL(ctx, "cache:user:1001").Result()
	if ttl == -1 {
		fmt.Println("键永不过期")
	} else if ttl == -2 {
		fmt.Println("键不存在")
	} else {
		fmt.Printf("剩余过期时间: %v\n", ttl)
	}

	// 移除过期时间（变为永不过期）
	rdb.Persist(ctx, "cache:user:1001")

	// 检查键是否存在
	exists, _ := rdb.Exists(ctx, "cache:user:1001").Result()
	fmt.Printf("键存在: %v\n", exists > 0)

	// 删除键
	rdb.Del(ctx, "cache:user:1001")

	// 批量删除
	rdb.Del(ctx, "k1", "k2", "k3")

	// 按模式查找键（注意：KEYS 在生产环境慎用，推荐 SCAN）
	// keys, _ := rdb.Keys(ctx, "cache:user:*").Result()

	// 使用 SCAN 安全地遍历键
	var cursor uint64
	var keys []string
	for {
		var batch []string
		var err error
		batch, cursor, err = rdb.Scan(ctx, cursor, "cache:*", 100).Result()
		if err != nil {
			break
		}
		keys = append(keys, batch...)
		if cursor == 0 {
			break
		}
	}
	fmt.Printf("匹配的键: %v\n", keys)
}
```

---

## 四、Pipeline 批量操作

Pipeline 将多个命令打包发送，减少网络往返，显著提高性能。

```go
func pipelineOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== 基本 Pipeline =====
	pipe := rdb.Pipeline()

	// 添加命令到 Pipeline（不会立即执行）
	incr := pipe.Incr(ctx, "pipeline:counter")
	pipe.Expire(ctx, "pipeline:counter", time.Hour)
	set := pipe.Set(ctx, "pipeline:name", "test", time.Minute)
	get := pipe.Get(ctx, "pipeline:name")

	// 执行所有命令
	_, err := pipe.Exec(ctx)
	if err != nil {
		fmt.Printf("Pipeline 执行失败: %v\n", err)
		return
	}

	// 获取各命令的结果
	fmt.Printf("INCR 结果: %d\n", incr.Val())
	fmt.Printf("SET 结果: %s\n", set.Val())
	fmt.Printf("GET 结果: %s\n", get.Val())

	// ===== Pipelined 便捷方法 =====
	var getResult *redis.StringCmd
	cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
		pipe.Set(ctx, "key1", "value1", time.Minute)
		pipe.Set(ctx, "key2", "value2", time.Minute)
		pipe.Set(ctx, "key3", "value3", time.Minute)
		getResult = pipe.Get(ctx, "key1")
		return nil
	})
	if err != nil {
		fmt.Printf("错误: %v\n", err)
	}
	fmt.Printf("命令数: %d\n", len(cmds))
	fmt.Printf("key1 = %s\n", getResult.Val())

	// ===== 批量设置缓存 =====
	type CacheItem struct {
		Key   string
		Value interface{}
		TTL   time.Duration
	}

	items := []CacheItem{
		{Key: "cache:user:1", Value: `{"name":"张三"}`, TTL: time.Hour},
		{Key: "cache:user:2", Value: `{"name":"李四"}`, TTL: time.Hour},
		{Key: "cache:user:3", Value: `{"name":"王五"}`, TTL: time.Hour},
	}

	pipe2 := rdb.Pipeline()
	for _, item := range items {
		pipe2.Set(ctx, item.Key, item.Value, item.TTL)
	}
	_, err = pipe2.Exec(ctx)
	if err != nil {
		fmt.Printf("批量缓存失败: %v\n", err)
	}

	// ===== 批量读取缓存 =====
	keys := []string{"cache:user:1", "cache:user:2", "cache:user:3"}
	pipe3 := rdb.Pipeline()
	cmdsMap := make(map[string]*redis.StringCmd)
	for _, key := range keys {
		cmdsMap[key] = pipe3.Get(ctx, key)
	}
	pipe3.Exec(ctx)

	for key, cmd := range cmdsMap {
		val, err := cmd.Result()
		if err == redis.Nil {
			fmt.Printf("%s: 未命中\n", key)
		} else if err != nil {
			fmt.Printf("%s: 错误 %v\n", key, err)
		} else {
			fmt.Printf("%s: %s\n", key, val)
		}
	}
}
```

---

## 五、事务（TxPipeline）

Redis 事务使用 MULTI/EXEC 保证命令的原子性执行。

```go
func transactionOperations(rdb *redis.Client) {
	ctx := context.Background()

	// ===== TxPipeline：事务 Pipeline =====
	// 所有命令在 EXEC 时原子执行
	txPipe := rdb.TxPipeline()
	txPipe.Set(ctx, "tx:key1", "value1", 0)
	txPipe.Set(ctx, "tx:key2", "value2", 0)
	txPipe.Incr(ctx, "tx:counter")
	_, err := txPipe.Exec(ctx)
	if err != nil {
		fmt.Printf("事务失败: %v\n", err)
	}

	// ===== TxPipelined 便捷方法 =====
	_, err = rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
		pipe.Set(ctx, "tx:balance:A", 1000, 0)
		pipe.Set(ctx, "tx:balance:B", 500, 0)
		return nil
	})

	// ===== Watch：乐观锁事务 =====
	// 用于实现"检查再操作"的原子性

	// 示例：安全的转账操作
	transferAmount := int64(100)
	err = rdb.Watch(ctx, func(tx *redis.Tx) error {
		// 读取当前余额
		balanceA, err := tx.Get(ctx, "tx:balance:A").Int64()
		if err != nil {
			return err
		}

		if balanceA < transferAmount {
			return fmt.Errorf("余额不足: %d < %d", balanceA, transferAmount)
		}

		// 如果在这期间 balance:A 或 balance:B 被其他客户端修改，
		// 事务会失败并返回 redis.TxFailedErr
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.DecrBy(ctx, "tx:balance:A", transferAmount)
			pipe.IncrBy(ctx, "tx:balance:B", transferAmount)
			return nil
		})
		return err
	}, "tx:balance:A", "tx:balance:B") // 监视这两个键

	if err == redis.TxFailedErr {
		fmt.Println("事务失败（键被其他客户端修改），需要重试")
	} else if err != nil {
		fmt.Printf("转账失败: %v\n", err)
	} else {
		fmt.Println("转账成功")
	}
}
```

---

## 六、Pub/Sub 发布订阅

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// Publisher 发布者
func Publisher(rdb *redis.Client) {
	ctx := context.Background()

	// 发布消息到频道
	for i := 0; i < 5; i++ {
		msg := fmt.Sprintf(`{"type":"alert","level":"warning","message":"CPU 使用率 %d%%","time":"%s"}`,
			80+i, time.Now().Format(time.RFC3339))

		err := rdb.Publish(ctx, "alerts", msg).Err()
		if err != nil {
			fmt.Printf("发布失败: %v\n", err)
		}
		fmt.Printf("已发布: %s\n", msg)
		time.Sleep(time.Second)
	}

	// 发布到不同频道
	rdb.Publish(ctx, "alerts:critical", `{"message":"数据库连接丢失"}`)
	rdb.Publish(ctx, "alerts:info", `{"message":"部署完成"}`)
	rdb.Publish(ctx, "metrics", `{"cpu":85,"memory":72,"disk":45}`)
}

// Subscriber 订阅者
func Subscriber(rdb *redis.Client) {
	ctx := context.Background()

	// 订阅单个频道
	sub := rdb.Subscribe(ctx, "alerts")
	defer sub.Close()

	// 等待订阅确认
	_, err := sub.Receive(ctx)
	if err != nil {
		fmt.Printf("订阅失败: %v\n", err)
		return
	}
	fmt.Println("已订阅 alerts 频道")

	// 获取消息通道
	ch := sub.Channel()

	// 接收消息
	for msg := range ch {
		fmt.Printf("收到消息 [%s]: %s\n", msg.Channel, msg.Payload)
	}
}

// PatternSubscriber 模式订阅
func PatternSubscriber(rdb *redis.Client) {
	ctx := context.Background()

	// 使用通配符订阅多个频道
	sub := rdb.PSubscribe(ctx, "alerts:*")
	defer sub.Close()

	ch := sub.Channel()

	for msg := range ch {
		fmt.Printf("模式订阅收到 [%s -> %s]: %s\n",
			msg.Pattern, msg.Channel, msg.Payload)
	}
}

func main() {
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer rdb.Close()

	// 启动订阅者（在 goroutine 中）
	go Subscriber(rdb)
	go PatternSubscriber(rdb)

	// 等待订阅者就绪
	time.Sleep(time.Second)

	// 发布消息
	Publisher(rdb)

	time.Sleep(2 * time.Second)
}
```

---

## 七、分布式锁

### 7.1 SetNX 简单分布式锁

```go
package redislock

import (
	"context"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

// DistributedLock 分布式锁
type DistributedLock struct {
	client *redis.Client
	key    string
	value  string // 唯一标识，用于安全释放
	ttl    time.Duration
}

// NewDistributedLock 创建分布式锁
func NewDistributedLock(client *redis.Client, key string, ttl time.Duration) *DistributedLock {
	return &DistributedLock{
		client: client,
		key:    "lock:" + key,
		value:  uuid.New().String(), // 唯一持有者标识
		ttl:    ttl,
	}
}

// Lock 获取锁
func (l *DistributedLock) Lock(ctx context.Context) (bool, error) {
	ok, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
	return ok, err
}

// LockWithRetry 获取锁（带重试）
func (l *DistributedLock) LockWithRetry(ctx context.Context, retryCount int, retryDelay time.Duration) (bool, error) {
	for i := 0; i < retryCount; i++ {
		ok, err := l.Lock(ctx)
		if err != nil {
			return false, err
		}
		if ok {
			return true, nil
		}

		// 等待后重试
		select {
		case <-ctx.Done():
			return false, ctx.Err()
		case <-time.After(retryDelay):
			// 继续重试
		}
	}
	return false, nil
}

// Unlock 释放锁（使用 Lua 脚本保证原子性）
func (l *DistributedLock) Unlock(ctx context.Context) (bool, error) {
	// Lua 脚本：只有持有者才能释放锁
	script := redis.NewScript(`
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("del", KEYS[1])
		else
			return 0
		end
	`)

	result, err := script.Run(ctx, l.client, []string{l.key}, l.value).Int64()
	if err != nil {
		return false, err
	}
	return result == 1, nil
}

// Extend 续期锁
func (l *DistributedLock) Extend(ctx context.Context, extraTTL time.Duration) (bool, error) {
	script := redis.NewScript(`
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("pexpire", KEYS[1], ARGV[2])
		else
			return 0
		end
	`)

	result, err := script.Run(ctx, l.client, []string{l.key},
		l.value, extraTTL.Milliseconds()).Int64()
	if err != nil {
		return false, err
	}
	return result == 1, nil
}

// ===== 使用示例 =====

func DeployWithLock(rdb *redis.Client) error {
	ctx := context.Background()

	// 创建部署锁（30 秒过期）
	lock := NewDistributedLock(rdb, "deploy:production", 30*time.Second)

	// 尝试获取锁（最多重试 5 次，每次间隔 1 秒）
	ok, err := lock.LockWithRetry(ctx, 5, time.Second)
	if err != nil {
		return fmt.Errorf("获取锁失败: %w", err)
	}
	if !ok {
		return fmt.Errorf("无法获取部署锁，可能有其他部署正在进行")
	}

	// 确保释放锁
	defer func() {
		unlocked, err := lock.Unlock(ctx)
		if err != nil {
			fmt.Printf("释放锁失败: %v\n", err)
		}
		if !unlocked {
			fmt.Println("警告：锁可能已过期或被其他进程释放")
		}
	}()

	// 执行部署操作
	fmt.Println("开始部署...")
	time.Sleep(5 * time.Second) // 模拟部署
	fmt.Println("部署完成")

	return nil
}
```

### 7.2 RedLock 算法（多节点分布式锁）

```go
package redislock

import (
	"context"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

// RedLock 基于多个 Redis 节点的分布式锁
type RedLock struct {
	clients []*redis.Client
	key     string
	value   string
	ttl     time.Duration
}

// NewRedLock 创建 RedLock
func NewRedLock(clients []*redis.Client, key string, ttl time.Duration) *RedLock {
	return &RedLock{
		clients: clients,
		key:     "lock:" + key,
		value:   uuid.New().String(),
		ttl:     ttl,
	}
}

// Lock 获取 RedLock
func (rl *RedLock) Lock(ctx context.Context) (bool, error) {
	n := len(rl.clients)
	quorum := n/2 + 1 // 多数派

	start := time.Now()
	successCount := 0

	// 尝试在所有节点上获取锁
	for _, client := range rl.clients {
		ok, err := client.SetNX(ctx, rl.key, rl.value, rl.ttl).Result()
		if err == nil && ok {
			successCount++
		}
	}

	elapsed := time.Since(start)

	// 检查是否在多数派节点上获取成功，且总耗时小于 TTL
	if successCount >= quorum && elapsed < rl.ttl {
		return true, nil
	}

	// 获取失败，释放已获取的锁
	rl.Unlock(ctx)
	return false, nil
}

// Unlock 释放所有节点上的锁
func (rl *RedLock) Unlock(ctx context.Context) {
	script := redis.NewScript(`
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("del", KEYS[1])
		end
		return 0
	`)

	for _, client := range rl.clients {
		script.Run(ctx, client, []string{rl.key}, rl.value)
	}
}
```

---

## 八、缓存模式

### 8.1 Cache Aside（旁路缓存）

```go
package cache

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"gorm.io/gorm"
)

// UserCache 用户缓存（Cache Aside 模式）
type UserCache struct {
	rdb      *redis.Client
	db       *gorm.DB
	cacheTTL time.Duration
}

// User 用户模型
type User struct {
	ID    uint   `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func NewUserCache(rdb *redis.Client, db *gorm.DB) *UserCache {
	return &UserCache{
		rdb:      rdb,
		db:       db,
		cacheTTL: 30 * time.Minute,
	}
}

func (uc *UserCache) cacheKey(id uint) string {
	return fmt.Sprintf("cache:user:%d", id)
}

// GetUser Cache Aside 读取流程
// 1. 先读缓存
// 2. 缓存命中 -> 直接返回
// 3. 缓存未命中 -> 查数据库 -> 写入缓存 -> 返回
func (uc *UserCache) GetUser(ctx context.Context, id uint) (*User, error) {
	key := uc.cacheKey(id)

	// 1. 尝试从缓存读取
	cached, err := uc.rdb.Get(ctx, key).Result()
	if err == nil {
		// 缓存命中
		var user User
		if err := json.Unmarshal([]byte(cached), &user); err == nil {
			return &user, nil
		}
	}
	if err != nil && err != redis.Nil {
		// Redis 错误，降级到数据库
		fmt.Printf("Redis 读取错误（已降级）: %v\n", err)
	}

	// 2. 缓存未命中，查数据库
	var user User
	if err := uc.db.First(&user, id).Error; err != nil {
		if err == gorm.ErrRecordNotFound {
			// 防止缓存穿透：缓存空值
			uc.rdb.Set(ctx, key, "null", 5*time.Minute)
			return nil, nil
		}
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}

	// 3. 写入缓存
	data, _ := json.Marshal(user)
	uc.rdb.Set(ctx, key, data, uc.cacheTTL)

	return &user, nil
}

// UpdateUser Cache Aside 更新流程
// 1. 更新数据库
// 2. 删除缓存（不是更新缓存！）
func (uc *UserCache) UpdateUser(ctx context.Context, user *User) error {
	// 1. 更新数据库
	if err := uc.db.Save(user).Error; err != nil {
		return fmt.Errorf("更新用户失败: %w", err)
	}

	// 2. 删除缓存
	key := uc.cacheKey(user.ID)
	uc.rdb.Del(ctx, key)

	return nil
}

// DeleteUser 删除用户
func (uc *UserCache) DeleteUser(ctx context.Context, id uint) error {
	// 1. 删除数据库记录
	if err := uc.db.Delete(&User{}, id).Error; err != nil {
		return fmt.Errorf("删除用户失败: %w", err)
	}

	// 2. 删除缓存
	uc.rdb.Del(ctx, uc.cacheKey(id))

	return nil
}
```

### 8.2 缓存穿透/雪崩/击穿防护

```go
package cache

import (
	"context"
	"math/rand"
	"sync"
	"time"

	"github.com/redis/go-redis/v9"
)

// CacheGuard 缓存防护器
type CacheGuard struct {
	rdb       *redis.Client
	mu        sync.Mutex
	singleFlight map[string]*sync.WaitGroup
}

// 防止缓存雪崩：给 TTL 添加随机偏移
func randomTTL(baseTTL time.Duration) time.Duration {
	// 在基础 TTL 的基础上增加 0-20% 的随机偏移
	jitter := time.Duration(rand.Int63n(int64(baseTTL) / 5))
	return baseTTL + jitter
}

// 防止缓存击穿：使用 singleflight 模式
func (cg *CacheGuard) GetOrLoad(ctx context.Context, key string, loadFn func() (string, error), ttl time.Duration) (string, error) {
	// 先尝试从缓存获取
	val, err := cg.rdb.Get(ctx, key).Result()
	if err == nil {
		return val, nil
	}

	// singleflight：同一时间同一个 key 只有一个请求去加载数据
	cg.mu.Lock()
	if cg.singleFlight == nil {
		cg.singleFlight = make(map[string]*sync.WaitGroup)
	}

	if wg, ok := cg.singleFlight[key]; ok {
		cg.mu.Unlock()
		// 等待正在加载的请求完成
		wg.Wait()
		// 再次尝试从缓存获取
		return cg.rdb.Get(ctx, key).Result()
	}

	// 创建 WaitGroup 并开始加载
	wg := &sync.WaitGroup{}
	wg.Add(1)
	cg.singleFlight[key] = wg
	cg.mu.Unlock()

	defer func() {
		wg.Done()
		cg.mu.Lock()
		delete(cg.singleFlight, key)
		cg.mu.Unlock()
	}()

	// 从数据源加载
	val, err = loadFn()
	if err != nil {
		return "", err
	}

	// 写入缓存（带随机 TTL 偏移防止雪崩）
	cg.rdb.Set(ctx, key, val, randomTTL(ttl))

	return val, nil
}
```

---

## 九、连接池配置

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,

		// ===== 连接池配置 =====
		PoolSize:     20,                // 最大连接数（默认: runtime.GOMAXPROCS * 10）
		MinIdleConns: 5,                 // 最小空闲连接数
		MaxIdleConns: 10,                // 最大空闲连接数
		PoolFIFO:     false,             // 连接池是否使用 FIFO（默认 LIFO）

		// ===== 超时配置 =====
		DialTimeout:  5 * time.Second,   // 建立连接超时
		ReadTimeout:  3 * time.Second,   // 读超时（-1 表示无限）
		WriteTimeout: 3 * time.Second,   // 写超时（-1 表示无限）
		PoolTimeout:  4 * time.Second,   // 从连接池获取连接的超时

		// ===== 重试配置 =====
		MaxRetries:      3,              // 最大重试次数（-1 表示不重试）
		MinRetryBackoff: 8 * time.Millisecond,   // 最小重试间隔
		MaxRetryBackoff: 512 * time.Millisecond, // 最大重试间隔

		// ===== 连接健康检查 =====
		ConnMaxIdleTime: 30 * time.Minute, // 空闲连接最大存活时间
		ConnMaxLifetime: 0,                // 连接最大存活时间（0 表示不限制）
	})
	defer rdb.Close()

	ctx := context.Background()

	// 查看连接池状态
	stats := rdb.PoolStats()
	fmt.Printf("连接池状态:\n")
	fmt.Printf("  总连接数: %d\n", stats.TotalConns)
	fmt.Printf("  空闲连接: %d\n", stats.IdleConns)
	fmt.Printf("  过期连接: %d\n", stats.StaleConns)
	fmt.Printf("  命中次数: %d\n", stats.Hits)
	fmt.Printf("  未命中次数: %d\n", stats.Misses)
	fmt.Printf("  超时次数: %d\n", stats.Timeouts)
}
```

---

## 十、Sentinel 和 Cluster 模式

### 10.1 Sentinel 哨兵模式

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func connectSentinel() *redis.Client {
	rdb := redis.NewFailoverClient(&redis.FailoverOptions{
		MasterName:    "mymaster",                                     // 主节点名称
		SentinelAddrs: []string{"sentinel1:26379", "sentinel2:26379", "sentinel3:26379"}, // 哨兵地址
		Password:      "",                                             // Redis 密码
		DB:            0,

		// 可选：哨兵认证
		SentinelPassword: "",

		// 连接池配置
		PoolSize:     20,
		MinIdleConns: 5,

		// 读写分离：从从节点读取
		// RouteByLatency: true,
		// RouteRandomly:  true,
	})

	return rdb
}
```

### 10.2 Cluster 集群模式

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

func connectCluster() *redis.ClusterClient {
	rdb := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{
			"redis-node1:6379",
			"redis-node2:6379",
			"redis-node3:6379",
			"redis-node4:6379",
			"redis-node5:6379",
			"redis-node6:6379",
		},
		Password: "",

		// 连接池配置（每个节点）
		PoolSize:     10,
		MinIdleConns: 3,

		// 超时配置
		DialTimeout:  5 * time.Second,
		ReadTimeout:  3 * time.Second,
		WriteTimeout: 3 * time.Second,

		// 重试配置
		MaxRetries: 3,

		// 读写分离
		ReadOnly:       false, // 是否只从从节点读取
		RouteByLatency: true,  // 按延迟路由读请求
		RouteRandomly:  false, // 随机路由读请求
	})

	return rdb
}

func clusterOperations(rdb *redis.ClusterClient) {
	ctx := context.Background()

	// 基本操作与单机 Redis 相同
	rdb.Set(ctx, "cluster:key1", "value1", time.Hour)
	val, _ := rdb.Get(ctx, "cluster:key1").Result()
	fmt.Printf("集群读取: %s\n", val)

	// Pipeline 在集群模式下也可用（go-redis 自动按 slot 分组）
	pipe := rdb.Pipeline()
	pipe.Set(ctx, "ck1", "v1", time.Hour)
	pipe.Set(ctx, "ck2", "v2", time.Hour)
	pipe.Set(ctx, "ck3", "v3", time.Hour)
	pipe.Exec(ctx)

	// 获取集群信息
	info := rdb.ClusterInfo(ctx)
	fmt.Printf("集群信息: %s\n", info.Val())

	// 遍历所有节点
	rdb.ForEachShard(ctx, func(ctx context.Context, client *redis.Client) error {
		return client.Ping(ctx).Err()
	})
}
```

---

## 十一、常见坑点

### 坑点 1：忘记处理 redis.Nil

```go
// 错误：不区分"不存在"和真正的错误
val, err := rdb.Get(ctx, "key").Result()
if err != nil {
	log.Fatal(err) // key 不存在也会 Fatal！
}

// 正确：区分处理
val, err := rdb.Get(ctx, "key").Result()
if err == redis.Nil {
	// 键不存在——正常业务逻辑
	fmt.Println("键不存在")
} else if err != nil {
	// 真正的 Redis 错误
	log.Printf("Redis 错误: %v", err)
} else {
	fmt.Printf("值: %s\n", val)
}
```

### 坑点 2：大 Key 问题

```go
// 错误：存储超大 value
bigData := strings.Repeat("x", 100*1024*1024) // 100MB
rdb.Set(ctx, "big_key", bigData, 0) // 会阻塞 Redis

// 正确：拆分大 Key
// - 大字符串：拆分为多个小 key
// - 大 Hash：拆分为多个 Hash
// - 大 List：使用分片 List
// - 大 Set：使用分片 Set

// 检查 Key 大小
debug, _ := rdb.DebugObject(ctx, "some_key").Result()
fmt.Println(debug)
```

### 坑点 3：KEYS 命令在生产环境

```go
// 错误：生产环境使用 KEYS（会阻塞 Redis）
keys, _ := rdb.Keys(ctx, "cache:*").Result()

// 正确：使用 SCAN 命令
var cursor uint64
for {
	keys, cursor, err := rdb.Scan(ctx, cursor, "cache:*", 100).Result()
	if err != nil {
		break
	}
	// 处理 keys...
	_ = keys
	if cursor == 0 {
		break
	}
}
```

### 坑点 4：分布式锁不安全释放

```go
// 错误：直接 DEL（可能释放别人的锁）
rdb.SetNX(ctx, "lock:job", "holder1", 10*time.Second)
// ... 执行时间超过 10 秒，锁过期
// holder2 获取了锁
rdb.Del(ctx, "lock:job") // 错！释放了 holder2 的锁

// 正确：使用 Lua 脚本原子性检查+删除
// 参见上面的 DistributedLock.Unlock() 实现
```

### 坑点 5：缓存更新时先删缓存再更数据库

```go
// 错误顺序：先删缓存，再更新数据库
rdb.Del(ctx, "cache:user:1") // 1. 删缓存
db.Save(&user)                // 2. 更新数据库
// 问题：在步骤1和2之间，另一个请求可能读到旧数据并写入缓存

// 正确顺序：先更新数据库，再删缓存（Cache Aside 模式）
db.Save(&user)                // 1. 更新数据库
rdb.Del(ctx, "cache:user:1") // 2. 删缓存
```

---

## 十二、速查表

### 常用命令映射

| Redis 命令 | go-redis 方法 |
|------------|---------------|
| `SET key value EX 60` | `rdb.Set(ctx, key, value, 60*time.Second)` |
| `GET key` | `rdb.Get(ctx, key).Result()` |
| `DEL key` | `rdb.Del(ctx, key)` |
| `EXPIRE key 60` | `rdb.Expire(ctx, key, 60*time.Second)` |
| `TTL key` | `rdb.TTL(ctx, key).Result()` |
| `INCR key` | `rdb.Incr(ctx, key)` |
| `HSET key f v` | `rdb.HSet(ctx, key, f, v)` |
| `HGET key f` | `rdb.HGet(ctx, key, f).Result()` |
| `HGETALL key` | `rdb.HGetAll(ctx, key).Result()` |
| `LPUSH key v` | `rdb.LPush(ctx, key, v)` |
| `RPOP key` | `rdb.RPop(ctx, key).Result()` |
| `SADD key m` | `rdb.SAdd(ctx, key, m)` |
| `SMEMBERS key` | `rdb.SMembers(ctx, key).Result()` |
| `ZADD key score m` | `rdb.ZAdd(ctx, key, redis.Z{Score, Member})` |
| `ZREVRANGE key 0 9` | `rdb.ZRevRange(ctx, key, 0, 9).Result()` |
| `PUBLISH ch msg` | `rdb.Publish(ctx, ch, msg)` |
| `SUBSCRIBE ch` | `rdb.Subscribe(ctx, ch)` |

### 缓存策略对比

| 策略 | 读流程 | 写流程 | 优点 | 缺点 |
|------|--------|--------|------|------|
| Cache Aside | 先读缓存，未命中读DB写缓存 | 先写DB，再删缓存 | 简单，适用广 | 有短暂不一致 |
| Read Through | 缓存层自动从DB加载 | 同 Cache Aside | 业务代码简洁 | 缓存层复杂 |
| Write Through | 同 Read Through | 缓存层同步写DB | 一致性好 | 写入延迟高 |
| Write Behind | 同 Read Through | 缓存层异步写DB | 写入快 | 可能丢数据 |
