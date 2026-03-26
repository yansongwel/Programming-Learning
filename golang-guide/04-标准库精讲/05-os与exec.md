# 05 - os 与 exec

> **适用版本**: Go 1.24 | **面向读者**: 有编程经验的 SRE / DevOps 工程师

---

## 导读

SRE/DevOps 工程师的日常离不开系统交互：读取环境变量、解析命令行参数、处理系统信号实现优雅退出、调用外部命令（`kubectl`、`docker`、`ansible`）……Go 的 `os` 和 `os/exec` 包提供了完整的操作系统交互能力，是编写运维工具的基石。

本篇涵盖：

1. `os.Args` —— 命令行参数
2. `os.Getenv` / `os.Setenv` —— 环境变量
3. `os.Exit` —— 退出进程
4. `os/signal` —— 信号处理（SIGINT/SIGTERM 优雅退出）
5. `os/exec.Command` —— 执行外部命令
6. 管道连接多个命令
7. 实时输出读取（StdoutPipe）
8. SRE 实战场景
9. 对比 Python subprocess / Shell 脚本
10. 常见坑点与速查表

---

## 1. os.Args —— 命令行参数

### 1.1 基本使用

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.Args 是一个字符串切片
    // os.Args[0] 是程序本身的路径
    // os.Args[1:] 是传入的参数

    fmt.Println("程序路径:", os.Args[0])
    fmt.Println("参数数量:", len(os.Args)-1)

    if len(os.Args) > 1 {
        fmt.Println("所有参数:")
        for i, arg := range os.Args[1:] {
            fmt.Printf("  参数 %d: %s\n", i+1, arg)
        }
    } else {
        fmt.Println("用法: program <arg1> <arg2> ...")
    }
}
// 运行: go run main.go hello world 123
// 输出:
//   程序路径: /tmp/go-build.../main
//   参数数量: 3
//   所有参数:
//     参数 1: hello
//     参数 2: world
//     参数 3: 123
```

### 1.2 简单的命令行工具

```go
package main

import (
    "fmt"
    "os"
    "strconv"
    "strings"
)

func printUsage() {
    fmt.Println(`SRE 工具箱 v1.0

用法:
  sre-tool <命令> [参数...]

命令:
  status <host>         检查主机状态
  restart <service>     重启服务
  scale <service> <n>   扩缩容
  help                  显示帮助`)
}

func main() {
    if len(os.Args) < 2 {
        printUsage()
        os.Exit(1)
    }

    command := os.Args[1]

    switch command {
    case "status":
        if len(os.Args) < 3 {
            fmt.Println("错误: status 命令需要指定主机名")
            os.Exit(1)
        }
        host := os.Args[2]
        fmt.Printf("检查主机 %s 状态...\n", host)
        fmt.Printf("  CPU: 45.2%%\n  内存: 60.1%%\n  状态: 正常\n")

    case "restart":
        if len(os.Args) < 3 {
            fmt.Println("错误: restart 命令需要指定服务名")
            os.Exit(1)
        }
        service := os.Args[2]
        fmt.Printf("重启服务 %s ...\n", service)
        fmt.Println("服务已重启")

    case "scale":
        if len(os.Args) < 4 {
            fmt.Println("错误: scale 命令需要指定服务名和副本数")
            os.Exit(1)
        }
        service := os.Args[2]
        n, err := strconv.Atoi(os.Args[3])
        if err != nil {
            fmt.Println("错误: 副本数必须是整数")
            os.Exit(1)
        }
        fmt.Printf("将服务 %s 扩展到 %d 个副本...\n", service, n)

    case "help":
        printUsage()

    default:
        fmt.Printf("未知命令: %s\n", command)
        fmt.Printf("可用命令: %s\n", strings.Join([]string{"status", "restart", "scale", "help"}, ", "))
        os.Exit(1)
    }
}
```

### 1.3 flag 包 —— 标准命令行解析

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // 定义命令行标志
    host := flag.String("host", "localhost", "目标主机")
    port := flag.Int("port", 8080, "目标端口")
    timeout := flag.Int("timeout", 30, "超时时间(秒)")
    verbose := flag.Bool("verbose", false, "详细输出")
    tags := flag.String("tags", "", "标签(逗号分隔)")

    // 解析命令行
    flag.Parse()

    // 使用参数
    fmt.Printf("主机: %s\n", *host)
    fmt.Printf("端口: %d\n", *port)
    fmt.Printf("超时: %d秒\n", *timeout)
    fmt.Printf("详细: %v\n", *verbose)
    fmt.Printf("标签: %s\n", *tags)

    // 剩余未解析的参数
    fmt.Printf("其他参数: %v\n", flag.Args())
}
// 运行: go run main.go -host=10.0.1.1 -port=9090 -verbose -tags=prod,web extra1 extra2
```

---

## 2. os.Getenv / os.Setenv —— 环境变量

### 2.1 读取环境变量

```go
package main

import (
    "fmt"
    "os"
    "strings"
)

func main() {
    // 读取单个环境变量
    home := os.Getenv("HOME")
    fmt.Println("HOME:", home)

    user := os.Getenv("USER")
    fmt.Println("USER:", user)

    path := os.Getenv("PATH")
    fmt.Println("PATH 目录数:", len(strings.Split(path, ":")))

    // 读取不存在的环境变量（返回空字符串）
    missing := os.Getenv("NOT_EXIST")
    fmt.Printf("NOT_EXIST: '%s' (空字符串)\n", missing)

    // LookupEnv：区分 "未设置" 和 "设置为空"
    val, ok := os.LookupEnv("HOME")
    if ok {
        fmt.Println("HOME 已设置:", val)
    } else {
        fmt.Println("HOME 未设置")
    }

    val2, ok2 := os.LookupEnv("NOT_EXIST")
    if ok2 {
        fmt.Println("NOT_EXIST 已设置:", val2)
    } else {
        fmt.Println("NOT_EXIST 未设置")
    }
}
```

### 2.2 带默认值的环境变量读取

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

// 通用环境变量读取函数
func getEnv(key, defaultValue string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    str := getEnv(key, "")
    if str == "" {
        return defaultValue
    }
    val, err := strconv.Atoi(str)
    if err != nil {
        return defaultValue
    }
    return val
}

func getEnvBool(key string, defaultValue bool) bool {
    str := getEnv(key, "")
    if str == "" {
        return defaultValue
    }
    val, err := strconv.ParseBool(str)
    if err != nil {
        return defaultValue
    }
    return val
}

type Config struct {
    Host     string
    Port     int
    LogLevel string
    Debug    bool
    Workers  int
}

func loadConfig() Config {
    return Config{
        Host:     getEnv("APP_HOST", "0.0.0.0"),
        Port:     getEnvInt("APP_PORT", 8080),
        LogLevel: getEnv("APP_LOG_LEVEL", "info"),
        Debug:    getEnvBool("APP_DEBUG", false),
        Workers:  getEnvInt("APP_WORKERS", 4),
    }
}

func main() {
    cfg := loadConfig()
    fmt.Printf("配置:\n")
    fmt.Printf("  Host:     %s\n", cfg.Host)
    fmt.Printf("  Port:     %d\n", cfg.Port)
    fmt.Printf("  LogLevel: %s\n", cfg.LogLevel)
    fmt.Printf("  Debug:    %v\n", cfg.Debug)
    fmt.Printf("  Workers:  %d\n", cfg.Workers)
}
```

### 2.3 设置和清除环境变量

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 设置环境变量
    os.Setenv("MY_SERVICE", "web-api")
    os.Setenv("MY_ENV", "production")

    fmt.Println("MY_SERVICE:", os.Getenv("MY_SERVICE"))
    fmt.Println("MY_ENV:", os.Getenv("MY_ENV"))

    // 清除环境变量
    os.Unsetenv("MY_SERVICE")
    fmt.Println("清除后 MY_SERVICE:", os.Getenv("MY_SERVICE"))

    // 列出所有环境变量
    fmt.Println("\n前 5 个环境变量:")
    envs := os.Environ() // 返回 []string，格式为 "KEY=VALUE"
    for i, env := range envs {
        if i >= 5 {
            break
        }
        fmt.Printf("  %s\n", env)
    }
    fmt.Printf("  ... 共 %d 个\n", len(envs))
}
```

---

## 3. os.Exit —— 退出进程

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.Exit 立即终止程序
    // 不会执行 defer 语句！
    // 不会运行 finally 或 cleanup 逻辑

    defer fmt.Println("这行不会执行!") // defer 在 os.Exit 后不执行

    fmt.Println("程序正在运行...")

    // 检查环境变量决定是否退出
    if os.Getenv("FORCE_EXIT") == "1" {
        fmt.Println("强制退出")
        os.Exit(1) // 非零状态码表示异常退出
    }

    // 正常退出
    fmt.Println("正常结束")
    // os.Exit(0) // 0 表示成功
}

// 退出码约定：
// 0  - 成功
// 1  - 一般错误
// 2  - 命令行使用错误
// 126 - 权限不足
// 127 - 命令未找到
// 128+N - 被信号 N 终止
```

---

## 4. os/signal —— 信号处理

### 4.1 基本信号捕获

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // 创建信号通道
    sigChan := make(chan os.Signal, 1)

    // 注册要捕获的信号
    // SIGINT:  Ctrl+C
    // SIGTERM: kill 命令（K8s Pod 终止时发送）
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    fmt.Println("程序已启动，按 Ctrl+C 退出...")
    fmt.Println("PID:", os.Getpid())

    // 模拟后台工作
    go func() {
        for i := 1; ; i++ {
            fmt.Printf("工作中... (第 %d 秒)\n", i)
            time.Sleep(1 * time.Second)
        }
    }()

    // 等待信号
    sig := <-sigChan
    fmt.Printf("\n收到信号: %v\n", sig)
    fmt.Println("开始清理...")
    time.Sleep(500 * time.Millisecond) // 模拟清理操作
    fmt.Println("清理完成，程序退出")
}
```

### 4.2 优雅退出模式（SRE 必备）

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

// 模拟后台服务组件
type Worker struct {
    name string
    done chan struct{}
}

func NewWorker(name string) *Worker {
    return &Worker{name: name, done: make(chan struct{})}
}

func (w *Worker) Start() {
    fmt.Printf("[%s] 启动\n", w.name)
    go func() {
        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                fmt.Printf("[%s] 运行中...\n", w.name)
            case <-w.done:
                fmt.Printf("[%s] 收到停止信号，清理中...\n", w.name)
                time.Sleep(500 * time.Millisecond) // 模拟清理
                fmt.Printf("[%s] 已停止\n", w.name)
                return
            }
        }
    }()
}

func (w *Worker) Stop() {
    close(w.done)
}

func main() {
    fmt.Println("=== SRE 服务优雅退出演示 ===")
    fmt.Println("PID:", os.Getpid())

    // 启动 HTTP 服务
    mux := http.NewServeMux()
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, `{"status":"ok"}`)
    })
    server := &http.Server{Addr: ":8080", Handler: mux}

    // 启动后台 Worker
    workers := []*Worker{
        NewWorker("指标采集器"),
        NewWorker("日志处理器"),
        NewWorker("告警引擎"),
    }

    // 启动所有组件
    go func() {
        fmt.Println("[HTTP] 启动在 :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            fmt.Printf("[HTTP] 异常: %v\n", err)
        }
    }()

    for _, w := range workers {
        w.Start()
    }

    // 等待终止信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    fmt.Printf("\n收到信号: %v，开始优雅关闭...\n", sig)

    // 阶段 1：停止接受新请求
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    fmt.Println("\n--- 阶段 1: 停止 HTTP 服务 ---")
    if err := server.Shutdown(shutdownCtx); err != nil {
        fmt.Printf("[HTTP] 关闭失败: %v\n", err)
    } else {
        fmt.Println("[HTTP] 已关闭")
    }

    // 阶段 2：停止后台 Worker
    fmt.Println("\n--- 阶段 2: 停止后台组件 ---")
    var wg sync.WaitGroup
    for _, w := range workers {
        wg.Add(1)
        go func(worker *Worker) {
            defer wg.Done()
            worker.Stop()
        }(w)
    }

    // 等待所有 Worker 停止（带超时）
    doneChan := make(chan struct{})
    go func() {
        wg.Wait()
        close(doneChan)
    }()

    select {
    case <-doneChan:
        fmt.Println("\n所有组件已优雅停止")
    case <-time.After(10 * time.Second):
        fmt.Println("\n超时！强制退出")
    }

    fmt.Println("程序退出")
}
```

### 4.3 signal.NotifyContext (Go 1.16+)

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // signal.NotifyContext: 收到信号时自动取消 context
    // 比手动创建 channel + signal.Notify 更简洁
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    fmt.Println("程序启动，按 Ctrl+C 退出")
    fmt.Println("PID:", os.Getpid())

    // 用 context 控制所有操作
    go func() {
        ticker := time.NewTicker(1 * time.Second)
        defer ticker.Stop()
        count := 0
        for {
            select {
            case <-ticker.C:
                count++
                fmt.Printf("心跳 #%d\n", count)
            case <-ctx.Done():
                fmt.Println("后台任务收到取消信号")
                return
            }
        }
    }()

    // 等待 context 被取消（即收到信号）
    <-ctx.Done()
    fmt.Println("\n收到终止信号，退出原因:", ctx.Err())
    fmt.Println("执行清理...")
    time.Sleep(500 * time.Millisecond)
    fmt.Println("程序退出")
}
```

---

## 5. os/exec.Command —— 执行外部命令

### 5.1 基本用法：Run / Output / CombinedOutput

```go
package main

import (
    "fmt"
    "os/exec"
)

func main() {
    // ---- Run: 执行命令，只返回 error ----
    fmt.Println("=== Run ===")
    cmd := exec.Command("echo", "Hello from Go!")
    err := cmd.Run()
    if err != nil {
        fmt.Println("执行失败:", err)
    }
    // Run 不捕获输出，输出直接到 os.Stdout/os.Stderr
    // 如果需要在控制台看到，设置 cmd.Stdout = os.Stdout

    // ---- Output: 执行命令并捕获标准输出 ----
    fmt.Println("\n=== Output ===")
    cmd2 := exec.Command("hostname")
    output, err := cmd2.Output()
    if err != nil {
        fmt.Println("执行失败:", err)
        return
    }
    fmt.Printf("主机名: %s", string(output))

    // ---- CombinedOutput: 捕获 stdout + stderr ----
    fmt.Println("\n=== CombinedOutput ===")
    cmd3 := exec.Command("ls", "-la", "/tmp")
    combined, err := cmd3.CombinedOutput()
    if err != nil {
        fmt.Println("执行失败:", err)
        fmt.Println("输出:", string(combined))
        return
    }
    lines := 0
    for _, b := range combined {
        if b == '\n' {
            lines++
        }
    }
    fmt.Printf("ls /tmp: %d 行输出\n", lines)

    // ---- 执行不存在的命令 ----
    fmt.Println("\n=== 错误处理 ===")
    cmd4 := exec.Command("nonexistent_command")
    _, err = cmd4.Output()
    if err != nil {
        fmt.Println("错误:", err)
        // 判断错误类型
        if exitErr, ok := err.(*exec.ExitError); ok {
            fmt.Println("退出码:", exitErr.ExitCode())
            fmt.Println("Stderr:", string(exitErr.Stderr))
        }
    }
}
```

### 5.2 Start + Wait —— 异步执行

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "time"
)

func main() {
    // Start 启动命令但不等待完成
    cmd := exec.Command("sleep", "2")
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    fmt.Println("启动命令...")
    start := time.Now()

    err := cmd.Start()
    if err != nil {
        fmt.Println("启动失败:", err)
        return
    }

    fmt.Printf("命令已启动，PID: %d\n", cmd.Process.Pid)
    fmt.Println("主程序继续执行其他工作...")

    // 做一些其他工作
    time.Sleep(500 * time.Millisecond)
    fmt.Println("其他工作完成")

    // 等待命令完成
    err = cmd.Wait()
    elapsed := time.Since(start)

    if err != nil {
        fmt.Println("命令执行失败:", err)
    } else {
        fmt.Printf("命令执行完成，耗时: %v\n", elapsed)
    }
}
```

### 5.3 设置工作目录和环境变量

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    cmd := exec.Command("env")

    // 设置工作目录
    cmd.Dir = "/tmp"

    // 设置环境变量（覆盖默认值）
    cmd.Env = append(os.Environ(),
        "MY_APP=sre-tool",
        "MY_ENV=production",
        "MY_VERSION=1.0.0",
    )

    output, err := cmd.Output()
    if err != nil {
        fmt.Println("执行失败:", err)
        return
    }

    // 只打印我们设置的环境变量
    fmt.Println("自定义环境变量:")
    for _, line := range splitLines(string(output)) {
        if len(line) > 3 && line[:3] == "MY_" {
            fmt.Println(" ", line)
        }
    }
}

func splitLines(s string) []string {
    var lines []string
    start := 0
    for i, c := range s {
        if c == '\n' {
            lines = append(lines, s[start:i])
            start = i + 1
        }
    }
    if start < len(s) {
        lines = append(lines, s[start:])
    }
    return lines
}
```

### 5.4 带超时的命令执行

```go
package main

import (
    "context"
    "fmt"
    "os/exec"
    "time"
)

func runWithTimeout(timeout time.Duration, name string, args ...string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    cmd := exec.CommandContext(ctx, name, args...)
    output, err := cmd.CombinedOutput()

    if ctx.Err() == context.DeadlineExceeded {
        return string(output), fmt.Errorf("命令超时 (超过 %v)", timeout)
    }

    return string(output), err
}

func main() {
    // 正常执行
    fmt.Println("=== 正常命令 ===")
    output, err := runWithTimeout(5*time.Second, "echo", "hello")
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Print("输出: ", output)
    }

    // 超时命令
    fmt.Println("\n=== 超时命令 ===")
    output, err = runWithTimeout(1*time.Second, "sleep", "10")
    if err != nil {
        fmt.Println("错误:", err) // 命令超时
    }
    _ = output
}
```

---

## 6. 管道连接多个命令

### 6.1 模拟 Shell 管道

```go
package main

import (
    "bytes"
    "fmt"
    "os/exec"
)

func main() {
    // 模拟: cat /etc/passwd | grep root | wc -l

    // 命令 1: cat /etc/passwd
    cmd1 := exec.Command("cat", "/etc/passwd")

    // 命令 2: grep root
    cmd2 := exec.Command("grep", "root")

    // 命令 3: wc -l
    cmd3 := exec.Command("wc", "-l")

    // 连接管道
    var err error
    cmd2.Stdin, err = cmd1.StdoutPipe()
    if err != nil {
        panic(err)
    }

    cmd3.Stdin, err = cmd2.StdoutPipe()
    if err != nil {
        panic(err)
    }

    // 捕获最终输出
    var output bytes.Buffer
    cmd3.Stdout = &output

    // 按顺序启动（从后往前，避免管道阻塞）
    cmd3.Start()
    cmd2.Start()
    cmd1.Start()

    // 按顺序等待完成
    cmd1.Wait()
    cmd2.Wait()
    cmd3.Wait()

    fmt.Printf("包含 'root' 的行数: %s", output.String())
}
```

### 6.2 简化管道函数

```go
package main

import (
    "bytes"
    "fmt"
    "os/exec"
    "strings"
)

// pipeline 执行管道命令链
func pipeline(commands ...*exec.Cmd) (string, error) {
    if len(commands) == 0 {
        return "", fmt.Errorf("至少需要一个命令")
    }

    // 连接管道
    for i := 1; i < len(commands); i++ {
        pipe, err := commands[i-1].StdoutPipe()
        if err != nil {
            return "", fmt.Errorf("创建管道失败: %w", err)
        }
        commands[i].Stdin = pipe
    }

    // 捕获最后一个命令的输出
    var output bytes.Buffer
    var stderr bytes.Buffer
    commands[len(commands)-1].Stdout = &output
    commands[len(commands)-1].Stderr = &stderr

    // 启动所有命令（从最后一个开始）
    for i := len(commands) - 1; i >= 0; i-- {
        if err := commands[i].Start(); err != nil {
            return "", fmt.Errorf("启动命令 %d 失败: %w", i, err)
        }
    }

    // 等待所有命令完成
    for i, cmd := range commands {
        if err := cmd.Wait(); err != nil {
            return "", fmt.Errorf("命令 %d 执行失败: %w (stderr: %s)", i, err, stderr.String())
        }
    }

    return strings.TrimSpace(output.String()), nil
}

func main() {
    // 示例 1: ps aux | grep -v grep | wc -l
    result, err := pipeline(
        exec.Command("ps", "aux"),
        exec.Command("wc", "-l"),
    )
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Println("总进程数:", result)
    }

    // 示例 2: cat /etc/passwd | sort | head -5
    result2, err := pipeline(
        exec.Command("cat", "/etc/passwd"),
        exec.Command("sort"),
        exec.Command("head", "-5"),
    )
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Println("\n/etc/passwd 前5行(排序后):")
        fmt.Println(result2)
    }

    // 示例 3: df -h | grep /dev
    result3, err := pipeline(
        exec.Command("df", "-h"),
        exec.Command("grep", "/dev"),
    )
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Println("\n磁盘使用情况:")
        fmt.Println(result3)
    }
}
```

---

## 7. 实时输出读取（StdoutPipe）

### 7.1 逐行读取命令输出

```go
package main

import (
    "bufio"
    "fmt"
    "os/exec"
    "time"
)

func main() {
    // 模拟实时读取长时间运行命令的输出
    // 使用 ping 命令作为示例（发送 5 个包）
    cmd := exec.Command("ping", "-c", "5", "127.0.0.1")

    // 获取标准输出管道
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        panic(err)
    }

    // 启动命令
    if err := cmd.Start(); err != nil {
        panic(err)
    }

    // 实时逐行读取
    scanner := bufio.NewScanner(stdout)
    lineNum := 0
    for scanner.Scan() {
        lineNum++
        fmt.Printf("[%s] 行%d: %s\n",
            time.Now().Format("15:04:05.000"),
            lineNum,
            scanner.Text(),
        )
    }

    // 等待命令完成
    if err := cmd.Wait(); err != nil {
        fmt.Println("命令退出错误:", err)
    } else {
        fmt.Println("命令执行完成")
    }
}
```

### 7.2 同时读取 stdout 和 stderr

```go
package main

import (
    "bufio"
    "fmt"
    "os/exec"
    "sync"
)

func main() {
    // 执行一个会同时产生 stdout 和 stderr 的命令
    cmd := exec.Command("bash", "-c", `
        echo "stdout: 第1行"
        echo "stderr: 错误信息" >&2
        echo "stdout: 第2行"
        echo "stderr: 警告信息" >&2
        echo "stdout: 第3行"
    `)

    stdout, _ := cmd.StdoutPipe()
    stderr, _ := cmd.StderrPipe()

    cmd.Start()

    var wg sync.WaitGroup

    // 读取 stdout
    wg.Add(1)
    go func() {
        defer wg.Done()
        scanner := bufio.NewScanner(stdout)
        for scanner.Scan() {
            fmt.Printf("[STDOUT] %s\n", scanner.Text())
        }
    }()

    // 读取 stderr
    wg.Add(1)
    go func() {
        defer wg.Done()
        scanner := bufio.NewScanner(stderr)
        for scanner.Scan() {
            fmt.Printf("[STDERR] %s\n", scanner.Text())
        }
    }()

    wg.Wait()
    cmd.Wait()
    fmt.Println("\n命令执行完成")
}
```

### 7.3 交互式命令（stdin + stdout）

```go
package main

import (
    "bufio"
    "fmt"
    "io"
    "os/exec"
)

func main() {
    // 通过 stdin 写入数据，从 stdout 读取结果
    cmd := exec.Command("sort")

    // 获取 stdin
    stdin, err := cmd.StdinPipe()
    if err != nil {
        panic(err)
    }

    // 获取 stdout
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        panic(err)
    }

    // 启动命令
    cmd.Start()

    // 写入数据到 stdin
    go func() {
        defer stdin.Close() // 关闭 stdin 表示输入结束
        data := []string{
            "nginx",
            "apache",
            "redis",
            "mysql",
            "postgres",
            "elasticsearch",
        }
        for _, line := range data {
            io.WriteString(stdin, line+"\n")
        }
    }()

    // 读取排序后的输出
    fmt.Println("排序结果:")
    scanner := bufio.NewScanner(stdout)
    for scanner.Scan() {
        fmt.Println(" ", scanner.Text())
    }

    cmd.Wait()
}
```

---

## 8. SRE 实战场景

### 8.1 服务健康检查脚本

```go
package main

import (
    "context"
    "fmt"
    "os/exec"
    "strings"
    "time"
)

type ServiceCheck struct {
    Name    string
    Command string
    Args    []string
}

type CheckResult struct {
    Name    string
    Status  string
    Output  string
    Elapsed time.Duration
}

func checkService(ctx context.Context, check ServiceCheck) CheckResult {
    start := time.Now()

    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    cmd := exec.CommandContext(ctx, check.Command, check.Args...)
    output, err := cmd.CombinedOutput()

    result := CheckResult{
        Name:    check.Name,
        Output:  strings.TrimSpace(string(output)),
        Elapsed: time.Since(start),
    }

    if ctx.Err() == context.DeadlineExceeded {
        result.Status = "TIMEOUT"
    } else if err != nil {
        result.Status = "FAIL"
    } else {
        result.Status = "OK"
    }

    return result
}

func main() {
    checks := []ServiceCheck{
        {Name: "磁盘空间", Command: "df", Args: []string{"-h", "/"}},
        {Name: "内存使用", Command: "free", Args: []string{"-h"}},
        {Name: "系统运行时间", Command: "uptime"},
        {Name: "DNS 解析", Command: "nslookup", Args: []string{"localhost"}},
        {Name: "本地连通性", Command: "ping", Args: []string{"-c", "1", "-W", "2", "127.0.0.1"}},
    }

    fmt.Println("=== 服务健康检查 ===")
    fmt.Printf("时间: %s\n\n", time.Now().Format("2006-01-02 15:04:05"))

    ctx := context.Background()
    for _, check := range checks {
        result := checkService(ctx, check)
        statusIcon := "+"
        if result.Status != "OK" {
            statusIcon = "-"
        }
        fmt.Printf("[%s] %s: %s (耗时 %v)\n", statusIcon, result.Name, result.Status, result.Elapsed.Round(time.Millisecond))

        // 失败时显示详细输出
        if result.Status != "OK" && result.Output != "" {
            fmt.Printf("    输出: %s\n", result.Output)
        }
    }
}
```

### 8.2 批量远程命令执行器

```go
package main

import (
    "context"
    "fmt"
    "os/exec"
    "strings"
    "sync"
    "time"
)

type RemoteHost struct {
    Name string
    Addr string
}

type RemoteResult struct {
    Host    string
    Output  string
    Error   string
    Elapsed time.Duration
}

// 通过 ssh 执行远程命令（示例用本地命令替代）
func executeRemote(ctx context.Context, host RemoteHost, command string) RemoteResult {
    start := time.Now()

    // 实际场景中使用 ssh:
    // cmd := exec.CommandContext(ctx, "ssh", "-o", "StrictHostKeyChecking=no",
    //     "-o", "ConnectTimeout=5", host.Addr, command)
    // 这里用本地命令模拟
    cmd := exec.CommandContext(ctx, "bash", "-c", command)

    output, err := cmd.CombinedOutput()

    result := RemoteResult{
        Host:    host.Name,
        Output:  strings.TrimSpace(string(output)),
        Elapsed: time.Since(start),
    }

    if err != nil {
        result.Error = err.Error()
    }

    return result
}

func main() {
    hosts := []RemoteHost{
        {Name: "web-01", Addr: "10.0.1.1"},
        {Name: "web-02", Addr: "10.0.1.2"},
        {Name: "db-01", Addr: "10.0.2.1"},
    }

    command := "hostname && uptime"
    maxConcurrent := 5

    fmt.Printf("在 %d 台主机上执行: %s\n\n", len(hosts), command)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 并发限制
    sem := make(chan struct{}, maxConcurrent)
    var mu sync.Mutex
    var results []RemoteResult
    var wg sync.WaitGroup

    for _, host := range hosts {
        wg.Add(1)
        go func(h RemoteHost) {
            defer wg.Done()
            sem <- struct{}{}        // 获取信号量
            defer func() { <-sem }() // 释放信号量

            result := executeRemote(ctx, h, command)
            mu.Lock()
            results = append(results, result)
            mu.Unlock()
        }(host)
    }

    wg.Wait()

    // 打印结果
    for _, r := range results {
        fmt.Printf("=== %s ===\n", r.Host)
        if r.Error != "" {
            fmt.Printf("  错误: %s\n", r.Error)
        }
        if r.Output != "" {
            for _, line := range strings.Split(r.Output, "\n") {
                fmt.Printf("  %s\n", line)
            }
        }
        fmt.Printf("  耗时: %v\n\n", r.Elapsed.Round(time.Millisecond))
    }
}
```

### 8.3 进程监控与自动重启

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "time"
)

// 简单的进程看守器（类似 supervisord 的核心逻辑）
func supervise(name string, args []string, maxRestarts int) {
    restartCount := 0

    for restartCount <= maxRestarts {
        fmt.Printf("[Supervisor] 启动进程: %s %v (第 %d 次)\n", name, args, restartCount+1)

        cmd := exec.Command(name, args...)
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stderr

        start := time.Now()
        err := cmd.Run()
        elapsed := time.Since(start)

        if err != nil {
            fmt.Printf("[Supervisor] 进程退出: %v (运行了 %v)\n", err, elapsed)
        } else {
            fmt.Printf("[Supervisor] 进程正常退出 (运行了 %v)\n", elapsed)
            return // 正常退出，不重启
        }

        restartCount++

        if restartCount <= maxRestarts {
            // 退避策略：重启间隔递增
            backoff := time.Duration(restartCount) * 2 * time.Second
            fmt.Printf("[Supervisor] %v 后重启...\n\n", backoff)
            time.Sleep(backoff)
        }
    }

    fmt.Printf("[Supervisor] 达到最大重启次数 (%d)，放弃重启\n", maxRestarts)
}

func main() {
    // 示例：监控一个会失败的命令
    supervise("bash", []string{"-c", "echo '服务运行中...'; sleep 1; exit 1"}, 3)
}
```

---

## 9. 对比 Python subprocess / Shell 脚本

### 9.1 执行命令

| 功能 | Go os/exec | Python subprocess | Shell |
|------|------------|-------------------|-------|
| 运行命令 | `exec.Command("ls","-la").Run()` | `subprocess.run(["ls","-la"])` | `ls -la` |
| 获取输出 | `cmd.Output()` | `subprocess.check_output(...)` | `output=$(cmd)` |
| stdout+stderr | `cmd.CombinedOutput()` | `subprocess.run(..., capture_output=True)` | `cmd 2>&1` |
| 设置超时 | `exec.CommandContext(ctx,...)` | `subprocess.run(..., timeout=5)` | `timeout 5 cmd` |
| 设置工作目录 | `cmd.Dir = "/tmp"` | `subprocess.run(..., cwd="/tmp")` | `cd /tmp && cmd` |
| 设置环境变量 | `cmd.Env = append(os.Environ(),...)` | `subprocess.run(..., env={...})` | `KEY=val cmd` |
| 管道连接 | 手动连接 StdoutPipe → Stdin | `subprocess.PIPE` | `cmd1 \| cmd2` |
| 异步执行 | `cmd.Start()` + `cmd.Wait()` | `subprocess.Popen(...)` | `cmd &` |

### 9.2 代码对比

```go
// Go: 执行命令并获取输出
output, err := exec.Command("kubectl", "get", "pods", "-o", "json").Output()
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(output))
```

```python
# Python: 等效代码
result = subprocess.run(["kubectl", "get", "pods", "-o", "json"],
                       capture_output=True, text=True, check=True)
print(result.stdout)
```

```bash
# Shell: 等效代码
kubectl get pods -o json
```

### 9.3 信号处理对比

```go
// Go
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
<-sigChan
// 清理...
```

```python
# Python
import signal
def handler(signum, frame):
    # 清理...
    sys.exit(0)
signal.signal(signal.SIGINT, handler)
signal.signal(signal.SIGTERM, handler)
```

```bash
# Shell
trap 'echo "清理..."; exit 0' INT TERM
```

---

## 常见坑点

### 坑 1：Shell 命令不能直接传

```go
// 错误：不能直接传 shell 管道/重定向
cmd := exec.Command("ls -la | grep go")
// exec 会把整个字符串当作一个命令名！

// 正确方式一：用 bash -c 包装
cmd := exec.Command("bash", "-c", "ls -la | grep go")

// 正确方式二：分别执行并手动连接管道
cmd1 := exec.Command("ls", "-la")
cmd2 := exec.Command("grep", "go")
```

### 坑 2：Command 不可复用

```go
// 错误：同一个 Cmd 不能执行两次
cmd := exec.Command("echo", "hello")
cmd.Run()
cmd.Run() // panic: exec: already started

// 正确：每次创建新的 Cmd
exec.Command("echo", "hello").Run()
exec.Command("echo", "world").Run()
```

### 坑 3：忘记读取 stdout/stderr 导致死锁

```go
// 如果命令输出大量数据到 stdout/stderr，
// 但没有人读取，管道缓冲区会满，命令阻塞

// 错误：
cmd := exec.Command("find", "/", "-name", "*.go")
cmd.Start()
cmd.Wait() // 可能死锁！

// 正确：确保有人读取管道
cmd := exec.Command("find", "/", "-name", "*.go")
cmd.Stdout = os.Stdout // 转发到标准输出
cmd.Stderr = os.Stderr
cmd.Start()
cmd.Wait()
```

### 坑 4：CommandContext 不保证子进程被杀

```go
// exec.CommandContext 超时时只杀主进程
// 如果主进程 fork 了子进程，子进程可能成为孤儿进程

// 解决：使用 Process Group
cmd := exec.CommandContext(ctx, "bash", "-c", "sleep 100")
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
cmd.Start()
// 超时后杀掉整个进程组：
// syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)
```

### 坑 5：os.Exit 不执行 defer

```go
func main() {
    f, _ := os.Create("/tmp/data.txt")
    defer f.Close() // os.Exit 后不执行！

    if err := doWork(); err != nil {
        fmt.Println("错误:", err)
        os.Exit(1) // f.Close() 不会被调用
    }
}

// 正确做法：在 run() 函数中用 return，main 根据返回值 exit
func run() error {
    f, _ := os.Create("/tmp/data.txt")
    defer f.Close() // 正常执行

    return doWork()
}

func main() {
    if err := run(); err != nil {
        fmt.Println("错误:", err)
        os.Exit(1)
    }
}
```

---

## 速查表

```text
┌──────────────────────────────────────────────────────────────────────┐
│                    os / exec 速查表                                   │
├──────────────────────────────────────────────────────────────────────┤
│ 命令行参数                                                            │
│   程序路径       │ os.Args[0]                                        │
│   所有参数       │ os.Args[1:]                                       │
│   flag 解析      │ flag.String/Int/Bool → flag.Parse()               │
├──────────────────────────────────────────────────────────────────────┤
│ 环境变量                                                              │
│   读取           │ os.Getenv("KEY")                                  │
│   读取(判断存在) │ val, ok := os.LookupEnv("KEY")                   │
│   设置           │ os.Setenv("KEY", "VALUE")                         │
│   清除           │ os.Unsetenv("KEY")                                │
│   全部           │ os.Environ() → []string{"KEY=VALUE"}             │
├──────────────────────────────────────────────────────────────────────┤
│ 进程                                                                  │
│   退出           │ os.Exit(code)  (不执行 defer!)                    │
│   PID            │ os.Getpid()                                       │
│   父PID          │ os.Getppid()                                      │
│   主机名         │ os.Hostname()                                     │
├──────────────────────────────────────────────────────────────────────┤
│ 信号处理                                                              │
│   注册           │ signal.Notify(ch, syscall.SIGINT, SIGTERM)        │
│   context版      │ signal.NotifyContext(ctx, SIGINT, SIGTERM)        │
│   忽略信号       │ signal.Ignore(syscall.SIGHUP)                     │
│   重置           │ signal.Reset(syscall.SIGINT)                      │
├──────────────────────────────────────────────────────────────────────┤
│ 执行命令                                                              │
│   创建命令       │ exec.Command("ls", "-la")                         │
│   带超时         │ exec.CommandContext(ctx, "cmd", args...)           │
│   执行(无输出)   │ cmd.Run()                                         │
│   执行(获取输出) │ cmd.Output()                                      │
│   执行(stdout+err)│ cmd.CombinedOutput()                            │
│   异步执行       │ cmd.Start() → cmd.Wait()                         │
│   实时输出       │ cmd.StdoutPipe() → bufio.NewScanner(pipe)        │
│   设置工作目录   │ cmd.Dir = "/path"                                 │
│   设置环境变量   │ cmd.Env = append(os.Environ(), "K=V")            │
│   传入 stdin     │ cmd.Stdin = strings.NewReader("data")             │
│   Shell 命令     │ exec.Command("bash", "-c", "cmd1 | cmd2")        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 小结

- `os.Args` 适合简单的命令行工具；复杂场景用 `flag` 包或第三方库（如 `cobra`）。
- 环境变量用 `os.LookupEnv` 区分 "未设置" 和 "设置为空"。
- `os.Exit` 不执行 `defer`，应将逻辑放入 `run()` 函数，`main()` 仅做退出处理。
- 信号处理是 SRE 工具的必备功能，`signal.NotifyContext`（Go 1.16+）最简洁。
- `exec.Command` 执行外部命令时，务必设置超时（`CommandContext`）、读取输出（避免死锁）。
- Shell 管道需要用 `bash -c` 包装或手动连接 `StdoutPipe`。
- `Cmd` 对象不可复用，每次调用需重新创建。
