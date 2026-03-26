# 01-Gin框架

## 导读

Gin 是 Go 语言生态中最流行的 Web 框架之一，以高性能和简洁的 API 著称。它基于 `httprouter` 构建，性能远超标准库的 `net/http` 默认路由。对于 SRE/DevOps 工程师而言，无论是构建内部运维平台、监控告警 API、还是自动化接口，Gin 都是首选框架。

本章将系统讲解：

- Gin 框架介绍与安装
- 路由定义（GET/POST/PUT/DELETE）
- 路由参数（`:name` 和 `*action`）与路由分组
- 中间件（Logger/Recovery/自定义/全局 vs 分组 vs 单路由）
- 请求参数获取（Query/PostForm/Bind/ShouldBind）
- 参数校验（binding tag、自定义验证器）
- JSON/HTML/XML 响应
- 文件上传
- Gin Context 详解

> 基于 Go 1.24 + Gin v1.10，所有代码完整可运行。

---

## 一、Gin 框架介绍与安装

### 1.1 为什么选择 Gin

| 特性 | Gin | net/http |
|------|-----|---------|
| 路由性能 | 基于 Radix 树，极快 | 线性匹配 |
| 路由参数 | 原生支持 | 需手动解析 |
| 中间件 | 链式中间件 | 需手动实现 |
| 参数绑定 | 自动绑定 + 校验 | 手动解析 |
| JSON 处理 | 内置封装 | 手动编码 |
| 错误恢复 | Recovery 中间件 | 需手动捕获 panic |

### 1.2 安装

```bash
# 初始化项目
mkdir gin-demo && cd gin-demo
go mod init gin-demo

# 安装 Gin
go get -u github.com/gin-gonic/gin
```

### 1.3 Hello World

```go
// 文件: main.go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	// 创建默认引擎（包含 Logger 和 Recovery 中间件）
	r := gin.Default()

	// 定义路由
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "pong",
		})
	})

	// 启动服务（默认监听 :8080）
	r.Run(":8080")
}
```

```bash
# 运行
go run main.go

# 测试
curl http://localhost:8080/ping
# {"message":"pong"}
```

---

## 二、路由定义

### 2.1 HTTP 方法路由

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// GET 请求
	r.GET("/users", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"action": "获取用户列表"})
	})

	// POST 请求
	r.POST("/users", func(c *gin.Context) {
		c.JSON(http.StatusCreated, gin.H{"action": "创建用户"})
	})

	// PUT 请求
	r.PUT("/users/:id", func(c *gin.Context) {
		id := c.Param("id")
		c.JSON(http.StatusOK, gin.H{"action": "更新用户", "id": id})
	})

	// PATCH 请求
	r.PATCH("/users/:id", func(c *gin.Context) {
		id := c.Param("id")
		c.JSON(http.StatusOK, gin.H{"action": "部分更新用户", "id": id})
	})

	// DELETE 请求
	r.DELETE("/users/:id", func(c *gin.Context) {
		id := c.Param("id")
		c.JSON(http.StatusOK, gin.H{"action": "删除用户", "id": id})
	})

	// HEAD 请求
	r.HEAD("/users", func(c *gin.Context) {
		c.Header("X-Total-Count", "100")
		c.Status(http.StatusOK)
	})

	// OPTIONS 请求
	r.OPTIONS("/users", func(c *gin.Context) {
		c.Header("Allow", "GET, POST, OPTIONS")
		c.Status(http.StatusNoContent)
	})

	// Any 匹配所有 HTTP 方法
	r.Any("/anything", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"method": c.Request.Method,
			"path":   c.Request.URL.Path,
		})
	})

	r.Run(":8080")
}
```

### 2.2 路由参数

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// ===== 命名参数（:name）—— 匹配单个路径段 =====

	// /user/张三 -> name = "张三"
	// /user/123  -> name = "123"
	r.GET("/user/:name", func(c *gin.Context) {
		name := c.Param("name")
		c.JSON(http.StatusOK, gin.H{"name": name})
	})

	// 多个命名参数
	// /repos/gin-gonic/gin -> owner = "gin-gonic", repo = "gin"
	r.GET("/repos/:owner/:repo", func(c *gin.Context) {
		owner := c.Param("owner")
		repo := c.Param("repo")
		c.JSON(http.StatusOK, gin.H{
			"owner": owner,
			"repo":  repo,
		})
	})

	// ===== 通配参数（*action）—— 匹配剩余的所有路径 =====

	// /files/documents/report.pdf -> filepath = "/documents/report.pdf"
	// /files/images/logo.png      -> filepath = "/images/logo.png"
	r.GET("/files/*filepath", func(c *gin.Context) {
		filepath := c.Param("filepath")
		c.JSON(http.StatusOK, gin.H{"filepath": filepath})
	})

	// 注意：命名参数和通配参数不能混用在同一位置
	// 通配参数必须在路由的最后

	r.Run(":8080")
}
```

### 2.3 路由分组（Group）

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// ===== 基本路由分组 =====
	// /api/v1/users, /api/v1/posts ...
	v1 := r.Group("/api/v1")
	{
		v1.GET("/users", getUsers)
		v1.GET("/users/:id", getUserByID)
		v1.POST("/users", createUser)
		v1.PUT("/users/:id", updateUser)
		v1.DELETE("/users/:id", deleteUser)

		v1.GET("/posts", getPosts)
		v1.POST("/posts", createPost)
	}

	// /api/v2/users ...
	v2 := r.Group("/api/v2")
	{
		v2.GET("/users", getUsersV2) // 新版本的 API
	}

	// ===== 嵌套分组 =====
	admin := r.Group("/admin")
	{
		// /admin/dashboard
		admin.GET("/dashboard", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"page": "dashboard"})
		})

		// /admin/settings/...
		settings := admin.Group("/settings")
		{
			settings.GET("/profile", func(c *gin.Context) {
				c.JSON(http.StatusOK, gin.H{"page": "profile settings"})
			})
			settings.GET("/security", func(c *gin.Context) {
				c.JSON(http.StatusOK, gin.H{"page": "security settings"})
			})
		}
	}

	// ===== 分组 + 中间件 =====
	authorized := r.Group("/api")
	authorized.Use(AuthMiddleware()) // 该分组下的所有路由都需要认证
	{
		authorized.GET("/profile", getProfile)
		authorized.POST("/settings", updateSettings)
	}

	r.Run(":8080")
}

// 示例 handler 函数
func getUsers(c *gin.Context)     { c.JSON(200, gin.H{"action": "获取用户列表"}) }
func getUserByID(c *gin.Context)  { c.JSON(200, gin.H{"action": "获取用户", "id": c.Param("id")}) }
func createUser(c *gin.Context)   { c.JSON(201, gin.H{"action": "创建用户"}) }
func updateUser(c *gin.Context)   { c.JSON(200, gin.H{"action": "更新用户"}) }
func deleteUser(c *gin.Context)   { c.JSON(200, gin.H{"action": "删除用户"}) }
func getPosts(c *gin.Context)     { c.JSON(200, gin.H{"action": "获取文章列表"}) }
func createPost(c *gin.Context)   { c.JSON(200, gin.H{"action": "创建文章"}) }
func getUsersV2(c *gin.Context)   { c.JSON(200, gin.H{"action": "获取用户列表 V2"}) }
func getProfile(c *gin.Context)   { c.JSON(200, gin.H{"action": "获取个人信息"}) }
func updateSettings(c *gin.Context) { c.JSON(200, gin.H{"action": "更新设置"}) }

func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		token := c.GetHeader("Authorization")
		if token == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "未授权"})
			c.Abort()
			return
		}
		c.Next()
	}
}
```

---

## 三、中间件

### 3.1 内置中间件

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// gin.New() 创建不带任何中间件的引擎
	r := gin.New()

	// 手动添加内置中间件
	r.Use(gin.Logger())   // 请求日志中间件
	r.Use(gin.Recovery()) // Panic 恢复中间件

	// 等效于 gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{"message": "pong"})
	})

	r.Run(":8080")
}
```

### 3.2 自定义中间件

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

// RequestTimer 请求计时中间件
func RequestTimer() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		// 处理请求（调用后续的处理器和中间件）
		c.Next()

		// 请求处理完成后记录耗时
		duration := time.Since(start)
		status := c.Writer.Status()

		log.Printf("[%s] %s %s - %d (%v)",
			c.Request.Method,
			c.Request.URL.Path,
			c.ClientIP(),
			status,
			duration,
		)
	}
}

// RequestID 请求 ID 中间件
func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 尝试从请求头获取
		requestID := c.GetHeader("X-Request-ID")
		if requestID == "" {
			requestID = fmt.Sprintf("req-%d", time.Now().UnixNano())
		}

		// 设置到 Context 中，后续处理器可以使用
		c.Set("request_id", requestID)

		// 设置到响应头中
		c.Header("X-Request-ID", requestID)

		c.Next()
	}
}

// CORS 跨域中间件
func CORS() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Header("Access-Control-Allow-Origin", "*")
		c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
		c.Header("Access-Control-Max-Age", "86400")

		// 预检请求直接返回
		if c.Request.Method == "OPTIONS" {
			c.AbortWithStatus(http.StatusNoContent)
			return
		}

		c.Next()
	}
}

// RateLimit 简单限流中间件
func RateLimit(maxRequests int, window time.Duration) gin.HandlerFunc {
	// 简单的固定窗口计数器（生产环境建议使用 Redis）
	type clientInfo struct {
		count    int
		resetAt  time.Time
	}
	clients := make(map[string]*clientInfo)

	return func(c *gin.Context) {
		ip := c.ClientIP()
		now := time.Now()

		info, exists := clients[ip]
		if !exists || now.After(info.resetAt) {
			clients[ip] = &clientInfo{count: 1, resetAt: now.Add(window)}
		} else {
			info.count++
			if info.count > maxRequests {
				c.JSON(http.StatusTooManyRequests, gin.H{
					"error":       "请求太频繁",
					"retry_after": info.resetAt.Sub(now).Seconds(),
				})
				c.Abort()
				return
			}
		}

		c.Next()
	}
}

// ErrorHandler 统一错误处理中间件
func ErrorHandler() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next()

		// 检查是否有错误
		if len(c.Errors) > 0 {
			// 获取最后一个错误
			err := c.Errors.Last()
			c.JSON(http.StatusInternalServerError, gin.H{
				"error":      err.Error(),
				"request_id": c.GetString("request_id"),
			})
		}
	}
}

func main() {
	r := gin.New()

	// ===== 全局中间件 =====
	r.Use(gin.Recovery())
	r.Use(RequestID())
	r.Use(RequestTimer())
	r.Use(CORS())
	r.Use(ErrorHandler())

	// ===== 公共路由（无需认证）=====
	r.GET("/health", func(c *gin.Context) {
		c.JSON(200, gin.H{"status": "ok"})
	})

	// ===== 分组中间件 =====
	api := r.Group("/api")
	api.Use(RateLimit(100, time.Minute))
	{
		api.GET("/public", func(c *gin.Context) {
			requestID, _ := c.Get("request_id")
			c.JSON(200, gin.H{
				"message":    "公共 API",
				"request_id": requestID,
			})
		})
	}

	// ===== 需要认证的分组 =====
	auth := r.Group("/api/admin")
	auth.Use(AuthRequired())
	{
		auth.GET("/dashboard", func(c *gin.Context) {
			userID, _ := c.Get("user_id")
			c.JSON(200, gin.H{"user_id": userID})
		})
	}

	// ===== 单路由中间件 =====
	r.GET("/special", SpecialMiddleware(), func(c *gin.Context) {
		c.JSON(200, gin.H{"message": "特殊路由"})
	})

	r.Run(":8080")
}

// AuthRequired 认证中间件
func AuthRequired() gin.HandlerFunc {
	return func(c *gin.Context) {
		token := c.GetHeader("Authorization")
		if token == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "需要认证"})
			c.Abort() // 终止请求链
			return
		}
		// 模拟 token 验证
		if token != "Bearer valid-token" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "无效的 token"})
			c.Abort()
			return
		}
		// 设置用户信息到 Context
		c.Set("user_id", "admin-001")
		c.Set("role", "admin")
		c.Next() // 继续处理
	}
}

// SpecialMiddleware 单路由中间件
func SpecialMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set("special", true)
		c.Next()
	}
}
```

### 3.3 中间件执行顺序

```
请求 → 中间件1(前) → 中间件2(前) → 中间件3(前) → Handler
                                                      ↓
响应 ← 中间件1(后) ← 中间件2(后) ← 中间件3(后) ← Handler

c.Next() 之前的代码 = "前置处理"
c.Next() 之后的代码 = "后置处理"
c.Abort() = 终止后续中间件和 Handler 的执行
```

```go
func MiddlewareDemo() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("前置处理: 认证检查")

		c.Next() // 调用下一个处理器

		fmt.Println("后置处理: 记录日志")
	}
}
```

---

## 四、请求参数获取

### 4.1 Query 参数、PostForm、路由参数

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// ===== Query 参数（URL ?key=value）=====
	// GET /search?q=golang&page=1&limit=10
	r.GET("/search", func(c *gin.Context) {
		// 获取参数，带默认值
		query := c.DefaultQuery("q", "")
		page := c.DefaultQuery("page", "1")
		limit := c.DefaultQuery("limit", "10")

		// 获取参数（不带默认值）
		sort := c.Query("sort") // 不存在返回空字符串

		// 获取数组参数
		// GET /search?tags=go&tags=web&tags=api
		tags := c.QueryArray("tags")

		// 获取 Map 参数
		// GET /search?filter[name]=test&filter[age]=25
		filters := c.QueryMap("filter")

		c.JSON(http.StatusOK, gin.H{
			"query":   query,
			"page":    page,
			"limit":   limit,
			"sort":    sort,
			"tags":    tags,
			"filters": filters,
		})
	})

	// ===== PostForm 参数（表单提交）=====
	// POST /login  Content-Type: application/x-www-form-urlencoded
	r.POST("/login", func(c *gin.Context) {
		username := c.PostForm("username")
		password := c.PostForm("password")
		remember := c.DefaultPostForm("remember", "false")

		// PostFormArray 获取表单数组
		hobbies := c.PostFormArray("hobbies")

		// PostFormMap 获取表单 Map
		address := c.PostFormMap("address")

		c.JSON(http.StatusOK, gin.H{
			"username": username,
			"password": "***",
			"remember": remember,
			"hobbies":  hobbies,
			"address":  address,
		})
	})

	// ===== 路由参数 =====
	r.GET("/users/:id/orders/:order_id", func(c *gin.Context) {
		userID := c.Param("id")
		orderID := c.Param("order_id")

		c.JSON(http.StatusOK, gin.H{
			"user_id":  userID,
			"order_id": orderID,
		})
	})

	r.Run(":8080")
}
```

### 4.2 Bind / ShouldBind 参数绑定

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// CreateUserRequest 创建用户请求
type CreateUserRequest struct {
	Name     string `json:"name" form:"name" binding:"required"`
	Email    string `json:"email" form:"email" binding:"required,email"`
	Age      int    `json:"age" form:"age" binding:"required,gte=1,lte=150"`
	Password string `json:"password" form:"password" binding:"required,min=6,max=64"`
}

// UpdateUserRequest 更新用户请求（部分字段可选）
type UpdateUserRequest struct {
	Name  *string `json:"name" form:"name" binding:"omitempty,min=1"`
	Email *string `json:"email" form:"email" binding:"omitempty,email"`
	Age   *int    `json:"age" form:"age" binding:"omitempty,gte=1,lte=150"`
}

// SearchRequest 搜索请求（Query 参数绑定）
type SearchRequest struct {
	Keyword string `form:"q" binding:"required"`
	Page    int    `form:"page" binding:"omitempty,gte=1"`
	Limit   int    `form:"limit" binding:"omitempty,gte=1,lte=100"`
}

// URIParam URI 参数绑定
type URIParam struct {
	ID   int    `uri:"id" binding:"required,gte=1"`
	Name string `uri:"name" binding:"required"`
}

// HeaderParam 请求头绑定
type HeaderParam struct {
	Authorization string `header:"Authorization" binding:"required"`
	ContentType   string `header:"Content-Type"`
}

func main() {
	r := gin.Default()

	// ===== ShouldBindJSON：绑定 JSON 请求体 =====
	r.POST("/users", func(c *gin.Context) {
		var req CreateUserRequest

		// ShouldBind 系列：出错时不会自动返回 400，由你决定如何处理
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"error":   "参数校验失败",
				"details": err.Error(),
			})
			return
		}

		c.JSON(http.StatusCreated, gin.H{
			"message": "用户创建成功",
			"user": gin.H{
				"name":  req.Name,
				"email": req.Email,
				"age":   req.Age,
			},
		})
	})

	// ===== ShouldBindQuery：绑定 Query 参数 =====
	r.GET("/search", func(c *gin.Context) {
		var req SearchRequest
		if err := c.ShouldBindQuery(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		// 设置默认值
		if req.Page == 0 {
			req.Page = 1
		}
		if req.Limit == 0 {
			req.Limit = 20
		}

		c.JSON(http.StatusOK, gin.H{
			"keyword": req.Keyword,
			"page":    req.Page,
			"limit":   req.Limit,
		})
	})

	// ===== ShouldBindUri：绑定 URI 参数 =====
	r.GET("/users/:id/:name", func(c *gin.Context) {
		var param URIParam
		if err := c.ShouldBindUri(&param); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, gin.H{
			"id":   param.ID,
			"name": param.Name,
		})
	})

	// ===== ShouldBindHeader：绑定请求头 =====
	r.GET("/protected", func(c *gin.Context) {
		var header HeaderParam
		if err := c.ShouldBindHeader(&header); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, gin.H{
			"auth":         header.Authorization,
			"content_type": header.ContentType,
		})
	})

	// ===== ShouldBind：自动根据 Content-Type 选择绑定方式 =====
	// Content-Type: application/json -> 绑定 JSON
	// Content-Type: application/x-www-form-urlencoded -> 绑定表单
	// Content-Type: multipart/form-data -> 绑定表单
	r.POST("/auto-bind", func(c *gin.Context) {
		var req CreateUserRequest
		if err := c.ShouldBind(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, gin.H{"user": req.Name})
	})

	r.Run(":8080")
}
```

---

## 五、参数校验

### 5.1 内置验证标签

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// ServerConfig 服务器配置（演示各种验证标签）
type ServerConfig struct {
	// 必填
	Name string `json:"name" binding:"required"`

	// 字符串长度
	Host string `json:"host" binding:"required,min=1,max=255"`

	// 数值范围
	Port int `json:"port" binding:"required,gte=1,lte=65535"`

	// 枚举值
	Protocol string `json:"protocol" binding:"required,oneof=http https tcp udp"`

	// 邮箱格式
	AdminEmail string `json:"admin_email" binding:"required,email"`

	// URL 格式
	WebhookURL string `json:"webhook_url" binding:"omitempty,url"`

	// IP 地址
	BindIP string `json:"bind_ip" binding:"omitempty,ip"`

	// 正则匹配（非 Gin 原生，需要自定义验证器或使用 validator 包）
	// Version string `json:"version" binding:"required,regexp=^v[0-9]+\\.[0-9]+\\.[0-9]+$"`

	// 嵌套结构体验证
	Database DatabaseConfig `json:"database" binding:"required"`

	// 切片验证
	Tags []string `json:"tags" binding:"omitempty,max=10,dive,min=1,max=50"`

	// Map 验证
	Labels map[string]string `json:"labels" binding:"omitempty,max=20"`
}

// DatabaseConfig 数据库配置
type DatabaseConfig struct {
	DSN      string `json:"dsn" binding:"required"`
	MaxConns int    `json:"max_conns" binding:"required,gte=1,lte=1000"`
	MaxIdle  int    `json:"max_idle" binding:"required,gte=0,ltefield=MaxConns"`
}

func main() {
	r := gin.Default()

	r.POST("/config", func(c *gin.Context) {
		var config ServerConfig
		if err := c.ShouldBindJSON(&config); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"error":   "配置验证失败",
				"details": err.Error(),
			})
			return
		}

		c.JSON(http.StatusOK, gin.H{
			"message": "配置有效",
			"config":  config,
		})
	})

	r.Run(":8080")
}
```

### 5.2 自定义验证器

```go
package main

import (
	"net/http"
	"regexp"
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

// 自定义验证函数：验证中文姓名
func validateChineseName(fl validator.FieldLevel) bool {
	name := fl.Field().String()
	// 2-20 个中文字符
	matched, _ := regexp.MatchString(`^[\x{4e00}-\x{9fa5}]{2,20}$`, name)
	return matched
}

// 自定义验证函数：验证手机号
func validateMobile(fl validator.FieldLevel) bool {
	mobile := fl.Field().String()
	matched, _ := regexp.MatchString(`^1[3-9]\d{9}$`, mobile)
	return matched
}

// 自定义验证函数：验证不包含敏感词
func validateNoSensitiveWords(fl validator.FieldLevel) bool {
	text := strings.ToLower(fl.Field().String())
	sensitiveWords := []string{"admin", "root", "system", "password"}
	for _, word := range sensitiveWords {
		if strings.Contains(text, word) {
			return false
		}
	}
	return true
}

// RegisterRequest 注册请求
type RegisterRequest struct {
	Name     string `json:"name" binding:"required,chinese_name"`
	Mobile   string `json:"mobile" binding:"required,mobile"`
	Email    string `json:"email" binding:"required,email"`
	Nickname string `json:"nickname" binding:"required,min=2,max=20,no_sensitive"`
	Password string `json:"password" binding:"required,min=8,max=64"`
}

func main() {
	r := gin.Default()

	// 注册自定义验证器
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		v.RegisterValidation("chinese_name", validateChineseName)
		v.RegisterValidation("mobile", validateMobile)
		v.RegisterValidation("no_sensitive", validateNoSensitiveWords)
	}

	r.POST("/register", func(c *gin.Context) {
		var req RegisterRequest
		if err := c.ShouldBindJSON(&req); err != nil {
			// 解析验证错误，返回友好的错误信息
			errs, ok := err.(validator.ValidationErrors)
			if !ok {
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return
			}

			// 将验证错误转换为友好的消息
			errorMessages := make(map[string]string)
			for _, e := range errs {
				switch e.Tag() {
				case "required":
					errorMessages[e.Field()] = "该字段不能为空"
				case "chinese_name":
					errorMessages[e.Field()] = "请输入2-20个中文字符的姓名"
				case "mobile":
					errorMessages[e.Field()] = "请输入有效的手机号码"
				case "email":
					errorMessages[e.Field()] = "请输入有效的邮箱地址"
				case "no_sensitive":
					errorMessages[e.Field()] = "不能包含敏感词"
				case "min":
					errorMessages[e.Field()] = "长度不能小于 " + e.Param()
				case "max":
					errorMessages[e.Field()] = "长度不能大于 " + e.Param()
				default:
					errorMessages[e.Field()] = "验证失败: " + e.Tag()
				}
			}

			c.JSON(http.StatusBadRequest, gin.H{
				"error":   "参数验证失败",
				"details": errorMessages,
			})
			return
		}

		c.JSON(http.StatusCreated, gin.H{
			"message": "注册成功",
			"user": gin.H{
				"name":     req.Name,
				"mobile":   req.Mobile,
				"email":    req.Email,
				"nickname": req.Nickname,
			},
		})
	})

	r.Run(":8080")
}
```

---

## 六、响应处理

### 6.1 JSON / XML / HTML / 纯文本 / 文件响应

```go
package main

import (
	"html/template"
	"net/http"

	"github.com/gin-gonic/gin"
)

// Product 产品结构体
type Product struct {
	ID    int     `json:"id" xml:"id"`
	Name  string  `json:"name" xml:"name"`
	Price float64 `json:"price" xml:"price"`
}

func main() {
	r := gin.Default()

	// ===== JSON 响应 =====
	r.GET("/json", func(c *gin.Context) {
		// 方式1：gin.H（本质是 map[string]any）
		c.JSON(http.StatusOK, gin.H{
			"message": "hello",
			"count":   42,
		})
	})

	r.GET("/json/struct", func(c *gin.Context) {
		// 方式2：结构体
		product := Product{ID: 1, Name: "Go 编程指南", Price: 59.9}
		c.JSON(http.StatusOK, product)
	})

	r.GET("/json/array", func(c *gin.Context) {
		// 方式3：切片
		products := []Product{
			{ID: 1, Name: "Go 编程指南", Price: 59.9},
			{ID: 2, Name: "Kubernetes 实战", Price: 79.9},
		}
		c.JSON(http.StatusOK, products)
	})

	// SecureJSON（防止 JSON 劫持，前缀 "while(1);"）
	r.GET("/json/secure", func(c *gin.Context) {
		c.SecureJSON(http.StatusOK, []string{"a", "b", "c"})
		// 输出: while(1);["a","b","c"]
	})

	// PureJSON（不转义 HTML 字符）
	r.GET("/json/pure", func(c *gin.Context) {
		c.PureJSON(http.StatusOK, gin.H{
			"html": "<b>加粗文本</b>",
		})
		// 输出: {"html":"<b>加粗文本</b>"} （不转义 < >）
	})

	// IndentedJSON（格式化输出，开发调试用）
	r.GET("/json/pretty", func(c *gin.Context) {
		c.IndentedJSON(http.StatusOK, gin.H{
			"name": "test",
			"items": []int{1, 2, 3},
		})
	})

	// ===== XML 响应 =====
	r.GET("/xml", func(c *gin.Context) {
		product := Product{ID: 1, Name: "Go 编程指南", Price: 59.9}
		c.XML(http.StatusOK, product)
	})

	// ===== HTML 响应 =====
	// 加载模板文件
	r.LoadHTMLGlob("templates/*")
	// 或者加载指定模板
	// r.LoadHTMLFiles("templates/index.html", "templates/about.html")

	r.GET("/html", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", gin.H{
			"title":   "首页",
			"message": "欢迎使用 Gin",
		})
	})

	// ===== 纯文本响应 =====
	r.GET("/text", func(c *gin.Context) {
		c.String(http.StatusOK, "Hello, %s! 你好!", "Gin")
	})

	// ===== YAML 响应 =====
	r.GET("/yaml", func(c *gin.Context) {
		c.YAML(http.StatusOK, gin.H{
			"name": "gin-app",
			"port": 8080,
		})
	})

	// ===== 文件下载 =====
	r.GET("/download", func(c *gin.Context) {
		// 直接发送文件
		c.File("./files/report.pdf")
	})

	r.GET("/download/attachment", func(c *gin.Context) {
		// 以附件形式下载（设置 Content-Disposition）
		c.FileAttachment("./files/report.pdf", "月度报告.pdf")
	})

	// ===== 重定向 =====
	r.GET("/redirect", func(c *gin.Context) {
		c.Redirect(http.StatusMovedPermanently, "https://example.com")
	})

	r.GET("/redirect/internal", func(c *gin.Context) {
		// 内部重定向（不改变 URL）
		c.Request.URL.Path = "/json"
		r.HandleContext(c)
	})

	// ===== 自定义响应头 =====
	r.GET("/custom-header", func(c *gin.Context) {
		c.Header("X-Custom-Header", "custom-value")
		c.Header("Cache-Control", "no-cache")
		c.JSON(http.StatusOK, gin.H{"message": "带自定义头的响应"})
	})

	r.Run(":8080")
}
```

HTML 模板文件示例：

```html
<!-- 文件: templates/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{ .title }}</title>
</head>
<body>
    <h1>{{ .message }}</h1>
    <p>由 Gin 框架渲染</p>
</body>
</html>
```

---

## 七、文件上传

```go
package main

import (
	"fmt"
	"net/http"
	"os"
	"path/filepath"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// 设置上传文件大小限制（默认 32 MiB）
	r.MaxMultipartMemory = 8 << 20 // 8 MiB

	// ===== 单文件上传 =====
	r.POST("/upload", func(c *gin.Context) {
		file, err := c.FormFile("file")
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "文件获取失败: " + err.Error()})
			return
		}

		// 获取文件信息
		filename := file.Filename
		size := file.Size

		// 验证文件大小
		if size > 10*1024*1024 { // 10MB
			c.JSON(http.StatusBadRequest, gin.H{"error": "文件大小不能超过 10MB"})
			return
		}

		// 验证文件类型
		ext := filepath.Ext(filename)
		allowedExts := map[string]bool{".jpg": true, ".png": true, ".gif": true, ".pdf": true}
		if !allowedExts[ext] {
			c.JSON(http.StatusBadRequest, gin.H{"error": "不支持的文件类型: " + ext})
			return
		}

		// 生成唯一文件名
		newFilename := fmt.Sprintf("%d_%s", time.Now().UnixNano(), filename)
		savePath := filepath.Join("uploads", newFilename)

		// 确保目录存在
		os.MkdirAll("uploads", 0755)

		// 保存文件
		if err := c.SaveUploadedFile(file, savePath); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "文件保存失败"})
			return
		}

		c.JSON(http.StatusOK, gin.H{
			"message":  "上传成功",
			"filename": newFilename,
			"size":     size,
		})
	})

	// ===== 多文件上传 =====
	r.POST("/upload/multiple", func(c *gin.Context) {
		form, err := c.MultipartForm()
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		files := form.File["files"] // 表单字段名
		if len(files) == 0 {
			c.JSON(http.StatusBadRequest, gin.H{"error": "请选择文件"})
			return
		}

		os.MkdirAll("uploads", 0755)

		var uploaded []string
		for _, file := range files {
			newFilename := fmt.Sprintf("%d_%s", time.Now().UnixNano(), file.Filename)
			savePath := filepath.Join("uploads", newFilename)

			if err := c.SaveUploadedFile(file, savePath); err != nil {
				c.JSON(http.StatusInternalServerError, gin.H{
					"error":    "部分文件保存失败",
					"uploaded": uploaded,
				})
				return
			}
			uploaded = append(uploaded, newFilename)
		}

		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("成功上传 %d 个文件", len(uploaded)),
			"files":   uploaded,
		})
	})

	r.Run(":8080")
}
```

---

## 八、Gin Context 详解

`gin.Context` 是 Gin 的核心类型，贯穿整个请求生命周期。

### 8.1 Context 核心方法

```go
package main

import (
	"context"
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/context-demo", func(c *gin.Context) {
		// ===== 请求信息 =====
		method := c.Request.Method       // HTTP 方法
		path := c.Request.URL.Path       // 请求路径
		fullPath := c.FullPath()         // 路由模式（如 /users/:id）
		clientIP := c.ClientIP()         // 客户端 IP
		userAgent := c.Request.UserAgent() // User-Agent
		contentType := c.ContentType()   // Content-Type

		// ===== Key-Value 存储（在中间件和处理器之间传递数据）=====
		c.Set("start_time", time.Now())
		c.Set("user_id", "admin")

		// 获取值
		startTime, exists := c.Get("start_time")
		if exists {
			log.Printf("开始时间: %v", startTime)
		}

		// 获取指定类型的值
		userID := c.GetString("user_id")
		// 还有: c.GetInt(), c.GetBool(), c.GetFloat64(),
		//       c.GetTime(), c.GetDuration(), c.GetStringSlice(),
		//       c.GetStringMap(), c.GetStringMapString()

		// ===== 获取标准 context.Context =====
		// 可以用于传递给需要 context.Context 的函数
		ctx := c.Request.Context()
		_ = ctx

		// ===== 复制 Context（用于 goroutine）=====
		// 重要：不能在 goroutine 中直接使用原始 c
		cCopy := c.Copy()
		go func() {
			time.Sleep(100 * time.Millisecond)
			log.Printf("异步操作完成, path: %s", cCopy.Request.URL.Path)
		}()

		c.JSON(http.StatusOK, gin.H{
			"method":       method,
			"path":         path,
			"full_path":    fullPath,
			"client_ip":    clientIP,
			"user_agent":   userAgent,
			"content_type": contentType,
			"user_id":      userID,
		})
	})

	// ===== Context 与超时控制 =====
	r.GET("/slow-query", func(c *gin.Context) {
		// 创建带超时的 context
		ctx, cancel := context.WithTimeout(c.Request.Context(), 3*time.Second)
		defer cancel()

		// 模拟慢查询
		result, err := slowDatabaseQuery(ctx)
		if err != nil {
			c.JSON(http.StatusGatewayTimeout, gin.H{"error": "查询超时"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"result": result})
	})

	// ===== 流程控制 =====
	r.GET("/abort-demo", func(c *gin.Context) {
		token := c.GetHeader("Authorization")
		if token == "" {
			// Abort 终止后续处理器，但不终止当前函数
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "需要认证",
			})
			return // 必须 return，否则下面的代码还会执行
		}

		// 检查是否已经 Abort
		if c.IsAborted() {
			return
		}

		c.JSON(http.StatusOK, gin.H{"message": "认证通过"})
	})

	r.Run(":8080")
}

func slowDatabaseQuery(ctx context.Context) (string, error) {
	select {
	case <-time.After(2 * time.Second):
		return "查询结果", nil
	case <-ctx.Done():
		return "", ctx.Err()
	}
}
```

---

## 九、常见坑点

### 坑点 1：在 goroutine 中使用原始 Context

```go
// 错误：在 goroutine 中直接使用 c
r.GET("/bad", func(c *gin.Context) {
	go func() {
		// 危险！c 可能在请求结束后被回收复用
		c.JSON(200, gin.H{"message": "bad"})  // panic 或数据竞争
	}()
	c.JSON(200, gin.H{"message": "ok"})
})

// 正确：使用 c.Copy()
r.GET("/good", func(c *gin.Context) {
	cCopy := c.Copy()
	go func() {
		// 使用 copy 是安全的
		log.Printf("异步处理: %s", cCopy.Request.URL.Path)
	}()
	c.JSON(200, gin.H{"message": "ok"})
})
```

### 坑点 2：Abort 后忘记 return

```go
// 错误：Abort 后继续执行
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		if c.GetHeader("Token") == "" {
			c.AbortWithStatus(401)
			// 忘记 return！后面的 c.Next() 虽然不会执行（Abort 阻止了），
			// 但当前函数中的其他代码还会执行
		}
		// 这行在 Token 为空时也会执行！
		c.Set("user", "authenticated")
		c.Next()
	}
}

// 正确：Abort 后立即 return
func AuthMiddlewareCorrect() gin.HandlerFunc {
	return func(c *gin.Context) {
		if c.GetHeader("Token") == "" {
			c.AbortWithStatus(401)
			return // 关键！
		}
		c.Set("user", "authenticated")
		c.Next()
	}
}
```

### 坑点 3：ShouldBind 只能调用一次

```go
// 错误：多次绑定 Body（Body 是 io.ReadCloser，只能读一次）
r.POST("/bad", func(c *gin.Context) {
	var req1 CreateUserRequest
	c.ShouldBindJSON(&req1) // 第一次读取 Body

	var req2 CreateUserRequest
	c.ShouldBindJSON(&req2) // 错误！Body 已经被读取完了
})

// 正确：只绑定一次，或使用 ShouldBindBodyWith
r.POST("/good", func(c *gin.Context) {
	var req1 CreateUserRequest
	if err := c.ShouldBindBodyWith(&req1, binding.JSON); err != nil {
		// ...
	}

	var req2 AnotherRequest
	if err := c.ShouldBindBodyWith(&req2, binding.JSON); err != nil {
		// 可以再次读取，因为 Body 被缓存了
	}
})
```

### 坑点 4：gin.H 的类型

```go
// gin.H 是 map[string]any 的别名
// 注意 JSON 数字类型

c.JSON(200, gin.H{
	"count": 42,      // int -> JSON number
	"price": 19.99,   // float64 -> JSON number
	"id":    int64(1), // int64 -> 在某些情况下可能精度丢失
})

// 对于大数字，考虑使用字符串
c.JSON(200, gin.H{
	"big_id": fmt.Sprintf("%d", bigInt64Value),
})
```

---

## 十、速查表

### Gin 路由方法

| 方法 | 说明 |
|------|------|
| `r.GET(path, handler)` | GET 请求 |
| `r.POST(path, handler)` | POST 请求 |
| `r.PUT(path, handler)` | PUT 请求 |
| `r.DELETE(path, handler)` | DELETE 请求 |
| `r.PATCH(path, handler)` | PATCH 请求 |
| `r.HEAD(path, handler)` | HEAD 请求 |
| `r.OPTIONS(path, handler)` | OPTIONS 请求 |
| `r.Any(path, handler)` | 所有 HTTP 方法 |
| `r.Group(prefix)` | 路由分组 |
| `r.Use(middleware)` | 注册中间件 |
| `r.Static(path, dir)` | 静态文件服务 |
| `r.StaticFS(path, fs)` | 自定义文件系统 |
| `r.NoRoute(handler)` | 404 处理 |
| `r.NoMethod(handler)` | 405 处理 |

### Context 参数获取

| 方法 | 说明 | 示例 |
|------|------|------|
| `c.Param(key)` | 路由参数 | `/user/:id` |
| `c.Query(key)` | Query 参数 | `?key=value` |
| `c.DefaultQuery(key, def)` | Query + 默认值 | |
| `c.PostForm(key)` | 表单参数 | |
| `c.FormFile(name)` | 上传文件 | |
| `c.ShouldBindJSON(&obj)` | 绑定 JSON | |
| `c.ShouldBindQuery(&obj)` | 绑定 Query | |
| `c.ShouldBindUri(&obj)` | 绑定 URI | |
| `c.ShouldBind(&obj)` | 自动绑定 | |
| `c.GetHeader(key)` | 请求头 | |
| `c.ClientIP()` | 客户端 IP | |

### Context 响应

| 方法 | 说明 |
|------|------|
| `c.JSON(code, obj)` | JSON 响应 |
| `c.XML(code, obj)` | XML 响应 |
| `c.HTML(code, name, obj)` | HTML 模板渲染 |
| `c.String(code, format, ...)` | 纯文本响应 |
| `c.YAML(code, obj)` | YAML 响应 |
| `c.File(path)` | 发送文件 |
| `c.FileAttachment(path, name)` | 文件下载 |
| `c.Redirect(code, url)` | 重定向 |
| `c.Status(code)` | 仅设置状态码 |
| `c.Header(key, value)` | 设置响应头 |

### 常用验证标签

| 标签 | 说明 | 示例 |
|------|------|------|
| `required` | 必填 | `binding:"required"` |
| `email` | 邮箱格式 | `binding:"email"` |
| `url` | URL 格式 | `binding:"url"` |
| `ip` | IP 地址 | `binding:"ip"` |
| `min` | 最小值/长度 | `binding:"min=6"` |
| `max` | 最大值/长度 | `binding:"max=100"` |
| `gte` | 大于等于 | `binding:"gte=0"` |
| `lte` | 小于等于 | `binding:"lte=130"` |
| `oneof` | 枚举 | `binding:"oneof=male female"` |
| `omitempty` | 空值跳过验证 | `binding:"omitempty,email"` |
| `dive` | 校验切片元素 | `binding:"dive,min=1"` |
| `ltefield` | 小于等于其他字段 | `binding:"ltefield=Max"` |
