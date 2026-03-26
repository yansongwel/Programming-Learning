# 04 - time 与 context

> **适用版本**: Go 1.24 | **面向读者**: 有编程经验的 SRE / DevOps 工程师

---

## 导读

时间和超时控制是 SRE 工作的核心主题。监控告警有时间窗口、API 调用需要超时、定时任务需要调度、日志需要时间戳……`time` 包提供完整的时间操作能力，而 `context` 包则是 Go 超时控制和请求链路传播的基石。

本篇涵盖：

1. `time.Time` / `Duration` / `Location` —— 时间基础
2. 时间格式化 —— Go 独特的参考时间 `"2006-01-02 15:04:05"`
3. `time.Parse` / `Format` —— 解析与格式化
4. 定时器 —— `Timer` / `Ticker` / `After` / `AfterFunc`
5. `context` 包详解 —— Background / TODO / WithCancel / WithTimeout / WithDeadline / WithValue
6. context 传播链
7. context 在 HTTP 服务中的使用
8. 超时控制最佳实践
9. SRE 实战场景
10. 常见坑点与速查表

---

## 1. time.Time / Duration / Location

### 1.1 time.Time —— 时间点

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 获取当前时间
    now := time.Now()
    fmt.Println("当前时间:", now)
    fmt.Println("Unix 时间戳:", now.Unix())
    fmt.Println("Unix 毫秒:", now.UnixMilli())
    fmt.Println("Unix 纳秒:", now.UnixNano())

    // 时间组成部分
    fmt.Printf("年: %d, 月: %d, 日: %d\n", now.Year(), now.Month(), now.Day())
    fmt.Printf("时: %d, 分: %d, 秒: %d\n", now.Hour(), now.Minute(), now.Second())
    fmt.Printf("星期: %s\n", now.Weekday())
    fmt.Printf("一年中第几天: %d\n", now.YearDay())

    // 创建指定时间
    specific := time.Date(2024, time.June, 15, 10, 30, 0, 0, time.UTC)
    fmt.Println("\n指定时间:", specific)

    // 从 Unix 时间戳创建
    fromUnix := time.Unix(1700000000, 0) // 2023-11-14
    fmt.Println("从时间戳:", fromUnix)

    // 零值时间
    var zero time.Time
    fmt.Println("\n零值时间:", zero)
    fmt.Println("是否为零值:", zero.IsZero())
}
```

### 1.2 time.Duration —— 时间段

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Duration 预定义常量
    fmt.Println("1 纳秒:  ", time.Nanosecond)
    fmt.Println("1 微秒:  ", time.Microsecond)
    fmt.Println("1 毫秒:  ", time.Millisecond)
    fmt.Println("1 秒:    ", time.Second)
    fmt.Println("1 分钟:  ", time.Minute)
    fmt.Println("1 小时:  ", time.Hour)

    // 组合使用
    timeout := 30 * time.Second
    fmt.Println("\n超时时间:", timeout)

    retryInterval := 2*time.Second + 500*time.Millisecond
    fmt.Println("重试间隔:", retryInterval)

    // Duration 转换
    d := 90 * time.Second
    fmt.Printf("\n%v = %.1f 分钟\n", d, d.Minutes())
    fmt.Printf("%v = %.0f 秒\n", d, d.Seconds())
    fmt.Printf("%v = %d 毫秒\n", d, d.Milliseconds())

    // 从字符串解析 Duration
    parsed, err := time.ParseDuration("2h30m15s")
    if err != nil {
        panic(err)
    }
    fmt.Println("\n解析的 Duration:", parsed)
    fmt.Printf("= %.2f 小时\n", parsed.Hours())

    // 更多格式
    d1, _ := time.ParseDuration("100ms")
    d2, _ := time.ParseDuration("1.5h")
    d3, _ := time.ParseDuration("2m30s")
    fmt.Println(d1, d2, d3)
}
```

### 1.3 时间运算

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // 加减时间
    oneHourLater := now.Add(1 * time.Hour)
    oneDayAgo := now.Add(-24 * time.Hour)
    fmt.Println("1小时后:", oneHourLater.Format("15:04:05"))
    fmt.Println("1天前:", oneDayAgo.Format("2006-01-02"))

    // AddDate 加减年月日
    nextMonth := now.AddDate(0, 1, 0)
    nextYear := now.AddDate(1, 0, 0)
    fmt.Println("下个月:", nextMonth.Format("2006-01-02"))
    fmt.Println("明年:", nextYear.Format("2006-01-02"))

    // 两个时间的差值
    start := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
    end := time.Date(2024, 6, 15, 12, 30, 0, 0, time.UTC)
    diff := end.Sub(start)
    fmt.Printf("\n时间差: %v (%.1f 天)\n", diff, diff.Hours()/24)

    // Since 和 Until（常用于计时）
    startTime := time.Now()
    // ... 模拟耗时操作
    time.Sleep(100 * time.Millisecond)
    elapsed := time.Since(startTime) // 等价于 time.Now().Sub(startTime)
    fmt.Printf("耗时: %v\n", elapsed)

    deadline := time.Now().Add(5 * time.Minute)
    remaining := time.Until(deadline) // 等价于 deadline.Sub(time.Now())
    fmt.Printf("剩余: %v\n", remaining)

    // 时间比较
    t1 := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
    t2 := time.Date(2024, 6, 1, 0, 0, 0, 0, time.UTC)
    fmt.Printf("\nt1.Before(t2) = %v\n", t1.Before(t2))
    fmt.Printf("t1.After(t2) = %v\n", t1.After(t2))
    fmt.Printf("t1.Equal(t2) = %v\n", t1.Equal(t2))
}
```

### 1.4 time.Location —— 时区

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // UTC 时间
    utc := now.UTC()
    fmt.Println("UTC:", utc)
    fmt.Println("本地:", now)

    // 加载时区
    shanghai, err := time.LoadLocation("Asia/Shanghai")
    if err != nil {
        fmt.Println("加载时区失败:", err)
        return
    }
    fmt.Println("上海:", now.In(shanghai))

    tokyo, _ := time.LoadLocation("Asia/Tokyo")
    fmt.Println("东京:", now.In(tokyo))

    newYork, _ := time.LoadLocation("America/New_York")
    fmt.Println("纽约:", now.In(newYork))

    // 固定偏移时区
    cst := time.FixedZone("CST", 8*60*60) // UTC+8
    fmt.Println("\nCST(固定偏移):", now.In(cst))

    // 在指定时区创建时间
    shanghaiTime := time.Date(2024, 6, 15, 10, 0, 0, 0, shanghai)
    fmt.Println("\n上海时间:", shanghaiTime)
    fmt.Println("对应UTC:", shanghaiTime.UTC())
}
```

---

## 2. 时间格式化 —— Go 的参考时间

Go 的时间格式化不使用 `%Y-%m-%d` 这样的占位符，而是使用一个特定的**参考时间**：

```
Mon Jan 2 15:04:05 MST 2006
```

这个时间对应的数字可以用来记忆：
- **1** 月 **2** 日 下午 **3**(**15**):0**4**:0**5** **2006** 年，时区偏移 **-0700**
- 简记：`1 2 3 4 5 6 7`

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // 常用格式
    fmt.Println("标准格式:")
    fmt.Println("  RFC3339:    ", now.Format(time.RFC3339))
    fmt.Println("  RFC3339Nano:", now.Format(time.RFC3339Nano))
    fmt.Println("  RFC1123:    ", now.Format(time.RFC1123))
    fmt.Println("  日期时间:   ", now.Format("2006-01-02 15:04:05"))
    fmt.Println("  仅日期:     ", now.Format("2006-01-02"))
    fmt.Println("  仅时间:     ", now.Format("15:04:05"))

    // 自定义格式
    fmt.Println("\n自定义格式:")
    fmt.Println("  中文格式:   ", now.Format("2006年01月02日 15时04分05秒"))
    fmt.Println("  日志格式:   ", now.Format("2006/01/02 15:04:05.000"))
    fmt.Println("  紧凑格式:   ", now.Format("20060102150405"))
    fmt.Println("  带时区:     ", now.Format("2006-01-02T15:04:05-07:00"))
    fmt.Println("  12小时制:   ", now.Format("2006-01-02 03:04:05 PM"))
    fmt.Println("  带星期:     ", now.Format("Mon, 02 Jan 2006 15:04:05"))

    // 预定义常量
    fmt.Println("\n预定义常量:")
    fmt.Println("  time.DateTime:  ", now.Format(time.DateTime))     // "2006-01-02 15:04:05"
    fmt.Println("  time.DateOnly:  ", now.Format(time.DateOnly))     // "2006-01-02"
    fmt.Println("  time.TimeOnly:  ", now.Format(time.TimeOnly))     // "15:04:05"
}
```

---

## 3. time.Parse / Format

### 3.1 时间解析

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Parse: 默认 UTC 时区
    t1, err := time.Parse("2006-01-02 15:04:05", "2024-06-15 10:30:00")
    if err != nil {
        panic(err)
    }
    fmt.Println("Parse 结果 (UTC):", t1)

    // ParseInLocation: 指定时区解析
    shanghai, _ := time.LoadLocation("Asia/Shanghai")
    t2, err := time.ParseInLocation("2006-01-02 15:04:05", "2024-06-15 10:30:00", shanghai)
    if err != nil {
        panic(err)
    }
    fmt.Println("ParseInLocation (上海):", t2)
    fmt.Println("对应 UTC:", t2.UTC())

    // 解析各种格式
    formats := map[string]string{
        time.RFC3339:             "2024-06-15T10:30:00Z",
        "2006-01-02":             "2024-06-15",
        "02/Jan/2006:15:04:05":  "15/Jun/2024:10:30:00", // Nginx 日志格式
        "Jan _2 15:04:05":       "Jun 15 10:30:00",       // Syslog 格式
        "2006/01/02 15:04:05":   "2024/06/15 10:30:00",   // 自定义日志格式
    }

    fmt.Println("\n解析各种日志时间格式:")
    for layout, value := range formats {
        t, err := time.Parse(layout, value)
        if err != nil {
            fmt.Printf("  %-30s → 解析失败: %v\n", value, err)
            continue
        }
        fmt.Printf("  %-30s → %s\n", value, t.Format(time.RFC3339))
    }
}
```

### 3.2 日志时间解析实战

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

// 解析不同格式的日志时间
func parseLogTime(line string) (time.Time, error) {
    // 尝试多种常见日志时间格式
    layouts := []string{
        time.RFC3339,                         // 2024-06-15T10:30:00Z
        "2006-01-02T15:04:05.000Z",          // 带毫秒的 ISO
        "2006-01-02 15:04:05",               // 标准日期时间
        "2006/01/02 15:04:05",               // 斜杠分隔
        "02/Jan/2006:15:04:05 -0700",        // Nginx/Apache
        "Jan _2 15:04:05",                    // Syslog
        "Mon Jan _2 15:04:05 2006",          // ctime 格式
    }

    // 尝试从行首提取时间字符串
    for _, layout := range layouts {
        // 尝试解析整行或行首部分
        t, err := time.Parse(layout, line)
        if err == nil {
            return t, nil
        }
        // 尝试截取与 layout 等长的前缀
        if len(line) >= len(layout) {
            prefix := line[:len(layout)+5] // 多取几个字符
            t, err := time.Parse(layout, strings.TrimSpace(prefix))
            if err == nil {
                return t, nil
            }
        }
    }

    return time.Time{}, fmt.Errorf("无法解析时间: %s", line[:min(50, len(line))])
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    testLines := []string{
        "2024-06-15T10:30:00Z",
        "2024-06-15 10:30:00",
        "2024/06/15 10:30:00",
    }

    for _, line := range testLines {
        t, err := parseLogTime(line)
        if err != nil {
            fmt.Printf("解析失败: %s\n  错误: %v\n", line, err)
            continue
        }
        fmt.Printf("输入: %-40s → %s\n", line, t.Format(time.RFC3339))
    }
}
```

---

## 4. 定时器

### 4.1 time.NewTimer —— 一次性定时器

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    fmt.Println("启动定时器 (2秒后触发)...")

    timer := time.NewTimer(2 * time.Second)

    // 阻塞等待定时器触发
    <-timer.C
    fmt.Println("定时器触发!")

    // 重置定时器
    timer.Reset(1 * time.Second)
    <-timer.C
    fmt.Println("重置后的定时器触发!")

    // 停止定时器（避免泄漏）
    timer2 := time.NewTimer(10 * time.Second)
    timer2.Stop() // 不再需要时停止
    fmt.Println("定时器已停止")
}
```

### 4.2 time.NewTicker —— 周期性定时器

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 每秒触发一次
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop() // 重要：必须停止，否则 goroutine 泄漏

    count := 0
    for t := range ticker.C {
        count++
        fmt.Printf("Tick #%d 时间: %s\n", count, t.Format("15:04:05"))
        if count >= 5 {
            break
        }
    }
    fmt.Println("定时任务结束")
}
```

### 4.3 time.After / time.AfterFunc

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // time.After: 返回一个 channel，指定时间后发送值
    fmt.Println("等待 1 秒...")
    <-time.After(1 * time.Second)
    fmt.Println("1 秒已过")

    // 常用于 select 超时
    ch := make(chan string)
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "操作完成"
    }()

    select {
    case result := <-ch:
        fmt.Println(result)
    case <-time.After(3 * time.Second):
        fmt.Println("操作超时!")
    }

    // time.AfterFunc: 延迟执行函数
    fmt.Println("\n设置延迟任务...")
    done := make(chan bool)
    timer := time.AfterFunc(1*time.Second, func() {
        fmt.Println("延迟任务执行!")
        done <- true
    })
    _ = timer // timer.Stop() 可以取消

    <-done
    fmt.Println("程序结束")
}
```

### 4.4 周期性任务调度器

```go
package main

import (
    "fmt"
    "time"
)

// 简单的任务调度器
func scheduleTask(name string, interval time.Duration, task func(), done <-chan struct{}) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    // 立即执行一次
    fmt.Printf("[%s] 首次执行\n", name)
    task()

    for {
        select {
        case <-ticker.C:
            fmt.Printf("[%s] 定期执行 at %s\n", name, time.Now().Format("15:04:05"))
            task()
        case <-done:
            fmt.Printf("[%s] 任务停止\n", name)
            return
        }
    }
}

func main() {
    done := make(chan struct{})

    // 启动多个定时任务
    go scheduleTask("健康检查", 2*time.Second, func() {
        fmt.Println("  → 检查服务健康状态...")
    }, done)

    go scheduleTask("指标采集", 3*time.Second, func() {
        fmt.Println("  → 采集 CPU/内存指标...")
    }, done)

    // 运行 8 秒后停止
    time.Sleep(8 * time.Second)
    close(done) // 通知所有任务停止
    time.Sleep(500 * time.Millisecond) // 等待清理
    fmt.Println("所有任务已停止")
}
```

---

## 5. context 包详解

`context` 是 Go 并发编程的核心包之一，主要用于：

- **超时控制**：设定操作截止时间
- **取消传播**：取消一个操作时，自动取消所有派生操作
- **请求链路传值**：在请求链路中传递少量元数据

### 5.1 context.Background / context.TODO

```go
package main

import (
    "context"
    "fmt"
)

func main() {
    // Background: 根 context，永远不会被取消
    // 通常作为 main 函数、初始化代码、测试的顶层 context
    ctx := context.Background()
    fmt.Println("Background:", ctx)

    // TODO: 当不确定使用哪个 context 时的占位符
    // 代码审查时看到 TODO 就知道这里需要改进
    todoCtx := context.TODO()
    fmt.Println("TODO:", todoCtx)

    // 两者功能完全相同，区别仅在语义上
}
```

### 5.2 context.WithCancel

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: 收到取消信号，原因: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d: 工作中...\n", id)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    // 创建可取消的 context
    ctx, cancel := context.WithCancel(context.Background())

    // 启动多个 worker
    for i := 1; i <= 3; i++ {
        go worker(ctx, i)
    }

    // 运行 2 秒后取消
    time.Sleep(2 * time.Second)
    fmt.Println("\n--- 调用 cancel() ---\n")
    cancel() // 所有 worker 会收到取消信号

    time.Sleep(1 * time.Second) // 等待 worker 退出
    fmt.Println("所有 worker 已停止")
}
```

### 5.3 context.WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

// 模拟数据库查询
func queryDatabase(ctx context.Context, query string) (string, error) {
    // 模拟随机延迟 (0-3秒)
    delay := time.Duration(rand.Intn(3000)) * time.Millisecond
    fmt.Printf("查询 '%s' 预计耗时 %v\n", query, delay)

    select {
    case <-time.After(delay):
        return fmt.Sprintf("查询结果: %s 的数据", query), nil
    case <-ctx.Done():
        return "", fmt.Errorf("查询超时: %w", ctx.Err())
    }
}

func main() {
    // 设置 1.5 秒超时
    ctx, cancel := context.WithTimeout(context.Background(), 1500*time.Millisecond)
    defer cancel() // 重要：即使操作完成，也必须调用 cancel 释放资源

    // 查看截止时间
    deadline, ok := ctx.Deadline()
    fmt.Printf("超时设置: %v, 截止时间: %v\n\n", ok, deadline.Format("15:04:05.000"))

    result, err := queryDatabase(ctx, "SELECT * FROM servers")
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Println("成功:", result)
    }
}
```

### 5.4 context.WithDeadline

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // WithDeadline: 设置绝对截止时间
    deadline := time.Now().Add(3 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    fmt.Printf("截止时间: %s\n", deadline.Format("15:04:05.000"))
    fmt.Printf("当前时间: %s\n\n", time.Now().Format("15:04:05.000"))

    // 模拟分步操作
    for step := 1; step <= 5; step++ {
        select {
        case <-ctx.Done():
            fmt.Printf("步骤 %d: 已超过截止时间 (%v)\n", step, ctx.Err())
            return
        default:
            fmt.Printf("步骤 %d: 执行中... 剩余 %v\n", step,
                time.Until(deadline).Round(time.Millisecond))
            time.Sleep(1 * time.Second)
        }
    }
}
```

### 5.5 context.WithValue

```go
package main

import (
    "context"
    "fmt"
)

// 使用自定义类型作为 key，避免冲突
type contextKey string

const (
    requestIDKey contextKey = "request_id"
    userIDKey    contextKey = "user_id"
    traceIDKey   contextKey = "trace_id"
)

func processRequest(ctx context.Context) {
    // 从 context 中获取值
    requestID, _ := ctx.Value(requestIDKey).(string)
    userID, _ := ctx.Value(userIDKey).(string)
    traceID, _ := ctx.Value(traceIDKey).(string)

    fmt.Printf("处理请求: requestID=%s, userID=%s, traceID=%s\n",
        requestID, userID, traceID)
}

func main() {
    // 逐层添加值
    ctx := context.Background()
    ctx = context.WithValue(ctx, requestIDKey, "req-12345")
    ctx = context.WithValue(ctx, userIDKey, "user-admin")
    ctx = context.WithValue(ctx, traceIDKey, "trace-abc-def")

    processRequest(ctx)
}
```

### 5.6 context.WithoutCancel (Go 1.21+)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 创建一个有超时的 context
    parentCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    // WithoutCancel: 创建一个不受父 context 取消影响的子 context
    // 适用场景：即使请求取消了，仍需完成的清理操作（如写审计日志）
    cleanupCtx := context.WithoutCancel(parentCtx)

    // 取消父 context
    cancel()

    // 父已取消
    fmt.Println("父 context 已取消:", parentCtx.Err())

    // 子不受影响
    fmt.Println("清理 context 状态:", cleanupCtx.Err()) // nil

    // 可以安全地做清理操作
    fmt.Println("执行清理操作...")
    _ = cleanupCtx
}
```

---

## 6. context 传播链

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// 模拟 HTTP 请求链路: Handler → Service → Repository → Database

func httpHandler(ctx context.Context) {
    // 添加请求级别超时
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    fmt.Println("[Handler] 开始处理请求")
    result, err := serviceLayer(ctx)
    if err != nil {
        fmt.Println("[Handler] 错误:", err)
        return
    }
    fmt.Println("[Handler] 结果:", result)
}

func serviceLayer(ctx context.Context) (string, error) {
    fmt.Println("  [Service] 业务逻辑处理")

    // 继承父 context 的超时，但可以设更短的超时
    // 注意：不能设比父更长的超时（父超时时子自动超时）
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    return repositoryLayer(ctx)
}

func repositoryLayer(ctx context.Context) (string, error) {
    fmt.Println("    [Repository] 查询数据")
    return databaseQuery(ctx, "SELECT * FROM servers WHERE status='active'")
}

func databaseQuery(ctx context.Context, query string) (string, error) {
    fmt.Println("      [Database] 执行查询:", query)

    // 检查 context 是否已超时
    select {
    case <-ctx.Done():
        return "", fmt.Errorf("数据库查询超时: %w", ctx.Err())
    case <-time.After(1 * time.Second): // 模拟查询耗时
        return "3 台活跃服务器", nil
    }
}

func main() {
    rootCtx := context.Background()
    httpHandler(rootCtx)
}
```

---

## 7. context 在 HTTP 服务中的使用

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"
)

type contextKey string

const requestIDKey contextKey = "request_id"

// 请求 ID 中间件
func requestIDMiddleware(next http.Handler) http.Handler {
    var counter int
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        counter++
        requestID := fmt.Sprintf("req-%d-%d", time.Now().UnixMilli(), counter)

        // 将 requestID 添加到 context
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)

        // 设置响应头
        w.Header().Set("X-Request-ID", requestID)

        // 传递新的 context
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 超时中间件
func timeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// 模拟慢查询
func slowHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    requestID := ctx.Value(requestIDKey).(string)

    log.Printf("[%s] 开始处理慢查询", requestID)

    select {
    case <-time.After(3 * time.Second):
        // 正常完成
        json.NewEncoder(w).Encode(map[string]string{
            "request_id": requestID,
            "status":     "completed",
        })
    case <-ctx.Done():
        // 超时或客户端断开
        log.Printf("[%s] 请求超时: %v", requestID, ctx.Err())
        http.Error(w, "请求超时", http.StatusGatewayTimeout)
    }
}

func fastHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    requestID := ctx.Value(requestIDKey).(string)

    json.NewEncoder(w).Encode(map[string]string{
        "request_id": requestID,
        "status":     "ok",
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /fast", fastHandler)
    mux.HandleFunc("GET /slow", slowHandler)

    // 中间件链：requestID → timeout(2s) → handler
    handler := requestIDMiddleware(
        timeoutMiddleware(2 * time.Second)(mux),
    )

    log.Println("服务启动在 :8080")
    log.Println("  /fast - 快速响应")
    log.Println("  /slow - 慢查询 (会超时)")
    http.ListenAndServe(":8080", handler)
}
```

---

## 8. 超时控制最佳实践

### 8.1 分层超时策略

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// 最佳实践：外层超时 > 内层超时
// HTTP 总超时 > 业务超时 > DB 超时 > 单次查询超时

func bestPracticeTimeout() {
    // 第一层：HTTP 请求总超时 10s
    httpCtx, httpCancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer httpCancel()

    // 第二层：业务逻辑超时 8s（留 2s 给响应序列化等）
    bizCtx, bizCancel := context.WithTimeout(httpCtx, 8*time.Second)
    defer bizCancel()

    // 第三层：数据库查询超时 5s
    dbCtx, dbCancel := context.WithTimeout(bizCtx, 5*time.Second)
    defer dbCancel()

    // 第四层：单次重试超时 2s
    retryCtx, retryCancel := context.WithTimeout(dbCtx, 2*time.Second)
    defer retryCancel()

    _ = retryCtx
    fmt.Println("超时策略: HTTP(10s) > 业务(8s) > DB(5s) > 单次(2s)")
}

func main() {
    bestPracticeTimeout()

    // 演示：检查剩余超时时间
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        fmt.Printf("剩余超时: %v\n", remaining)
        if remaining < 1*time.Second {
            fmt.Println("剩余时间不足，跳过操作")
        }
    }
}
```

### 8.2 带超时的重试模式

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

func unreliableOperation() error {
    if rand.Float64() < 0.7 { // 70% 概率失败
        return fmt.Errorf("操作失败")
    }
    return nil
}

func retryWithContext(ctx context.Context, maxRetries int, operation func() error) error {
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        // 检查 context 是否已取消
        if ctx.Err() != nil {
            return fmt.Errorf("context 已取消: %w (最后错误: %v)", ctx.Err(), lastErr)
        }

        err := operation()
        if err == nil {
            if attempt > 0 {
                fmt.Printf("  第 %d 次重试成功\n", attempt)
            }
            return nil
        }

        lastErr = err
        fmt.Printf("  第 %d 次尝试失败: %v\n", attempt+1, err)

        if attempt < maxRetries {
            // 指数退避，但受 context 超时限制
            backoff := time.Duration(1<<uint(attempt)) * 200 * time.Millisecond
            select {
            case <-time.After(backoff):
                // 退避完成，继续重试
            case <-ctx.Done():
                return fmt.Errorf("退避期间 context 取消: %w", ctx.Err())
            }
        }
    }

    return fmt.Errorf("重试 %d 次后仍然失败: %v", maxRetries, lastErr)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    fmt.Println("开始带超时的重试操作...")
    err := retryWithContext(ctx, 10, unreliableOperation)
    if err != nil {
        fmt.Println("最终失败:", err)
    } else {
        fmt.Println("最终成功!")
    }
}
```

---

## 9. SRE 实战场景

### 9.1 SLA 计算器

```go
package main

import (
    "fmt"
    "time"
)

type Incident struct {
    Start time.Time
    End   time.Time
    Desc  string
}

func calculateSLA(month time.Month, year int, incidents []Incident) {
    // 计算该月总时间
    start := time.Date(year, month, 1, 0, 0, 0, 0, time.UTC)
    end := start.AddDate(0, 1, 0) // 下个月 1 号
    totalDuration := end.Sub(start)

    // 计算总停机时间
    var totalDowntime time.Duration
    for _, inc := range incidents {
        // 确保事件在目标月份内
        incStart := inc.Start
        incEnd := inc.End
        if incStart.Before(start) {
            incStart = start
        }
        if incEnd.After(end) {
            incEnd = end
        }
        if incStart.Before(incEnd) {
            downtime := incEnd.Sub(incStart)
            totalDowntime += downtime
            fmt.Printf("  事故: %s\n", inc.Desc)
            fmt.Printf("    持续: %v\n", downtime.Round(time.Minute))
        }
    }

    // 计算 SLA
    uptime := totalDuration - totalDowntime
    slaPercent := float64(uptime) / float64(totalDuration) * 100

    fmt.Printf("\n%d 年 %s 月 SLA 报告:\n", year, month)
    fmt.Printf("  总时间:   %v (%.0f 小时)\n", totalDuration, totalDuration.Hours())
    fmt.Printf("  停机时间: %v (%.1f 小时)\n", totalDowntime, totalDowntime.Hours())
    fmt.Printf("  可用时间: %v\n", uptime)
    fmt.Printf("  SLA:      %.4f%%\n", slaPercent)

    // SLA 等级判定
    switch {
    case slaPercent >= 99.99:
        fmt.Println("  等级:     四个9 (优秀)")
    case slaPercent >= 99.9:
        fmt.Println("  等级:     三个9 (良好)")
    case slaPercent >= 99.0:
        fmt.Println("  等级:     两个9 (需改进)")
    default:
        fmt.Println("  等级:     低于两个9 (严重)")
    }
}

func main() {
    incidents := []Incident{
        {
            Start: time.Date(2024, 6, 5, 14, 0, 0, 0, time.UTC),
            End:   time.Date(2024, 6, 5, 14, 30, 0, 0, time.UTC),
            Desc:  "数据库主从切换导致短暂不可用",
        },
        {
            Start: time.Date(2024, 6, 15, 3, 0, 0, 0, time.UTC),
            End:   time.Date(2024, 6, 15, 3, 15, 0, 0, time.UTC),
            Desc:  "证书过期导致 HTTPS 503",
        },
        {
            Start: time.Date(2024, 6, 22, 10, 0, 0, 0, time.UTC),
            End:   time.Date(2024, 6, 22, 11, 0, 0, 0, time.UTC),
            Desc:  "K8s 节点 OOM 导致 Pod 被驱逐",
        },
    }

    calculateSLA(time.June, 2024, incidents)
}
```

### 9.2 证书到期检查器

```go
package main

import (
    "fmt"
    "time"
)

type Certificate struct {
    Domain    string
    ExpiresAt time.Time
}

func checkCertExpiry(certs []Certificate) {
    now := time.Now()

    fmt.Println("SSL 证书到期检查:")
    fmt.Printf("%-30s %-20s %-15s %s\n", "域名", "到期时间", "剩余天数", "状态")
    fmt.Println("─────────────────────────────────────────────────────────────────────────────")

    for _, cert := range certs {
        remaining := cert.ExpiresAt.Sub(now)
        days := int(remaining.Hours() / 24)

        status := "正常"
        if days < 0 {
            status = "已过期"
        } else if days < 7 {
            status = "紧急"
        } else if days < 30 {
            status = "警告"
        }

        fmt.Printf("%-30s %-20s %-15d %s\n",
            cert.Domain,
            cert.ExpiresAt.Format("2006-01-02"),
            days,
            status,
        )
    }
}

func main() {
    now := time.Now()
    certs := []Certificate{
        {Domain: "api.example.com", ExpiresAt: now.Add(90 * 24 * time.Hour)},
        {Domain: "www.example.com", ExpiresAt: now.Add(25 * 24 * time.Hour)},
        {Domain: "admin.example.com", ExpiresAt: now.Add(5 * 24 * time.Hour)},
        {Domain: "old.example.com", ExpiresAt: now.Add(-3 * 24 * time.Hour)},
        {Domain: "new.example.com", ExpiresAt: now.Add(365 * 24 * time.Hour)},
    }

    checkCertExpiry(certs)
}
```

### 9.3 并发请求限时聚合

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

type ServiceStatus struct {
    Name    string
    Status  string
    Latency time.Duration
    Error   string
}

func checkService(ctx context.Context, name string) ServiceStatus {
    start := time.Now()

    // 模拟不同服务的响应延迟
    delay := time.Duration(rand.Intn(2000)) * time.Millisecond

    select {
    case <-time.After(delay):
        return ServiceStatus{
            Name:    name,
            Status:  "healthy",
            Latency: time.Since(start),
        }
    case <-ctx.Done():
        return ServiceStatus{
            Name:    name,
            Status:  "timeout",
            Latency: time.Since(start),
            Error:   ctx.Err().Error(),
        }
    }
}

func aggregateStatus(ctx context.Context, services []string) []ServiceStatus {
    // 所有服务必须在超时内返回
    ctx, cancel := context.WithTimeout(ctx, 1500*time.Millisecond)
    defer cancel()

    var mu sync.Mutex
    results := make([]ServiceStatus, 0, len(services))
    var wg sync.WaitGroup

    for _, svc := range services {
        wg.Add(1)
        go func(name string) {
            defer wg.Done()
            status := checkService(ctx, name)
            mu.Lock()
            results = append(results, status)
            mu.Unlock()
        }(svc)
    }

    wg.Wait()
    return results
}

func main() {
    services := []string{"nginx", "redis", "postgres", "elasticsearch", "rabbitmq"}

    fmt.Println("聚合服务状态 (超时 1.5s):\n")
    results := aggregateStatus(context.Background(), services)

    for _, r := range results {
        fmt.Printf("  %-18s 状态: %-10s 延迟: %v",
            r.Name, r.Status, r.Latency.Round(time.Millisecond))
        if r.Error != "" {
            fmt.Printf(" 错误: %s", r.Error)
        }
        fmt.Println()
    }
}
```

---

## 常见坑点

### 坑 1：格式化字符串用错参考时间

```go
// 错误：Go 不使用 %Y-%m-%d 格式
// t.Format("%Y-%m-%d")  // 输出的是字面量 "%Y-%m-%d"

// 正确：使用参考时间 2006-01-02 15:04:05
t.Format("2006-01-02 15:04:05")

// 记忆技巧：1 2 3 4 5 6 7
// Jan(1月) 2(日) 15(3PM) 04(分) 05(秒) 2006(年) -0700(时区)
```

### 坑 2：忘记调用 cancel

```go
// 错误：context 泄漏！
func handle(r *http.Request) {
    ctx, _ := context.WithTimeout(r.Context(), 5*time.Second)
    // cancel 被丢弃，Timer 不会被清理
    doWork(ctx)
}

// 正确：始终 defer cancel()
func handle(r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel() // 即使操作提前完成，也释放资源
    doWork(ctx)
}
```

### 坑 3：time.After 在循环中导致内存泄漏

```go
// 错误：每次循环创建一个新的 Timer，旧的不会被回收直到触发
for {
    select {
    case msg := <-ch:
        handle(msg)
    case <-time.After(5 * time.Second): // 每次循环分配新 Timer!
        fmt.Println("超时")
    }
}

// 正确：使用 time.NewTimer 并手动重置
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()
for {
    select {
    case msg := <-ch:
        if !timer.Stop() {
            <-timer.C
        }
        timer.Reset(5 * time.Second)
        handle(msg)
    case <-timer.C:
        fmt.Println("超时")
        timer.Reset(5 * time.Second)
    }
}
```

### 坑 4：Ticker 忘记 Stop

```go
// 错误：Ticker 不会被 GC，会一直运行
func startMetrics() {
    ticker := time.NewTicker(1 * time.Second)
    for range ticker.C {
        collectMetrics()
    }
    // ticker 永远不会被 Stop
}

// 正确：使用 defer Stop
func startMetrics(done <-chan struct{}) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            collectMetrics()
        case <-done:
            return
        }
    }
}
```

### 坑 5：context.WithValue 滥用

```go
// 反模式：用 context 传递业务参数
ctx = context.WithValue(ctx, "user", userObj)
ctx = context.WithValue(ctx, "order", orderObj)
ctx = context.WithValue(ctx, "config", configObj)
// context 变成了全局变量袋!

// 正确用法：只传递请求范围的元数据
// - request ID
// - trace ID
// - 认证信息
// - 截止时间（由 WithTimeout/WithDeadline 自动处理）
```

---

## 速查表

```text
┌──────────────────────────────────────────────────────────────────────┐
│                    time 速查表                                        │
├──────────────────────────────────────────────────────────────────────┤
│ 当前时间       │ time.Now()                                          │
│ 创建时间       │ time.Date(2024,6,15,10,0,0,0,time.UTC)             │
│ 从时间戳       │ time.Unix(sec, nsec)                                │
│ 格式化         │ t.Format("2006-01-02 15:04:05")                    │
│ 解析           │ time.Parse("2006-01-02", "2024-06-15")             │
│ 时间差         │ t1.Sub(t2) → Duration                              │
│ 经过时间       │ time.Since(start)                                   │
│ 剩余时间       │ time.Until(deadline)                                │
│ 加减时间       │ t.Add(duration) / t.AddDate(y,m,d)                 │
│ 比较           │ t.Before(t2) / t.After(t2) / t.Equal(t2)          │
│ 定时器(单次)   │ time.NewTimer(d)  → timer.C / timer.Stop()         │
│ 定时器(周期)   │ time.NewTicker(d) → ticker.C / ticker.Stop()       │
│ 延迟通道       │ <-time.After(d)                                    │
│ 延迟函数       │ time.AfterFunc(d, func(){})                        │
│ 解析Duration   │ time.ParseDuration("1h30m")                        │
│ 睡眠           │ time.Sleep(d)                                       │
├──────────────────────────────────────────────────────────────────────┤
│                    context 速查表                                     │
├──────────────────────────────────────────────────────────────────────┤
│ 根 context     │ context.Background()                                │
│ 占位 context   │ context.TODO()                                      │
│ 可取消         │ ctx, cancel := context.WithCancel(parent)           │
│ 超时(相对)     │ ctx, cancel := context.WithTimeout(parent, d)       │
│ 超时(绝对)     │ ctx, cancel := context.WithDeadline(parent, t)      │
│ 传值           │ ctx = context.WithValue(parent, key, val)           │
│ 不可取消副本   │ ctx = context.WithoutCancel(parent)  (Go 1.21+)     │
│ 检查取消       │ <-ctx.Done() / ctx.Err()                           │
│ 获取截止时间   │ deadline, ok := ctx.Deadline()                      │
│ 获取值         │ val := ctx.Value(key)                               │
└──────────────────────────────────────────────────────────────────────┘

参考时间记忆法: 1(月) 2(日) 3(15时) 4(分) 5(秒) 6(2006年) 7(-0700时区)
```

---

## 小结

- Go 的时间格式化使用**参考时间** `2006-01-02 15:04:05` 而非占位符，初看怪异但非常灵活。
- `time.Duration` 用于表示时间段，注意不要直接用整数（`time.Sleep(5)` 是 5 纳秒而非 5 秒）。
- `context` 是 Go 并发控制的核心：用 `WithCancel` 取消、`WithTimeout` 超时、`WithValue` 传元数据。
- **始终 `defer cancel()`**，避免 context 泄漏。
- 分层超时策略：外层超时 > 内层超时。
- `time.After` 在循环中会导致内存泄漏，应使用 `time.NewTimer` + `Reset`。
- `context.WithValue` 只传请求范围的元数据（request ID、trace ID），不要当全局变量用。
