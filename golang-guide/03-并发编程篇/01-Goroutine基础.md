# 01 - Goroutine 基础

## 导读

Go 语言最核心的特性之一就是内置的并发支持。与 Java 的线程、Python 的 threading/asyncio 不同，Go 通过 **goroutine** 提供了一种极其轻量级的并发原语，让你可以轻松创建成千上万个并发执行单元。

作为 SRE/DevOps 工程师，你会在以下场景频繁使用 goroutine：
- 并发执行多台服务器的健康检查
- 同时拉取多个监控指标
- 并行处理日志流
- 构建高并发的 HTTP 服务

本篇将从并发与并行的概念出发，深入讲解 goroutine 的创建、调度模型（GMP）、生命周期管理，并与 Python 的并发方案进行对比，帮助你建立扎实的 Go 并发编程基础。

---

## 1. 并发 vs 并行

### 1.1 概念区分

很多人混淆「并发」和「并行」，Rob Pike（Go 语言创始人之一）有一句经典总结：

> **Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.**
> 并发是同时处理多件事情的能力。并行是同时执行多件事情。

| 概念 | 英文 | 含义 | 类比 |
|------|------|------|------|
| 并发 | Concurrency | 多个任务交替执行，逻辑上同时 | 一个厨师在切菜和炒菜之间快速切换 |
| 并行 | Parallelism | 多个任务真正同时执行 | 两个厨师各自做一道菜 |

### 1.2 Go 的并发模型

Go 采用 **CSP（Communicating Sequential Processes）** 模型：
- goroutine 是并发执行的基本单元
- channel 是 goroutine 之间通信的桥梁
- 运行时调度器负责将 goroutine 映射到操作系统线程上执行

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	// 查看逻辑 CPU 数量（决定最大并行度）
	fmt.Printf("逻辑 CPU 数量: %d\n", runtime.NumCPU())
	// 查看当前 GOMAXPROCS 设置（默认等于 NumCPU）
	fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))

	// 启动一个 goroutine（并发）
	go func() {
		fmt.Println("我在另一个 goroutine 中执行")
	}()

	// 在单核上是并发（交替执行），在多核上可以是并行（真正同时执行）
	time.Sleep(100 * time.Millisecond)
}
```

### 1.3 单核并发 vs 多核并行示例

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func work(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	start := time.Now()
	// 模拟 CPU 密集型工作
	sum := 0
	for i := 0; i < 100_000_000; i++ {
		sum += i
	}
	fmt.Printf("Worker %d 完成，耗时: %v\n", id, time.Since(start))
}

func main() {
	// 实验1: 限制为单核（只能并发，不能并行）
	fmt.Println("=== 单核模式（GOMAXPROCS=1）===")
	runtime.GOMAXPROCS(1)
	var wg sync.WaitGroup
	start := time.Now()
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go work(i, &wg)
	}
	wg.Wait()
	fmt.Printf("单核总耗时: %v\n\n", time.Since(start))

	// 实验2: 使用全部 CPU（可以真正并行）
	fmt.Println("=== 多核模式（GOMAXPROCS=NumCPU）===")
	runtime.GOMAXPROCS(runtime.NumCPU())
	start = time.Now()
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go work(i, &wg)
	}
	wg.Wait()
	fmt.Printf("多核总耗时: %v\n", time.Since(start))
}
```

运行结果（4 核机器上）：
```
=== 单核模式（GOMAXPROCS=1）===
Worker 0 完成，耗时: 52ms
Worker 1 完成，耗时: 51ms
Worker 2 完成，耗时: 50ms
Worker 3 完成，耗时: 53ms
单核总耗时: 206ms

=== 多核模式（GOMAXPROCS=NumCPU）===
Worker 2 完成，耗时: 53ms
Worker 0 完成，耗时: 54ms
Worker 3 完成，耗时: 55ms
Worker 1 完成，耗时: 54ms
多核总耗时: 56ms
```

---

## 2. Goroutine 创建

### 2.1 go 关键字

创建 goroutine 只需在函数调用前加上 `go` 关键字：

```go
package main

import (
	"fmt"
	"time"
)

// 普通函数
func sayHello(name string) {
	fmt.Printf("Hello, %s!\n", name)
}

func main() {
	// 方式1: go + 具名函数
	go sayHello("Alice")

	// 方式2: go + 匿名函数
	go func() {
		fmt.Println("我是匿名 goroutine")
	}()

	// 方式3: go + 带参数的匿名函数
	message := "传递参数"
	go func(msg string) {
		fmt.Println(msg)
	}(message)

	// 方式4: go + 方法调用
	server := &Server{Name: "web-01"}
	go server.Start()

	// 等待 goroutine 完成（生产环境不要用 sleep，这里仅作演示）
	time.Sleep(100 * time.Millisecond)
}

// Server 结构体
type Server struct {
	Name string
}

// Start 是 Server 的方法
func (s *Server) Start() {
	fmt.Printf("服务器 %s 已启动\n", s.Name)
}
```

### 2.2 goroutine 没有返回值

`go` 语句不返回任何值。如果需要获取结果，必须通过 channel 或共享变量传递：

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
)

func main() {
	// 错误写法（编译不通过）：
	// result := go compute()

	// 正确写法1: 通过 channel 获取结果
	ch := make(chan int)
	go func() {
		result := rand.Intn(100)
		ch <- result // 将结果发送到 channel
	}()
	value := <-ch // 从 channel 接收结果
	fmt.Printf("通过 channel 获取结果: %d\n", value)

	// 正确写法2: 通过共享变量（需要同步）
	var result int
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		result = rand.Intn(100) // 写入共享变量
	}()
	wg.Wait() // 等待完成，确保写入已发生
	fmt.Printf("通过共享变量获取结果: %d\n", result)
}
```

---

## 3. Goroutine 的轻量级特性

### 3.1 goroutine vs OS 线程

| 特性 | Goroutine | OS 线程 |
|------|-----------|---------|
| 初始栈大小 | **2 KB**（可动态增长到 1 GB） | **1-8 MB**（固定） |
| 创建开销 | 极低（~0.3 us） | 高（~30 us） |
| 切换开销 | 低（用户态调度） | 高（需要内核态切换） |
| 数量限制 | 轻松创建百万级 | 通常千级 |
| 调度方式 | Go 运行时调度（协作式+抢占式） | 操作系统内核调度（抢占式） |
| 内存占用 | 极低 | 较高 |

### 3.2 创建大量 goroutine 的示例

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	// 创建 100 万个 goroutine
	const numGoroutines = 1_000_000

	var wg sync.WaitGroup
	wg.Add(numGoroutines)

	// 记录初始内存
	var memBefore runtime.MemStats
	runtime.ReadMemStats(&memBefore)

	start := time.Now()

	for i := 0; i < numGoroutines; i++ {
		go func(id int) {
			defer wg.Done()
			// 模拟一些轻量工作
			time.Sleep(time.Second)
		}(i)
	}

	createTime := time.Since(start)

	// 记录创建后的内存
	var memAfter runtime.MemStats
	runtime.ReadMemStats(&memAfter)

	fmt.Printf("创建 %d 个 goroutine\n", numGoroutines)
	fmt.Printf("创建耗时: %v\n", createTime)
	fmt.Printf("当前 goroutine 数量: %d\n", runtime.NumGoroutine())
	fmt.Printf("内存增长: %.2f MB\n", float64(memAfter.Alloc-memBefore.Alloc)/1024/1024)
	fmt.Printf("每个 goroutine 约占: %.2f KB\n",
		float64(memAfter.Alloc-memBefore.Alloc)/float64(numGoroutines)/1024)

	wg.Wait()
	fmt.Println("全部完成")
}
```

### 3.3 栈的动态增长

goroutine 的栈从 2KB 开始，按需自动增长（最大默认 1GB）：

```go
package main

import (
	"fmt"
	"runtime"
)

// 递归函数，每次调用增加栈深度
func recursive(depth int) int {
	if depth == 0 {
		// 在最深处打印 goroutine 栈信息
		var m runtime.MemStats
		runtime.ReadMemStats(&m)
		fmt.Printf("递归深度 %d 层时，栈使用: %d KB\n", depth, m.StackInuse/1024)
		return 0
	}
	// 在栈上分配一些空间
	var arr [64]byte
	_ = arr
	return recursive(depth-1) + 1
}

func main() {
	fmt.Println("=== goroutine 栈动态增长演示 ===")
	fmt.Printf("初始 goroutine 数: %d\n", runtime.NumGoroutine())

	// 浅递归
	recursive(100)

	// 深递归（栈会自动增长）
	recursive(10000)

	// 更深的递归
	recursive(100000)
}
```

---

## 4. GMP 调度模型详解

### 4.1 GMP 三大组件

Go 运行时的调度器基于 **GMP 模型**：

```
                    +-----------+
                    | Go 程序    |
                    +-----+-----+
                          |
              +-----------+-----------+
              |                       |
         +----+----+            +----+----+
         |    P    |            |    P    |    P = Processor（逻辑处理器）
         +----+----+            +----+----+    数量 = GOMAXPROCS
              |                       |
    +---------+---------+       +-----+-----+
    |    本地队列 LRQ    |       |  本地队列   |
    | [G] [G] [G] [G]  |       | [G] [G]   |  G = Goroutine
    +---------+---------+       +-----+-----+
              |                       |
         +----+----+            +----+----+
         |    M    |            |    M    |    M = Machine（OS 线程）
         +----+----+            +----+----+
              |                       |
    +---------+---------+       +-----+-----+
    |   OS 线程          |       |  OS 线程    |
    +-------------------+       +-----------+

              全局队列 GRQ: [G] [G] [G] ...
```

**G（Goroutine）**：
- Go 代码的并发执行单元
- 包含栈、程序计数器、状态等信息
- 状态：_Gidle、_Grunnable、_Grunning、_Gsyscall、_Gwaiting、_Gdead

**M（Machine/Thread）**：
- 操作系统线程，由操作系统管理
- 真正执行代码的载体
- M 必须绑定一个 P 才能执行 G

**P（Processor）**：
- 逻辑处理器，是 G 和 M 之间的桥梁
- 数量由 GOMAXPROCS 决定
- 每个 P 有一个本地运行队列（Local Run Queue）

### 4.2 调度流程

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	// GOMAXPROCS 决定了 P 的数量
	// P 的数量决定了最大并行度
	numP := runtime.GOMAXPROCS(0) // 0 表示获取当前值，不修改
	fmt.Printf("P 的数量 (GOMAXPROCS): %d\n", numP)
	fmt.Printf("M 的理论最大数量: 10000 (runtime/debug.SetMaxThreads)\n")
	fmt.Printf("G 的当前数量: %d\n", runtime.NumGoroutine())

	var wg sync.WaitGroup
	// 创建一些 goroutine 来观察调度
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			// Gosched 让出当前 goroutine 的执行权，重新排队
			// 类似于 Python 的 asyncio.sleep(0)
			runtime.Gosched()
			fmt.Printf("G-%d 在 goroutine %d 中执行\n", id, runtime.NumGoroutine())
		}(i)
	}
	wg.Wait()
}
```

### 4.3 调度策略详解

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// 演示调度器的工作窃取（Work Stealing）机制
// 当一个 P 的本地队列为空时，它会：
// 1. 先从全局队列取
// 2. 再从其他 P 的本地队列「偷」一半
func main() {
	runtime.GOMAXPROCS(4) // 设置 4 个 P
	fmt.Println("=== GMP 调度策略演示 ===")

	var wg sync.WaitGroup

	// 场景1: 正常调度
	fmt.Println("\n--- 场景1: 正常的 goroutine 调度 ---")
	for i := 0; i < 8; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			fmt.Printf("Goroutine %d 开始执行\n", id)
			time.Sleep(10 * time.Millisecond)
			fmt.Printf("Goroutine %d 执行完成\n", id)
		}(i)
	}
	wg.Wait()

	// 场景2: 系统调用导致 M 阻塞
	// 当 G 进行系统调用（如文件I/O）时，M 会被阻塞
	// 调度器会将 P 从阻塞的 M 上分离，绑定到新的/空闲的 M 上
	fmt.Println("\n--- 场景2: Hand-off 机制 ---")
	fmt.Println("当 M 因系统调用阻塞时，P 会被转交给其他 M")
	fmt.Println("这确保了其他 goroutine 不会因为一个阻塞而停滞")

	// 场景3: 抢占式调度（Go 1.14+ 基于信号的抢占）
	fmt.Println("\n--- 场景3: 抢占式调度 ---")
	fmt.Println("Go 1.14 之前：只在函数调用时检查抢占标记（协作式）")
	fmt.Println("Go 1.14 之后：基于信号的异步抢占，即使死循环也能被抢占")

	// 演示：在 Go 1.14+ 中，死循环不会卡死调度器
	done := make(chan bool)
	go func() {
		count := 0
		for {
			count++
			if count > 100_000_000 {
				break
			}
			// 即使没有函数调用，Go 1.14+ 也能抢占这个 goroutine
		}
		done <- true
	}()

	select {
	case <-done:
		fmt.Println("密集计算 goroutine 完成（已被正确调度）")
	case <-time.After(5 * time.Second):
		fmt.Println("超时（调度器可能存在问题）")
	}
}
```

### 4.4 GMP 状态查看（GODEBUG）

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// 运行方式：GODEBUG=schedtrace=1000 go run main.go
// 这会每 1000ms 打印一次调度器状态
//
// 输出示例：
// SCHED 0ms: gomaxprocs=8 idleprocs=6 threads=4 spinningthreads=1
//   idlethreads=0 runqueue=0 [0 0 0 0 0 0 0 0]
//
// 字段说明：
//   gomaxprocs  - P 的数量
//   idleprocs   - 空闲的 P 数量
//   threads     - M 的总数
//   spinningthreads - 自旋状态的 M 数量（在寻找可运行的 G）
//   idlethreads - 空闲的 M 数量
//   runqueue    - 全局队列中的 G 数量
//   [0 0 ...]   - 每个 P 的本地队列中的 G 数量

func main() {
	runtime.GOMAXPROCS(4)
	fmt.Println("启动 GMP 调度观察...")
	fmt.Println("提示: 使用 GODEBUG=schedtrace=1000 go run main.go 查看调度信息")

	var wg sync.WaitGroup

	// 创建不同类型的 goroutine 来观察调度器行为
	// CPU 密集型
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			sum := 0
			for j := 0; j < 50_000_000; j++ {
				sum += j
			}
			fmt.Printf("[CPU] Worker %d 完成, sum=%d\n", id, sum)
		}(i)
	}

	// I/O 型（sleep 模拟）
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			time.Sleep(2 * time.Second)
			fmt.Printf("[I/O] Worker %d 完成\n", id)
		}(i)
	}

	wg.Wait()
	fmt.Println("所有 worker 完成")
}
```

---

## 5. GOMAXPROCS 设置

### 5.1 基本用法

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// 获取逻辑 CPU 数量
	cpus := runtime.NumCPU()
	fmt.Printf("逻辑 CPU 数: %d\n", cpus)

	// 获取当前 GOMAXPROCS（传入 0 不修改，只读取）
	current := runtime.GOMAXPROCS(0)
	fmt.Printf("当前 GOMAXPROCS: %d\n", current)

	// 设置 GOMAXPROCS（返回旧值）
	old := runtime.GOMAXPROCS(2)
	fmt.Printf("旧的 GOMAXPROCS: %d, 新的: %d\n", old, runtime.GOMAXPROCS(0))

	// 恢复默认值
	runtime.GOMAXPROCS(runtime.NumCPU())
	fmt.Printf("恢复后 GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
}
```

### 5.2 容器环境下的 GOMAXPROCS

在 Kubernetes / Docker 容器中，`runtime.NumCPU()` 返回的是**宿主机**的 CPU 数量，而非容器分配的 CPU 限制。这可能导致过多的 P 竞争有限的 CPU 资源。

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"strconv"
	"strings"
)

// 从 cgroup 中读取容器 CPU 限制
// 在 Kubernetes 中：resources.limits.cpu
func getContainerCPULimit() (float64, error) {
	// cgroup v2 路径
	data, err := os.ReadFile("/sys/fs/cgroup/cpu.max")
	if err == nil {
		parts := strings.Fields(string(data))
		if len(parts) == 2 && parts[0] != "max" {
			quota, _ := strconv.ParseFloat(parts[0], 64)
			period, _ := strconv.ParseFloat(parts[1], 64)
			return quota / period, nil
		}
	}

	// cgroup v1 路径
	quotaData, err := os.ReadFile("/sys/fs/cgroup/cpu/cpu.cfs_quota_us")
	if err != nil {
		return 0, err
	}
	periodData, err := os.ReadFile("/sys/fs/cgroup/cpu/cpu.cfs_period_us")
	if err != nil {
		return 0, err
	}

	quota, _ := strconv.ParseFloat(strings.TrimSpace(string(quotaData)), 64)
	period, _ := strconv.ParseFloat(strings.TrimSpace(string(periodData)), 64)

	if quota <= 0 {
		return float64(runtime.NumCPU()), nil // 无限制
	}
	return quota / period, nil
}

func main() {
	fmt.Printf("runtime.NumCPU() = %d (宿主机 CPU 数)\n", runtime.NumCPU())

	cpuLimit, err := getContainerCPULimit()
	if err != nil {
		fmt.Printf("无法读取容器 CPU 限制: %v\n", err)
		fmt.Println("可能不在容器环境中运行")
	} else {
		fmt.Printf("容器 CPU 限制: %.1f 核\n", cpuLimit)
		// 设置合理的 GOMAXPROCS
		maxProcs := int(cpuLimit)
		if maxProcs < 1 {
			maxProcs = 1
		}
		runtime.GOMAXPROCS(maxProcs)
		fmt.Printf("已设置 GOMAXPROCS = %d\n", maxProcs)
	}

	// 生产环境推荐使用 uber-go/automaxprocs
	// import _ "go.uber.org/automaxprocs"
	// 它会自动根据容器 CPU 限制设置 GOMAXPROCS
	fmt.Println("\n生产环境推荐：")
	fmt.Println("  import _ \"go.uber.org/automaxprocs\"")
	fmt.Println("  在 main() 的 init 阶段自动设置正确的 GOMAXPROCS")
}
```

---

## 6. Goroutine 生命周期管理

### 6.1 主 goroutine 退出问题

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// 问题演示：主 goroutine 退出后，所有子 goroutine 会被强制终止
	go func() {
		time.Sleep(time.Second) // 模拟耗时操作
		fmt.Println("这行永远不会被打印！")
	}()

	fmt.Println("主 goroutine 即将退出")
	// 主 goroutine 退出，程序结束，子 goroutine 还没来得及执行
}
```

### 6.2 正确的等待方式

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	// 方式1: sync.WaitGroup（最常用）
	fmt.Println("=== 方式1: sync.WaitGroup ===")
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1) // 在启动 goroutine 之前 Add
		go func(id int) {
			defer wg.Done() // goroutine 结束时 Done
			time.Sleep(100 * time.Millisecond)
			fmt.Printf("Worker %d 完成\n", id)
		}(i)
	}
	wg.Wait() // 等待所有 goroutine 完成
	fmt.Println("所有 worker 已完成\n")

	// 方式2: channel 同步
	fmt.Println("=== 方式2: channel 同步 ===")
	done := make(chan struct{}) // 使用空结构体节省内存

	go func() {
		defer close(done)
		time.Sleep(100 * time.Millisecond)
		fmt.Println("工作完成")
	}()

	<-done // 阻塞等待 channel 关闭
	fmt.Println("收到完成信号\n")

	// 方式3: context 控制（适合取消和超时）
	fmt.Println("=== 方式3: context 控制 ===")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	go func(ctx context.Context) {
		for {
			select {
			case <-ctx.Done():
				fmt.Printf("收到取消信号: %v\n", ctx.Err())
				return
			default:
				fmt.Println("工作中...")
				time.Sleep(500 * time.Millisecond)
			}
		}
	}(ctx)

	time.Sleep(1500 * time.Millisecond)
	cancel() // 手动取消（也可以等超时自动取消）
	time.Sleep(100 * time.Millisecond)
	fmt.Println("程序结束")
}
```

### 6.3 优雅退出模式（SRE 实战必备）

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// Worker 模拟一个后台服务
type Worker struct {
	id   int
	quit chan struct{}
	wg   *sync.WaitGroup
}

func NewWorker(id int, wg *sync.WaitGroup) *Worker {
	return &Worker{
		id:   id,
		quit: make(chan struct{}),
		wg:   wg,
	}
}

func (w *Worker) Start() {
	w.wg.Add(1)
	go func() {
		defer w.wg.Done()
		ticker := time.NewTicker(500 * time.Millisecond)
		defer ticker.Stop()

		for {
			select {
			case <-w.quit:
				fmt.Printf("[Worker %d] 收到退出信号，正在清理...\n", w.id)
				time.Sleep(200 * time.Millisecond) // 模拟清理工作
				fmt.Printf("[Worker %d] 清理完成，退出\n", w.id)
				return
			case <-ticker.C:
				fmt.Printf("[Worker %d] 心跳 - %s\n", w.id, time.Now().Format("15:04:05"))
			}
		}
	}()
}

func (w *Worker) Stop() {
	close(w.quit)
}

func main() {
	fmt.Println("=== 优雅退出模式演示 ===")
	fmt.Println("按 Ctrl+C 发送终止信号")

	// 创建可取消的 context
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	var wg sync.WaitGroup

	// 启动多个 worker
	workers := make([]*Worker, 3)
	for i := 0; i < 3; i++ {
		workers[i] = NewWorker(i, &wg)
		workers[i].Start()
	}

	// 监听系统信号
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

	// 等待信号或 context 取消
	select {
	case sig := <-sigCh:
		fmt.Printf("\n收到信号: %v，开始优雅退出...\n", sig)
	case <-ctx.Done():
		fmt.Println("Context 已取消，开始退出...")
	}

	// 停止所有 worker
	for _, w := range workers {
		w.Stop()
	}

	// 等待所有 worker 完成，但设置超时
	doneCh := make(chan struct{})
	go func() {
		wg.Wait()
		close(doneCh)
	}()

	select {
	case <-doneCh:
		fmt.Println("所有 worker 已优雅退出")
	case <-time.After(5 * time.Second):
		fmt.Println("等待超时，强制退出")
	}
}
```

---

## 7. Goroutine 与 Python 并发方案对比

### 7.1 对照表

| 特性 | Go goroutine | Python threading | Python asyncio |
|------|-------------|------------------|----------------|
| 并发模型 | CSP | 共享内存 + 锁 | 事件循环 + 协程 |
| 创建方式 | `go func()` | `Thread(target=func)` | `asyncio.create_task()` |
| 调度方式 | 运行时调度（M:N） | OS 调度（1:1） | 事件循环调度 |
| 真正并行 | 是（多核） | 否（GIL 限制） | 否（单线程） |
| 适用场景 | CPU 密集 + I/O 密集 | I/O 密集 | I/O 密集 |
| 通信方式 | channel | 共享变量 + 锁 | asyncio.Queue |
| 数量级 | 百万级 | 千级 | 万级 |
| 内存开销 | ~2 KB/goroutine | ~8 MB/thread | ~数 KB/coroutine |
| 取消机制 | context | Event / 自定义标记 | task.cancel() |

### 7.2 等效代码对比

**场景：并发请求 5 个 URL**

Go 版本：
```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"sync"
	"time"
)

func fetchURL(ctx context.Context, url string) (int, error) {
	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return 0, err
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return 0, err
	}
	defer resp.Body.Close()
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return 0, err
	}
	return len(body), nil
}

func main() {
	urls := []string{
		"https://httpbin.org/get",
		"https://httpbin.org/ip",
		"https://httpbin.org/headers",
		"https://httpbin.org/user-agent",
		"https://httpbin.org/delay/1",
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	var wg sync.WaitGroup
	results := make([]string, len(urls))

	for i, url := range urls {
		wg.Add(1)
		go func(idx int, u string) {
			defer wg.Done()
			start := time.Now()
			size, err := fetchURL(ctx, u)
			elapsed := time.Since(start)

			if err != nil {
				results[idx] = fmt.Sprintf("[%d] %s -> 错误: %v (%v)", idx, u, err, elapsed)
			} else {
				results[idx] = fmt.Sprintf("[%d] %s -> %d bytes (%v)", idx, u, size, elapsed)
			}
		}(i, url)
	}

	wg.Wait()

	for _, r := range results {
		fmt.Println(r)
	}
}
```

等效的 Python asyncio 版本（对比参考）：
```python
# Python 等效代码（仅供对比，不可运行于 Go 环境）
# import asyncio
# import aiohttp
#
# async def fetch_url(session, url):
#     async with session.get(url) as resp:
#         return len(await resp.read())
#
# async def main():
#     urls = [
#         "https://httpbin.org/get",
#         "https://httpbin.org/ip",
#         "https://httpbin.org/headers",
#         "https://httpbin.org/user-agent",
#         "https://httpbin.org/delay/1",
#     ]
#     async with aiohttp.ClientSession() as session:
#         tasks = [fetch_url(session, url) for url in urls]
#         results = await asyncio.gather(*tasks)
#         for url, size in zip(urls, results):
#             print(f"{url} -> {size} bytes")
#
# asyncio.run(main())
```

### 7.3 关键差异总结

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Go goroutine vs Python 并发方案关键差异 ===")
	fmt.Println()

	differences := []struct {
		aspect string
		golang string
		python string
	}{
		{
			"真正并行",
			"goroutine 可以在多核上真正并行执行",
			"threading 受 GIL 限制无法并行; asyncio 是单线程",
		},
		{
			"CPU 密集任务",
			"直接用 goroutine，天然支持多核并行",
			"需要用 multiprocessing 多进程才能利用多核",
		},
		{
			"I/O 密集任务",
			"goroutine + channel，M:N 调度自动处理阻塞",
			"asyncio 性能好，但需要 async/await 生态",
		},
		{
			"通信方式",
			"推荐 channel（CSP 模型）",
			"threading 用 Queue/Lock; asyncio 用 asyncio.Queue",
		},
		{
			"错误处理",
			"需要手动传递 error（通过 channel 或 errgroup）",
			"asyncio.gather 可以自动收集异常",
		},
		{
			"内存效率",
			"goroutine 栈初始 2KB，可动态增长",
			"线程栈 8MB 固定分配",
		},
	}

	for _, d := range differences {
		fmt.Printf("【%s】\n", d.aspect)
		fmt.Printf("  Go:     %s\n", d.golang)
		fmt.Printf("  Python: %s\n\n", d.python)
	}
}
```

---

## 8. SRE 实战场景

### 8.1 并发健康检查

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// HealthStatus 健康检查结果
type HealthStatus struct {
	Host    string
	Healthy bool
	Latency time.Duration
	Error   error
}

// checkHealth 检查单个主机健康状态
func checkHealth(ctx context.Context, host string) HealthStatus {
	start := time.Now()

	// 模拟网络请求（实际生产中用 http.Client）
	delay := time.Duration(rand.Intn(500)) * time.Millisecond

	select {
	case <-ctx.Done():
		return HealthStatus{
			Host:    host,
			Healthy: false,
			Latency: time.Since(start),
			Error:   ctx.Err(),
		}
	case <-time.After(delay):
		// 模拟随机失败
		healthy := rand.Float64() > 0.2
		var err error
		if !healthy {
			err = fmt.Errorf("连接被拒绝")
		}
		return HealthStatus{
			Host:    host,
			Healthy: healthy,
			Latency: time.Since(start),
			Error:   err,
		}
	}
}

// batchHealthCheck 批量并发健康检查
func batchHealthCheck(hosts []string, timeout time.Duration) []HealthStatus {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	results := make([]HealthStatus, len(hosts))
	var wg sync.WaitGroup

	for i, host := range hosts {
		wg.Add(1)
		go func(idx int, h string) {
			defer wg.Done()
			results[idx] = checkHealth(ctx, h)
		}(i, host)
	}

	wg.Wait()
	return results
}

func main() {
	hosts := []string{
		"web-01.prod.internal:8080",
		"web-02.prod.internal:8080",
		"web-03.prod.internal:8080",
		"api-01.prod.internal:9090",
		"api-02.prod.internal:9090",
		"db-01.prod.internal:5432",
		"db-02.prod.internal:5432",
		"cache-01.prod.internal:6379",
		"cache-02.prod.internal:6379",
		"mq-01.prod.internal:5672",
	}

	fmt.Printf("并发检查 %d 台主机的健康状态...\n\n", len(hosts))
	start := time.Now()

	results := batchHealthCheck(hosts, 3*time.Second)

	fmt.Printf("%-35s %-8s %-10s %s\n", "HOST", "STATUS", "LATENCY", "ERROR")
	fmt.Println("---------------------------------------------------------------")

	healthyCount := 0
	for _, r := range results {
		status := "UP"
		if !r.Healthy {
			status = "DOWN"
		} else {
			healthyCount++
		}

		errMsg := "-"
		if r.Error != nil {
			errMsg = r.Error.Error()
		}

		fmt.Printf("%-35s %-8s %-10v %s\n", r.Host, status, r.Latency.Round(time.Millisecond), errMsg)
	}

	fmt.Printf("\n总计: %d/%d 健康, 耗时: %v\n", healthyCount, len(hosts), time.Since(start).Round(time.Millisecond))
}
```

### 8.2 并发 SSH 命令执行

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// SSHResult SSH 执行结果
type SSHResult struct {
	Host   string
	Output string
	Error  error
}

// executeSSH 模拟 SSH 命令执行
func executeSSH(ctx context.Context, host, command string) SSHResult {
	// 实际生产中使用 golang.org/x/crypto/ssh 包
	delay := time.Duration(100+rand.Intn(900)) * time.Millisecond

	select {
	case <-ctx.Done():
		return SSHResult{Host: host, Error: ctx.Err()}
	case <-time.After(delay):
		if rand.Float64() > 0.1 { // 90% 成功率
			return SSHResult{
				Host:   host,
				Output: fmt.Sprintf("uptime: %d days, load: %.2f", rand.Intn(365), rand.Float64()*4),
			}
		}
		return SSHResult{Host: host, Error: fmt.Errorf("ssh: connection timeout")}
	}
}

// parallelSSH 并发 SSH 执行，带并发数控制
func parallelSSH(hosts []string, command string, maxConcurrency int, timeout time.Duration) []SSHResult {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	results := make([]SSHResult, len(hosts))
	var wg sync.WaitGroup

	// 使用带缓冲的 channel 作为信号量，控制并发数
	semaphore := make(chan struct{}, maxConcurrency)

	for i, host := range hosts {
		wg.Add(1)
		go func(idx int, h string) {
			defer wg.Done()

			semaphore <- struct{}{}        // 获取信号量（可能阻塞）
			defer func() { <-semaphore }() // 释放信号量

			results[idx] = executeSSH(ctx, h, command)
		}(i, host)
	}

	wg.Wait()
	return results
}

func main() {
	hosts := make([]string, 20)
	for i := range hosts {
		hosts[i] = fmt.Sprintf("server-%02d.prod.internal", i+1)
	}

	fmt.Printf("在 %d 台服务器上执行命令 (最大并发: 5)\n", len(hosts))
	fmt.Println("命令: uptime")
	fmt.Println()

	start := time.Now()
	results := parallelSSH(hosts, "uptime", 5, 10*time.Second)

	successCount := 0
	for _, r := range results {
		if r.Error == nil {
			fmt.Printf("[OK]   %-30s %s\n", r.Host, r.Output)
			successCount++
		} else {
			fmt.Printf("[FAIL] %-30s %s\n", r.Host, r.Error)
		}
	}

	fmt.Printf("\n成功: %d/%d, 耗时: %v\n", successCount, len(hosts), time.Since(start).Round(time.Millisecond))
}
```

---

## 9. 常见坑点

### 9.1 循环变量捕获问题

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 注意：Go 1.22+ 已修复循环变量问题，每次迭代创建新变量
	// 但理解这个问题仍然很重要，因为你可能维护旧代码

	// 旧版本 Go（< 1.22）的问题代码:
	fmt.Println("=== 潜在问题（Go < 1.22）===")
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		// 在 Go < 1.22 中，所有 goroutine 共享同一个 i 变量
		// 可能全部打印 "5"
		go func() {
			defer wg.Done()
			fmt.Printf("i = %d\n", i)
		}()
	}
	wg.Wait()

	// 安全写法（兼容所有版本）:
	fmt.Println("\n=== 安全写法 ===")
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(val int) { // 通过参数传递值的拷贝
			defer wg.Done()
			fmt.Printf("val = %d\n", val)
		}(i)
	}
	wg.Wait()
}
```

### 9.2 WaitGroup 使用错误

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 错误1: 在 goroutine 内部 Add（可能来不及 Add 就 Wait 了）
	fmt.Println("=== 错误示范 ===")
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		go func(id int) {
			wg.Add(1) // 错误！应该在 go 之前 Add
			defer wg.Done()
			fmt.Printf("Worker %d\n", id)
		}(i)
	}
	wg.Wait() // 可能在某些 goroutine Add 之前就返回了

	// 正确写法:
	fmt.Println("\n=== 正确写法 ===")
	var wg2 sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg2.Add(1) // 正确：在 go 之前 Add
		go func(id int) {
			defer wg2.Done()
			fmt.Printf("Worker %d\n", id)
		}(i)
	}
	wg2.Wait()

	// 错误2: Add 和 Done 次数不匹配会 panic
	// wg3.Add(1)  // 只 Add 1 次
	// wg3.Done()  // Done 1 次
	// wg3.Done()  // 再 Done 1 次 -> panic: sync: negative WaitGroup counter
}
```

### 9.3 goroutine 泄漏

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

func leakyGoroutine() {
	ch := make(chan int) // 无缓冲 channel
	go func() {
		val := <-ch // 永远阻塞在这里，因为没人往 ch 发送数据
		fmt.Println(val)
	}()
	// 函数返回，ch 不可达，但 goroutine 仍然在等待
	// 这就是 goroutine 泄漏！
}

func safeGoroutine() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	ch := make(chan int)
	go func() {
		select {
		case val := <-ch:
			fmt.Println(val)
		case <-ctx.Done():
			fmt.Println("超时退出，避免泄漏")
			return
		}
	}()
}

func main() {
	fmt.Printf("初始 goroutine 数: %d\n", runtime.NumGoroutine())

	// 制造泄漏
	for i := 0; i < 100; i++ {
		leakyGoroutine()
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("泄漏后 goroutine 数: %d (多了约 100 个)\n", runtime.NumGoroutine())

	// 安全的方式
	for i := 0; i < 100; i++ {
		safeGoroutine()
	}
	time.Sleep(1500 * time.Millisecond) // 等待超时退出
	fmt.Printf("安全退出后 goroutine 数: %d\n", runtime.NumGoroutine())
}
```

---

## 10. 速查表 / 小结

### Goroutine 速查表

```
创建 goroutine:
  go funcName(args)           // 具名函数
  go func() { ... }()        // 匿名函数（无参数）
  go func(x int) { ... }(v)  // 匿名函数（带参数）
  go obj.Method()             // 方法调用

等待 goroutine 完成:
  sync.WaitGroup              // 等待一组 goroutine
  <-done                      // channel 同步
  context.WithCancel/Timeout  // 超时控制

查看运行时信息:
  runtime.NumGoroutine()      // 当前 goroutine 数量
  runtime.NumCPU()            // 逻辑 CPU 数量
  runtime.GOMAXPROCS(n)       // 设置/获取 P 数量
  runtime.Gosched()           // 让出执行权

调试工具:
  GODEBUG=schedtrace=1000     // 打印调度器状态
  go run -race main.go        // 数据竞争检测
  runtime/pprof               // 性能分析

容器环境:
  go.uber.org/automaxprocs    // 自动设置 GOMAXPROCS
```

### 核心要点总结

| 要点 | 说明 |
|------|------|
| goroutine 极其轻量 | 初始栈 2KB，可创建百万个 |
| 主 goroutine 退出 = 程序退出 | 必须等待子 goroutine 完成 |
| 优先使用 WaitGroup | Add 在 go 之前，Done 用 defer |
| 容器中注意 GOMAXPROCS | 使用 automaxprocs 库自动适配 |
| 循环变量要注意 | Go 1.22+ 已修复，旧版本用参数传递 |
| 防止 goroutine 泄漏 | 使用 context 超时/取消机制 |
| GMP 模型是核心 | G-goroutine, M-thread, P-processor |
| 抢占式调度 | Go 1.14+ 基于信号的异步抢占 |

### 从 Python 迁移的要点

```
Python threading.Thread  ->  go func()
Python threading.Event   ->  channel / context
Python queue.Queue       ->  channel
Python GIL 限制          ->  Go 无此限制，天然多核并行
Python asyncio           ->  goroutine（更简洁，无需 async/await）
Python ProcessPoolExec.  ->  直接用 goroutine（已经是多核并行）
```

---

> **下一篇**：[02-Channel.md](./02-Channel.md) - 深入学习 Go 并发编程的核心通信机制
