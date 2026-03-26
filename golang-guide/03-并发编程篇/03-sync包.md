# 03 - sync 包

## 导读

虽然 Go 推荐使用 channel 进行 goroutine 间通信（CSP 模型），但在很多实际场景中，使用传统的同步原语（锁、等待组、原子操作等）更加直观和高效。Go 的 `sync` 包和 `sync/atomic` 包提供了一套完整的并发同步工具。

作为 SRE/DevOps 工程师，你会在以下场景使用 sync 包：
- 保护共享配置数据的并发读写（RWMutex）
- 等待一组并发任务完成（WaitGroup）
- 确保初始化只执行一次（Once）
- 高性能的并发安全 Map（sync.Map）
- 减少 GC 压力的对象池（sync.Pool）
- 无锁的原子计数器和状态标记（atomic）

本篇将逐一讲解这些工具的用法、适用场景和性能特点。

---

## 1. sync.Mutex 互斥锁

### 1.1 基本用法

Mutex 是最基本的锁，同一时间只有一个 goroutine 可以持有锁。

```go
package main

import (
	"fmt"
	"sync"
)

// 没有锁的危险示例
func unsafeCounter() {
	var count int
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			count++ // 数据竞争！多个 goroutine 同时读写 count
		}()
	}
	wg.Wait()
	// 结果不确定，可能不是 1000
	fmt.Printf("不安全计数器: %d（期望 1000）\n", count)
}

// 使用 Mutex 保护的安全示例
func safeCounter() {
	var count int
	var mu sync.Mutex
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()   // 加锁
			count++     // 安全地修改
			mu.Unlock() // 解锁
		}()
	}
	wg.Wait()
	fmt.Printf("安全计数器: %d（期望 1000）\n", count)
}

func main() {
	unsafeCounter()
	safeCounter()
}
```

### 1.2 在结构体中使用 Mutex

```go
package main

import (
	"fmt"
	"sync"
)

// SafeMap 线程安全的 Map
type SafeMap struct {
	mu   sync.Mutex
	data map[string]string
}

func NewSafeMap() *SafeMap {
	return &SafeMap{
		data: make(map[string]string),
	}
}

func (m *SafeMap) Set(key, value string) {
	m.mu.Lock()
	defer m.mu.Unlock() // 使用 defer 确保一定会解锁
	m.data[key] = value
}

func (m *SafeMap) Get(key string) (string, bool) {
	m.mu.Lock()
	defer m.mu.Unlock()
	val, ok := m.data[key]
	return val, ok
}

func (m *SafeMap) Delete(key string) {
	m.mu.Lock()
	defer m.mu.Unlock()
	delete(m.data, key)
}

func (m *SafeMap) Len() int {
	m.mu.Lock()
	defer m.mu.Unlock()
	return len(m.data)
}

func main() {
	sm := NewSafeMap()
	var wg sync.WaitGroup

	// 并发写入
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			key := fmt.Sprintf("key-%d", id)
			sm.Set(key, fmt.Sprintf("value-%d", id))
		}(i)
	}

	wg.Wait()
	fmt.Printf("Map 中有 %d 个元素\n", sm.Len())

	// 并发读取
	if val, ok := sm.Get("key-42"); ok {
		fmt.Println("key-42 =", val)
	}
}
```

### 1.3 Mutex 注意事项

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var mu sync.Mutex

	// 注意1: 不要复制 Mutex
	// mu2 := mu  // 错误！复制 Mutex 会导致未定义行为
	// 正确的做法是传递指针

	// 注意2: 不要在未加锁时 Unlock
	// mu.Unlock() // panic: sync: unlock of unlocked mutex

	// 注意3: 不要重复加锁（Go 的 Mutex 不可重入）
	// mu.Lock()
	// mu.Lock() // 死锁！同一个 goroutine 不能对同一把锁加两次

	// 注意4: Lock/Unlock 必须在同一个 goroutine 中配对
	// （虽然技术上可以在不同 goroutine 中 Lock/Unlock，但强烈不推荐）

	// 注意5: 使用 defer 确保解锁
	mu.Lock()
	defer mu.Unlock()
	fmt.Println("使用 defer 确保解锁，即使发生 panic 也能正确释放锁")
}
```

---

## 2. sync.RWMutex 读写锁

### 2.1 基本用法

RWMutex 允许多个读操作并发执行，但写操作是排他的：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Config 使用读写锁保护的配置
type Config struct {
	mu       sync.RWMutex
	settings map[string]string
}

func NewConfig() *Config {
	return &Config{
		settings: map[string]string{
			"timeout":    "30s",
			"max_retry":  "3",
			"log_level":  "info",
		},
	}
}

// Get 读操作 - 使用读锁，多个 goroutine 可以同时读
func (c *Config) Get(key string) string {
	c.mu.RLock()         // 读锁
	defer c.mu.RUnlock() // 释放读锁
	return c.settings[key]
}

// Set 写操作 - 使用写锁，排他的
func (c *Config) Set(key, value string) {
	c.mu.Lock()         // 写锁
	defer c.mu.Unlock() // 释放写锁
	c.settings[key] = value
}

// GetAll 读取所有配置
func (c *Config) GetAll() map[string]string {
	c.mu.RLock()
	defer c.mu.RUnlock()
	// 返回副本，避免外部修改
	result := make(map[string]string, len(c.settings))
	for k, v := range c.settings {
		result[k] = v
	}
	return result
}

func main() {
	config := NewConfig()
	var wg sync.WaitGroup

	// 启动多个读 goroutine
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 100; j++ {
				_ = config.Get("timeout")
			}
			fmt.Printf("Reader %d 完成 100 次读取\n", id)
		}(i)
	}

	// 启动少量写 goroutine
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 10; j++ {
				config.Set("timeout", fmt.Sprintf("%ds", 30+j))
				time.Sleep(10 * time.Millisecond)
			}
			fmt.Printf("Writer %d 完成 10 次写入\n", id)
		}(i)
	}

	wg.Wait()
	fmt.Println("最终配置:", config.GetAll())
}
```

### 2.2 读写锁 vs 互斥锁性能对比

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// 使用 Mutex
func benchMutex(readers, writers int, iterations int) time.Duration {
	var mu sync.Mutex
	data := make(map[string]string)
	data["key"] = "value"
	var wg sync.WaitGroup

	start := time.Now()

	for i := 0; i < readers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations; j++ {
				mu.Lock()
				_ = data["key"]
				mu.Unlock()
			}
		}()
	}

	for i := 0; i < writers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations/10; j++ {
				mu.Lock()
				data["key"] = "value"
				mu.Unlock()
			}
		}()
	}

	wg.Wait()
	return time.Since(start)
}

// 使用 RWMutex
func benchRWMutex(readers, writers int, iterations int) time.Duration {
	var mu sync.RWMutex
	data := make(map[string]string)
	data["key"] = "value"
	var wg sync.WaitGroup

	start := time.Now()

	for i := 0; i < readers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations; j++ {
				mu.RLock()
				_ = data["key"]
				mu.RUnlock()
			}
		}()
	}

	for i := 0; i < writers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations/10; j++ {
				mu.Lock()
				data["key"] = "value"
				mu.Unlock()
			}
		}()
	}

	wg.Wait()
	return time.Since(start)
}

func main() {
	readers := 100
	writers := 2
	iterations := 10000

	fmt.Printf("测试条件: %d 读者, %d 写者, 每个读者 %d 次操作\n\n", readers, writers, iterations)

	mutexTime := benchMutex(readers, writers, iterations)
	fmt.Printf("sync.Mutex:   %v\n", mutexTime)

	rwMutexTime := benchRWMutex(readers, writers, iterations)
	fmt.Printf("sync.RWMutex: %v\n", rwMutexTime)

	if rwMutexTime < mutexTime {
		speedup := float64(mutexTime) / float64(rwMutexTime)
		fmt.Printf("\nRWMutex 快 %.1fx（读多写少场景）\n", speedup)
	}

	fmt.Println("\n结论：读多写少用 RWMutex，读写均衡用 Mutex")
}
```

---

## 3. sync.WaitGroup 等待组

### 3.1 基本用法

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup

	tasks := []string{"备份数据库", "清理日志", "更新证书", "重启服务", "验证健康"}

	for _, task := range tasks {
		wg.Add(1) // 在启动 goroutine 之前 Add
		go func(t string) {
			defer wg.Done() // 完成时 Done
			fmt.Printf("开始: %s\n", t)
			time.Sleep(time.Duration(100+len(t)*50) * time.Millisecond)
			fmt.Printf("完成: %s\n", t)
		}(task)
	}

	fmt.Println("等待所有任务完成...")
	wg.Wait() // 阻塞直到计数器归零
	fmt.Println("所有任务已完成！")
}
```

### 3.2 WaitGroup 的正确和错误用法

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 正确用法: 在 go 之前 Add
	fmt.Println("=== 正确用法 ===")
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1) // 正确位置
		go func(id int) {
			defer wg.Done()
			fmt.Printf("Worker %d\n", id)
		}(i)
	}
	wg.Wait()

	// 也可以一次性 Add
	fmt.Println("\n=== 一次性 Add ===")
	n := 5
	wg.Add(n) // 一次性 Add n
	for i := 0; i < n; i++ {
		go func(id int) {
			defer wg.Done()
			fmt.Printf("Worker %d\n", id)
		}(i)
	}
	wg.Wait()

	// 嵌套 WaitGroup
	fmt.Println("\n=== 嵌套 WaitGroup ===")
	var outerWg sync.WaitGroup
	for i := 0; i < 3; i++ {
		outerWg.Add(1)
		go func(groupID int) {
			defer outerWg.Done()
			var innerWg sync.WaitGroup

			for j := 0; j < 3; j++ {
				innerWg.Add(1)
				go func(taskID int) {
					defer innerWg.Done()
					fmt.Printf("Group %d, Task %d\n", groupID, taskID)
				}(j)
			}

			innerWg.Wait() // 等待组内所有任务
			fmt.Printf("Group %d 全部完成\n", groupID)
		}(i)
	}
	outerWg.Wait()
	fmt.Println("全部组完成")
}
```

### 3.3 带超时的 WaitGroup

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// WaitGroupWithTimeout 带超时的等待
func WaitGroupWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		return true // 正常完成
	case <-time.After(timeout):
		return false // 超时
	}
}

func main() {
	var wg sync.WaitGroup

	// 场景1: 快速完成
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(100 * time.Millisecond)
	}()

	if WaitGroupWithTimeout(&wg, time.Second) {
		fmt.Println("场景1: 任务在超时前完成")
	} else {
		fmt.Println("场景1: 超时")
	}

	// 场景2: 超时
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(2 * time.Second)
	}()

	if WaitGroupWithTimeout(&wg, 500*time.Millisecond) {
		fmt.Println("场景2: 任务在超时前完成")
	} else {
		fmt.Println("场景2: 超时！部分任务未完成")
	}

	time.Sleep(2 * time.Second) // 等待泄漏的 goroutine 完成
}
```

---

## 4. sync.Once 只执行一次

### 4.1 基本用法

```go
package main

import (
	"fmt"
	"sync"
)

var (
	instance *Database
	once     sync.Once
)

// Database 模拟数据库连接
type Database struct {
	DSN string
}

// GetDatabase 获取数据库单例（线程安全的懒加载）
func GetDatabase() *Database {
	once.Do(func() {
		fmt.Println("初始化数据库连接（只执行一次）")
		instance = &Database{DSN: "postgres://localhost:5432/mydb"}
	})
	return instance
}

func main() {
	var wg sync.WaitGroup

	// 多个 goroutine 同时获取数据库实例
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			db := GetDatabase()
			fmt.Printf("Goroutine %d: DB=%p DSN=%s\n", id, db, db.DSN)
		}(i)
	}

	wg.Wait()
	// 所有 goroutine 获取到的是同一个实例
}
```

### 4.2 Once 的注意事项

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 注意1: Once.Do 中如果 panic，后续调用不会重新执行
	fmt.Println("=== Once + panic ===")
	var once1 sync.Once
	func() {
		defer func() { recover() }()
		once1.Do(func() {
			fmt.Println("第一次执行（会 panic）")
			panic("初始化失败")
		})
	}()

	once1.Do(func() {
		fmt.Println("第二次调用（不会执行！）")
	})
	fmt.Println("即使第一次 panic，Once 也认为已执行过")

	// 注意2: Go 1.21+ 新增 OnceFunc, OnceValue, OnceValues
	fmt.Println("\n=== Go 1.21+ OnceFunc / OnceValue ===")

	// sync.OnceFunc - 将函数包装为只执行一次
	init := sync.OnceFunc(func() {
		fmt.Println("OnceFunc: 初始化")
	})
	init() // 执行
	init() // 不执行
	init() // 不执行

	// sync.OnceValue - 只计算一次并缓存结果
	getConfig := sync.OnceValue(func() string {
		fmt.Println("OnceValue: 加载配置")
		return "production"
	})
	fmt.Println("Config:", getConfig()) // 执行并返回
	fmt.Println("Config:", getConfig()) // 直接返回缓存值

	// sync.OnceValues - 返回两个值（常用于 value, error 模式）
	loadDB := sync.OnceValues(func() (*Database2, error) {
		fmt.Println("OnceValues: 连接数据库")
		return &Database2{Connected: true}, nil
	})
	db, err := loadDB()
	fmt.Printf("DB: %+v, Err: %v\n", db, err)
	db2, err2 := loadDB() // 直接返回缓存
	fmt.Printf("DB: %+v, Err: %v (同一实例: %v)\n", db2, err2, db == db2)
}

type Database2 struct {
	Connected bool
}
```

---

## 5. sync.Pool 对象池

### 5.1 基本用法

sync.Pool 用于缓存和复用临时对象，减少内存分配和 GC 压力。

```go
package main

import (
	"bytes"
	"fmt"
	"sync"
)

func main() {
	// 创建对象池
	bufferPool := sync.Pool{
		New: func() any {
			fmt.Println("创建新的 Buffer")
			return new(bytes.Buffer)
		},
	}

	// 从池中获取（如果池为空，调用 New 创建）
	buf := bufferPool.Get().(*bytes.Buffer)
	buf.WriteString("Hello, Pool!")
	fmt.Println("Buffer 内容:", buf.String())

	// 用完后放回池中（注意要 Reset）
	buf.Reset()
	bufferPool.Put(buf)

	// 再次获取（可能是之前放回的对象）
	buf2 := bufferPool.Get().(*bytes.Buffer)
	fmt.Printf("获取的 Buffer 是否为空: %v\n", buf2.Len() == 0)

	// 注意：Pool 中的对象可能在任何时候被 GC 回收
	// 不要依赖 Pool 中的对象一定存在
}
```

### 5.2 SRE 场景：高性能日志格式化

```go
package main

import (
	"bytes"
	"fmt"
	"sync"
	"time"
)

// LogFormatter 高性能日志格式化器
type LogFormatter struct {
	bufPool sync.Pool
}

func NewLogFormatter() *LogFormatter {
	return &LogFormatter{
		bufPool: sync.Pool{
			New: func() any {
				// 预分配 256 字节，减少后续扩容
				return bytes.NewBuffer(make([]byte, 0, 256))
			},
		},
	}
}

// Format 格式化日志（复用 buffer，减少 GC）
func (f *LogFormatter) Format(level, message string, fields map[string]string) string {
	buf := f.bufPool.Get().(*bytes.Buffer)
	defer func() {
		buf.Reset()
		f.bufPool.Put(buf)
	}()

	// 写入时间戳
	buf.WriteString(time.Now().Format("2006-01-02T15:04:05.000Z07:00"))
	buf.WriteString(" [")
	buf.WriteString(level)
	buf.WriteString("] ")
	buf.WriteString(message)

	// 写入字段
	for k, v := range fields {
		buf.WriteString(" ")
		buf.WriteString(k)
		buf.WriteString("=")
		buf.WriteString(v)
	}

	return buf.String()
}

func main() {
	formatter := NewLogFormatter()
	var wg sync.WaitGroup

	// 模拟高并发日志写入
	start := time.Now()
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			log := formatter.Format("INFO", "请求处理完成", map[string]string{
				"request_id": fmt.Sprintf("req-%d", id),
				"latency":    "15ms",
				"status":     "200",
			})
			_ = log // 实际中写入文件或发送到日志系统
		}(i)
	}

	wg.Wait()
	fmt.Printf("格式化 10000 条日志耗时: %v\n", time.Since(start))

	// 对比不使用 Pool 的版本
	start = time.Now()
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			buf := new(bytes.Buffer) // 每次分配新的 Buffer
			buf.WriteString(time.Now().Format("2006-01-02T15:04:05.000Z07:00"))
			buf.WriteString(fmt.Sprintf(" [INFO] 请求处理完成 request_id=req-%d", id))
			_ = buf.String()
		}(i)
	}
	wg.Wait()
	fmt.Printf("不使用 Pool 耗时: %v\n", time.Since(start))
}
```

---

## 6. sync.Map 并发安全 Map

### 6.1 基本用法

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map

	// 存储
	m.Store("name", "web-server")
	m.Store("port", 8080)
	m.Store("version", "1.2.3")

	// 读取
	if val, ok := m.Load("name"); ok {
		fmt.Println("name:", val)
	}

	// 读取或存储（如果不存在则存储）
	actual, loaded := m.LoadOrStore("name", "default")
	fmt.Printf("LoadOrStore: value=%v, loaded=%v\n", actual, loaded) // 已存在，返回旧值

	actual, loaded = m.LoadOrStore("region", "us-east-1")
	fmt.Printf("LoadOrStore: value=%v, loaded=%v\n", actual, loaded) // 不存在，存储并返回新值

	// 删除
	m.Delete("port")

	// 遍历
	fmt.Println("\n所有键值对:")
	m.Range(func(key, value any) bool {
		fmt.Printf("  %v: %v\n", key, value)
		return true // 返回 false 停止遍历
	})

	// Go 1.20+: LoadAndDelete
	if val, loaded := m.LoadAndDelete("version"); loaded {
		fmt.Printf("\n删除并返回: %v\n", val)
	}

	// Go 1.20+: Swap
	old, loaded := m.Swap("name", "api-server")
	fmt.Printf("Swap: old=%v, loaded=%v\n", old, loaded)

	// Go 1.20+: CompareAndSwap
	swapped := m.CompareAndSwap("name", "api-server", "gateway")
	fmt.Printf("CompareAndSwap: swapped=%v\n", swapped)

	// Go 1.20+: CompareAndDelete
	deleted := m.CompareAndDelete("name", "gateway")
	fmt.Printf("CompareAndDelete: deleted=%v\n", deleted)
}
```

### 6.2 sync.Map 适用场景

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// sync.Map 最适合的两种场景：
// 1. 键值对只写入一次但读取多次（缓存）
// 2. 多个 goroutine 读写不同的键（无竞争）

// 场景1: DNS 缓存（写少读多）
type DNSCache struct {
	cache sync.Map
}

type DNSEntry struct {
	IP        string
	ExpiresAt time.Time
}

func (c *DNSCache) Resolve(hostname string) string {
	// 尝试从缓存读取
	if entry, ok := c.cache.Load(hostname); ok {
		e := entry.(*DNSEntry)
		if time.Now().Before(e.ExpiresAt) {
			return e.IP // 缓存命中
		}
		// 已过期，删除
		c.cache.Delete(hostname)
	}

	// 缓存未命中，模拟 DNS 查询
	ip := fmt.Sprintf("10.0.%d.%d", len(hostname)%256, len(hostname)%128)
	c.cache.Store(hostname, &DNSEntry{
		IP:        ip,
		ExpiresAt: time.Now().Add(5 * time.Minute),
	})
	return ip
}

// 场景2: 连接池注册表（不同 goroutine 操作不同的键）
type ConnectionRegistry struct {
	connections sync.Map
}

func (r *ConnectionRegistry) Register(service string, conn string) {
	r.connections.Store(service, conn)
}

func (r *ConnectionRegistry) Get(service string) (string, bool) {
	val, ok := r.connections.Load(service)
	if !ok {
		return "", false
	}
	return val.(string), true
}

func (r *ConnectionRegistry) Unregister(service string) {
	r.connections.Delete(service)
}

func main() {
	// DNS 缓存示例
	dns := &DNSCache{}
	var wg sync.WaitGroup

	hosts := []string{"api.example.com", "db.example.com", "cache.example.com"}
	for _, host := range hosts {
		for i := 0; i < 100; i++ {
			wg.Add(1)
			go func(h string) {
				defer wg.Done()
				ip := dns.Resolve(h)
				_ = ip
			}(host)
		}
	}
	wg.Wait()

	fmt.Println("DNS 缓存内容:")
	dns.cache.Range(func(key, value any) bool {
		entry := value.(*DNSEntry)
		fmt.Printf("  %s -> %s\n", key, entry.IP)
		return true
	})

	// 什么时候不用 sync.Map？
	fmt.Println("\n=== sync.Map vs map+Mutex 选择指南 ===")
	fmt.Println("sync.Map 适用:")
	fmt.Println("  - 键值对一次写入多次读取")
	fmt.Println("  - 多个 goroutine 操作不相交的键集合")
	fmt.Println()
	fmt.Println("map+Mutex 适用:")
	fmt.Println("  - 需要对 map 做复合操作（如 check-then-set）")
	fmt.Println("  - 需要遍历一致性快照")
	fmt.Println("  - 键数量已知且固定")
}
```

---

## 7. sync.Cond 条件变量

### 7.1 基本用法

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var mu sync.Mutex
	cond := sync.NewCond(&mu)

	ready := false

	// 等待者 goroutine
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			cond.L.Lock()
			for !ready { // 必须用 for 循环检查条件，而不是 if
				fmt.Printf("Worker %d: 等待信号...\n", id)
				cond.Wait() // 释放锁并等待信号，被唤醒后重新获取锁
			}
			fmt.Printf("Worker %d: 收到信号，开始工作！\n", id)
			cond.L.Unlock()
		}(i)
	}

	// 给等待者一些时间开始等待
	time.Sleep(100 * time.Millisecond)

	// 发送信号
	fmt.Println("\n设置 ready=true 并广播")
	cond.L.Lock()
	ready = true
	cond.L.Unlock()
	cond.Broadcast() // 唤醒所有等待者（Signal 只唤醒一个）

	wg.Wait()
	fmt.Println("全部完成")
}
```

### 7.2 SRE 场景：配置热加载通知

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ConfigWatcher 配置热加载监控器
type ConfigWatcher struct {
	mu      sync.RWMutex
	cond    *sync.Cond
	config  map[string]string
	version int
}

func NewConfigWatcher(initial map[string]string) *ConfigWatcher {
	cw := &ConfigWatcher{
		config:  initial,
		version: 1,
	}
	cw.cond = sync.NewCond(cw.mu.RLocker()) // 使用读锁作为 Cond 的锁
	return cw
}

// Update 更新配置并通知所有监听者
func (cw *ConfigWatcher) Update(newConfig map[string]string) {
	cw.mu.Lock()
	cw.config = newConfig
	cw.version++
	v := cw.version
	cw.mu.Unlock()

	fmt.Printf("[ConfigWatcher] 配置更新到版本 %d\n", v)
	cw.cond.Broadcast() // 通知所有等待者
}

// WaitForUpdate 等待配置更新（阻塞直到版本变化）
func (cw *ConfigWatcher) WaitForUpdate(currentVersion int) (map[string]string, int) {
	cw.mu.RLock()
	defer cw.mu.RUnlock()

	for cw.version == currentVersion {
		cw.cond.Wait() // 等待更新通知
	}

	// 返回配置副本
	result := make(map[string]string, len(cw.config))
	for k, v := range cw.config {
		result[k] = v
	}
	return result, cw.version
}

// GetConfig 获取当前配置
func (cw *ConfigWatcher) GetConfig() (map[string]string, int) {
	cw.mu.RLock()
	defer cw.mu.RUnlock()
	result := make(map[string]string, len(cw.config))
	for k, v := range cw.config {
		result[k] = v
	}
	return result, cw.version
}

func main() {
	watcher := NewConfigWatcher(map[string]string{
		"log_level": "info",
		"port":      "8080",
	})

	var wg sync.WaitGroup

	// 启动多个配置订阅者
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			_, version := watcher.GetConfig()
			for j := 0; j < 2; j++ { // 每个订阅者等待 2 次更新
				newConfig, newVersion := watcher.WaitForUpdate(version)
				fmt.Printf("[Subscriber %d] 配置更新: v%d -> v%d, config=%v\n",
					id, version, newVersion, newConfig)
				version = newVersion
			}
		}(i)
	}

	// 模拟配置更新
	time.Sleep(200 * time.Millisecond)
	watcher.Update(map[string]string{
		"log_level": "debug",
		"port":      "8080",
	})

	time.Sleep(200 * time.Millisecond)
	watcher.Update(map[string]string{
		"log_level": "warn",
		"port":      "9090",
	})

	wg.Wait()
}
```

---

## 8. atomic 包原子操作

### 8.1 基本类型原子操作（Go 1.19+）

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	// Go 1.19+ 引入了类型化的原子类型
	fmt.Println("=== atomic 类型化 API (Go 1.19+) ===")

	// atomic.Int64
	var counter atomic.Int64
	counter.Store(0) // 设置值

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Add(1) // 原子加 1
		}()
	}
	wg.Wait()
	fmt.Printf("atomic.Int64 计数器: %d（期望 1000）\n", counter.Load())

	// atomic.Bool
	var isRunning atomic.Bool
	isRunning.Store(true)
	fmt.Printf("isRunning: %v\n", isRunning.Load())

	// CompareAndSwap (CAS)
	swapped := isRunning.CompareAndSwap(true, false) // 如果是 true 就改为 false
	fmt.Printf("CAS 结果: swapped=%v, value=%v\n", swapped, isRunning.Load())

	// Swap
	old := isRunning.Swap(true)
	fmt.Printf("Swap: old=%v, new=%v\n", old, isRunning.Load())

	// atomic.Int32, atomic.Uint32, atomic.Uint64, atomic.Uintptr 用法类似

	// atomic.Pointer[T] (Go 1.19+)
	type Config struct {
		LogLevel string
		Port     int
	}

	var configPtr atomic.Pointer[Config]
	configPtr.Store(&Config{LogLevel: "info", Port: 8080})

	// 原子地读取配置
	cfg := configPtr.Load()
	fmt.Printf("Config: %+v\n", cfg)

	// 原子地更新配置
	newCfg := &Config{LogLevel: "debug", Port: 9090}
	configPtr.Store(newCfg)
	fmt.Printf("Updated Config: %+v\n", configPtr.Load())
}
```

### 8.2 原子操作 vs 锁的性能对比

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

const iterations = 1_000_000

func benchAtomic() time.Duration {
	var counter atomic.Int64
	var wg sync.WaitGroup

	start := time.Now()
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations; j++ {
				counter.Add(1)
			}
		}()
	}
	wg.Wait()
	return time.Since(start)
}

func benchMutex() time.Duration {
	var counter int64
	var mu sync.Mutex
	var wg sync.WaitGroup

	start := time.Now()
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < iterations; j++ {
				mu.Lock()
				counter++
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	return time.Since(start)
}

func main() {
	fmt.Printf("每个 goroutine 执行 %d 次计数操作（4 个 goroutine）\n\n", iterations)

	atomicTime := benchAtomic()
	fmt.Printf("atomic.Int64: %v\n", atomicTime)

	mutexTime := benchMutex()
	fmt.Printf("sync.Mutex:   %v\n", mutexTime)

	fmt.Printf("\natomic 比 Mutex 快 %.1fx\n", float64(mutexTime)/float64(atomicTime))
	fmt.Println("\n结论: 简单的计数器/标志位用 atomic，复杂的临界区用 Mutex")
}
```

### 8.3 SRE 场景：无锁指标收集

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// Metrics 无锁的指标收集器
type Metrics struct {
	RequestCount  atomic.Int64
	ErrorCount    atomic.Int64
	TotalLatency  atomic.Int64  // 纳秒
	ActiveConns   atomic.Int64
	IsHealthy     atomic.Bool
}

func NewMetrics() *Metrics {
	m := &Metrics{}
	m.IsHealthy.Store(true)
	return m
}

// RecordRequest 记录一次请求
func (m *Metrics) RecordRequest(latency time.Duration, isError bool) {
	m.RequestCount.Add(1)
	m.TotalLatency.Add(int64(latency))
	if isError {
		m.ErrorCount.Add(1)
	}
}

// ConnOpen 连接打开
func (m *Metrics) ConnOpen() {
	m.ActiveConns.Add(1)
}

// ConnClose 连接关闭
func (m *Metrics) ConnClose() {
	m.ActiveConns.Add(-1)
}

// Snapshot 获取指标快照
func (m *Metrics) Snapshot() map[string]interface{} {
	reqCount := m.RequestCount.Load()
	avgLatency := int64(0)
	if reqCount > 0 {
		avgLatency = m.TotalLatency.Load() / reqCount
	}

	return map[string]interface{}{
		"request_count":   reqCount,
		"error_count":     m.ErrorCount.Load(),
		"error_rate":      float64(m.ErrorCount.Load()) / float64(max(reqCount, 1)) * 100,
		"avg_latency_ms":  float64(avgLatency) / float64(time.Millisecond),
		"active_conns":    m.ActiveConns.Load(),
		"is_healthy":      m.IsHealthy.Load(),
	}
}

func max(a, b int64) int64 {
	if a > b {
		return a
	}
	return b
}

func main() {
	metrics := NewMetrics()
	var wg sync.WaitGroup

	// 模拟并发请求
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			metrics.ConnOpen()
			defer metrics.ConnClose()

			for j := 0; j < 100; j++ {
				latency := time.Duration(rand.Intn(100)) * time.Millisecond
				isError := rand.Float64() < 0.05 // 5% 错误率
				time.Sleep(time.Duration(rand.Intn(5)) * time.Millisecond)
				metrics.RecordRequest(latency, isError)
			}
		}()
	}

	// 每 500ms 打印一次指标
	done := make(chan struct{})
	go func() {
		ticker := time.NewTicker(500 * time.Millisecond)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				snap := metrics.Snapshot()
				fmt.Printf("指标: 请求=%d, 错误=%d(%.1f%%), 平均延迟=%.1fms, 活跃连接=%d\n",
					snap["request_count"],
					snap["error_count"],
					snap["error_rate"],
					snap["avg_latency_ms"],
					snap["active_conns"],
				)
			case <-done:
				return
			}
		}
	}()

	wg.Wait()
	close(done)

	// 最终指标
	fmt.Println("\n=== 最终指标 ===")
	for k, v := range metrics.Snapshot() {
		fmt.Printf("  %s: %v\n", k, v)
	}
}
```

---

## 9. 锁的粒度与性能

### 9.1 粗粒度 vs 细粒度锁

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// 粗粒度锁：整个结构体一把锁
type CoarseGrainedCache struct {
	mu   sync.RWMutex
	data map[string]string
}

func (c *CoarseGrainedCache) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	v, ok := c.data[key]
	return v, ok
}

func (c *CoarseGrainedCache) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = value
}

// 细粒度锁：分片（Sharding），每个分片一把锁
const numShards = 32

type ShardedCache struct {
	shards [numShards]struct {
		mu   sync.RWMutex
		data map[string]string
	}
}

func NewShardedCache() *ShardedCache {
	c := &ShardedCache{}
	for i := range c.shards {
		c.shards[i].data = make(map[string]string)
	}
	return c
}

func (c *ShardedCache) getShard(key string) int {
	// 简单的哈希分片
	hash := 0
	for _, ch := range key {
		hash = hash*31 + int(ch)
	}
	if hash < 0 {
		hash = -hash
	}
	return hash % numShards
}

func (c *ShardedCache) Get(key string) (string, bool) {
	shard := c.getShard(key)
	c.shards[shard].mu.RLock()
	defer c.shards[shard].mu.RUnlock()
	v, ok := c.shards[shard].data[key]
	return v, ok
}

func (c *ShardedCache) Set(key, value string) {
	shard := c.getShard(key)
	c.shards[shard].mu.Lock()
	defer c.shards[shard].mu.Unlock()
	c.shards[shard].data[key] = value
}

func benchmark(name string, getter func(string) (string, bool), setter func(string, string)) time.Duration {
	var wg sync.WaitGroup
	start := time.Now()

	// 预填充数据
	for i := 0; i < 1000; i++ {
		setter(fmt.Sprintf("key-%d", i), fmt.Sprintf("value-%d", i))
	}

	// 并发读写
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 10000; j++ {
				key := fmt.Sprintf("key-%d", j%1000)
				if j%10 == 0 {
					setter(key, fmt.Sprintf("new-value-%d", j))
				} else {
					getter(key)
				}
			}
		}(i)
	}

	wg.Wait()
	elapsed := time.Since(start)
	fmt.Printf("%s: %v\n", name, elapsed)
	return elapsed
}

func main() {
	fmt.Println("=== 锁粒度性能对比 ===")
	fmt.Println("100 个 goroutine，每个 10000 次操作（90% 读 + 10% 写）")
	fmt.Println()

	// 粗粒度锁
	coarse := &CoarseGrainedCache{data: make(map[string]string)}
	coarseTime := benchmark("粗粒度锁", coarse.Get, coarse.Set)

	// 细粒度锁（分片）
	sharded := NewShardedCache()
	shardedTime := benchmark("细粒度锁(32分片)", sharded.Get, sharded.Set)

	if shardedTime < coarseTime {
		fmt.Printf("\n细粒度锁快 %.1fx\n", float64(coarseTime)/float64(shardedTime))
	}
}
```

### 9.2 锁的使用建议

```go
package main

import "fmt"

func main() {
	fmt.Println("=== 锁的使用建议 ===")
	fmt.Println()

	tips := []struct {
		title string
		desc  string
	}{
		{"最小化临界区", "锁保护的代码越少越好，不要在锁内做 I/O 或计算密集操作"},
		{"优先用 RWMutex", "读多写少场景用 RWMutex，允许并发读"},
		{"避免锁嵌套", "嵌套加锁容易死锁，如果必须嵌套，确保加锁顺序一致"},
		{"用 defer Unlock", "防止 panic 导致的死锁，但注意 defer 有微小开销"},
		{"简单计数用 atomic", "计数器、标志位等简单操作用 atomic，性能远优于锁"},
		{"考虑分片", "高竞争场景下，将数据分片减少锁竞争"},
		{"避免复制锁", "sync.Mutex/RWMutex 不能复制，传递时用指针"},
		{"用 go vet 检查", "go vet 可以检测锁的复制等问题"},
	}

	for i, tip := range tips {
		fmt.Printf("%d. 【%s】\n   %s\n\n", i+1, tip.title, tip.desc)
	}
}
```

---

## 10. 常见坑点

### 10.1 锁复制问题

```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu    sync.Mutex
	count int
}

func (c SafeCounter) GetCount() int {
	// 错误！值接收者会复制 SafeCounter，包括复制了 Mutex
	// 如果调用时 c 的 Mutex 处于锁定状态，复制出来的也是锁定的
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}

func (c *SafeCounter) GetCountCorrect() int {
	// 正确！使用指针接收者，不会复制 Mutex
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}

func main() {
	counter := &SafeCounter{}
	// 使用 go vet 可以检测到值接收者复制 Mutex 的问题
	// go vet ./...
	fmt.Println("提示: 包含 sync.Mutex 的结构体应该用指针传递")
	fmt.Println("提示: 方法应该用指针接收者")
	fmt.Println("运行 go vet 检查: go vet ./...")
	_ = counter
}
```

### 10.2 WaitGroup 的坑

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 坑1: 传值导致 WaitGroup 复制
	var wg sync.WaitGroup
	wg.Add(1)

	// 错误：传值会复制 WaitGroup
	// go func(w sync.WaitGroup) {
	//     defer w.Done() // 操作的是副本，原始的 wg 永远不会 Done
	// }(wg)
	// wg.Wait() // 永远阻塞（死锁）

	// 正确：传指针
	go func(w *sync.WaitGroup) {
		defer w.Done()
		fmt.Println("使用指针传递 WaitGroup")
	}(&wg)
	wg.Wait()

	// 坑2: Add 的数值必须与 Done 的调用次数匹配
	// wg.Add(2)
	// wg.Done() // 只 Done 一次
	// wg.Wait() // 永远阻塞

	// 坑3: Done 过多导致 panic
	// wg.Add(1)
	// wg.Done()
	// wg.Done() // panic: sync: negative WaitGroup counter

	fmt.Println("WaitGroup 注意事项:")
	fmt.Println("  1. 传递 WaitGroup 必须用指针")
	fmt.Println("  2. Add 和 Done 次数必须匹配")
	fmt.Println("  3. Add 在 go 之前调用")
}
```

### 10.3 sync.Pool 的坑

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 坑1: Pool 中的对象可能被 GC 回收
	pool := &sync.Pool{
		New: func() any { return "new" },
	}
	pool.Put("cached")
	// GC 后，cached 可能被回收
	// runtime.GC()
	// pool.Get() // 可能返回 "new" 而非 "cached"

	// 坑2: 放回 Pool 的对象必须重置状态
	type Buffer struct {
		data []byte
	}
	bufPool := &sync.Pool{
		New: func() any { return &Buffer{data: make([]byte, 0, 1024)} },
	}

	buf := bufPool.Get().(*Buffer)
	buf.data = append(buf.data, []byte("sensitive data")...)

	// 错误：没有清理就放回
	// bufPool.Put(buf) // 下次 Get 可能获取到残留数据

	// 正确：清理后放回
	buf.data = buf.data[:0] // 重置但保留底层数组
	bufPool.Put(buf)

	next := bufPool.Get().(*Buffer)
	fmt.Printf("重用的 Buffer 长度: %d (应该为 0)\n", len(next.data))
	fmt.Printf("重用的 Buffer 容量: %d (保留了底层数组)\n", cap(next.data))

	// 坑3: 不要在 Pool 中存储带有外部引用的对象
	fmt.Println("\nPool 使用建议:")
	fmt.Println("  1. 不要依赖 Pool 中对象的持久性")
	fmt.Println("  2. 放回前必须重置对象状态")
	fmt.Println("  3. 适合短生命周期、频繁分配的对象")
	fmt.Println("  4. 不适合连接池等需要持久化的场景")
}
```

---

## 11. 速查表 / 小结

### sync 包速查表

```
sync.Mutex（互斥锁）:
  var mu sync.Mutex
  mu.Lock()                    // 加锁
  mu.Unlock()                  // 解锁
  // 不可重入，不可复制

sync.RWMutex（读写锁）:
  var rw sync.RWMutex
  rw.RLock() / rw.RUnlock()    // 读锁（可多个并发）
  rw.Lock()  / rw.Unlock()     // 写锁（排他）

sync.WaitGroup（等待组）:
  var wg sync.WaitGroup
  wg.Add(n)                    // 增加计数
  wg.Done()                    // 计数 -1（等于 Add(-1)）
  wg.Wait()                    // 阻塞直到计数为 0

sync.Once（只执行一次）:
  var once sync.Once
  once.Do(func() { ... })      // 只执行一次
  sync.OnceFunc(f)             // Go 1.21+ 包装函数
  sync.OnceValue(f)            // Go 1.21+ 缓存返回值
  sync.OnceValues(f)           // Go 1.21+ 缓存两个返回值

sync.Pool（对象池）:
  pool := &sync.Pool{New: func() any { ... }}
  obj := pool.Get()            // 获取（可能调用 New）
  pool.Put(obj)                // 放回（记得重置状态）

sync.Map（并发 Map）:
  var m sync.Map
  m.Store(key, value)          // 存储
  m.Load(key)                  // 读取
  m.Delete(key)                // 删除
  m.Range(func(k, v any) bool) // 遍历
  m.LoadOrStore(key, value)    // 读取或存储
  m.CompareAndSwap(k, old, new)// CAS

sync.Cond（条件变量）:
  cond := sync.NewCond(&mu)
  cond.Wait()                  // 等待信号（必须在锁内）
  cond.Signal()                // 唤醒一个等待者
  cond.Broadcast()             // 唤醒所有等待者

atomic（原子操作，Go 1.19+）:
  var n atomic.Int64
  n.Store(v)                   // 设置
  n.Load()                     // 读取
  n.Add(delta)                 // 原子加
  n.Swap(new)                  // 交换
  n.CompareAndSwap(old, new)   // CAS

  var p atomic.Pointer[T]
  p.Store(ptr) / p.Load()
```

### 选择指南

| 场景 | 推荐工具 |
|------|----------|
| 保护共享变量 | sync.Mutex |
| 读多写少 | sync.RWMutex |
| 等待多个 goroutine | sync.WaitGroup |
| 单例/一次性初始化 | sync.Once |
| 频繁分配临时对象 | sync.Pool |
| 并发安全 Map（写少读多）| sync.Map |
| 简单计数器/标志位 | atomic.Int64 / atomic.Bool |
| 等待条件满足 | sync.Cond |
| 配置热加载 | atomic.Pointer[T] |

### 性能排序（从快到慢）

```
atomic > sync.Mutex > sync.RWMutex > channel

适用范围（从窄到宽）：
atomic（简单值）< Mutex（临界区）< channel（通信+同步）
```

---

> **下一篇**：[04-并发模式.md](./04-并发模式.md) - 学习 Go 中常用的并发设计模式
