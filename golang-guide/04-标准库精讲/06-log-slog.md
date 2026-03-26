# 06 - log 与 slog 日志

> **适用版本**: Go 1.24 | **面向读者**: 有编程经验的 SRE / DevOps 工程师

---

## 导读

日志是 SRE 的"眼睛"——线上问题排查、审计追踪、性能分析都离不开日志。Go 标准库提供了两套日志方案：传统的 `log` 包和 Go 1.21 引入的 `log/slog` 结构化日志包。`slog` 代表了现代日志的发展方向：结构化、可搜索、高性能。

本篇涵盖：

1. `log` 标准包 —— Println / Printf / Fatal / Panic、自定义 Logger、输出到文件
2. `log/slog` 结构化日志（Go 1.21+）
3. slog Handler: TextHandler / JSONHandler
4. slog.With 和 LogValuer 接口
5. 自定义 slog Handler
6. 日志级别控制
7. slog.Group 分组属性
8. 日志上下文传递
9. SRE 实战场景
10. 对比 Python logging / loguru

---

## 1. log 标准包

### 1.1 基本使用

```go
package main

import (
    "log"
)

func main() {
    // 默认输出到 os.Stderr，前缀为日期时间
    log.Println("服务启动")
    log.Println("监听端口 8080")

    // Printf 格式化输出
    host := "10.0.1.1"
    port := 8080
    log.Printf("服务启动在 %s:%d\n", host, port)

    // Print 不自动换行
    log.Print("这是一条普通日志\n")

    // Fatal: 打印日志后调用 os.Exit(1)
    // log.Fatal("致命错误: 无法连接数据库")  // 会导致程序退出

    // Panic: 打印日志后调用 panic()
    // log.Panic("不可恢复的错误")  // 会触发 panic
}
// 输出示例：
// 2024/06/15 10:30:00 服务启动
// 2024/06/15 10:30:00 监听端口 8080
// 2024/06/15 10:30:00 服务启动在 10.0.1.1:8080
```

### 1.2 设置 log 标志（Flags）

```go
package main

import (
    "log"
)

func main() {
    // 默认 flags: Ldate | Ltime
    log.Println("默认格式") // 2024/06/15 10:30:00 默认格式

    // 设置更详细的标志
    log.SetFlags(log.Ldate | log.Ltime | log.Lmicroseconds | log.Lshortfile)
    log.Println("带微秒和文件名")
    // 2024/06/15 10:30:00.123456 main.go:12: 带微秒和文件名

    // 常用标志组合
    // log.Ldate          日期 2024/06/15
    // log.Ltime          时间 10:30:00
    // log.Lmicroseconds  微秒 10:30:00.123456
    // log.Llongfile      完整文件路径和行号 /home/user/main.go:12
    // log.Lshortfile     短文件名和行号 main.go:12
    // log.LUTC           使用 UTC 时区
    // log.Lmsgprefix     将前缀放在消息前而不是行首

    // 设置前缀
    log.SetPrefix("[SRE-TOOL] ")
    log.Println("带前缀的日志")
    // [SRE-TOOL] 2024/06/15 10:30:00.123456 main.go:25: 带前缀的日志

    // 只显示时间（不显示日期）
    log.SetFlags(log.Ltime | log.Lmicroseconds)
    log.SetPrefix("")
    log.Println("简洁格式")
    // 10:30:00.123456 简洁格式
}
```

### 1.3 自定义 Logger

```go
package main

import (
    "log"
    "os"
)

func main() {
    // 创建不同级别的 Logger
    infoLogger := log.New(os.Stdout, "[INFO]  ", log.Ldate|log.Ltime|log.Lshortfile)
    warnLogger := log.New(os.Stdout, "[WARN]  ", log.Ldate|log.Ltime|log.Lshortfile)
    errorLogger := log.New(os.Stderr, "[ERROR] ", log.Ldate|log.Ltime|log.Lshortfile)

    infoLogger.Println("服务启动成功")
    infoLogger.Printf("监听端口: %d\n", 8080)
    warnLogger.Println("内存使用率偏高: 85%")
    errorLogger.Println("数据库连接超时")

    // 输出:
    // [INFO]  2024/06/15 10:30:00 main.go:12: 服务启动成功
    // [INFO]  2024/06/15 10:30:00 main.go:13: 监听端口: 8080
    // [WARN]  2024/06/15 10:30:00 main.go:14: 内存使用率偏高: 85%
    // [ERROR] 2024/06/15 10:30:00 main.go:15: 数据库连接超时
}
```

### 1.4 输出到文件

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    // ---- 输出到文件 ----
    logFile, err := os.OpenFile("/tmp/app.log",
        os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal("打开日志文件失败:", err)
    }
    defer logFile.Close()

    fileLogger := log.New(logFile, "", log.Ldate|log.Ltime|log.Lmicroseconds)
    fileLogger.Println("这条日志写入文件")
    fileLogger.Printf("磁盘使用: %d%%\n", 75)

    // ---- 同时输出到文件和控制台 ----
    multiWriter := io.MultiWriter(os.Stdout, logFile)
    multiLogger := log.New(multiWriter, "[MULTI] ", log.Ldate|log.Ltime)
    multiLogger.Println("同时输出到控制台和文件")

    // ---- 修改默认 logger 的输出目标 ----
    log.SetOutput(multiWriter)
    log.SetPrefix("[DEFAULT] ")
    log.Println("默认 logger 也同时输出了")

    // 读取日志文件验证
    data, _ := os.ReadFile("/tmp/app.log")
    fmt.Println("\n=== 日志文件内容 ===")
    fmt.Println(string(data))
}
```

### 1.5 log 包的局限性

```go
// log 包的问题：
// 1. 没有日志级别（DEBUG, INFO, WARN, ERROR）
// 2. 非结构化——纯文本，难以解析和搜索
// 3. 无法方便地添加上下文字段（如 request_id, user_id）
// 4. 无法轻松切换输出格式（text/json）
// 5. 没有采样、限速等高级功能

// 这就是为什么 Go 1.21 引入了 log/slog
```

---

## 2. log/slog 结构化日志（Go 1.21+）

### 2.1 基本使用

```go
package main

import (
    "log/slog"
)

func main() {
    // 默认使用 TextHandler，输出到 os.Stderr
    slog.Info("服务启动",
        "host", "10.0.1.1",
        "port", 8080,
    )

    slog.Warn("内存使用率偏高",
        "usage_percent", 85.5,
        "threshold", 80.0,
    )

    slog.Error("数据库连接失败",
        "host", "db-master",
        "port", 5432,
        "error", "connection refused",
    )

    slog.Debug("调试信息", "key", "value")
    // Debug 级别默认不输出（默认级别为 Info）
}
// 输出示例：
// time=2024-06-15T10:30:00.000+08:00 level=INFO msg=服务启动 host=10.0.1.1 port=8080
// time=2024-06-15T10:30:00.000+08:00 level=WARN msg=内存使用率偏高 usage_percent=85.5 threshold=80
// time=2024-06-15T10:30:00.000+08:00 level=ERROR msg=数据库连接失败 host=db-master port=5432 error="connection refused"
```

### 2.2 强类型属性 —— slog.Attr

```go
package main

import (
    "log/slog"
    "time"
)

func main() {
    // 使用 slog.Attr 可以避免 key/value 配对错误
    slog.Info("请求处理完成",
        slog.String("method", "GET"),
        slog.String("path", "/api/users"),
        slog.Int("status", 200),
        slog.Duration("latency", 150*time.Millisecond),
        slog.Bool("cached", true),
        slog.Float64("response_size_kb", 12.5),
        slog.Time("timestamp", time.Now()),
    )

    // slog.Any: 任意类型
    tags := []string{"production", "web", "api"}
    slog.Info("服务信息",
        slog.Any("tags", tags),
    )
}
```

---

## 3. Handler: TextHandler / JSONHandler

### 3.1 TextHandler

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // TextHandler: 人类可读的文本格式
    handler := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug, // 显示所有级别
    })
    logger := slog.New(handler)

    logger.Debug("调试信息", "module", "auth")
    logger.Info("用户登录", "user", "admin", "ip", "192.168.1.1")
    logger.Warn("磁盘空间不足", "disk", "/dev/sda1", "usage", 92)
    logger.Error("服务不可用", "service", "redis", "error", "connection timeout")
}
// 输出：
// time=... level=DEBUG msg=调试信息 module=auth
// time=... level=INFO msg=用户登录 user=admin ip=192.168.1.1
// time=... level=WARN msg=磁盘空间不足 disk=/dev/sda1 usage=92
// time=... level=ERROR msg=服务不可用 service=redis error="connection timeout"
```

### 3.2 JSONHandler

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // JSONHandler: 结构化 JSON 格式（推荐生产环境使用）
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    logger := slog.New(handler)

    logger.Info("服务启动",
        "host", "10.0.1.1",
        "port", 8080,
        "version", "1.0.0",
    )

    logger.Error("请求失败",
        "method", "POST",
        "path", "/api/deploy",
        "status", 500,
        "error", "deployment timeout",
    )
}
// 输出：
// {"time":"2024-06-15T10:30:00.000+08:00","level":"INFO","msg":"服务启动","host":"10.0.1.1","port":8080,"version":"1.0.0"}
// {"time":"2024-06-15T10:30:00.000+08:00","level":"ERROR","msg":"请求失败","method":"POST","path":"/api/deploy","status":500,"error":"deployment timeout"}
```

### 3.3 设置为默认 Logger

```go
package main

import (
    "log"
    "log/slog"
    "os"
)

func main() {
    // 创建 JSON Logger 并设为全局默认
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelInfo,
        AddSource: true, // 添加源代码位置
    })
    logger := slog.New(handler)
    slog.SetDefault(logger)

    // 现在 slog.Info 等全局函数都使用 JSON 格式
    slog.Info("全局日志测试", "key", "value")

    // 传统 log 包也会通过 slog 输出
    log.Println("传统 log 包的输出也变成 JSON 了")
}
```

---

## 4. slog.With —— 预设属性

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    baseHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })

    // With: 创建带预设属性的 Logger
    // 每条日志自动附带这些属性
    logger := slog.New(baseHandler).With(
        "service", "api-gateway",
        "version", "2.1.0",
        "env", "production",
    )

    logger.Info("服务启动", "port", 8080)
    logger.Warn("流量激增", "qps", 5000)
    logger.Error("上游超时", "upstream", "user-service")

    // 继续叠加属性
    requestLogger := logger.With(
        "request_id", "req-abc-123",
        "user_id", "user-456",
    )
    requestLogger.Info("处理请求", "method", "GET", "path", "/api/users")
    requestLogger.Info("请求完成", "status", 200, "latency_ms", 45)
}
// 每条日志都自动包含 service, version, env 字段
// requestLogger 的日志还额外包含 request_id, user_id
```

---

## 5. LogValuer 接口

```go
package main

import (
    "log/slog"
    "os"
    "time"
)

// LogValuer 接口：自定义类型的日志输出格式
type User struct {
    ID       string
    Name     string
    Email    string
    Password string // 敏感信息，不应出现在日志中
}

// 实现 LogValuer 接口
func (u User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.String("id", u.ID),
        slog.String("name", u.Name),
        // 注意：故意不包含 Password 和 Email
        slog.String("email", maskEmail(u.Email)),
    )
}

func maskEmail(email string) string {
    for i, c := range email {
        if c == '@' {
            if i > 2 {
                return email[:2] + "***" + email[i:]
            }
            return "***" + email[i:]
        }
    }
    return "***"
}

// 敏感数据包装器
type Secret string

func (s Secret) LogValue() slog.Value {
    return slog.StringValue("[REDACTED]")
}

// 延迟计算的值
type LazyValue struct {
    compute func() interface{}
}

func (lv LazyValue) LogValue() slog.Value {
    return slog.AnyValue(lv.compute())
}

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    logger := slog.New(handler)

    user := User{
        ID:       "u-123",
        Name:     "张三",
        Email:    "zhangsan@example.com",
        Password: "super-secret-123",
    }

    // LogValuer 自动脱敏
    logger.Info("用户登录", "user", user)

    // Secret 类型自动隐藏
    apiKey := Secret("sk-abc123def456ghi789")
    logger.Info("API 调用", "api_key", apiKey)

    // 延迟计算
    logger.Info("系统状态",
        "uptime", LazyValue{compute: func() interface{} {
            return time.Since(time.Now().Add(-24 * time.Hour)).String()
        }},
    )
}
```

---

## 6. 自定义 slog Handler

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log/slog"
    "os"
    "runtime"
    "sync"
    "time"
)

// 自定义 Handler：带颜色的控制台输出
type ColorHandler struct {
    opts   slog.HandlerOptions
    output io.Writer
    attrs  []slog.Attr
    groups []string
    mu     sync.Mutex
}

func NewColorHandler(w io.Writer, opts *slog.HandlerOptions) *ColorHandler {
    if opts == nil {
        opts = &slog.HandlerOptions{}
    }
    return &ColorHandler{opts: *opts, output: w}
}

// 颜色常量
const (
    colorReset  = "\033[0m"
    colorRed    = "\033[31m"
    colorGreen  = "\033[32m"
    colorYellow = "\033[33m"
    colorBlue   = "\033[34m"
    colorGray   = "\033[90m"
)

func levelColor(level slog.Level) string {
    switch {
    case level >= slog.LevelError:
        return colorRed
    case level >= slog.LevelWarn:
        return colorYellow
    case level >= slog.LevelInfo:
        return colorGreen
    default:
        return colorBlue
    }
}

func (h *ColorHandler) Enabled(_ context.Context, level slog.Level) bool {
    minLevel := slog.LevelInfo
    if h.opts.Level != nil {
        minLevel = h.opts.Level.Level()
    }
    return level >= minLevel
}

func (h *ColorHandler) Handle(_ context.Context, r slog.Record) error {
    h.mu.Lock()
    defer h.mu.Unlock()

    // 时间
    timeStr := r.Time.Format("15:04:05.000")

    // 级别（带颜色）
    color := levelColor(r.Level)
    levelStr := fmt.Sprintf("%s%-5s%s", color, r.Level.String(), colorReset)

    // 消息
    msg := r.Message

    // 源文件
    sourceStr := ""
    if h.opts.AddSource {
        // 获取调用源
        fs := runtime.CallersFrames([]uintptr{r.PC})
        f, _ := fs.Next()
        sourceStr = fmt.Sprintf(" %s%s:%d%s", colorGray, f.File, f.Line, colorReset)
    }

    // 输出主行
    fmt.Fprintf(h.output, "%s%s%s %s %s%s",
        colorGray, timeStr, colorReset,
        levelStr, msg, sourceStr)

    // 输出预设属性
    for _, attr := range h.attrs {
        fmt.Fprintf(h.output, " %s%s%s=%v",
            colorGray, attr.Key, colorReset, attr.Value)
    }

    // 输出记录属性
    r.Attrs(func(a slog.Attr) bool {
        fmt.Fprintf(h.output, " %s%s%s=%v",
            colorGray, a.Key, colorReset, a.Value)
        return true
    })

    fmt.Fprintln(h.output)
    return nil
}

func (h *ColorHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &ColorHandler{
        opts:   h.opts,
        output: h.output,
        attrs:  append(h.attrs, attrs...),
        groups: h.groups,
    }
}

func (h *ColorHandler) WithGroup(name string) slog.Handler {
    return &ColorHandler{
        opts:   h.opts,
        output: h.output,
        attrs:  h.attrs,
        groups: append(h.groups, name),
    }
}

func main() {
    handler := NewColorHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelDebug,
        AddSource: false,
    })

    logger := slog.New(handler).With("service", "api")

    logger.Debug("初始化配置", "file", "config.yaml")
    logger.Info("服务启动", "port", 8080)
    logger.Warn("内存使用率偏高", "percent", 85.5)
    logger.Error("数据库连接失败", "host", "db-master", "error", "timeout")

    // 带更多上下文
    reqLogger := logger.With("request_id", "req-123")
    reqLogger.Info("处理请求", "method", "POST", "path", "/api/deploy")

    _ = time.Now() // 占位
}
```

---

## 7. 日志级别控制

### 7.1 静态级别设置

```go
package main

import (
    "fmt"
    "log/slog"
    "os"
)

func main() {
    // 设置不同级别
    levels := []slog.Level{
        slog.LevelDebug,
        slog.LevelInfo,
        slog.LevelWarn,
        slog.LevelError,
    }

    for _, level := range levels {
        handler := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: level,
        })
        logger := slog.New(handler)

        fmt.Printf("\n--- 最低级别: %s ---\n", level)

        logger.Debug("DEBUG 日志")
        logger.Info("INFO 日志")
        logger.Warn("WARN 日志")
        logger.Error("ERROR 日志")
    }
}
```

### 7.2 动态级别调整 —— slog.LevelVar

```go
package main

import (
    "fmt"
    "log/slog"
    "os"
    "time"
)

func main() {
    // LevelVar 允许在运行时动态调整日志级别
    logLevel := new(slog.LevelVar)
    logLevel.Set(slog.LevelInfo) // 初始级别

    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: logLevel,
    })
    logger := slog.New(handler)

    fmt.Println("=== 当前级别: INFO ===")
    logger.Debug("这条不会显示")
    logger.Info("这条会显示")

    // 动态降低到 DEBUG 级别（用于排查问题）
    fmt.Println("\n=== 切换到 DEBUG ===")
    logLevel.Set(slog.LevelDebug)
    logger.Debug("现在这条也会显示了")
    logger.Info("INFO 也会显示")

    // 提升到 ERROR 级别（减少日志量）
    fmt.Println("\n=== 切换到 ERROR ===")
    logLevel.Set(slog.LevelError)
    logger.Info("这条不会显示")
    logger.Error("只有 ERROR 会显示")

    // 实际场景：通过 HTTP API 动态调整日志级别
    // 参见 SRE 实战场景部分
    _ = time.Now()
}
```

### 7.3 自定义日志级别

```go
package main

import (
    "log/slog"
    "os"
)

// 自定义级别
const (
    LevelTrace  = slog.Level(-8) // 比 Debug 更详细
    LevelNotice = slog.Level(2)  // 介于 Info 和 Warn 之间
    LevelFatal  = slog.Level(12) // 比 Error 更严重
)

func main() {
    // 自定义级别名称
    opts := &slog.HandlerOptions{
        Level: LevelTrace,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            if a.Key == slog.LevelKey {
                level := a.Value.Any().(slog.Level)
                switch level {
                case LevelTrace:
                    a.Value = slog.StringValue("TRACE")
                case LevelNotice:
                    a.Value = slog.StringValue("NOTICE")
                case LevelFatal:
                    a.Value = slog.StringValue("FATAL")
                }
            }
            return a
        },
    }

    handler := slog.NewTextHandler(os.Stdout, opts)
    logger := slog.New(handler)

    logger.Log(nil, LevelTrace, "非常详细的追踪信息")
    logger.Debug("调试信息")
    logger.Info("一般信息")
    logger.Log(nil, LevelNotice, "需要注意但不是警告")
    logger.Warn("警告信息")
    logger.Error("错误信息")
    logger.Log(nil, LevelFatal, "致命错误")
}
```

---

## 8. slog.Group —— 分组属性

```go
package main

import (
    "log/slog"
    "os"
    "time"
)

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    logger := slog.New(handler)

    // 使用 Group 将相关属性分组
    logger.Info("HTTP 请求",
        slog.Group("request",
            slog.String("method", "POST"),
            slog.String("path", "/api/deploy"),
            slog.String("remote_addr", "192.168.1.100"),
        ),
        slog.Group("response",
            slog.Int("status", 200),
            slog.Duration("latency", 150*time.Millisecond),
            slog.Int64("bytes", 1024),
        ),
        slog.Group("user",
            slog.String("id", "u-123"),
            slog.String("role", "admin"),
        ),
    )
    // JSON 输出:
    // {
    //   "time": "...",
    //   "level": "INFO",
    //   "msg": "HTTP 请求",
    //   "request": {"method": "POST", "path": "/api/deploy", "remote_addr": "192.168.1.100"},
    //   "response": {"status": 200, "latency": 150000000, "bytes": 1024},
    //   "user": {"id": "u-123", "role": "admin"}
    // }

    // WithGroup: 为所有后续日志添加分组前缀
    httpLogger := logger.WithGroup("http")
    httpLogger.Info("请求到达",
        "method", "GET",
        "path", "/health",
    )
    // JSON 输出:
    // {"time":"...","level":"INFO","msg":"请求到达","http":{"method":"GET","path":"/health"}}
}
```

---

## 9. 日志上下文传递

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

// 使用 context 传递 logger 的模式

type ctxKey string

const loggerKey ctxKey = "logger"

// 将 logger 存入 context
func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

// 从 context 获取 logger
func FromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default() // 回退到默认 logger
}

// 模拟 HTTP Handler
func handleRequest(ctx context.Context) {
    logger := FromContext(ctx)
    logger.Info("开始处理请求")

    // 传递给下层
    processOrder(ctx)
}

func processOrder(ctx context.Context) {
    logger := FromContext(ctx)
    logger.Info("处理订单", "step", "validate")
    logger.Info("处理订单", "step", "save")

    // 进一步传递
    sendNotification(ctx)
}

func sendNotification(ctx context.Context) {
    logger := FromContext(ctx)
    logger.Info("发送通知", "channel", "email")
}

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })

    // 创建带请求上下文的 logger
    logger := slog.New(handler).With(
        "request_id", "req-abc-123",
        "user_id", "user-456",
        "trace_id", "trace-xyz-789",
    )

    ctx := WithLogger(context.Background(), logger)

    // 整个请求链路共享同一个 logger（带相同的 request_id）
    handleRequest(ctx)
}
// 所有日志都自动带有 request_id, user_id, trace_id
```

---

## 10. ReplaceAttr —— 属性替换

```go
package main

import (
    "log/slog"
    "os"
    "time"
)

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // 自定义时间格式
            if a.Key == slog.TimeKey {
                a.Value = slog.StringValue(a.Value.Time().Format("2006-01-02 15:04:05.000"))
            }
            // 自定义级别名称（全小写）
            if a.Key == slog.LevelKey {
                level := a.Value.Any().(slog.Level)
                switch level {
                case slog.LevelDebug:
                    a.Value = slog.StringValue("debug")
                case slog.LevelInfo:
                    a.Value = slog.StringValue("info")
                case slog.LevelWarn:
                    a.Value = slog.StringValue("warn")
                case slog.LevelError:
                    a.Value = slog.StringValue("error")
                }
            }
            // 移除某个属性
            if a.Key == "password" {
                return slog.Attr{} // 返回空 Attr 表示移除
            }
            return a
        },
    })

    logger := slog.New(handler)

    logger.Info("用户登录",
        "username", "admin",
        "password", "secret123", // 会被移除
        "ip", "192.168.1.1",
    )

    _ = time.Now()
}
// 输出（注意时间格式和级别名称）:
// {"time":"2024-06-15 10:30:00.000","level":"info","msg":"用户登录","username":"admin","ip":"192.168.1.1"}
// password 字段已被移除
```

---

## 11. SRE 实战场景

### 11.1 HTTP 请求日志中间件

```go
package main

import (
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "time"
)

// statusRecorder 记录 HTTP 状态码
type statusRecorder struct {
    http.ResponseWriter
    statusCode int
    bytes      int
}

func (sr *statusRecorder) WriteHeader(code int) {
    sr.statusCode = code
    sr.ResponseWriter.WriteHeader(code)
}

func (sr *statusRecorder) Write(b []byte) (int, error) {
    n, err := sr.ResponseWriter.Write(b)
    sr.bytes += n
    return n, err
}

// 结构化日志中间件
func slogMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // 包装 ResponseWriter
            recorder := &statusRecorder{
                ResponseWriter: w,
                statusCode:     200,
            }

            // 执行请求
            next.ServeHTTP(recorder, r)

            // 记录结构化日志
            duration := time.Since(start)
            level := slog.LevelInfo
            if recorder.statusCode >= 500 {
                level = slog.LevelError
            } else if recorder.statusCode >= 400 {
                level = slog.LevelWarn
            }

            logger.Log(r.Context(), level, "HTTP 请求",
                slog.Group("request",
                    slog.String("method", r.Method),
                    slog.String("path", r.URL.Path),
                    slog.String("query", r.URL.RawQuery),
                    slog.String("remote_addr", r.RemoteAddr),
                    slog.String("user_agent", r.UserAgent()),
                ),
                slog.Group("response",
                    slog.Int("status", recorder.statusCode),
                    slog.Int("bytes", recorder.bytes),
                    slog.Duration("duration", duration),
                ),
            )
        })
    }
}

func main() {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    logger := slog.New(handler).With("service", "api-gateway")

    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, `{"status":"ok"}`)
    })
    mux.HandleFunc("GET /api/users", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, `[{"name":"admin"}]`)
    })

    wrappedHandler := slogMiddleware(logger)(mux)

    fmt.Println("服务启动在 :8080 (结构化日志)")
    http.ListenAndServe(":8080", wrappedHandler)
}
```

### 11.2 动态日志级别 HTTP API

```go
package main

import (
    "encoding/json"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "strings"
)

func main() {
    // 动态级别控制
    logLevel := new(slog.LevelVar)
    logLevel.Set(slog.LevelInfo)

    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: logLevel,
    })
    logger := slog.New(handler)
    slog.SetDefault(logger)

    mux := http.NewServeMux()

    // GET /admin/log-level - 查看当前级别
    mux.HandleFunc("GET /admin/log-level", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{
            "level": logLevel.Level().String(),
        })
    })

    // PUT /admin/log-level - 动态修改级别
    mux.HandleFunc("PUT /admin/log-level", func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            Level string `json:"level"`
        }
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "JSON 解析失败", http.StatusBadRequest)
            return
        }

        switch strings.ToUpper(req.Level) {
        case "DEBUG":
            logLevel.Set(slog.LevelDebug)
        case "INFO":
            logLevel.Set(slog.LevelInfo)
        case "WARN":
            logLevel.Set(slog.LevelWarn)
        case "ERROR":
            logLevel.Set(slog.LevelError)
        default:
            http.Error(w, "无效的级别，可选: DEBUG, INFO, WARN, ERROR", http.StatusBadRequest)
            return
        }

        slog.Info("日志级别已修改", "new_level", logLevel.Level().String())
        json.NewEncoder(w).Encode(map[string]string{
            "level":   logLevel.Level().String(),
            "message": "日志级别已更新",
        })
    })

    // 测试路由
    mux.HandleFunc("GET /test", func(w http.ResponseWriter, r *http.Request) {
        slog.Debug("调试信息")
        slog.Info("普通信息")
        slog.Warn("警告信息")
        slog.Error("错误信息")
        fmt.Fprint(w, "日志已写入，检查控制台")
    })

    fmt.Println("服务启动在 :8080")
    fmt.Println("  GET  /admin/log-level     - 查看日志级别")
    fmt.Println("  PUT  /admin/log-level     - 修改日志级别")
    fmt.Println("  GET  /test                - 生成测试日志")
    http.ListenAndServe(":8080", mux)
}
```

### 11.3 多目标日志输出

```go
package main

import (
    "context"
    "io"
    "log/slog"
    "os"
)

// multiHandler 将日志同时输出到多个 Handler
type multiHandler struct {
    handlers []slog.Handler
}

func (mh *multiHandler) Enabled(ctx context.Context, level slog.Level) bool {
    for _, h := range mh.handlers {
        if h.Enabled(ctx, level) {
            return true
        }
    }
    return false
}

func (mh *multiHandler) Handle(ctx context.Context, r slog.Record) error {
    for _, h := range mh.handlers {
        if h.Enabled(ctx, r.Level) {
            if err := h.Handle(ctx, r.Clone()); err != nil {
                return err
            }
        }
    }
    return nil
}

func (mh *multiHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    handlers := make([]slog.Handler, len(mh.handlers))
    for i, h := range mh.handlers {
        handlers[i] = h.WithAttrs(attrs)
    }
    return &multiHandler{handlers: handlers}
}

func (mh *multiHandler) WithGroup(name string) slog.Handler {
    handlers := make([]slog.Handler, len(mh.handlers))
    for i, h := range mh.handlers {
        handlers[i] = h.WithGroup(name)
    }
    return &multiHandler{handlers: handlers}
}

func main() {
    // 控制台：Text 格式，DEBUG 级别
    consoleHandler := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    })

    // 文件：JSON 格式，INFO 级别
    logFile, err := os.OpenFile("/tmp/slog_app.log",
        os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        panic(err)
    }
    defer logFile.Close()

    fileHandler := slog.NewJSONHandler(logFile, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })

    // 组合
    logger := slog.New(&multiHandler{
        handlers: []slog.Handler{consoleHandler, fileHandler},
    })

    logger.Debug("调试信息 - 只在控制台")
    logger.Info("服务启动 - 控制台和文件都有")
    logger.Error("严重错误 - 控制台和文件都有")

    _ = io.Discard // 占位
}
```

### 11.4 审计日志

```go
package main

import (
    "encoding/json"
    "fmt"
    "log/slog"
    "os"
    "time"
)

type AuditEvent struct {
    Timestamp time.Time `json:"timestamp"`
    Actor     string    `json:"actor"`
    Action    string    `json:"action"`
    Resource  string    `json:"resource"`
    Details   string    `json:"details"`
    Result    string    `json:"result"` // success, failure
    SourceIP  string    `json:"source_ip"`
}

func (ae AuditEvent) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Time("timestamp", ae.Timestamp),
        slog.String("actor", ae.Actor),
        slog.String("action", ae.Action),
        slog.String("resource", ae.Resource),
        slog.String("details", ae.Details),
        slog.String("result", ae.Result),
        slog.String("source_ip", ae.SourceIP),
    )
}

func main() {
    // 审计日志专用 Logger
    auditFile, err := os.OpenFile("/tmp/audit.log",
        os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0600) // 严格权限
    if err != nil {
        panic(err)
    }
    defer auditFile.Close()

    auditLogger := slog.New(slog.NewJSONHandler(auditFile, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })).With("log_type", "audit")

    // 记录审计事件
    events := []AuditEvent{
        {
            Timestamp: time.Now(),
            Actor:     "admin@company.com",
            Action:    "deploy",
            Resource:  "production/web-api",
            Details:   "部署版本 v2.1.0",
            Result:    "success",
            SourceIP:  "10.0.1.100",
        },
        {
            Timestamp: time.Now(),
            Actor:     "dev@company.com",
            Action:    "scale",
            Resource:  "staging/worker",
            Details:   "副本数 3 → 5",
            Result:    "success",
            SourceIP:  "10.0.1.101",
        },
        {
            Timestamp: time.Now(),
            Actor:     "unknown@external.com",
            Action:    "delete",
            Resource:  "production/database",
            Details:   "尝试删除生产数据库",
            Result:    "denied",
            SourceIP:  "203.0.113.50",
        },
    }

    for _, event := range events {
        auditLogger.Info("审计事件", "event", event)
    }

    // 读取审计日志
    data, _ := os.ReadFile("/tmp/audit.log")
    fmt.Println("=== 审计日志 ===")
    // 格式化输出每行
    for _, line := range splitLines(string(data)) {
        if line == "" {
            continue
        }
        var m map[string]interface{}
        json.Unmarshal([]byte(line), &m)
        pretty, _ := json.MarshalIndent(m, "", "  ")
        fmt.Println(string(pretty))
    }
}

func splitLines(s string) []string {
    var lines []string
    start := 0
    for i := range s {
        if s[i] == '\n' {
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

---

## 12. 对比 Python logging / loguru

### 12.1 基本日志

```go
// Go slog
slog.Info("用户登录", "user", "admin", "ip", "192.168.1.1")
```

```python
# Python logging
import logging
logging.info("用户登录", extra={"user": "admin", "ip": "192.168.1.1"})

# Python loguru
from loguru import logger
logger.info("用户登录 user={user} ip={ip}", user="admin", ip="192.168.1.1")
```

### 12.2 功能对比

| 功能 | Go slog | Python logging | Python loguru |
|------|---------|----------------|---------------|
| 结构化日志 | 原生支持 | 需 json-log-formatter | 原生支持 |
| 日志级别 | Debug/Info/Warn/Error | DEBUG~CRITICAL (5级) | TRACE~CRITICAL (7级) |
| 动态级别 | `slog.LevelVar` | `logger.setLevel()` | `logger.level()` |
| JSON 输出 | `slog.NewJSONHandler` | 需第三方 Formatter | `serialize=True` |
| 分组属性 | `slog.Group` | 无 | 无 |
| 上下文传递 | `slog.With` / context | `LoggerAdapter` | `logger.bind()` |
| 敏感数据 | `LogValuer` 接口 | 自定义 Filter | 自定义 `format` |
| 文件轮转 | 需第三方 (lumberjack) | `RotatingFileHandler` | `rotation="500 MB"` |
| 性能 | 高 (零分配优化) | 中 | 中 |

### 12.3 配置对比

```go
// Go: 创建带 JSON 输出的 logger
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
```

```python
# Python logging: 等效配置
import logging
import json

handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter(
    '{"time":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s"}'
))
logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

```python
# Python loguru: 等效配置（更简洁）
from loguru import logger
logger.add(sys.stdout, format="{time} {level} {message}", serialize=True)
```

---

## 常见坑点

### 坑 1：slog 键值对必须成对出现

```go
// 错误：奇数个参数，最后一个会被当作 !BADKEY
slog.Info("测试", "key1", "value1", "key2") // key2 没有值！
// 输出: msg=测试 key1=value1 !BADKEY=key2

// 正确：始终成对传递
slog.Info("测试", "key1", "value1", "key2", "value2")

// 最佳实践：使用 slog.String/Int 等强类型函数避免配对错误
slog.Info("测试",
    slog.String("key1", "value1"),
    slog.String("key2", "value2"),
)
```

### 坑 2：log.Fatal / log.Panic 会中断程序

```go
// log.Fatal 调用 os.Exit(1)，defer 不执行
func handler() {
    f, _ := os.Open("file")
    defer f.Close() // 不会执行！

    log.Fatal("出错了") // 直接退出
}

// slog 没有 Fatal/Panic 方法（有意为之）
// 正确做法：记录错误日志后优雅退出
slog.Error("致命错误", "error", err)
os.Exit(1)
```

### 坑 3：日志文件未同步就退出

```go
// 如果使用 bufio.Writer 包装日志文件，
// 程序退出前需要 Flush/Sync
logFile.Sync()  // 确保日志刷到磁盘
logFile.Close()
// 或使用 defer 确保清理
```

### 坑 4：高频日志影响性能

```go
// 在热路径（如每个请求）中记录大量 DEBUG 日志
// 即使级别设为 INFO，参数仍会被求值

// 低效：即使不输出，也会执行 expensiveComputation()
slog.Debug("详细信息", "data", expensiveComputation())

// 高效：先检查级别
if slog.Default().Enabled(nil, slog.LevelDebug) {
    slog.Debug("详细信息", "data", expensiveComputation())
}

// 或使用 LogValuer 接口延迟计算
```

### 坑 5：slog.Default() 的 Handler 不是线程安全的自定义 Handler

```go
// 如果自定义 Handler 内部有可变状态，必须加锁
type MyHandler struct {
    mu     sync.Mutex // 必须保护共享状态
    output io.Writer
}

func (h *MyHandler) Handle(ctx context.Context, r slog.Record) error {
    h.mu.Lock()
    defer h.mu.Unlock()
    // 写入操作
    return nil
}
```

---

## 速查表

```text
┌──────────────────────────────────────────────────────────────────────┐
│                    log / slog 速查表                                  │
├──────────────────────────────────────────────────────────────────────┤
│ 传统 log 包                                                          │
│   基本输出       │ log.Println("msg") / log.Printf("fmt", args)     │
│   致命退出       │ log.Fatal("msg")  → 调用 os.Exit(1)              │
│   触发 panic     │ log.Panic("msg")  → 调用 panic()                 │
│   自定义 Logger  │ log.New(writer, prefix, flags)                    │
│   设置标志       │ log.SetFlags(log.Ldate | log.Ltime | ...)        │
│   设置前缀       │ log.SetPrefix("[APP] ")                           │
│   设置输出       │ log.SetOutput(writer)                             │
├──────────────────────────────────────────────────────────────────────┤
│ slog (Go 1.21+)                                                      │
│   基本输出       │ slog.Info("msg", "key", value)                    │
│   级别方法       │ slog.Debug / Info / Warn / Error                  │
│   强类型属性     │ slog.String("k","v") / slog.Int("k",1)           │
│   分组           │ slog.Group("name", slog.String(...), ...)         │
│   设置默认       │ slog.SetDefault(logger)                           │
│   预设属性       │ logger.With("key", "value")                       │
│   预设分组       │ logger.WithGroup("name")                          │
│   TextHandler    │ slog.NewTextHandler(w, opts)                      │
│   JSONHandler    │ slog.NewJSONHandler(w, opts)                      │
│   动态级别       │ var lv slog.LevelVar; lv.Set(slog.LevelDebug)    │
│   添加源码位置   │ HandlerOptions{AddSource: true}                   │
│   属性替换       │ HandlerOptions{ReplaceAttr: func(...){}}          │
│   自定义类型日志 │ 实现 LogValuer 接口                                │
│   检查级别       │ logger.Enabled(ctx, level)                        │
│   自定义级别     │ logger.Log(ctx, customLevel, "msg")               │
├──────────────────────────────────────────────────────────────────────┤
│ Handler 接口方法                                                      │
│   是否启用       │ Enabled(context.Context, slog.Level) bool         │
│   处理记录       │ Handle(context.Context, slog.Record) error        │
│   添加属性       │ WithAttrs([]slog.Attr) slog.Handler               │
│   添加分组       │ WithGroup(string) slog.Handler                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 小结

- **传统 `log` 包**适合简单脚本和小工具，但缺乏日志级别和结构化能力。
- **`log/slog`** 是 Go 官方推荐的现代日志方案（Go 1.21+），支持结构化、分级、可扩展的 Handler。
- 生产环境推荐使用 `JSONHandler`，便于日志采集工具（ELK、Loki、Datadog）解析。
- `slog.LevelVar` 支持**运行时动态调整日志级别**，非常适合线上排查问题。
- `LogValuer` 接口可以自动脱敏敏感数据（密码、Token、Email）。
- `slog.With` 创建带预设属性的 Logger，在请求链路中传递 request_id、trace_id 等上下文。
- `slog.Group` 将相关属性分组，在 JSON 输出中呈现为嵌套对象。
- 自定义 Handler 必须线程安全（加 `sync.Mutex`）。
- 日志文件轮转不在标准库范围内，推荐使用第三方库 `lumberjack`。
