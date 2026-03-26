# Go pprof 性能分析完全指南

## 导读

在 SRE/DevOps 日常工作中，性能问题是最常见也是最棘手的挑战之一。Go 语言内置了强大的性能分析工具链 —— pprof，它能帮助我们精准定位 CPU 热点、内存泄漏、Goroutine 泄漏等问题。

本篇将系统讲解 pprof 的所有核心功能，从本地程序分析到线上服务实时诊断，覆盖 SRE 日常遇到的各类性能场景。

**本篇核心知识点：**
- `runtime/pprof` 包：离线采集 CPU/Memory/Goroutine/Block Profile
- `net/http/pprof` 包：通过 HTTP 暴露在线分析端点
- `go tool pprof` 交互式命令：top/list/web/peek
- 火焰图生成与分析方法
- `go tool trace` 执行追踪器
- 实际性能问题排查的完整流程
- SRE 实战：线上服务性能分析案例

---

## 一、pprof 简介与原理

### 1.1 什么是 pprof

pprof 是 Go 内置的性能分析工具，名字来源于 "program profiling"。它通过采样（sampling）的方式收集程序运行时的数据，开销极低，适合在生产环境使用。

### 1.2 pprof 的采样原理

```go
// pprof_intro.go
// 演示 pprof 的基本采样原理
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	// Go 的 CPU profiler 默认每秒采样 100 次（每 10ms 一次）
	// 采样时记录当前所有 goroutine 的调用栈
	// 通过统计每个函数出现在栈顶的次数，推断 CPU 时间分布

	// 内存 profiler 则是在每分配 512KB 时采样一次（可通过 runtime.MemProfileRate 调整）
	fmt.Printf("当前内存采样率: %d bytes\n", runtime.MemProfileRate)

	// 可以调整采样率（设为 1 表示每次分配都记录，仅用于调试）
	// runtime.MemProfileRate = 1

	// goroutine profile 不需要采样，它直接获取所有 goroutine 的栈信息
	fmt.Printf("当前 Goroutine 数量: %d\n", runtime.NumGoroutine())

	// 演示 GC 统计信息
	var stats runtime.MemStats
	runtime.ReadMemStats(&stats)
	fmt.Printf("堆内存分配: %d MB\n", stats.HeapAlloc/1024/1024)
	fmt.Printf("GC 次数: %d\n", stats.NumGC)
	fmt.Printf("上次 GC 暂停时间: %v\n", time.Duration(stats.PauseNs[(stats.NumGC+255)%256]))
}
```

### 1.3 Profile 类型总览

| Profile 类型 | 说明 | 采样方式 | 适用场景 |
|---|---|---|---|
| CPU Profile | CPU 使用分布 | 定时采样（10ms） | 定位 CPU 热点函数 |
| Heap Profile | 堆内存分配 | 按分配量采样 | 内存泄漏、分配热点 |
| Goroutine Profile | 所有 goroutine 栈 | 全量快照 | goroutine 泄漏 |
| Block Profile | 阻塞操作统计 | 全量记录 | 锁竞争、channel 阻塞 |
| Mutex Profile | 互斥锁竞争 | 采样 | 锁竞争分析 |
| Threadcreate Profile | 线程创建 | 全量记录 | 线程泄漏 |

---

## 二、runtime/pprof — 离线性能分析

### 2.1 CPU Profile

```go
// cpu_profile.go
// 演示如何使用 runtime/pprof 进行 CPU 性能分析
package main

import (
	"fmt"
	"math"
	"os"
	"runtime/pprof"
	"sync"
	"time"
)

// cpuIntensiveTask 模拟一个 CPU 密集型任务
func cpuIntensiveTask(iterations int) float64 {
	result := 0.0
	for i := 0; i < iterations; i++ {
		// 大量浮点运算
		result += math.Sqrt(float64(i)) * math.Sin(float64(i))
	}
	return result
}

// stringProcessing 模拟字符串处理（常见的 CPU 热点）
func stringProcessing(n int) string {
	result := ""
	for i := 0; i < n; i++ {
		// 字符串拼接是常见的性能问题
		result += fmt.Sprintf("item-%d,", i)
	}
	return result
}

// concurrentWork 模拟并发计算任务
func concurrentWork(workers int) {
	var wg sync.WaitGroup
	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cpuIntensiveTask(1000000)
		}(i)
	}
	wg.Wait()
}

func main() {
	// 创建 CPU profile 文件
	f, err := os.Create("cpu.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建 CPU profile 文件失败: %v\n", err)
		os.Exit(1)
	}
	defer f.Close()

	// 开始 CPU 采样
	if err := pprof.StartCPUProfile(f); err != nil {
		fmt.Fprintf(os.Stderr, "启动 CPU profile 失败: %v\n", err)
		os.Exit(1)
	}
	defer pprof.StopCPUProfile()

	fmt.Println("开始执行性能测试任务...")
	start := time.Now()

	// 执行各种任务，让 profiler 采集数据
	// 任务 1：CPU 密集型计算
	fmt.Println("执行 CPU 密集型计算...")
	result := cpuIntensiveTask(10000000)
	fmt.Printf("计算结果: %.2f\n", result)

	// 任务 2：字符串处理
	fmt.Println("执行字符串处理...")
	s := stringProcessing(10000)
	fmt.Printf("字符串长度: %d\n", len(s))

	// 任务 3：并发计算
	fmt.Println("执行并发计算...")
	concurrentWork(4)

	fmt.Printf("全部任务完成，耗时: %v\n", time.Since(start))
	fmt.Println("CPU profile 已保存到 cpu.prof")
	fmt.Println("使用命令分析: go tool pprof cpu.prof")
}
```

### 2.2 Memory Profile（堆内存分析）

```go
// memory_profile.go
// 演示堆内存 Profile 的采集与分析
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/pprof"
)

// allocateSlices 模拟大量切片分配
func allocateSlices() [][]byte {
	// 这个函数会产生大量堆分配
	var slices [][]byte
	for i := 0; i < 1000; i++ {
		// 每次分配 1KB 的切片
		s := make([]byte, 1024)
		for j := range s {
			s[j] = byte(j % 256)
		}
		slices = append(slices, s)
	}
	return slices
}

// allocateStrings 模拟字符串分配
func allocateStrings() []string {
	var strings []string
	for i := 0; i < 5000; i++ {
		// 字符串是不可变的，每次修改都会分配新内存
		s := fmt.Sprintf("这是第 %d 个字符串，包含一些额外的内容用于增加分配量", i)
		strings = append(strings, s)
	}
	return strings
}

// allocateMaps 模拟 Map 分配
func allocateMaps() map[string][]byte {
	m := make(map[string][]byte)
	for i := 0; i < 1000; i++ {
		key := fmt.Sprintf("key-%d", i)
		value := make([]byte, 2048)
		m[key] = value
	}
	return m
}

// leakyBuffer 模拟一个潜在的内存泄漏场景
type leakyBuffer struct {
	data []byte
}

var globalBuffers []*leakyBuffer // 全局变量持有引用，导致无法被 GC

func createLeakyBuffers(count int) {
	for i := 0; i < count; i++ {
		buf := &leakyBuffer{
			data: make([]byte, 4096), // 每个 4KB
		}
		globalBuffers = append(globalBuffers, buf)
	}
}

func main() {
	// 设置内存采样率（1 = 每次分配都采样，仅调试使用）
	// 默认是 512KB，即每分配 512KB 采样一次
	runtime.MemProfileRate = 1

	fmt.Println("开始内存分配测试...")

	// 执行各种内存分配操作
	slices := allocateSlices()
	strings := allocateStrings()
	maps := allocateMaps()
	createLeakyBuffers(500)

	// 避免编译器优化掉变量
	fmt.Printf("切片数量: %d\n", len(slices))
	fmt.Printf("字符串数量: %d\n", len(strings))
	fmt.Printf("Map 条目数: %d\n", len(maps))
	fmt.Printf("全局缓冲区数量: %d\n", len(globalBuffers))

	// 触发 GC，确保内存统计准确
	runtime.GC()

	// 写入堆内存 profile
	heapFile, err := os.Create("heap.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建 heap profile 失败: %v\n", err)
		os.Exit(1)
	}
	defer heapFile.Close()

	// 写入堆 profile
	// pprof.WriteHeapProfile 等价于 pprof.Lookup("heap").WriteTo(f, 0)
	if err := pprof.WriteHeapProfile(heapFile); err != nil {
		fmt.Fprintf(os.Stderr, "写入 heap profile 失败: %v\n", err)
		os.Exit(1)
	}

	// 也可以写入 allocs profile（记录所有分配，而非仅存活对象）
	allocsFile, err := os.Create("allocs.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建 allocs profile 失败: %v\n", err)
		os.Exit(1)
	}
	defer allocsFile.Close()

	if err := pprof.Lookup("allocs").WriteTo(allocsFile, 0); err != nil {
		fmt.Fprintf(os.Stderr, "写入 allocs profile 失败: %v\n", err)
		os.Exit(1)
	}

	// 打印内存统计
	var memStats runtime.MemStats
	runtime.ReadMemStats(&memStats)
	fmt.Println("\n--- 内存统计 ---")
	fmt.Printf("堆内存使用: %.2f MB\n", float64(memStats.HeapAlloc)/1024/1024)
	fmt.Printf("堆内存系统申请: %.2f MB\n", float64(memStats.HeapSys)/1024/1024)
	fmt.Printf("堆对象数量: %d\n", memStats.HeapObjects)
	fmt.Printf("累计分配: %.2f MB\n", float64(memStats.TotalAlloc)/1024/1024)
	fmt.Printf("GC 次数: %d\n", memStats.NumGC)

	fmt.Println("\nProfile 文件已生成:")
	fmt.Println("  heap.prof   - 当前堆内存使用（inuse_space）")
	fmt.Println("  allocs.prof - 累计内存分配（alloc_space）")
	fmt.Println("\n分析命令:")
	fmt.Println("  go tool pprof heap.prof")
	fmt.Println("  go tool pprof -inuse_space heap.prof   # 当前使用中的内存")
	fmt.Println("  go tool pprof -alloc_space allocs.prof # 累计分配的内存")
}
```

### 2.3 Goroutine Profile

```go
// goroutine_profile.go
// 演示 Goroutine Profile 的采集，用于排查 goroutine 泄漏
package main

import (
	"fmt"
	"net/http"
	"os"
	"runtime"
	"runtime/pprof"
	"sync"
	"time"
)

// leakyGoroutine 模拟 goroutine 泄漏
// 常见原因：channel 没有关闭、context 没有取消
func leakyGoroutine(ch chan struct{}) {
	// 这个 goroutine 会一直阻塞在 channel 接收上
	// 如果没有人向 channel 发送数据或关闭 channel，这个 goroutine 就泄漏了
	<-ch
	fmt.Println("这行永远不会执行")
}

// leakyHTTPHandler 模拟 HTTP 请求导致的 goroutine 泄漏
func leakyHTTPHandler() {
	for i := 0; i < 10; i++ {
		go func(id int) {
			// 模拟一个永远不会返回的 HTTP 请求
			client := &http.Client{
				// 没有设置超时！这是常见的 goroutine 泄漏原因
				// Timeout: 10 * time.Second, // 应该设置超时
			}
			// 这里只是模拟，不会真正发起请求
			_ = client
			select {} // 永远阻塞
		}(i)
	}
}

// blockedOnMutex 模拟互斥锁导致的 goroutine 阻塞
func blockedOnMutex() {
	var mu sync.Mutex
	mu.Lock() // 主 goroutine 持有锁

	for i := 0; i < 5; i++ {
		go func(id int) {
			mu.Lock() // 这些 goroutine 会一直阻塞
			defer mu.Unlock()
			fmt.Printf("goroutine %d 获得了锁\n", id)
		}(i)
	}

	// 故意不释放锁，制造阻塞
	time.Sleep(100 * time.Millisecond)
}

// blockedOnChannel 模拟 channel 阻塞
func blockedOnChannel() {
	ch := make(chan int) // 无缓冲 channel

	for i := 0; i < 5; i++ {
		go func(id int) {
			ch <- id // 没有接收者，发送会阻塞
		}(i)
	}

	time.Sleep(100 * time.Millisecond)
}

func main() {
	fmt.Printf("初始 Goroutine 数量: %d\n", runtime.NumGoroutine())

	// 制造各种 goroutine 泄漏场景
	fmt.Println("\n制造 goroutine 泄漏场景...")

	// 场景 1：channel 泄漏
	for i := 0; i < 20; i++ {
		ch := make(chan struct{})
		go leakyGoroutine(ch)
		// 没有关闭 ch，goroutine 泄漏
	}

	// 场景 2：互斥锁阻塞
	blockedOnMutex()

	// 场景 3：channel 发送阻塞
	blockedOnChannel()

	time.Sleep(200 * time.Millisecond)
	fmt.Printf("当前 Goroutine 数量: %d\n", runtime.NumGoroutine())

	// 写入 goroutine profile
	goroutineFile, err := os.Create("goroutine.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建文件失败: %v\n", err)
		os.Exit(1)
	}
	defer goroutineFile.Close()

	// debug=0 输出 protobuf 格式（用于 go tool pprof）
	if err := pprof.Lookup("goroutine").WriteTo(goroutineFile, 0); err != nil {
		fmt.Fprintf(os.Stderr, "写入 goroutine profile 失败: %v\n", err)
		os.Exit(1)
	}

	// 同时输出可读的文本格式（debug=2）
	goroutineTextFile, err := os.Create("goroutine_debug.txt")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建文件失败: %v\n", err)
		os.Exit(1)
	}
	defer goroutineTextFile.Close()

	// debug=2 输出人类可读的详细栈信息
	if err := pprof.Lookup("goroutine").WriteTo(goroutineTextFile, 2); err != nil {
		fmt.Fprintf(os.Stderr, "写入 goroutine debug 失败: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("\nProfile 文件已生成:")
	fmt.Println("  goroutine.prof      - protobuf 格式（用于 pprof 工具）")
	fmt.Println("  goroutine_debug.txt - 可读文本格式（直接查看）")
	fmt.Println("\n分析命令:")
	fmt.Println("  go tool pprof goroutine.prof")
	fmt.Println("  cat goroutine_debug.txt | head -100")
}
```

### 2.4 Block Profile

```go
// block_profile.go
// 演示 Block Profile 的采集，用于分析阻塞操作
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/pprof"
	"sync"
	"time"
)

func init() {
	// 必须开启 Block Profiling，参数为采样率（纳秒）
	// 设为 1 表示记录所有阻塞事件
	runtime.SetBlockProfileRate(1)

	// 同时开启 Mutex Profiling
	// 参数为采样比例，1/n 的锁竞争事件会被记录
	runtime.SetMutexProfileFraction(1)
}

// mutexContention 模拟互斥锁竞争
func mutexContention() {
	var mu sync.Mutex
	var wg sync.WaitGroup
	counter := 0

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				mu.Lock()
				counter++
				// 模拟持锁时的一些操作
				time.Sleep(time.Microsecond)
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Printf("计数器最终值: %d\n", counter)
}

// channelBlocking 模拟 channel 阻塞
func channelBlocking() {
	ch := make(chan int, 10)
	var wg sync.WaitGroup

	// 生产者（快）
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 100; i++ {
			ch <- i // 缓冲区满时会阻塞
		}
		close(ch)
	}()

	// 消费者（慢）
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range ch {
			_ = v
			time.Sleep(time.Millisecond) // 模拟慢消费
		}
	}()

	wg.Wait()
}

// condWaiting 模拟条件变量等待
func condWaiting() {
	var mu sync.Mutex
	cond := sync.NewCond(&mu)
	ready := false
	var wg sync.WaitGroup

	// 等待者
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			mu.Lock()
			for !ready {
				cond.Wait() // 在此阻塞
			}
			mu.Unlock()
		}(i)
	}

	time.Sleep(100 * time.Millisecond) // 让等待者先运行

	// 通知
	mu.Lock()
	ready = true
	mu.Unlock()
	cond.Broadcast()

	wg.Wait()
}

func main() {
	fmt.Println("执行阻塞分析测试...")

	fmt.Println("1. 互斥锁竞争测试...")
	mutexContention()

	fmt.Println("2. Channel 阻塞测试...")
	channelBlocking()

	fmt.Println("3. 条件变量等待测试...")
	condWaiting()

	// 写入 Block Profile
	blockFile, err := os.Create("block.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建文件失败: %v\n", err)
		os.Exit(1)
	}
	defer blockFile.Close()

	if err := pprof.Lookup("block").WriteTo(blockFile, 0); err != nil {
		fmt.Fprintf(os.Stderr, "写入 block profile 失败: %v\n", err)
		os.Exit(1)
	}

	// 写入 Mutex Profile
	mutexFile, err := os.Create("mutex.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建文件失败: %v\n", err)
		os.Exit(1)
	}
	defer mutexFile.Close()

	if err := pprof.Lookup("mutex").WriteTo(mutexFile, 0); err != nil {
		fmt.Fprintf(os.Stderr, "写入 mutex profile 失败: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("\nProfile 文件已生成:")
	fmt.Println("  block.prof - 阻塞事件分析")
	fmt.Println("  mutex.prof - 互斥锁竞争分析")
	fmt.Println("\n分析命令:")
	fmt.Println("  go tool pprof block.prof")
	fmt.Println("  go tool pprof mutex.prof")
}
```

---

## 三、net/http/pprof — 在线性能分析

### 3.1 基本使用

```go
// http_pprof_basic.go
// 演示通过 HTTP 端点暴露 pprof 分析能力
// 这是线上服务最常用的方式
package main

import (
	"fmt"
	"log"
	"math/rand"
	"net/http"
	_ "net/http/pprof" // 只需导入即可自动注册路由
	"sync"
	"time"
)

// MemoryHog 模拟一个不断消耗内存的服务
type MemoryHog struct {
	mu     sync.Mutex
	caches map[string][]byte
}

func NewMemoryHog() *MemoryHog {
	return &MemoryHog{
		caches: make(map[string][]byte),
	}
}

func (m *MemoryHog) AddEntry(key string, size int) {
	m.mu.Lock()
	defer m.mu.Unlock()
	data := make([]byte, size)
	for i := range data {
		data[i] = byte(rand.Intn(256))
	}
	m.caches[key] = data
}

func (m *MemoryHog) Size() int {
	m.mu.Lock()
	defer m.mu.Unlock()
	return len(m.caches)
}

// simulateWorkload 模拟真实的工作负载
func simulateWorkload() {
	hog := NewMemoryHog()

	// 不断添加缓存条目（模拟内存增长）
	go func() {
		for i := 0; ; i++ {
			key := fmt.Sprintf("cache-key-%d", i)
			size := rand.Intn(10240) + 1024 // 1KB-11KB
			hog.AddEntry(key, size)
			time.Sleep(10 * time.Millisecond)
		}
	}()

	// CPU 密集型工作
	go func() {
		for {
			result := 0.0
			for i := 0; i < 100000; i++ {
				result += float64(i) * 1.001
			}
			_ = result
			time.Sleep(time.Millisecond)
		}
	}()

	// 启动一些阻塞的 goroutine
	for i := 0; i < 10; i++ {
		ch := make(chan struct{})
		go func() {
			<-ch // 永远阻塞，模拟 goroutine 泄漏
		}()
	}
}

func main() {
	// 启动模拟工作负载
	simulateWorkload()

	// 自定义业务路由
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "服务正在运行\n")
	})

	http.HandleFunc("/api/status", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, `{"status": "ok", "time": "%s"}`, time.Now().Format(time.RFC3339))
	})

	// net/http/pprof 自动注册了以下路由:
	// /debug/pprof/              - pprof 索引页
	// /debug/pprof/cmdline       - 命令行参数
	// /debug/pprof/profile       - CPU profile（默认 30 秒）
	// /debug/pprof/symbol        - 符号查找
	// /debug/pprof/trace         - 执行追踪
	// /debug/pprof/heap          - 堆内存 profile
	// /debug/pprof/goroutine     - goroutine 栈信息
	// /debug/pprof/block         - 阻塞 profile
	// /debug/pprof/mutex         - 互斥锁 profile
	// /debug/pprof/allocs        - 内存分配 profile
	// /debug/pprof/threadcreate  - 线程创建 profile

	addr := ":6060"
	fmt.Printf("服务启动在 http://localhost%s\n", addr)
	fmt.Println("\npprof 端点:")
	fmt.Println("  http://localhost:6060/debug/pprof/")
	fmt.Println("\n使用方法:")
	fmt.Println("  # CPU 分析（采样 30 秒）")
	fmt.Println("  go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30")
	fmt.Println()
	fmt.Println("  # 堆内存分析")
	fmt.Println("  go tool pprof http://localhost:6060/debug/pprof/heap")
	fmt.Println()
	fmt.Println("  # Goroutine 分析")
	fmt.Println("  go tool pprof http://localhost:6060/debug/pprof/goroutine")
	fmt.Println()
	fmt.Println("  # 阻塞分析")
	fmt.Println("  go tool pprof http://localhost:6060/debug/pprof/block")
	fmt.Println()
	fmt.Println("  # 执行追踪（5 秒）")
	fmt.Println("  curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5")
	fmt.Println("  go tool trace trace.out")

	log.Fatal(http.ListenAndServe(addr, nil))
}
```

### 3.2 独立 pprof 端口（生产环境推荐）

```go
// http_pprof_separate.go
// 生产环境推荐：将 pprof 端点部署在独立端口
// 避免 pprof 端点暴露给外部用户
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"net/http/pprof"
	"os"
	"os/signal"
	"runtime"
	"syscall"
	"time"
)

// registerPprofRoutes 将 pprof 路由注册到指定的 ServeMux
func registerPprofRoutes(mux *http.ServeMux) {
	mux.HandleFunc("/debug/pprof/", pprof.Index)
	mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
}

func main() {
	// 业务服务使用默认 ServeMux
	bizMux := http.NewServeMux()
	bizMux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// 模拟业务处理
		time.Sleep(time.Duration(10+int(time.Now().UnixNano()%50)) * time.Millisecond)
		fmt.Fprintf(w, `{"message": "hello", "goroutines": %d}`, runtime.NumGoroutine())
	})

	bizMux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, `{"status": "healthy"}`)
	})

	// pprof 使用独立的 ServeMux 和端口
	pprofMux := http.NewServeMux()
	registerPprofRoutes(pprofMux)

	// 添加额外的调试信息端点
	pprofMux.HandleFunc("/debug/info", func(w http.ResponseWriter, r *http.Request) {
		var stats runtime.MemStats
		runtime.ReadMemStats(&stats)
		fmt.Fprintf(w, "Goroutines: %d\n", runtime.NumGoroutine())
		fmt.Fprintf(w, "HeapAlloc: %.2f MB\n", float64(stats.HeapAlloc)/1024/1024)
		fmt.Fprintf(w, "HeapObjects: %d\n", stats.HeapObjects)
		fmt.Fprintf(w, "GC Cycles: %d\n", stats.NumGC)
		fmt.Fprintf(w, "GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
		fmt.Fprintf(w, "NumCPU: %d\n", runtime.NumCPU())
	})

	// 启动业务服务
	bizServer := &http.Server{
		Addr:    ":8080",
		Handler: bizMux,
	}

	// 启动 pprof 服务（独立端口，仅内网可访问）
	pprofServer := &http.Server{
		Addr:    ":6060",
		Handler: pprofMux,
	}

	go func() {
		fmt.Println("业务服务启动: http://localhost:8080")
		if err := bizServer.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("业务服务异常退出: %v", err)
		}
	}()

	go func() {
		fmt.Println("pprof 服务启动: http://localhost:6060/debug/pprof/")
		if err := pprofServer.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("pprof 服务异常退出: %v", err)
		}
	}()

	// 优雅关闭
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	fmt.Println("\n收到关闭信号，优雅关闭服务...")
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	bizServer.Shutdown(ctx)
	pprofServer.Shutdown(ctx)
	fmt.Println("服务已关闭")
}
```

---

## 四、go tool pprof 交互式分析

### 4.1 常用命令速查

```bash
# === go tool pprof 常用命令 ===

# 分析本地 profile 文件
go tool pprof cpu.prof
go tool pprof heap.prof

# 分析远程服务（会自动下载 profile）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap

# 直接生成火焰图（不进入交互模式）
go tool pprof -http=:8081 cpu.prof
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30

# 指定分析模式
go tool pprof -inuse_space heap.prof    # 当前使用中的内存（默认）
go tool pprof -inuse_objects heap.prof  # 当前存活的对象数
go tool pprof -alloc_space heap.prof    # 累计分配的内存总量
go tool pprof -alloc_objects heap.prof  # 累计分配的对象总数

# === 交互模式常用命令 ===

# top - 显示资源消耗最多的函数
# (pprof) top          # 默认显示 top 10
# (pprof) top 20       # 显示 top 20
# (pprof) top -cum     # 按累积值排序

# list - 显示函数的源码级别分析
# (pprof) list funcName        # 显示某个函数的逐行分析
# (pprof) list main.           # 显示 main 包所有函数

# peek - 显示函数的调用者和被调用者
# (pprof) peek funcName

# web - 在浏览器中打开调用图
# (pprof) web

# weblist - 在浏览器中显示源码级别分析
# (pprof) weblist funcName

# tree - 显示调用树
# (pprof) tree

# traces - 显示所有的调用栈
# (pprof) traces

# focus/ignore/hide - 过滤
# (pprof) top -focus=mypackage
# (pprof) top -ignore=runtime

# === 对比两个 profile ===
# go tool pprof -base before.prof after.prof
# 这对于分析优化效果非常有用
```

### 4.2 top 命令详解

```go
// pprof_top_example.go
// 用于演示 top 命令输出的示例程序
package main

import (
	"crypto/sha256"
	"encoding/json"
	"fmt"
	"math/rand"
	"os"
	"runtime/pprof"
	"sort"
)

// 模拟数据结构
type Record struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Score float64 `json:"score"`
	Hash  string `json:"hash"`
}

// generateRecords 生成大量记录
func generateRecords(n int) []Record {
	records := make([]Record, 0, n)
	for i := 0; i < n; i++ {
		records = append(records, Record{
			ID:    i,
			Name:  fmt.Sprintf("user_%d", i),
			Score: rand.Float64() * 100,
		})
	}
	return records
}

// computeHashes 计算哈希（CPU 密集）
func computeHashes(records []Record) []Record {
	for i := range records {
		data := fmt.Sprintf("%d:%s:%.2f", records[i].ID, records[i].Name, records[i].Score)
		hash := sha256.Sum256([]byte(data))
		records[i].Hash = fmt.Sprintf("%x", hash)
	}
	return records
}

// sortRecords 排序
func sortRecords(records []Record) {
	sort.Slice(records, func(i, j int) bool {
		return records[i].Score > records[j].Score
	})
}

// serializeRecords JSON 序列化
func serializeRecords(records []Record) []byte {
	data, _ := json.Marshal(records)
	return data
}

func main() {
	f, _ := os.Create("top_demo.prof")
	defer f.Close()
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()

	// 执行工作负载
	records := generateRecords(100000)
	records = computeHashes(records)
	sortRecords(records)
	data := serializeRecords(records)

	fmt.Printf("生成 %d 条记录，JSON 大小: %d KB\n", len(records), len(data)/1024)
	fmt.Println("\n使用 go tool pprof top_demo.prof 分析")
	fmt.Println("在交互模式下输入 top 查看结果")

	// top 命令输出格式说明:
	// flat     flat%   sum%   cum      cum%   函数名
	// ----     -----   ----   ---      ----   ------
	// 1.20s    40%     40%    1.20s    40%    main.computeHashes
	// 0.80s    27%     67%    0.80s    27%    crypto/sha256.block
	// 0.50s    17%     83%    0.60s    20%    encoding/json.Marshal
	// ...
	//
	// flat:  函数自身消耗的时间（不含调用的子函数）
	// flat%: flat 占总采样的百分比
	// sum%:  从上到下 flat% 的累积
	// cum:   函数及其调用的所有子函数消耗的总时间
	// cum%:  cum 占总采样的百分比
}
```

---

## 五、火焰图生成与分析

### 5.1 使用 go tool pprof 内置 Web UI

```bash
# 方法 1：go tool pprof 内置 Web UI（推荐，Go 1.11+ 自带）
# 前提：需要安装 graphviz（用于生成调用图）
# sudo apt-get install graphviz   # Debian/Ubuntu
# brew install graphviz           # macOS

# 分析本地 profile 文件，启动 Web UI
go tool pprof -http=:8081 cpu.prof

# 分析远程服务
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30

# Web UI 提供以下视图:
# 1. Top     - 类似 top 命令的表格视图
# 2. Graph   - 调用关系图（节点大小表示资源消耗）
# 3. Flame Graph - 火焰图
# 4. Peek    - 函数调用上下文
# 5. Source  - 源码级别分析

# 方法 2：使用 go-torch（较老的方法，现在推荐使用内置 Web UI）
# go install github.com/uber/go-torch@latest
# go-torch --url http://localhost:6060 --suffix /debug/pprof/profile
```

### 5.2 火焰图阅读方法

```go
// flamegraph_demo.go
// 一个包含多层调用栈的程序，适合用火焰图分析
package main

import (
	"crypto/md5"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"math"
	"net/http"
	_ "net/http/pprof"
	"strings"
	"sync"
	"time"
)

/*
火焰图阅读要点:

1. X 轴: 表示采样比例，越宽的函数占用 CPU 越多
   - 注意: X 轴的排列顺序是字母序，不是时间序
   - 宽度 = 该函数（及其子函数）在采样中出现的次数

2. Y 轴: 表示调用栈深度
   - 最底层是入口函数（如 main.main）
   - 往上是被调用的函数
   - 栈越深表示调用层次越多

3. 颜色: 随机分配的暖色调，没有特殊含义
   - 如果看到大片相同颜色的宽块，说明该函数占用大量资源

4. 分析技巧:
   - 找最宽的 "平顶" - 这些是真正消耗资源的叶子函数
   - 找突然变宽的层 - 说明这个函数调用了很多不同的子函数
   - 对比优化前后的火焰图，看宽度变化
*/

// 模拟多层调用，制造有趣的火焰图
func handleRequest() {
	validateInput()
	processData()
	generateResponse()
}

func validateInput() {
	// 模拟输入验证
	for i := 0; i < 10000; i++ {
		_ = strings.Contains("some long validation string", "check")
	}
}

func processData() {
	transformData()
	enrichData()
	computeMetrics()
}

func transformData() {
	for i := 0; i < 50000; i++ {
		data := fmt.Sprintf("data-item-%d", i)
		hash := md5.Sum([]byte(data))
		_ = hex.EncodeToString(hash[:])
	}
}

func enrichData() {
	for i := 0; i < 30000; i++ {
		data := fmt.Sprintf("enrich-%d", i)
		hash := sha256.Sum256([]byte(data))
		_ = hex.EncodeToString(hash[:])
	}
}

func computeMetrics() {
	for i := 0; i < 100000; i++ {
		_ = math.Sqrt(float64(i)) * math.Log(float64(i+1))
	}
}

func generateResponse() {
	var builder strings.Builder
	for i := 0; i < 20000; i++ {
		builder.WriteString(fmt.Sprintf(`{"id":%d,"value":"item"},`, i))
	}
	_ = builder.String()
}

func main() {
	// 启动后台工作负载
	var wg sync.WaitGroup
	go func() {
		for {
			wg.Add(1)
			go func() {
				defer wg.Done()
				handleRequest()
			}()
			time.Sleep(10 * time.Millisecond)
		}
	}()

	fmt.Println("火焰图演示服务启动: http://localhost:6060")
	fmt.Println("\n生成火焰图:")
	fmt.Println("  go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=10")
	fmt.Println("\n然后在浏览器中选择 'Flame Graph' 视图")

	http.ListenAndServe(":6060", nil)
}
```

---

## 六、go tool trace — 执行追踪

### 6.1 Trace 基础

```go
// trace_basic.go
// 演示 go tool trace 的使用
// trace 与 pprof 不同：trace 记录的是事件时间线，而非采样统计
package main

import (
	"context"
	"fmt"
	"math/rand"
	"os"
	"runtime/trace"
	"sync"
	"time"
)

// fetchData 模拟从远程获取数据
func fetchData(ctx context.Context, id int) []byte {
	// 创建一个 trace region
	defer trace.StartRegion(ctx, "fetchData").End()

	// 模拟网络延迟
	time.Sleep(time.Duration(rand.Intn(50)+10) * time.Millisecond)

	return []byte(fmt.Sprintf("data-%d", id))
}

// processData 模拟数据处理
func processData(ctx context.Context, data []byte) string {
	defer trace.StartRegion(ctx, "processData").End()

	// 模拟 CPU 计算
	result := 0.0
	for i := 0; i < 100000; i++ {
		result += float64(data[0]) * 1.001
	}

	return fmt.Sprintf("processed: %s (%.2f)", string(data), result)
}

// aggregateResults 模拟结果聚合
func aggregateResults(ctx context.Context, results []string) string {
	defer trace.StartRegion(ctx, "aggregateResults").End()

	combined := ""
	for _, r := range results {
		combined += r + "\n"
	}
	return combined
}

func main() {
	// 创建 trace 文件
	f, err := os.Create("trace.out")
	if err != nil {
		fmt.Fprintf(os.Stderr, "创建 trace 文件失败: %v\n", err)
		os.Exit(1)
	}
	defer f.Close()

	// 开始 trace
	if err := trace.Start(f); err != nil {
		fmt.Fprintf(os.Stderr, "启动 trace 失败: %v\n", err)
		os.Exit(1)
	}
	defer trace.Stop()

	// 创建带有 task 的 context
	ctx, task := trace.NewTask(context.Background(), "main-workflow")
	defer task.End()

	fmt.Println("执行 trace 演示...")

	// 模拟并发工作流
	var wg sync.WaitGroup
	var mu sync.Mutex
	var results []string

	// 并发获取和处理数据
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			// 每个 goroutine 创建自己的 task
			taskCtx, subTask := trace.NewTask(ctx, fmt.Sprintf("worker-%d", id))
			defer subTask.End()

			// 添加日志事件
			trace.Log(taskCtx, "worker", fmt.Sprintf("worker %d 开始执行", id))

			// 获取数据
			data := fetchData(taskCtx, id)

			// 处理数据
			result := processData(taskCtx, data)

			// 保存结果
			mu.Lock()
			results = append(results, result)
			mu.Unlock()

			trace.Log(taskCtx, "worker", fmt.Sprintf("worker %d 完成", id))
		}(i)
	}

	wg.Wait()

	// 聚合结果
	final := aggregateResults(ctx, results)
	fmt.Printf("完成，结果长度: %d\n", len(final))

	fmt.Println("\ntrace 文件已保存到 trace.out")
	fmt.Println("分析命令: go tool trace trace.out")
	fmt.Println("\ngo tool trace 提供以下视图:")
	fmt.Println("  1. View trace            - 时间线视图（最常用）")
	fmt.Println("  2. Goroutine analysis     - Goroutine 分析")
	fmt.Println("  3. Network blocking       - 网络阻塞分析")
	fmt.Println("  4. Synchronization blocking - 同步阻塞分析")
	fmt.Println("  5. Syscall blocking       - 系统调用阻塞分析")
	fmt.Println("  6. Scheduler latency      - 调度器延迟分析")
	fmt.Println("  7. User-defined tasks     - 自定义任务分析")
	fmt.Println("  8. User-defined regions   - 自定义区域分析")
}
```

### 6.2 Trace 与 pprof 的区别

```go
// trace_vs_pprof.go
// 对比 trace 和 pprof 的适用场景
package main

import (
	"fmt"
)

/*
=== pprof vs trace 对比 ===

| 维度       | pprof                | trace                  |
|-----------|----------------------|------------------------|
| 数据类型   | 统计采样              | 事件序列               |
| 时间粒度   | 较粗（10ms 采样）     | 极细（纳秒级事件）      |
| 开销       | 很低（~5%）           | 较高（~10-20%）        |
| 分析内容   | CPU/内存热点函数       | 调度/阻塞/GC 时间线    |
| 适用场景   | 找到"谁消耗最多资源"   | 找到"为什么延迟高"      |
| 生产环境   | 安全使用              | 短时间采样可用          |
| 输出格式   | protobuf profile      | 二进制 trace 文件      |

什么时候用 pprof:
- CPU 使用率高，需要找到热点函数
- 内存使用高，需要找到分配源头
- goroutine 数量异常，需要找到泄漏点
- 锁竞争严重，需要找到竞争热点

什么时候用 trace:
- 请求延迟高但 CPU 不高（可能是阻塞问题）
- 需要理解 goroutine 的调度行为
- 需要分析 GC 对延迟的影响
- 需要理解并发程序的执行流程
- 需要找到 goroutine 之间的依赖关系
*/

func main() {
	fmt.Println("=== pprof vs trace 选择指南 ===")
	fmt.Println()
	fmt.Println("症状: CPU 使用率 > 80%")
	fmt.Println("  -> 使用 pprof CPU profile")
	fmt.Println()
	fmt.Println("症状: 内存持续增长不释放")
	fmt.Println("  -> 使用 pprof heap profile")
	fmt.Println()
	fmt.Println("症状: P99 延迟突然升高")
	fmt.Println("  -> 先用 pprof 排查热点，如果 CPU 不高则用 trace 分析阻塞")
	fmt.Println()
	fmt.Println("症状: goroutine 数量持续增长")
	fmt.Println("  -> 使用 pprof goroutine profile")
	fmt.Println()
	fmt.Println("症状: GC 暂停时间过长")
	fmt.Println("  -> 使用 trace 分析 GC 行为")
	fmt.Println()
	fmt.Println("症状: 并发程序行为不符合预期")
	fmt.Println("  -> 使用 trace 查看 goroutine 调度时间线")
}
```

---

## 七、实际性能问题排查流程

### 7.1 系统化排查流程

```go
// troubleshoot_workflow.go
// SRE 性能问题排查标准流程
package main

import (
	"context"
	"fmt"
	"log/slog"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"os"
	"os/signal"
	"runtime"
	"sync"
	"syscall"
	"time"
)

/*
=== SRE 性能问题排查标准流程 ===

第一步：确认问题
  1. 明确症状（CPU 高？内存高？延迟高？goroutine 多？）
  2. 确认问题发生的时间、频率、影响范围
  3. 检查监控指标（Prometheus/Grafana）

第二步：采集数据
  # CPU 问题
  go tool pprof http://服务地址:6060/debug/pprof/profile?seconds=30

  # 内存问题
  go tool pprof http://服务地址:6060/debug/pprof/heap

  # Goroutine 问题
  go tool pprof http://服务地址:6060/debug/pprof/goroutine
  # 或者获取可读文本
  curl http://服务地址:6060/debug/pprof/goroutine?debug=2

  # 延迟问题（且 CPU 不高）
  curl -o trace.out http://服务地址:6060/debug/pprof/trace?seconds=5

  # 锁竞争问题
  go tool pprof http://服务地址:6060/debug/pprof/mutex
  go tool pprof http://服务地址:6060/debug/pprof/block

第三步：分析数据
  1. 使用 go tool pprof -http=:8081 xxx.prof 查看火焰图
  2. 使用 top/list 命令找到热点
  3. 分析调用栈，定位问题代码

第四步：验证修复
  1. 在测试环境复现问题
  2. 应用修复
  3. 再次采集 profile 对比
  go tool pprof -base before.prof after.prof
*/

// MetricsCollector 模拟一个有性能问题的指标收集器
type MetricsCollector struct {
	mu      sync.Mutex
	metrics map[string]float64
	logger  *slog.Logger
}

func NewMetricsCollector(logger *slog.Logger) *MetricsCollector {
	return &MetricsCollector{
		metrics: make(map[string]float64),
		logger:  logger,
	}
}

// Record 记录指标（有锁竞争问题）
func (mc *MetricsCollector) Record(name string, value float64) {
	mc.mu.Lock()
	defer mc.mu.Unlock()
	mc.metrics[name] = value
}

// Snapshot 获取指标快照
func (mc *MetricsCollector) Snapshot() map[string]float64 {
	mc.mu.Lock()
	defer mc.mu.Unlock()
	snapshot := make(map[string]float64, len(mc.metrics))
	for k, v := range mc.metrics {
		snapshot[k] = v
	}
	return snapshot
}

// simulateServiceLoad 模拟真实服务负载
func simulateServiceLoad(ctx context.Context, collector *MetricsCollector) {
	ticker := time.NewTicker(time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			// 模拟请求处理
			start := time.Now()

			// 模拟一些计算
			result := 0.0
			for i := 0; i < 10000; i++ {
				result += rand.Float64()
			}

			// 记录指标
			duration := time.Since(start).Seconds()
			collector.Record("request_duration", duration)
			collector.Record("request_count", result)
		}
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	// 开启 block 和 mutex profiling
	runtime.SetBlockProfileRate(1)
	runtime.SetMutexProfileFraction(1)

	collector := NewMetricsCollector(logger)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 启动多个模拟负载
	for i := 0; i < 4; i++ {
		go simulateServiceLoad(ctx, collector)
	}

	// 定期输出运行状态
	go func() {
		ticker := time.NewTicker(5 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				var stats runtime.MemStats
				runtime.ReadMemStats(&stats)
				logger.Info("运行状态",
					"goroutines", runtime.NumGoroutine(),
					"heap_mb", fmt.Sprintf("%.2f", float64(stats.HeapAlloc)/1024/1024),
					"gc_count", stats.NumGC,
				)
			}
		}
	}()

	// HTTP 服务
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, `{"status":"ok","goroutines":%d}`, runtime.NumGoroutine())
	})

	http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		snapshot := collector.Snapshot()
		for k, v := range snapshot {
			fmt.Fprintf(w, "%s %.4f\n", k, v)
		}
	})

	go func() {
		logger.Info("服务启动", "addr", ":8080", "pprof", "http://localhost:8080/debug/pprof/")
		if err := http.ListenAndServe(":8080", nil); err != nil {
			logger.Error("服务启动失败", "error", err)
		}
	}()

	// 等待退出信号
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("服务关闭中...")
	cancel()
	time.Sleep(time.Second)
	logger.Info("服务已关闭")
}
```

### 7.2 Benchmark 与 Profile 结合

```go
// bench_with_profile_test.go
// 演示如何在 benchmark 测试中生成 profile
package main

import (
	"crypto/sha256"
	"fmt"
	"strings"
	"testing"
)

// hashString 被测试的函数
func hashString(s string) string {
	hash := sha256.Sum256([]byte(s))
	return fmt.Sprintf("%x", hash)
}

// concatStringSlow 低效的字符串拼接
func concatStringSlow(n int) string {
	result := ""
	for i := 0; i < n; i++ {
		result += fmt.Sprintf("item-%d,", i)
	}
	return result
}

// concatStringFast 高效的字符串拼接
func concatStringFast(n int) string {
	var builder strings.Builder
	builder.Grow(n * 10) // 预分配
	for i := 0; i < n; i++ {
		fmt.Fprintf(&builder, "item-%d,", i)
	}
	return builder.String()
}

func BenchmarkHashString(b *testing.B) {
	input := "hello world benchmark test"
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = hashString(input)
	}
}

func BenchmarkConcatStringSlow(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = concatStringSlow(100)
	}
}

func BenchmarkConcatStringFast(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = concatStringFast(100)
	}
}

/*
运行 benchmark 并生成 profile:

# 生成 CPU profile
go test -bench=BenchmarkConcatStringSlow -cpuprofile=cpu_bench.prof -benchmem -count=3

# 生成内存 profile
go test -bench=BenchmarkConcatStringSlow -memprofile=mem_bench.prof -benchmem -count=3

# 同时生成 CPU 和内存 profile
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof -benchmem

# 分析 benchmark profile
go tool pprof -http=:8081 cpu_bench.prof

# 对比两个 benchmark 的结果
go install golang.org/x/perf/cmd/benchstat@latest
go test -bench=BenchmarkConcatString -benchmem -count=5 > result.txt
benchstat result.txt

# benchstat 还可以对比两个文件
go test -bench=. -benchmem -count=5 > old.txt
# ... 修改代码 ...
go test -bench=. -benchmem -count=5 > new.txt
benchstat old.txt new.txt
*/
```

---

## 八、SRE 实战：线上服务性能分析

### 8.1 完整的可观测性服务

```go
// sre_observable_service.go
// SRE 实战：一个包含完整可观测性的服务模板
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"math/rand"
	"net/http"
	"net/http/pprof"
	"os"
	"os/signal"
	"runtime"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

// === 请求计数器 ===
type RequestMetrics struct {
	totalRequests   atomic.Int64
	failedRequests  atomic.Int64
	activeRequests  atomic.Int64
	totalDurationNs atomic.Int64
}

func (rm *RequestMetrics) RecordRequest(duration time.Duration, failed bool) {
	rm.totalRequests.Add(1)
	rm.totalDurationNs.Add(int64(duration))
	if failed {
		rm.failedRequests.Add(1)
	}
}

func (rm *RequestMetrics) IncrActive() { rm.activeRequests.Add(1) }
func (rm *RequestMetrics) DecrActive() { rm.activeRequests.Add(-1) }

func (rm *RequestMetrics) Snapshot() map[string]interface{} {
	total := rm.totalRequests.Load()
	avgDuration := time.Duration(0)
	if total > 0 {
		avgDuration = time.Duration(rm.totalDurationNs.Load() / total)
	}
	return map[string]interface{}{
		"total_requests":  total,
		"failed_requests": rm.failedRequests.Load(),
		"active_requests": rm.activeRequests.Load(),
		"avg_duration_ms": avgDuration.Milliseconds(),
	}
}

// === 模拟业务缓存（可能导致内存泄漏） ===
type BusinessCache struct {
	mu    sync.RWMutex
	store map[string]*CacheEntry
}

type CacheEntry struct {
	Data      []byte
	CreatedAt time.Time
	TTL       time.Duration
}

func NewBusinessCache() *BusinessCache {
	return &BusinessCache{
		store: make(map[string]*CacheEntry),
	}
}

func (c *BusinessCache) Set(key string, data []byte, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.store[key] = &CacheEntry{
		Data:      data,
		CreatedAt: time.Now(),
		TTL:       ttl,
	}
}

func (c *BusinessCache) Get(key string) ([]byte, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	entry, ok := c.store[key]
	if !ok {
		return nil, false
	}
	if time.Since(entry.CreatedAt) > entry.TTL {
		return nil, false // 过期了，但没有清理！（内存泄漏）
	}
	return entry.Data, true
}

func (c *BusinessCache) Size() int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return len(c.store)
}

// CleanExpired 清理过期条目（修复内存泄漏的方法）
func (c *BusinessCache) CleanExpired() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	cleaned := 0
	for key, entry := range c.store {
		if time.Since(entry.CreatedAt) > entry.TTL {
			delete(c.store, key)
			cleaned++
		}
	}
	return cleaned
}

// === 服务主体 ===
type Service struct {
	logger  *slog.Logger
	metrics *RequestMetrics
	cache   *BusinessCache
}

func NewService(logger *slog.Logger) *Service {
	return &Service{
		logger:  logger,
		metrics: &RequestMetrics{},
		cache:   NewBusinessCache(),
	}
}

// handleAPI 模拟 API 处理
func (s *Service) handleAPI(w http.ResponseWriter, r *http.Request) {
	s.metrics.IncrActive()
	defer s.metrics.DecrActive()

	start := time.Now()

	// 模拟业务处理
	delay := time.Duration(rand.Intn(100)+10) * time.Millisecond
	time.Sleep(delay)

	// 模拟缓存操作
	key := fmt.Sprintf("api-key-%d", rand.Intn(10000))
	if _, ok := s.cache.Get(key); !ok {
		// 缓存未命中，生成数据
		data := make([]byte, rand.Intn(4096)+512)
		s.cache.Set(key, data, 30*time.Second)
	}

	// 随机失败
	failed := rand.Float64() < 0.05
	duration := time.Since(start)
	s.metrics.RecordRequest(duration, failed)

	if failed {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"error": "internal server error"}`)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintf(w, `{"status": "ok", "duration_ms": %d}`, duration.Milliseconds())
}

// handleStatus 运行状态端点
func (s *Service) handleStatus(w http.ResponseWriter, r *http.Request) {
	var stats runtime.MemStats
	runtime.ReadMemStats(&stats)

	status := map[string]interface{}{
		"goroutines":     runtime.NumGoroutine(),
		"heap_alloc_mb":  float64(stats.HeapAlloc) / 1024 / 1024,
		"heap_objects":   stats.HeapObjects,
		"gc_cycles":      stats.NumGC,
		"gc_pause_total": time.Duration(stats.PauseTotalNs).String(),
		"cache_size":     s.cache.Size(),
		"requests":       s.metrics.Snapshot(),
		"go_version":     runtime.Version(),
		"num_cpu":        runtime.NumCPU(),
		"gomaxprocs":     runtime.GOMAXPROCS(0),
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(status)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	// 开启所有 profiling
	runtime.SetBlockProfileRate(1)
	runtime.SetMutexProfileFraction(1)

	svc := NewService(logger)

	// === 业务路由 ===
	bizMux := http.NewServeMux()
	bizMux.HandleFunc("/api/data", svc.handleAPI)
	bizMux.HandleFunc("/status", svc.handleStatus)
	bizMux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "ok")
	})

	// === pprof 路由（独立端口） ===
	pprofMux := http.NewServeMux()
	pprofMux.HandleFunc("/debug/pprof/", pprof.Index)
	pprofMux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	pprofMux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	pprofMux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	pprofMux.HandleFunc("/debug/pprof/trace", pprof.Trace)

	// 启动缓存清理（如果不启动，就是内存泄漏场景）
	ctx, cancel := context.WithCancel(context.Background())
	go func() {
		ticker := time.NewTicker(10 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				cleaned := svc.cache.CleanExpired()
				if cleaned > 0 {
					logger.Info("清理过期缓存", "cleaned", cleaned, "remaining", svc.cache.Size())
				}
			}
		}
	}()

	// 模拟持续流量
	go func() {
		client := &http.Client{Timeout: 5 * time.Second}
		for {
			select {
			case <-ctx.Done():
				return
			default:
				resp, err := client.Get("http://localhost:8080/api/data")
				if err == nil {
					resp.Body.Close()
				}
				time.Sleep(time.Duration(rand.Intn(10)+1) * time.Millisecond)
			}
		}
	}()

	bizServer := &http.Server{Addr: ":8080", Handler: bizMux}
	pprofServer := &http.Server{Addr: ":6060", Handler: pprofMux}

	go func() {
		logger.Info("业务服务启动", "addr", ":8080")
		if err := bizServer.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("业务服务异常", "error", err)
		}
	}()

	go func() {
		logger.Info("pprof 服务启动", "addr", ":6060")
		if err := pprofServer.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("pprof 服务异常", "error", err)
		}
	}()

	logger.Info("=== 性能分析命令 ===")
	logger.Info("CPU 分析: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30")
	logger.Info("内存分析: go tool pprof http://localhost:6060/debug/pprof/heap")
	logger.Info("Goroutine: go tool pprof http://localhost:6060/debug/pprof/goroutine")
	logger.Info("火焰图: go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=10")
	logger.Info("运行状态: curl http://localhost:8080/status | jq .")

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	cancel()
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer shutdownCancel()
	bizServer.Shutdown(shutdownCtx)
	pprofServer.Shutdown(shutdownCtx)
	logger.Info("服务已关闭")
}
```

---

## 九、常见坑点

### 9.1 pprof 使用常见错误

```go
// common_pitfalls.go
// pprof 使用中的常见错误和注意事项
package main

import "fmt"

func main() {
	fmt.Println("=== pprof 常见坑点 ===")
}

/*
坑点 1：CPU Profile 采集时间太短
  错误：go tool pprof http://...:6060/debug/pprof/profile?seconds=1
  原因：1 秒只采样约 100 次，数据量不够
  正确：至少采样 10-30 秒
  go tool pprof http://...:6060/debug/pprof/profile?seconds=30

坑点 2：忘记开启 Block/Mutex Profiling
  错误：直接获取 block/mutex profile，结果为空
  正确：需要在程序中开启
  runtime.SetBlockProfileRate(1)
  runtime.SetMutexProfileFraction(1)

坑点 3：heap profile 混淆 inuse 和 alloc
  - inuse_space: 当前正在使用的内存（找内存泄漏用这个）
  - alloc_space: 累计分配的内存总量（找分配热点用这个）
  - 默认是 inuse_space
  - 排查内存泄漏时：对比两个时间点的 inuse_space 差异

坑点 4：goroutine profile debug 级别选错
  - debug=0: protobuf 格式，用于 go tool pprof
  - debug=1: 文本格式，按栈合并显示
  - debug=2: 文本格式，每个 goroutine 单独显示（最详细）
  - 线上排查一般用 debug=2

坑点 5：pprof 端点暴露给公网
  - pprof 端点不应该对外暴露
  - 正确做法：使用独立端口，通过网络策略限制只有内网可访问
  - 或者通过 kubectl port-forward 临时转发

坑点 6：Profile 采样率设置不当
  - MemProfileRate=1 会记录每次分配，性能开销极大
  - 生产环境保持默认值（512KB）
  - BlockProfileRate=1 也会有性能影响，排查完要关闭

坑点 7：对比 Profile 时版本不一致
  - 对比两个 profile 时，确保二进制版本一致
  - 否则函数地址不匹配，对比结果无意义
  - go tool pprof -base before.prof after.prof

坑点 8：在容器中使用 pprof
  - 容器中 GOMAXPROCS 可能被错误设置
  - 使用 uber-go/automaxprocs 自动适配
  import _ "go.uber.org/automaxprocs"
  - 容器网络可能需要特殊配置才能访问 pprof 端口
*/
```

---

## 十、速查表

### pprof 命令速查

```
=== 采集 Profile ===
# CPU（30 秒采样）
go tool pprof http://HOST:PORT/debug/pprof/profile?seconds=30

# 堆内存
go tool pprof http://HOST:PORT/debug/pprof/heap

# Goroutine
go tool pprof http://HOST:PORT/debug/pprof/goroutine

# 阻塞
go tool pprof http://HOST:PORT/debug/pprof/block

# 互斥锁
go tool pprof http://HOST:PORT/debug/pprof/mutex

# Trace（5 秒）
curl -o trace.out http://HOST:PORT/debug/pprof/trace?seconds=5

=== 分析 Profile ===
# 交互模式
go tool pprof xxx.prof

# Web UI（推荐）
go tool pprof -http=:8081 xxx.prof

# 指定分析维度
go tool pprof -inuse_space heap.prof    # 当前使用的内存
go tool pprof -alloc_space heap.prof    # 累计分配的内存

# 对比两个 Profile
go tool pprof -base before.prof after.prof

=== 交互模式命令 ===
top [N]          # 显示前 N 个热点（默认 10）
top -cum         # 按累积值排序
list funcName    # 显示函数源码级别分析
peek funcName    # 显示调用者和被调用者
web              # 浏览器中显示调用图
weblist funcName # 浏览器中显示源码分析
tree             # 调用树
traces           # 所有调用栈
svg              # 输出 SVG 图
png              # 输出 PNG 图

=== Benchmark + Profile ===
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof -benchmem
go tool pprof cpu.prof

=== Trace ===
go tool trace trace.out
# 查看 goroutine 调度、GC 事件、网络阻塞等时间线

=== 实用脚本 ===
# 一键采集所有 profile（保存到带时间戳的目录）
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
mkdir -p profiles/$TIMESTAMP
curl -o profiles/$TIMESTAMP/heap.prof     http://localhost:6060/debug/pprof/heap
curl -o profiles/$TIMESTAMP/goroutine.txt "http://localhost:6060/debug/pprof/goroutine?debug=2"
go tool pprof -proto -output profiles/$TIMESTAMP/cpu.prof \
  http://localhost:6060/debug/pprof/profile?seconds=30
curl -o profiles/$TIMESTAMP/trace.out     "http://localhost:6060/debug/pprof/trace?seconds=5"
echo "Profile 已保存到 profiles/$TIMESTAMP/"
```

### 小结

| 场景 | 工具 | 命令 |
|---|---|---|
| CPU 热点 | pprof CPU | `go tool pprof .../profile?seconds=30` |
| 内存泄漏 | pprof Heap | `go tool pprof -inuse_space .../heap` |
| 分配热点 | pprof Allocs | `go tool pprof -alloc_space .../heap` |
| Goroutine 泄漏 | pprof Goroutine | `curl .../goroutine?debug=2` |
| 锁竞争 | pprof Block/Mutex | `go tool pprof .../block` |
| 延迟分析 | Trace | `go tool trace trace.out` |
| GC 影响 | Trace | 查看 trace 中的 GC 事件 |
| 优化对比 | pprof diff | `go tool pprof -base old.prof new.prof` |
| Benchmark 分析 | test + pprof | `go test -bench=. -cpuprofile=cpu.prof` |
| 火焰图 | pprof Web UI | `go tool pprof -http=:8081 xxx.prof` |
