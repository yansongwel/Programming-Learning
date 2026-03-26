# 02 - Channel

## 导读

Channel 是 Go 并发编程的核心通信机制，也是 Go 语言最具特色的设计之一。Go 的并发哲学来自 CSP（Communicating Sequential Processes）模型：

> **Don't communicate by sharing memory; share memory by communicating.**
> 不要通过共享内存来通信，而要通过通信来共享内存。

在传统的多线程编程中（如 Java、Python），线程之间通过共享变量 + 锁来通信。而 Go 推荐使用 channel 在 goroutine 之间传递数据，从根本上避免了许多并发问题。

作为 SRE/DevOps 工程师，你会在以下场景使用 channel：
- goroutine 之间传递任务和结果
- 实现超时控制和优雅退出
- 构建 Pipeline 数据处理流水线
- 实现发布/订阅模式
- 控制并发数量（信号量模式）

本篇将全面讲解 channel 的各种用法、特性和陷阱。

---

## 1. Channel 基本概念

### 1.1 什么是 Channel

Channel 是一个类型化的管道（typed conduit），用于在 goroutine 之间发送和接收值。

```go
package main

import "fmt"

func main() {
	// 创建一个传递 int 类型的 channel
	ch := make(chan int)

	// 在新的 goroutine 中发送数据
	go func() {
		ch <- 42 // 发送值到 channel
	}()

	// 在主 goroutine 中接收数据
	value := <-ch // 从 channel 接收值
	fmt.Println("收到:", value)
}
```

### 1.2 Channel 的三种操作

```go
package main

import "fmt"

func main() {
	ch := make(chan string, 1) // 创建带缓冲的 channel

	// 操作1: 发送（send）
	ch <- "hello" // 将值发送到 channel

	// 操作2: 接收（receive）
	msg := <-ch // 从 channel 接收值并赋给变量
	fmt.Println(msg)

	// 操作3: 关闭（close）
	close(ch) // 关闭 channel，不能再发送，但可以继续接收

	// 接收时可以检查 channel 是否已关闭
	value, ok := <-ch
	fmt.Printf("value=%q, ok=%v (ok=false 表示 channel 已关闭且无数据)\n", value, ok)
}
```

### 1.3 Channel 的零值是 nil

```go
package main

import "fmt"

func main() {
	// channel 的零值是 nil
	var ch chan int
	fmt.Printf("ch = %v, ch == nil: %v\n", ch, ch == nil)

	// 对 nil channel 的操作：
	// - 发送：永久阻塞
	// - 接收：永久阻塞
	// - 关闭：panic

	// 必须使用 make 初始化后才能使用
	ch = make(chan int)
	fmt.Printf("初始化后 ch = %v, ch == nil: %v\n", ch, ch == nil)
}
```

---

## 2. 无缓冲 Channel（同步通信）

无缓冲 channel 在发送和接收之间建立了一个同步点：发送方必须等待接收方准备好，反之亦然。

### 2.1 基本用法

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// 无缓冲 channel：make(chan Type)
	ch := make(chan string)

	go func() {
		fmt.Println("发送方：准备发送...")
		time.Sleep(time.Second) // 模拟一些工作
		ch <- "你好"             // 如果没人接收，会阻塞在这里
		fmt.Println("发送方：发送完成")
	}()

	fmt.Println("接收方：等待接收...")
	msg := <-ch // 如果没人发送，会阻塞在这里
	fmt.Println("接收方：收到 -", msg)
}
```

### 2.2 无缓冲 channel 的同步特性

```go
package main

import (
	"fmt"
	"time"
)

// 无缓冲 channel 可以用作同步信号
func main() {
	done := make(chan struct{}) // 空结构体不占内存，常用作信号

	// 场景：确保一个操作在另一个之后执行
	go func() {
		fmt.Println("步骤1: 初始化数据库连接")
		time.Sleep(500 * time.Millisecond)
		done <- struct{}{} // 发送信号：初始化完成
	}()

	<-done // 等待初始化完成
	fmt.Println("步骤2: 开始处理请求（确保在初始化之后）")

	// 场景：两个 goroutine 之间的握手（handshake）
	ping := make(chan string)
	pong := make(chan string)

	// Player 1
	go func() {
		for i := 0; i < 3; i++ {
			msg := <-ping
			fmt.Printf("Player 1 收到: %s\n", msg)
			time.Sleep(200 * time.Millisecond)
			pong <- fmt.Sprintf("pong-%d", i)
		}
	}()

	// Player 2（主 goroutine）
	for i := 0; i < 3; i++ {
		ping <- fmt.Sprintf("ping-%d", i)
		msg := <-pong
		fmt.Printf("Player 2 收到: %s\n", msg)
	}
}
```

### 2.3 无缓冲 channel 的阻塞行为

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)

	// 发送方
	go func() {
		for i := 1; i <= 5; i++ {
			fmt.Printf("[%s] 准备发送 %d\n", time.Now().Format("15:04:05.000"), i)
			ch <- i // 无缓冲 channel：必须等接收方就绪
			fmt.Printf("[%s] 已发送 %d\n", time.Now().Format("15:04:05.000"), i)
		}
		close(ch)
	}()

	// 接收方（故意慢一些）
	for v := range ch {
		fmt.Printf("[%s] 收到 %d，处理中...\n", time.Now().Format("15:04:05.000"), v)
		time.Sleep(500 * time.Millisecond) // 模拟慢处理
	}
}
```

---

## 3. 有缓冲 Channel（异步通信）

有缓冲 channel 允许在缓冲区未满时发送不阻塞，缓冲区非空时接收不阻塞。

### 3.1 基本用法

```go
package main

import "fmt"

func main() {
	// 有缓冲 channel：make(chan Type, capacity)
	ch := make(chan int, 3) // 缓冲区大小为 3

	// 发送不会阻塞（缓冲区未满）
	ch <- 1
	ch <- 2
	ch <- 3
	// ch <- 4 // 这里会阻塞，因为缓冲区已满

	fmt.Printf("channel 长度: %d, 容量: %d\n", len(ch), cap(ch))

	// 接收
	fmt.Println(<-ch) // 1
	fmt.Println(<-ch) // 2
	fmt.Println(<-ch) // 3
}
```

### 3.2 缓冲 channel 的行为

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// 缓冲区大小为 2
	ch := make(chan string, 2)

	go func() {
		messages := []string{"A", "B", "C", "D", "E"}
		for _, msg := range messages {
			fmt.Printf("[%s] 发送 %s (缓冲: %d/%d)\n",
				time.Now().Format("15:04:05.000"), msg, len(ch), cap(ch))
			ch <- msg
			fmt.Printf("[%s] 已发送 %s\n", time.Now().Format("15:04:05.000"), msg)
		}
		close(ch)
	}()

	// 慢速消费者
	for msg := range ch {
		fmt.Printf("[%s] 处理 %s\n", time.Now().Format("15:04:05.000"), msg)
		time.Sleep(500 * time.Millisecond)
	}
}
```

### 3.3 无缓冲 vs 有缓冲选择指南

```go
package main

import "fmt"

func main() {
	fmt.Println("=== 无缓冲 vs 有缓冲 Channel 选择指南 ===")
	fmt.Println()

	guidelines := []struct {
		scenario    string
		choice      string
		reason      string
	}{
		{"goroutine 间同步信号", "无缓冲", "需要确保接收方收到后才继续"},
		{"生产者-消费者模式", "有缓冲", "解耦生产和消费的速度差"},
		{"限制并发数（信号量）", "有缓冲", "缓冲区大小 = 最大并发数"},
		{"请求-响应模式", "无缓冲或缓冲1", "确保结果被接收"},
		{"事件通知", "缓冲1", "不阻塞发送方"},
		{"Worker Pool", "有缓冲", "任务队列需要缓冲"},
	}

	for _, g := range guidelines {
		fmt.Printf("场景: %-20s -> %-10s | 原因: %s\n", g.scenario, g.choice, g.reason)
	}
}
```

---

## 4. 关闭 Channel

### 4.1 close 的基本用法

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 5)

	// 发送一些值
	for i := 1; i <= 3; i++ {
		ch <- i
	}

	// 关闭 channel
	close(ch)

	// 关闭后仍然可以接收已发送的值
	fmt.Println(<-ch) // 1
	fmt.Println(<-ch) // 2
	fmt.Println(<-ch) // 3

	// 关闭后，接收操作返回零值和 false
	val, ok := <-ch
	fmt.Printf("val=%d, ok=%v\n", val, ok) // val=0, ok=false

	// 再次接收仍然是零值
	val, ok = <-ch
	fmt.Printf("val=%d, ok=%v\n", val, ok) // val=0, ok=false
}
```

### 4.2 关闭 channel 的规则

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Channel 关闭规则 ===")
	fmt.Println()
	fmt.Println("1. 只有发送方应该关闭 channel，接收方不应关闭")
	fmt.Println("2. 向已关闭的 channel 发送数据会 panic")
	fmt.Println("3. 重复关闭 channel 会 panic")
	fmt.Println("4. 从已关闭的 channel 接收数据不会 panic，返回零值")
	fmt.Println("5. 关闭 nil channel 会 panic")
	fmt.Println()

	// 演示规则4：安全地从已关闭的 channel 接收
	ch := make(chan string, 2)
	ch <- "hello"
	ch <- "world"
	close(ch)

	// 使用 for-range 自动检测关闭
	for msg := range ch {
		fmt.Println("收到:", msg)
	}
	fmt.Println("channel 已关闭，for-range 自动退出")

	// 演示 panic 情况（用 recover 捕获）
	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("捕获到 panic: %v\n", r)
			}
		}()
		ch2 := make(chan int)
		close(ch2)
		close(ch2) // panic: close of closed channel
	}()

	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("捕获到 panic: %v\n", r)
			}
		}()
		ch3 := make(chan int)
		close(ch3)
		ch3 <- 1 // panic: send on closed channel
	}()
}
```

### 4.3 安全关闭 channel 的模式

```go
package main

import (
	"fmt"
	"sync"
)

// 模式1: 单发送者 - 直接关闭
func singleSender() {
	ch := make(chan int)

	// 只有一个发送者，由发送者关闭
	go func() {
		defer close(ch) // 发送完毕后关闭
		for i := 0; i < 5; i++ {
			ch <- i
		}
	}()

	for v := range ch {
		fmt.Printf("单发送者模式: %d\n", v)
	}
}

// 模式2: 多发送者 - 使用 sync.WaitGroup + 单独 goroutine 关闭
func multipleSenders() {
	ch := make(chan int, 10)
	var wg sync.WaitGroup

	// 多个发送者
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 3; j++ {
				ch <- id*10 + j
			}
		}(i)
	}

	// 单独的 goroutine 等待所有发送者完成后关闭 channel
	go func() {
		wg.Wait()
		close(ch) // 所有发送者完成后关闭
	}()

	for v := range ch {
		fmt.Printf("多发送者模式: %d\n", v)
	}
}

// 模式3: 使用 done channel 通知发送者停止
func stopWithDone() {
	ch := make(chan int)
	done := make(chan struct{})

	// 发送者：监听 done 信号
	go func() {
		defer close(ch)
		i := 0
		for {
			select {
			case <-done:
				fmt.Println("发送者收到停止信号")
				return
			case ch <- i:
				i++
			}
		}
	}()

	// 接收前 5 个值后通知停止
	for i := 0; i < 5; i++ {
		fmt.Printf("done 模式: %d\n", <-ch)
	}
	close(done) // 通知发送者停止
}

func main() {
	fmt.Println("=== 模式1: 单发送者 ===")
	singleSender()

	fmt.Println("\n=== 模式2: 多发送者 ===")
	multipleSenders()

	fmt.Println("\n=== 模式3: done channel 停止 ===")
	stopWithDone()
}
```

---

## 5. Range 遍历 Channel

### 5.1 基本用法

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	go func() {
		for i := 1; i <= 5; i++ {
			ch <- i * i
		}
		close(ch) // 必须关闭，否则 range 会永远阻塞
	}()

	// range 会持续接收，直到 channel 被关闭
	for value := range ch {
		fmt.Printf("收到平方数: %d\n", value)
	}
	fmt.Println("channel 已关闭，循环结束")
}
```

### 5.2 生成器模式

```go
package main

import "fmt"

// fibonacci 生成斐波那契数列
func fibonacci(n int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		a, b := 0, 1
		for i := 0; i < n; i++ {
			ch <- a
			a, b = b, a+b
		}
	}()
	return ch
}

// primes 生成素数序列（埃拉托斯特尼筛法风格）
func primes(limit int) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		for n := 2; n <= limit; n++ {
			isPrime := true
			for i := 2; i*i <= n; i++ {
				if n%i == 0 {
					isPrime = false
					break
				}
			}
			if isPrime {
				ch <- n
			}
		}
	}()
	return ch
}

func main() {
	// 遍历斐波那契数列
	fmt.Println("斐波那契数列（前 10 个）：")
	for v := range fibonacci(10) {
		fmt.Printf("%d ", v)
	}
	fmt.Println()

	// 遍历素数
	fmt.Println("\n100 以内的素数：")
	for p := range primes(100) {
		fmt.Printf("%d ", p)
	}
	fmt.Println()
}
```

---

## 6. Select 多路复用

### 6.1 基本用法

select 类似于 switch，但专门用于 channel 操作：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go func() {
		time.Sleep(100 * time.Millisecond)
		ch1 <- "来自 channel 1"
	}()

	go func() {
		time.Sleep(200 * time.Millisecond)
		ch2 <- "来自 channel 2"
	}()

	// select 等待多个 channel 操作
	// 哪个先就绪就执行哪个
	for i := 0; i < 2; i++ {
		select {
		case msg := <-ch1:
			fmt.Println("收到:", msg)
		case msg := <-ch2:
			fmt.Println("收到:", msg)
		}
	}
}
```

### 6.2 select 的特性

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	// 特性1: 多个 case 同时就绪时，随机选择一个
	fmt.Println("=== 特性1: 随机选择 ===")
	ch1 := make(chan string, 1)
	ch2 := make(chan string, 1)

	counts := map[string]int{"ch1": 0, "ch2": 0}
	for i := 0; i < 1000; i++ {
		ch1 <- "a"
		ch2 <- "b"
		select {
		case <-ch1:
			counts["ch1"]++
		case <-ch2:
			counts["ch2"]++
		}
	}
	fmt.Printf("ch1 被选中 %d 次, ch2 被选中 %d 次（大致 50/50）\n", counts["ch1"], counts["ch2"])

	// 特性2: default 分支使 select 非阻塞
	fmt.Println("\n=== 特性2: 非阻塞 select ===")
	ch := make(chan int)
	select {
	case v := <-ch:
		fmt.Println("收到:", v)
	default:
		fmt.Println("没有数据可接收，不阻塞，直接执行 default")
	}

	// 特性3: 空 select 永久阻塞
	// select {} // 永久阻塞当前 goroutine

	// 特性4: nil channel 在 select 中被忽略
	fmt.Println("\n=== 特性4: nil channel ===")
	var nilCh chan int // nil channel
	normalCh := make(chan int, 1)
	normalCh <- 42

	select {
	case v := <-nilCh:
		fmt.Println("从 nil channel 收到:", v) // 永远不会执行
	case v := <-normalCh:
		fmt.Println("从 normal channel 收到:", v) // 会执行
	}

	_ = rand.Int() // 使用 rand 包
}
```

### 6.3 超时控制

```go
package main

import (
	"fmt"
	"time"
)

// 模拟一个可能很慢的操作
func slowOperation() <-chan string {
	ch := make(chan string)
	go func() {
		time.Sleep(2 * time.Second) // 模拟慢操作
		ch <- "操作完成"
	}()
	return ch
}

func fastOperation() <-chan string {
	ch := make(chan string)
	go func() {
		time.Sleep(100 * time.Millisecond) // 模拟快操作
		ch <- "操作完成"
	}()
	return ch
}

func main() {
	// 方式1: time.After 超时
	fmt.Println("=== 方式1: time.After ===")
	select {
	case result := <-slowOperation():
		fmt.Println("结果:", result)
	case <-time.After(1 * time.Second):
		fmt.Println("操作超时（1秒）")
	}

	// 方式2: time.After 未超时
	fmt.Println("\n=== 快操作不超时 ===")
	select {
	case result := <-fastOperation():
		fmt.Println("结果:", result)
	case <-time.After(1 * time.Second):
		fmt.Println("操作超时")
	}

	// 方式3: 带超时的轮询
	fmt.Println("\n=== 带超时的轮询 ===")
	dataCh := make(chan int)
	go func() {
		time.Sleep(500 * time.Millisecond)
		dataCh <- 42
	}()

	timeout := time.After(2 * time.Second)
	ticker := time.NewTicker(200 * time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case data := <-dataCh:
			fmt.Printf("收到数据: %d\n", data)
			goto done
		case <-ticker.C:
			fmt.Println("等待中...")
		case <-timeout:
			fmt.Println("超时")
			goto done
		}
	}
done:
	fmt.Println("完成")
}
```

### 6.4 定时器与 Ticker

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// time.Timer - 一次性定时器
	fmt.Println("=== Timer ===")
	timer := time.NewTimer(500 * time.Millisecond)
	fmt.Println("等待定时器触发...")
	<-timer.C
	fmt.Println("定时器触发！")

	// 重置定时器
	timer.Reset(300 * time.Millisecond)
	<-timer.C
	fmt.Println("重置后的定时器触发！")

	// 停止定时器
	timer2 := time.NewTimer(time.Hour)
	if timer2.Stop() {
		fmt.Println("定时器已停止（未触发）")
	}

	// time.Ticker - 周期性定时器
	fmt.Println("\n=== Ticker ===")
	ticker := time.NewTicker(200 * time.Millisecond)
	defer ticker.Stop() // 重要：不用时必须 Stop，否则会泄漏

	count := 0
	for t := range ticker.C {
		fmt.Printf("Tick at %s\n", t.Format("15:04:05.000"))
		count++
		if count >= 5 {
			break
		}
	}

	// SRE 场景：周期性健康检查 + 超时退出
	fmt.Println("\n=== 周期性健康检查 ===")
	healthTicker := time.NewTicker(300 * time.Millisecond)
	defer healthTicker.Stop()
	deadline := time.After(1500 * time.Millisecond)

	for {
		select {
		case <-healthTicker.C:
			fmt.Printf("[%s] 执行健康检查...\n", time.Now().Format("15:04:05.000"))
		case <-deadline:
			fmt.Println("检查周期结束")
			return
		}
	}
}
```

---

## 7. 单向 Channel

### 7.1 只读和只写 channel

```go
package main

import "fmt"

// 只能发送的 channel：chan<- Type
// 只能接收的 channel：<-chan Type

// producer 只能向 channel 发送数据
func producer(out chan<- int) {
	for i := 0; i < 5; i++ {
		out <- i
	}
	close(out)
}

// consumer 只能从 channel 接收数据
func consumer(in <-chan int) {
	for v := range in {
		fmt.Printf("消费: %d\n", v)
	}
}

// transformer 从一个 channel 读取，处理后发送到另一个
func transformer(in <-chan int, out chan<- string) {
	for v := range in {
		out <- fmt.Sprintf("值=%d, 平方=%d", v, v*v)
	}
	close(out)
}

func main() {
	// 双向 channel 可以隐式转换为单向 channel
	ch := make(chan int, 5)
	strCh := make(chan string, 5)

	go producer(ch)      // chan int -> chan<- int (自动转换)
	go transformer(ch, strCh) // chan int -> <-chan int

	// 注意：单向 channel 不能转回双向 channel
	for msg := range strCh {
		fmt.Println(msg)
	}
}
```

### 7.2 单向 channel 的设计价值

```go
package main

import (
	"fmt"
	"time"
)

// EventBus 事件总线
type EventBus struct {
	subscribers map[string][]chan<- string
}

func NewEventBus() *EventBus {
	return &EventBus{
		subscribers: make(map[string][]chan<- string),
	}
}

// Subscribe 返回一个只读 channel，调用者只能接收事件
func (eb *EventBus) Subscribe(topic string) <-chan string {
	ch := make(chan string, 10) // 有缓冲，避免阻塞发布者
	eb.subscribers[topic] = append(eb.subscribers[topic], ch)
	return ch // chan string 自动转为 <-chan string
}

// Publish 向所有订阅者发送事件
func (eb *EventBus) Publish(topic, message string) {
	for _, ch := range eb.subscribers[topic] {
		select {
		case ch <- message:
		default:
			fmt.Printf("警告: 订阅者缓冲区已满，丢弃消息\n")
		}
	}
}

func main() {
	bus := NewEventBus()

	// 订阅者1
	alertCh := bus.Subscribe("alert")
	go func() {
		for msg := range alertCh {
			fmt.Printf("[Alert订阅者] %s\n", msg)
		}
	}()

	// 订阅者2
	alertCh2 := bus.Subscribe("alert")
	go func() {
		for msg := range alertCh2 {
			fmt.Printf("[PagerDuty] %s\n", msg)
		}
	}()

	// 发布事件
	bus.Publish("alert", "CPU 使用率超过 90%")
	bus.Publish("alert", "磁盘空间不足 10%")

	time.Sleep(100 * time.Millisecond)
}
```

---

## 8. nil Channel 的特性

nil channel 有独特的行为，在某些高级模式中非常有用。

### 8.1 nil channel 行为总结

```go
package main

import "fmt"

func main() {
	fmt.Println("=== nil channel 行为 ===")
	fmt.Println()
	fmt.Println("操作          | nil channel | 非 nil 已关闭 | 非 nil 未关闭")
	fmt.Println("发送 ch <- v  | 永久阻塞    | panic        | 阻塞或成功")
	fmt.Println("接收 <- ch    | 永久阻塞    | 零值, false  | 阻塞或成功")
	fmt.Println("关闭 close(ch)| panic       | panic        | 成功")
	fmt.Println("len(ch)       | 0           | 0            | 缓冲中的元素数")
	fmt.Println("cap(ch)       | 0           | 缓冲区大小    | 缓冲区大小")
}
```

### 8.2 利用 nil channel 禁用 select 分支

```go
package main

import (
	"fmt"
	"time"
)

// 合并两个 channel，任一关闭后停止从该 channel 读取
func merge(ch1, ch2 <-chan int) <-chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for ch1 != nil || ch2 != nil {
			select {
			case v, ok := <-ch1:
				if !ok {
					ch1 = nil // 设为 nil，禁用这个 case
					fmt.Println("ch1 已关闭，禁用")
					continue
				}
				out <- v
			case v, ok := <-ch2:
				if !ok {
					ch2 = nil // 设为 nil，禁用这个 case
					fmt.Println("ch2 已关闭，禁用")
					continue
				}
				out <- v
			}
		}
		fmt.Println("两个 channel 都已关闭")
	}()

	return out
}

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	// ch1 发送 3 个值后关闭
	go func() {
		for i := 1; i <= 3; i++ {
			ch1 <- i * 10
			time.Sleep(100 * time.Millisecond)
		}
		close(ch1)
	}()

	// ch2 发送 5 个值后关闭
	go func() {
		for i := 1; i <= 5; i++ {
			ch2 <- i * 100
			time.Sleep(70 * time.Millisecond)
		}
		close(ch2)
	}()

	// 合并两个 channel
	for v := range merge(ch1, ch2) {
		fmt.Printf("收到: %d\n", v)
	}
	fmt.Println("全部完成")
}
```

---

## 9. SRE 实战场景

### 9.1 并发限速器（Semaphore）

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Semaphore 使用带缓冲的 channel 实现信号量
type Semaphore struct {
	ch chan struct{}
}

func NewSemaphore(maxConcurrency int) *Semaphore {
	return &Semaphore{
		ch: make(chan struct{}, maxConcurrency),
	}
}

func (s *Semaphore) Acquire() {
	s.ch <- struct{}{} // 缓冲区满时阻塞
}

func (s *Semaphore) Release() {
	<-s.ch // 释放一个位置
}

// 模拟 API 请求
func callAPI(id int) string {
	time.Sleep(200 * time.Millisecond) // 模拟网络延迟
	return fmt.Sprintf("Response from API call %d", id)
}

func main() {
	sem := NewSemaphore(3) // 最多同时 3 个请求
	var wg sync.WaitGroup

	fmt.Println("发起 10 个 API 请求（最大并发 3）：")
	start := time.Now()

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			sem.Acquire()
			defer sem.Release()

			fmt.Printf("[%s] 开始请求 %d\n", time.Since(start).Round(time.Millisecond), id)
			result := callAPI(id)
			fmt.Printf("[%s] 完成请求 %d: %s\n", time.Since(start).Round(time.Millisecond), id, result)
		}(i)
	}

	wg.Wait()
	fmt.Printf("\n全部完成，总耗时: %v\n", time.Since(start).Round(time.Millisecond))
}
```

### 9.2 超时保护的服务调用

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// ServiceResponse 服务响应
type ServiceResponse struct {
	Service string
	Data    string
	Latency time.Duration
}

// callService 模拟调用一个服务
func callService(name string) <-chan ServiceResponse {
	ch := make(chan ServiceResponse, 1) // 缓冲1，防止 goroutine 泄漏
	go func() {
		latency := time.Duration(rand.Intn(2000)) * time.Millisecond
		time.Sleep(latency)
		ch <- ServiceResponse{
			Service: name,
			Data:    fmt.Sprintf("%s 的数据", name),
			Latency: latency,
		}
	}()
	return ch
}

// callWithTimeout 带超时的服务调用
func callWithTimeout(name string, timeout time.Duration) (ServiceResponse, error) {
	select {
	case resp := <-callService(name):
		return resp, nil
	case <-time.After(timeout):
		return ServiceResponse{}, fmt.Errorf("调用 %s 超时（%v）", name, timeout)
	}
}

// callFirstResponse 获取最快响应（竞速模式）
func callFirstResponse(services []string, timeout time.Duration) (ServiceResponse, error) {
	ch := make(chan ServiceResponse, len(services))

	for _, svc := range services {
		go func(name string) {
			latency := time.Duration(rand.Intn(1000)) * time.Millisecond
			time.Sleep(latency)
			ch <- ServiceResponse{
				Service: name,
				Data:    fmt.Sprintf("%s 的数据", name),
				Latency: latency,
			}
		}(svc)
	}

	select {
	case resp := <-ch:
		return resp, nil
	case <-time.After(timeout):
		return ServiceResponse{}, fmt.Errorf("所有服务超时")
	}
}

func main() {
	// 场景1: 带超时的单个服务调用
	fmt.Println("=== 超时保护 ===")
	resp, err := callWithTimeout("用户服务", 500*time.Millisecond)
	if err != nil {
		fmt.Println("失败:", err)
	} else {
		fmt.Printf("成功: %s (延迟: %v)\n", resp.Data, resp.Latency)
	}

	// 场景2: 竞速模式 - 多个副本，取最快的
	fmt.Println("\n=== 竞速模式（取最快响应）===")
	replicas := []string{"replica-1", "replica-2", "replica-3"}
	resp, err = callFirstResponse(replicas, time.Second)
	if err != nil {
		fmt.Println("失败:", err)
	} else {
		fmt.Printf("最快响应来自 %s (延迟: %v)\n", resp.Service, resp.Latency)
	}

	// 场景3: 聚合多个服务的结果
	fmt.Println("\n=== 聚合多服务结果 ===")
	services := []string{"用户服务", "订单服务", "库存服务"}
	timeout := time.Second
	results := make(map[string]ServiceResponse)
	errors := make(map[string]error)

	// 为每个服务创建一个 channel
	type result struct {
		resp ServiceResponse
		err  error
	}
	channels := make(map[string]<-chan result)

	for _, svc := range services {
		name := svc
		ch := make(chan result, 1)
		channels[name] = ch
		go func() {
			resp, err := callWithTimeout(name, timeout)
			ch <- result{resp, err}
		}()
	}

	// 收集所有结果
	for name, ch := range channels {
		r := <-ch
		if r.err != nil {
			errors[name] = r.err
		} else {
			results[name] = r.resp
		}
	}

	for name, resp := range results {
		fmt.Printf("[OK] %s: %s (延迟: %v)\n", name, resp.Data, resp.Latency)
	}
	for name, err := range errors {
		fmt.Printf("[FAIL] %s: %v\n", name, err)
	}
}
```

### 9.3 日志收集 Pipeline

```go
package main

import (
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"time"
)

// LogEntry 日志条目
type LogEntry struct {
	Timestamp time.Time
	Level     string
	Host      string
	Message   string
}

// 阶段1: 日志生成（模拟从多台服务器收集日志）
func generateLogs(hosts []string, count int) <-chan LogEntry {
	out := make(chan LogEntry, 100)

	var wg sync.WaitGroup
	for _, host := range hosts {
		wg.Add(1)
		go func(h string) {
			defer wg.Done()
			levels := []string{"INFO", "WARN", "ERROR"}
			messages := []string{
				"请求处理完成",
				"连接超时",
				"内存使用率过高",
				"磁盘 I/O 延迟增加",
				"服务启动成功",
				"数据库连接池耗尽",
			}

			for i := 0; i < count; i++ {
				out <- LogEntry{
					Timestamp: time.Now(),
					Level:     levels[rand.Intn(len(levels))],
					Host:      h,
					Message:   messages[rand.Intn(len(messages))],
				}
				time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond)
			}
		}(host)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

// 阶段2: 过滤（只保留 WARN 和 ERROR）
func filterLogs(in <-chan LogEntry) <-chan LogEntry {
	out := make(chan LogEntry, 50)
	go func() {
		defer close(out)
		for entry := range in {
			if entry.Level == "WARN" || entry.Level == "ERROR" {
				out <- entry
			}
		}
	}()
	return out
}

// 阶段3: 格式化
func formatLogs(in <-chan LogEntry) <-chan string {
	out := make(chan string, 50)
	go func() {
		defer close(out)
		for entry := range in {
			formatted := fmt.Sprintf("[%s] [%-5s] [%s] %s",
				entry.Timestamp.Format("15:04:05.000"),
				entry.Level,
				entry.Host,
				entry.Message,
			)
			out <- formatted
		}
	}()
	return out
}

// 阶段4: 输出（模拟写入日志系统）
func outputLogs(in <-chan string) {
	var errorCount, warnCount int
	for msg := range in {
		fmt.Println(msg)
		if strings.Contains(msg, "ERROR") {
			errorCount++
		} else {
			warnCount++
		}
	}
	fmt.Printf("\n--- 统计 ---\n")
	fmt.Printf("ERROR: %d 条, WARN: %d 条\n", errorCount, warnCount)
}

func main() {
	hosts := []string{"web-01", "web-02", "api-01", "db-01"}

	fmt.Printf("从 %d 台主机收集日志（每台 20 条）...\n\n", len(hosts))

	// 构建 Pipeline: 生成 -> 过滤 -> 格式化 -> 输出
	logs := generateLogs(hosts, 20)
	filtered := filterLogs(logs)
	formatted := formatLogs(filtered)
	outputLogs(formatted) // 阻塞直到 pipeline 完成
}
```

---

## 10. 常见坑点

### 10.1 Channel 操作导致的 panic

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Channel Panic 场景 ===")

	// 坑1: 向已关闭的 channel 发送数据
	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("坑1 - 向已关闭 channel 发送: %v\n", r)
			}
		}()
		ch := make(chan int, 1)
		close(ch)
		ch <- 1 // panic!
	}()

	// 坑2: 重复关闭 channel
	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("坑2 - 重复关闭 channel: %v\n", r)
			}
		}()
		ch := make(chan int)
		close(ch)
		close(ch) // panic!
	}()

	// 坑3: 关闭 nil channel
	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("坑3 - 关闭 nil channel: %v\n", r)
			}
		}()
		var ch chan int
		close(ch) // panic!
	}()
}
```

### 10.2 Channel 导致的死锁

```go
package main

import "fmt"

func main() {
	// 死锁场景1: 单个 goroutine 中无缓冲 channel 的发送和接收
	// ch := make(chan int)
	// ch <- 1    // 阻塞！没有其他 goroutine 来接收
	// <-ch       // 永远执行不到

	// 死锁场景2: 两个 goroutine 互相等待
	// ch1 := make(chan int)
	// ch2 := make(chan int)
	// go func() {
	//     <-ch1      // 等待 ch1
	//     ch2 <- 1   // 向 ch2 发送
	// }()
	// <-ch2          // 等待 ch2（但 ch2 要等 ch1 先完成）
	// ch1 <- 1       // 永远执行不到

	// 正确写法
	ch := make(chan int, 1)
	ch <- 1     // 有缓冲，不会阻塞
	val := <-ch // 可以立即接收
	fmt.Println("正确:", val)

	// 或者用不同的 goroutine
	ch2 := make(chan int)
	go func() {
		ch2 <- 42
	}()
	fmt.Println("正确:", <-ch2)
}
```

### 10.3 goroutine 泄漏（channel 相关）

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	fmt.Printf("初始 goroutine 数: %d\n", runtime.NumGoroutine())

	// 泄漏场景: 缓冲 channel 未被读取完
	for i := 0; i < 10; i++ {
		ch := make(chan int, 1)
		go func() {
			ch <- 42
			// 如果没人来读 ch，这个 goroutine 会因为缓冲区满而永远阻塞
			ch <- 43 // 阻塞在这里，goroutine 泄漏
		}()
		<-ch // 只读了一个值
	}

	time.Sleep(100 * time.Millisecond)
	fmt.Printf("泄漏后 goroutine 数: %d（多了 10 个泄漏的）\n", runtime.NumGoroutine())

	// 防范措施: 使用 select + done channel
	done := make(chan struct{})
	ch := make(chan int)
	go func() {
		select {
		case ch <- 42:
		case <-done:
			return // 收到取消信号，退出
		}
	}()
	close(done) // 通知 goroutine 退出
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("使用 done channel 后 goroutine 数: %d\n", runtime.NumGoroutine())
}
```

---

## 11. 速查表 / 小结

### Channel 操作速查表

```
创建:
  ch := make(chan Type)        // 无缓冲
  ch := make(chan Type, size)  // 有缓冲

发送与接收:
  ch <- value                  // 发送
  value := <-ch                // 接收
  value, ok := <-ch            // 接收，ok=false 表示 channel 已关闭

关闭:
  close(ch)                    // 关闭（只有发送方关闭）

遍历:
  for v := range ch { ... }   // 持续接收直到关闭

Select:
  select {
  case v := <-ch1:             // 从 ch1 接收
  case ch2 <- v:               // 向 ch2 发送
  case <-time.After(timeout):  // 超时
  default:                     // 非阻塞
  }

方向限制:
  chan<- Type                  // 只发送
  <-chan Type                  // 只接收

查询:
  len(ch)                      // 缓冲区中的元素数
  cap(ch)                      // 缓冲区容量
```

### Channel 状态与操作结果

```
              | nil      | 非nil空        | 非nil非空      | 非nil满        | 已关闭
-----------+----------+--------------+--------------+--------------+----------
发送 ch<-v | 永久阻塞 | 无缓冲:阻塞   | 发送成功      | 阻塞          | panic
           |          | 有缓冲:发送   |              |              |
接收 <-ch  | 永久阻塞 | 阻塞          | 接收成功      | 接收成功      | 零值+false
关闭       | panic    | 成功          | 成功(可继续读) | 成功(可继续读) | panic
```

### 核心要点

| 要点 | 说明 |
|------|------|
| CSP 模型 | 通过通信共享内存，而非通过共享内存通信 |
| 无缓冲 = 同步 | 发送和接收必须同时就绪 |
| 有缓冲 = 异步 | 缓冲区未满时发送不阻塞 |
| 谁发送谁关闭 | 只有发送方关闭 channel |
| select 多路复用 | 同时等待多个 channel |
| nil channel 阻塞 | 在 select 中用于禁用分支 |
| 防止泄漏 | 使用 done channel 或 context 取消 |
| 缓冲区大小 | 通常从 0 或 1 开始，按需调整 |

---

> **下一篇**：[03-sync包.md](./03-sync包.md) - 学习 Go 标准库中的同步原语
