# 02 - net/http 网络编程

> **适用版本**: Go 1.24 | **面向读者**: 有编程经验的 SRE / DevOps 工程师

---

## 导读

`net/http` 是 Go 标准库中最强大的包之一。不依赖任何第三方框架，你就能构建生产级的 HTTP 服务和客户端。对 SRE 而言，无论是编写健康检查探针、API 网关、还是运维工具的 Web 界面，`net/http` 都是首选。

本篇涵盖：

1. HTTP 客户端 —— Get/Post/Do、自定义 Client、超时、连接池、TLS
2. HTTP 服务端 —— ListenAndServe、Handler、HandlerFunc、DefaultServeMux
3. Go 1.22+ 增强路由 —— 方法匹配、路径参数、通配符
4. 中间件模式
5. 优雅关闭 —— http.Server.Shutdown
6. JSON API 完整示例
7. SRE 实战场景
8. 对比 Python requests / Flask

---

## 1. HTTP 客户端

### 1.1 快速请求 —— http.Get / http.Post

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "strings"
)

func main() {
    // ---- GET 请求 ----
    resp, err := http.Get("https://httpbin.org/get")
    if err != nil {
        fmt.Println("GET 请求失败:", err)
        return
    }
    defer resp.Body.Close() // 必须关闭，否则连接泄漏！

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("状态码:", resp.StatusCode)
    fmt.Println("响应体 (前200字符):", string(body[:min(200, len(body))]))

    // ---- POST 请求 ----
    payload := strings.NewReader(`{"name":"sre-tool","version":"1.0"}`)
    resp2, err := http.Post(
        "https://httpbin.org/post",
        "application/json",
        payload,
    )
    if err != nil {
        fmt.Println("POST 请求失败:", err)
        return
    }
    defer resp2.Body.Close()

    body2, _ := io.ReadAll(resp2.Body)
    fmt.Println("\nPOST 状态码:", resp2.StatusCode)
    fmt.Println("POST 响应体 (前200字符):", string(body2[:min(200, len(body2))]))
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

### 1.2 自定义请求 —— http.NewRequest + Client.Do

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "time"
)

func main() {
    // 构造请求
    req, err := http.NewRequest("GET", "https://httpbin.org/headers", nil)
    if err != nil {
        panic(err)
    }

    // 设置请求头
    req.Header.Set("User-Agent", "SRE-Monitor/1.0")
    req.Header.Set("Authorization", "Bearer my-token-123")
    req.Header.Set("Accept", "application/json")

    // 自定义 Client（生产环境必须设置超时！）
    client := &http.Client{
        Timeout: 10 * time.Second, // 总超时（包括连接、TLS握手、读响应体）
    }

    resp, err := client.Do(req)
    if err != nil {
        fmt.Println("请求失败:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("状态码:", resp.StatusCode)
    fmt.Println("响应:", string(body))
}
```

### 1.3 超时控制详解

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net"
    "net/http"
    "time"
)

func main() {
    // 方式一：Client.Timeout（推荐，最简单）
    client1 := &http.Client{
        Timeout: 5 * time.Second, // 整个请求的总超时
    }
    _ = client1

    // 方式二：Transport 级别精细控制
    transport := &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   3 * time.Second, // TCP 连接超时
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   5 * time.Second,  // TLS 握手超时
        ResponseHeaderTimeout: 10 * time.Second,  // 等待响应头超时
        IdleConnTimeout:       90 * time.Second,  // 空闲连接超时
        MaxIdleConns:          100,                // 最大空闲连接数
        MaxIdleConnsPerHost:   10,                 // 每个主机最大空闲连接数
        MaxConnsPerHost:       20,                 // 每个主机最大连接数
    }

    client2 := &http.Client{
        Transport: transport,
        Timeout:   30 * time.Second,
    }

    // 方式三：通过 context 控制单个请求超时
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", "https://httpbin.org/delay/2", nil)
    resp, err := client2.Do(req)
    if err != nil {
        fmt.Println("请求失败:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("状态: %d, 长度: %d\n", resp.StatusCode, len(body))
}
```

### 1.4 连接池与复用

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

func main() {
    // 默认 http.DefaultTransport 已经有连接池
    // 关键：必须读完 resp.Body 并关闭，连接才会回到池中！
    transport := &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20, // 默认只有 2！对同一 host 高并发时务必调大
        IdleConnTimeout:     90 * time.Second,
    }

    client := &http.Client{
        Transport: transport,
        Timeout:   10 * time.Second,
    }

    // 并发请求演示
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            resp, err := client.Get("https://httpbin.org/get")
            if err != nil {
                fmt.Printf("请求 %d 失败: %v\n", id, err)
                return
            }
            // 关键：必须读完并关闭 Body！
            io.Copy(io.Discard, resp.Body) // 丢弃响应体但确保读完
            resp.Body.Close()

            fmt.Printf("请求 %d 完成: %d\n", id, resp.StatusCode)
        }(i)
    }
    wg.Wait()
    fmt.Println("所有请求完成")
}
```

### 1.5 TLS 配置

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)

func main() {
    // 场景一：跳过证书验证（仅限测试环境！）
    insecureTransport := &http.Transport{
        TLSClientConfig: &tls.Config{
            InsecureSkipVerify: true, // 危险：跳过证书验证
        },
    }
    insecureClient := &http.Client{
        Transport: insecureTransport,
        Timeout:   10 * time.Second,
    }
    _ = insecureClient // 仅测试环境使用

    // 场景二：使用自定义 CA 证书（如内部 PKI）
    caCert, err := os.ReadFile("/etc/ssl/certs/ca-certificates.crt")
    if err != nil {
        fmt.Println("读取 CA 证书失败:", err)
        // 使用系统默认 CA
        caCert = nil
    }

    caCertPool := x509.NewCertPool()
    if caCert != nil {
        caCertPool.AppendCertsFromPEM(caCert)
    }

    secureTransport := &http.Transport{
        TLSClientConfig: &tls.Config{
            RootCAs:    caCertPool,
            MinVersion: tls.VersionTLS12, // 最低 TLS 1.2
        },
    }

    secureClient := &http.Client{
        Transport: secureTransport,
        Timeout:   10 * time.Second,
    }

    resp, err := secureClient.Get("https://httpbin.org/get")
    if err != nil {
        fmt.Println("请求失败:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("状态:", resp.StatusCode)
    fmt.Println("TLS 版本:", resp.TLS.Version)
    _ = body
}
```

---

## 2. HTTP 服务端

### 2.1 最简服务

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    // 方式一：使用函数处理器
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, SRE! 你的请求方法: %s\n", r.Method)
    })

    // 方式二：使用 Handler 接口
    http.Handle("/health", &HealthHandler{})

    fmt.Println("服务启动在 :8080")
    err := http.ListenAndServe(":8080", nil) // nil 使用 DefaultServeMux
    if err != nil {
        fmt.Println("服务启动失败:", err)
    }
}

// 实现 http.Handler 接口
type HealthHandler struct{}

func (h *HealthHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, `{"status":"healthy"}`)
}
```

### 2.2 Handler 接口与 HandlerFunc

```go
package main

import (
    "fmt"
    "net/http"
)

// http.Handler 接口定义：
// type Handler interface {
//     ServeHTTP(ResponseWriter, *Request)
// }

// http.HandlerFunc 类型：将普通函数适配为 Handler
// type HandlerFunc func(ResponseWriter, *Request)
// func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

func greetHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "World"
    }
    fmt.Fprintf(w, "Hello, %s!\n", name)
}

func main() {
    mux := http.NewServeMux()

    // 以下两种写法等价：
    mux.HandleFunc("/greet", greetHandler)
    mux.Handle("/greet2", http.HandlerFunc(greetHandler))

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}
```

### 2.3 获取请求参数

```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/params", func(w http.ResponseWriter, r *http.Request) {
        // Query 参数
        name := r.URL.Query().Get("name")
        age := r.URL.Query().Get("age")
        fmt.Fprintf(w, "Query 参数: name=%s, age=%s\n", name, age)

        // 请求头
        fmt.Fprintf(w, "User-Agent: %s\n", r.Header.Get("User-Agent"))
        fmt.Fprintf(w, "Content-Type: %s\n", r.Header.Get("Content-Type"))

        // 请求方法和路径
        fmt.Fprintf(w, "方法: %s, 路径: %s\n", r.Method, r.URL.Path)

        // POST 表单数据
        if r.Method == http.MethodPost {
            // 必须先 ParseForm
            r.ParseForm()
            fmt.Fprintf(w, "表单参数: %v\n", r.Form)

            // 或读取原始 Body
            body, _ := io.ReadAll(r.Body)
            fmt.Fprintf(w, "Body: %s\n", string(body))
        }
    })

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}
```

---

## 3. Go 1.22+ 增强路由

Go 1.22 对 `net/http` 的路由进行了重大增强，支持 HTTP 方法匹配和路径参数，很多以前需要第三方路由库的功能现在可以直接用标准库实现。

### 3.1 方法匹配

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // Go 1.22+: 路由模式中可指定 HTTP 方法
    mux.HandleFunc("GET /api/users", listUsers)
    mux.HandleFunc("POST /api/users", createUser)
    mux.HandleFunc("DELETE /api/users", deleteAllUsers)

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}

func listUsers(w http.ResponseWriter, r *http.Request) {
    users := []map[string]string{
        {"name": "张三", "role": "SRE"},
        {"name": "李四", "role": "DevOps"},
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusCreated)
    fmt.Fprint(w, `{"message":"用户已创建"}`)
}

func deleteAllUsers(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNoContent)
}
```

### 3.2 路径参数 {name}

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // Go 1.22+: 路径参数使用 {name} 语法
    mux.HandleFunc("GET /api/users/{id}", getUser)
    mux.HandleFunc("PUT /api/users/{id}", updateUser)
    mux.HandleFunc("DELETE /api/users/{id}", deleteUser)

    // 嵌套路径参数
    mux.HandleFunc("GET /api/users/{userId}/posts/{postId}", getUserPost)

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    // 使用 r.PathValue() 获取路径参数
    id := r.PathValue("id")
    user := map[string]string{
        "id":   id,
        "name": "SRE-" + id,
        "role": "运维工程师",
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func updateUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintf(w, "更新用户 %s\n", id)
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintf(w, "删除用户 %s\n", id)
}

func getUserPost(w http.ResponseWriter, r *http.Request) {
    userId := r.PathValue("userId")
    postId := r.PathValue("postId")
    fmt.Fprintf(w, "用户 %s 的文章 %s\n", userId, postId)
}
```

### 3.3 通配符 {path...}

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // {path...} 匹配剩余所有路径
    mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
        filePath := r.PathValue("path")
        fmt.Fprintf(w, "请求的文件路径: %s\n", filePath)
        // 例如 /files/logs/2024/app.log → path = "logs/2024/app.log"
    })

    // 静态文件服务
    mux.HandleFunc("GET /static/{path...}", func(w http.ResponseWriter, r *http.Request) {
        path := r.PathValue("path")
        // 实际场景中使用 http.FileServer
        fmt.Fprintf(w, "静态文件: %s\n", path)
    })

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}
```

### 3.4 路由优先级

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // Go 1.22+ 路由匹配规则：
    // 1. 更具体的路径优先（如 /api/users/me 优先于 /api/users/{id}）
    // 2. 精确方法匹配优先（如 "GET /path" 优先于 "/path"）
    // 3. 更长的路径优先

    mux.HandleFunc("GET /api/users/me", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, "当前用户信息\n") // 精确路径优先
    })

    mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "用户 %s 的信息\n", r.PathValue("id")) // 参数路径
    })

    mux.HandleFunc("/api/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "通用 API 处理: %s %s\n", r.Method, r.URL.Path)
    })

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", mux)
}
```

---

## 4. 中间件模式

中间件就是一个接收 `http.Handler` 并返回 `http.Handler` 的函数。

### 4.1 日志中间件

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// 日志中间件
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // 包装 ResponseWriter 以获取状态码
        wrapped := &statusRecorder{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        duration := time.Since(start)
        log.Printf("%s %s %d %v\n", r.Method, r.URL.Path, wrapped.statusCode, duration)
    })
}

// 自定义 ResponseWriter 包装器，记录状态码
type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
    sr.statusCode = code
    sr.ResponseWriter.WriteHeader(code)
}

// 认证中间件
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, `{"error":"未授权"}`, http.StatusUnauthorized)
            return
        }
        // 简单 token 验证（实际场景用 JWT 等）
        if token != "Bearer secret-token" {
            http.Error(w, `{"error":"token 无效"}`, http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// 恢复中间件（防止 panic 崩溃）
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC 恢复: %v\n", err)
                http.Error(w, `{"error":"内部服务器错误"}`, http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /public", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, "公开页面\n")
    })

    mux.HandleFunc("GET /secret", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, "机密信息\n")
    })

    // 中间件链：recovery → logging → auth → handler
    handler := recoveryMiddleware(loggingMiddleware(authMiddleware(mux)))

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", handler)
}
```

### 4.2 中间件链组合器

```go
package main

import (
    "fmt"
    "net/http"
)

// Chain 将多个中间件组合成一个
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    // 从后往前包装，这样第一个中间件最先执行
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// CORS 中间件
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, `{"data":"hello"}`)
    })

    // 使用 Chain 组合中间件
    handler := Chain(mux, corsMiddleware)

    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", handler)
}
```

---

## 5. 优雅关闭

### 5.1 http.Server.Shutdown

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
        // 模拟耗时请求
        time.Sleep(2 * time.Second)
        fmt.Fprint(w, "Hello, 优雅关闭!\n")
    })

    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, `{"status":"ok"}`)
    })

    // 使用 http.Server 而不是 http.ListenAndServe
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // 在 goroutine 中启动服务
    go func() {
        log.Println("服务启动在 :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("服务异常退出: %v", err)
        }
    }()

    // 等待中断信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Printf("收到信号 %v，开始优雅关闭...\n", sig)

    // 创建超时 context
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Shutdown 会：
    // 1. 停止接受新连接
    // 2. 等待已有请求处理完毕
    // 3. 超时后强制关闭
    if err := server.Shutdown(ctx); err != nil {
        log.Printf("优雅关闭失败: %v\n", err)
    } else {
        log.Println("服务已优雅关闭")
    }
}
```

---

## 6. JSON API 完整示例

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"
)

// ---- 数据模型 ----

type Service struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Status    string    `json:"status"`     // running, stopped, degraded
    Port      int       `json:"port"`
    CreatedAt time.Time `json:"created_at"`
}

type APIResponse struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

// ---- 内存存储 ----

type ServiceStore struct {
    mu       sync.RWMutex
    services map[string]*Service
    nextID   int
}

func NewServiceStore() *ServiceStore {
    return &ServiceStore{
        services: make(map[string]*Service),
    }
}

// ---- JSON 响应工具 ----

func writeJSON(w http.ResponseWriter, status int, resp APIResponse) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(resp)
}

func main() {
    store := NewServiceStore()

    // 预填充一些数据
    store.services["svc-1"] = &Service{ID: "svc-1", Name: "nginx", Status: "running", Port: 80, CreatedAt: time.Now()}
    store.services["svc-2"] = &Service{ID: "svc-2", Name: "redis", Status: "running", Port: 6379, CreatedAt: time.Now()}
    store.nextID = 3

    mux := http.NewServeMux()

    // Go 1.22+ 增强路由
    mux.HandleFunc("GET /api/services", func(w http.ResponseWriter, r *http.Request) {
        store.mu.RLock()
        defer store.mu.RUnlock()

        services := make([]*Service, 0, len(store.services))
        for _, svc := range store.services {
            services = append(services, svc)
        }

        writeJSON(w, http.StatusOK, APIResponse{
            Code:    200,
            Message: "success",
            Data:    services,
        })
    })

    mux.HandleFunc("GET /api/services/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")

        store.mu.RLock()
        svc, ok := store.services[id]
        store.mu.RUnlock()

        if !ok {
            writeJSON(w, http.StatusNotFound, APIResponse{
                Code:    404,
                Message: fmt.Sprintf("服务 %s 不存在", id),
            })
            return
        }

        writeJSON(w, http.StatusOK, APIResponse{
            Code:    200,
            Message: "success",
            Data:    svc,
        })
    })

    mux.HandleFunc("POST /api/services", func(w http.ResponseWriter, r *http.Request) {
        var svc Service
        if err := json.NewDecoder(r.Body).Decode(&svc); err != nil {
            writeJSON(w, http.StatusBadRequest, APIResponse{
                Code:    400,
                Message: "JSON 解析失败: " + err.Error(),
            })
            return
        }

        store.mu.Lock()
        svc.ID = fmt.Sprintf("svc-%d", store.nextID)
        store.nextID++
        svc.CreatedAt = time.Now()
        if svc.Status == "" {
            svc.Status = "stopped"
        }
        store.services[svc.ID] = &svc
        store.mu.Unlock()

        writeJSON(w, http.StatusCreated, APIResponse{
            Code:    201,
            Message: "服务已创建",
            Data:    svc,
        })
    })

    mux.HandleFunc("DELETE /api/services/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")

        store.mu.Lock()
        _, ok := store.services[id]
        if ok {
            delete(store.services, id)
        }
        store.mu.Unlock()

        if !ok {
            writeJSON(w, http.StatusNotFound, APIResponse{
                Code:    404,
                Message: fmt.Sprintf("服务 %s 不存在", id),
            })
            return
        }

        writeJSON(w, http.StatusOK, APIResponse{
            Code:    200,
            Message: fmt.Sprintf("服务 %s 已删除", id),
        })
    })

    // 健康检查
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        writeJSON(w, http.StatusOK, APIResponse{
            Code:    200,
            Message: "healthy",
        })
    })

    log.Println("JSON API 服务启动在 :8080")
    log.Println("  GET    /api/services      - 列出所有服务")
    log.Println("  GET    /api/services/{id}  - 获取单个服务")
    log.Println("  POST   /api/services       - 创建服务")
    log.Println("  DELETE /api/services/{id}  - 删除服务")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 7. SRE 实战场景

### 7.1 HTTP 健康检查探针

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// 健康检查目标
type Target struct {
    Name string `json:"name"`
    URL  string `json:"url"`
}

// 健康检查结果
type CheckResult struct {
    Name       string        `json:"name"`
    URL        string        `json:"url"`
    Status     string        `json:"status"` // healthy, unhealthy, timeout
    StatusCode int           `json:"status_code,omitempty"`
    Latency    time.Duration `json:"latency_ms"`
    Error      string        `json:"error,omitempty"`
}

func checkHealth(ctx context.Context, client *http.Client, target Target) CheckResult {
    start := time.Now()

    req, err := http.NewRequestWithContext(ctx, "GET", target.URL, nil)
    if err != nil {
        return CheckResult{
            Name:    target.Name,
            URL:     target.URL,
            Status:  "unhealthy",
            Latency: time.Since(start),
            Error:   err.Error(),
        }
    }

    resp, err := client.Do(req)
    latency := time.Since(start)

    if err != nil {
        return CheckResult{
            Name:    target.Name,
            URL:     target.URL,
            Status:  "timeout",
            Latency: latency,
            Error:   err.Error(),
        }
    }
    defer resp.Body.Close()

    status := "healthy"
    if resp.StatusCode >= 400 {
        status = "unhealthy"
    }

    return CheckResult{
        Name:       target.Name,
        URL:        target.URL,
        Status:     status,
        StatusCode: resp.StatusCode,
        Latency:    latency,
    }
}

func main() {
    targets := []Target{
        {Name: "Google", URL: "https://www.google.com"},
        {Name: "GitHub", URL: "https://github.com"},
        {Name: "不存在的服务", URL: "http://localhost:9999"},
    }

    client := &http.Client{Timeout: 5 * time.Second}
    ctx := context.Background()

    var wg sync.WaitGroup
    results := make([]CheckResult, len(targets))

    for i, target := range targets {
        wg.Add(1)
        go func(idx int, t Target) {
            defer wg.Done()
            results[idx] = checkHealth(ctx, client, t)
        }(i, target)
    }

    wg.Wait()

    // 输出结果
    data, _ := json.MarshalIndent(results, "", "  ")
    fmt.Println(string(data))
}
```

### 7.2 带重试的 HTTP 客户端

```go
package main

import (
    "fmt"
    "io"
    "math"
    "net/http"
    "time"
)

// RetryClient 带重试和指数退避的 HTTP 客户端
type RetryClient struct {
    client    *http.Client
    maxRetry  int
    baseDelay time.Duration
}

func NewRetryClient(maxRetry int) *RetryClient {
    return &RetryClient{
        client: &http.Client{
            Timeout: 10 * time.Second,
        },
        maxRetry:  maxRetry,
        baseDelay: 500 * time.Millisecond,
    }
}

func (rc *RetryClient) Do(req *http.Request) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= rc.maxRetry; attempt++ {
        if attempt > 0 {
            // 指数退避：500ms, 1s, 2s, 4s, ...
            delay := time.Duration(float64(rc.baseDelay) * math.Pow(2, float64(attempt-1)))
            fmt.Printf("  第 %d 次重试，等待 %v\n", attempt, delay)
            time.Sleep(delay)
        }

        resp, err := rc.client.Do(req)
        if err != nil {
            lastErr = err
            fmt.Printf("  请求失败: %v\n", err)
            continue
        }

        // 5xx 服务端错误可以重试
        if resp.StatusCode >= 500 {
            body, _ := io.ReadAll(resp.Body)
            resp.Body.Close()
            lastErr = fmt.Errorf("服务端错误: %d, body: %s", resp.StatusCode, string(body))
            fmt.Printf("  收到 %d，准备重试\n", resp.StatusCode)
            continue
        }

        // 成功或客户端错误（4xx）不重试
        return resp, nil
    }

    return nil, fmt.Errorf("重试 %d 次后仍然失败: %v", rc.maxRetry, lastErr)
}

func main() {
    client := NewRetryClient(3)

    req, _ := http.NewRequest("GET", "https://httpbin.org/status/500", nil)
    fmt.Println("开始请求 (预期 500 错误):")

    resp, err := client.Do(req)
    if err != nil {
        fmt.Println("最终失败:", err)
        return
    }
    defer resp.Body.Close()
    fmt.Println("最终状态码:", resp.StatusCode)
}
```

---

## 8. 对比 Python requests / Flask

### 8.1 HTTP 客户端对比

| 功能 | Go net/http | Python requests |
|------|-------------|-----------------|
| GET 请求 | `http.Get(url)` | `requests.get(url)` |
| POST JSON | `http.Post(url, "application/json", body)` | `requests.post(url, json=data)` |
| 自定义头 | `req.Header.Set(k, v)` | `requests.get(url, headers={k:v})` |
| 超时 | `client.Timeout = 5s` | `requests.get(url, timeout=5)` |
| Session 复用 | `http.Client` (自带连接池) | `requests.Session()` |
| TLS 跳过验证 | `TLSClientConfig{InsecureSkipVerify}` | `verify=False` |

### 8.2 HTTP 服务端对比

```go
// Go: 完整的 REST API
mux := http.NewServeMux()
mux.HandleFunc("GET /api/users/{id}", getUser)
http.ListenAndServe(":8080", mux)
```

```python
# Python Flask: 等效代码
from flask import Flask
app = Flask(__name__)

@app.route('/api/users/<id>', methods=['GET'])
def get_user(id):
    return {"id": id}

app.run(port=8080)
```

| 功能 | Go net/http | Python Flask |
|------|-------------|-------------|
| 路由 | `mux.HandleFunc("GET /path/{id}", h)` | `@app.route('/path/<id>')` |
| 路径参数 | `r.PathValue("id")` | 函数参数 `id` |
| JSON 响应 | `json.NewEncoder(w).Encode(data)` | `return jsonify(data)` |
| 中间件 | 函数包装 `func(http.Handler)http.Handler` | `@app.before_request` |
| 优雅关闭 | `server.Shutdown(ctx)` | 需要额外处理 |
| 并发性能 | goroutine per request | 受 GIL 限制 |
| 部署 | 单二进制 | 需要 gunicorn/uwsgi |

---

## 常见坑点

### 坑 1：忘记关闭 Response Body

```go
// 错误：连接泄漏！
resp, err := http.Get(url)
if err != nil {
    return err
}
// 忘记 resp.Body.Close()
// 连接不会回到连接池，最终耗尽文件描述符

// 正确：
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
// 重要：即使不需要响应体，也必须读完并关闭
io.Copy(io.Discard, resp.Body) // 确保读完
```

### 坑 2：默认 Client 无超时

```go
// 危险：http.DefaultClient 没有超时限制！
// 如果远端不响应，goroutine 会永远阻塞
resp, err := http.Get("https://slow-server.example.com")

// 正确：始终使用自定义 Client
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get("https://slow-server.example.com")
```

### 坑 3：MaxIdleConnsPerHost 默认太小

```go
// http.DefaultTransport 的 MaxIdleConnsPerHost 默认只有 2
// 对同一个 host 高并发时，大量连接无法复用

// 解决：
transport := &http.Transport{
    MaxIdleConnsPerHost: 20, // 根据并发量调整
}
client := &http.Client{Transport: transport}
```

### 坑 4：Handler 中的 goroutine 泄漏

```go
// 错误：请求取消后 goroutine 不会停止
func handler(w http.ResponseWriter, r *http.Request) {
    go func() {
        time.Sleep(10 * time.Second)
        // 可能 client 已断开
        fmt.Fprint(w, "done") // panic: write on closed connection
    }()
}

// 正确：使用 r.Context() 感知请求取消
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    go func() {
        select {
        case <-time.After(10 * time.Second):
            // 完成工作
        case <-ctx.Done():
            // 请求已取消，清理资源
            return
        }
    }()
}
```

### 坑 5：WriteHeader 只能调用一次

```go
// 错误：重复设置状态码
func handler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    // ...
    w.WriteHeader(http.StatusInternalServerError) // 无效！会打印警告日志
}

// 注意：调用 Write() 会隐式调用 WriteHeader(200)
// 所以设置 Header 必须在 Write 之前
func handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json") // 必须在 Write 之前
    w.WriteHeader(http.StatusCreated)                   // 必须在 Write 之前
    fmt.Fprint(w, `{"ok":true}`)
}
```

---

## 速查表

```text
┌────────────────────────────────────────────────────────────────────────┐
│                    net/http 速查表                                       │
├────────────────────────────────────────────────────────────────────────┤
│ 客户端                                                                  │
│   GET 请求       │ http.Get(url)                                       │
│   POST 请求      │ http.Post(url, contentType, body)                   │
│   自定义请求     │ http.NewRequest(method, url, body) + client.Do(req) │
│   设置超时       │ client.Timeout = 10 * time.Second                   │
│   请求 context   │ http.NewRequestWithContext(ctx, method, url, body)  │
│   连接池         │ Transport{MaxIdleConnsPerHost: 20}                  │
│   跳过TLS验证    │ TLSClientConfig{InsecureSkipVerify: true}           │
├────────────────────────────────────────────────────────────────────────┤
│ 服务端                                                                  │
│   启动服务       │ http.ListenAndServe(":8080", handler)               │
│   新建路由       │ mux := http.NewServeMux()                           │
│   注册路由       │ mux.HandleFunc("GET /path/{id}", handler)           │
│   路径参数       │ r.PathValue("id")                                   │
│   通配符路径     │ "GET /files/{path...}"                              │
│   Query 参数     │ r.URL.Query().Get("key")                            │
│   JSON 响应      │ json.NewEncoder(w).Encode(data)                     │
│   优雅关闭       │ server.Shutdown(ctx)                                │
├────────────────────────────────────────────────────────────────────────┤
│ 中间件                                                                  │
│   签名           │ func(http.Handler) http.Handler                     │
│   链式调用       │ recovery(logging(auth(handler)))                    │
│   请求上下文     │ r.Context()                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 小结

- `net/http` 客户端：**务必设置超时**、**务必关闭 Body**、根据并发量调整连接池参数。
- `net/http` 服务端：Go 1.22+ 的增强路由大幅减少了对第三方路由库的依赖。
- 中间件模式是构建可维护 HTTP 服务的关键，遵循 `func(http.Handler) http.Handler` 签名。
- 优雅关闭使用 `http.Server.Shutdown`，配合 `os/signal` 监听 SIGINT/SIGTERM。
- 生产环境中，用 `http.Server` 代替 `http.ListenAndServe`，精确控制各种超时参数。
