# Go 代码规范与 Lint 工具

## 导读

代码规范是团队协作的基础。Go 语言从设计之初就内置了统一的格式化工具（gofmt），并通过社区形成了丰富的编码规范。本篇将系统讲解 Go 代码规范的核心要点、golangci-lint 的配置与使用，以及代码审查的关键检查项。

**本篇核心知识点：**
- gofmt/goimports 自动格式化
- Go 代码风格规范（Effective Go 要点）
- 命名规范（驼峰、缩写、接口命名）
- golangci-lint 安装与配置
- 常用 Linter 说明
- 代码审查要点清单
- Go Proverbs 箴言

---

## 一、gofmt/goimports 自动格式化

### 1.1 gofmt 基础

```bash
# gofmt 是 Go 官方的代码格式化工具
# 所有 Go 代码都应该用 gofmt 格式化，没有例外

# 检查未格式化的文件（只显示不修改）
gofmt -l .

# 格式化并覆盖文件
gofmt -w .

# 简化代码（-s 标志）
gofmt -s -w .
# -s 会做以下简化:
#   []int{int(1), int(2)} → []int{1, 2}
#   s[a:len(s)] → s[a:]
#   for x, _ = range v → for x = range v

# 查看格式化差异
gofmt -d .
```

### 1.2 goimports

```bash
# goimports = gofmt + 自动管理 import
# 安装
go install golang.org/x/tools/cmd/goimports@latest

# 格式化并自动添加/删除 import
goimports -w .

# 设置本地包前缀（用于 import 分组排序）
goimports -local github.com/myorg/myproject -w .

# import 分组规则（goimports 自动排序）:
# 1. 标准库
# 2. 第三方库
# 3. 本地包（-local 指定的前缀）
```

### 1.3 import 规范

```go
// import_convention.go
// Go import 的标准排序和分组
package main

import (
	// 第一组：标准库
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"time"

	// 第二组：第三方库（空行分隔）
	"github.com/go-chi/chi/v5"
	"github.com/redis/go-redis/v9"
	"go.uber.org/zap"

	// 第三组：本地包（空行分隔）
	// "github.com/myorg/myproject/internal/config"
	// "github.com/myorg/myproject/internal/model"
)

func main() {
	// 避免编译器报错
	_ = context.Background()
	_ = json.Marshal
	_ = fmt.Println
	_ = http.ListenAndServe
	_ = os.Exit
	_ = time.Now()
	_ = chi.NewRouter
	_ = redis.NewClient
	_ = zap.NewProduction

	fmt.Println("import 分组演示")
	fmt.Println("使用 goimports -local github.com/myorg 自动管理")
}
```

---

## 二、Go 代码风格规范

### 2.1 命名规范

```go
// naming_convention.go
// Go 命名规范完整示例
package main

import "fmt"

// === 1. 包名 ===
// 规则：小写、单个单词、不使用下划线、不使用 mixedCaps
// 好的包名: http, json, os, strconv, bufio
// 坏的包名: base_utils, common_helpers, myPackage

// === 2. 变量名 ===
// 规则：驼峰命名，首字母大写=导出，首字母小写=未导出

// 好的命名
var userCount int
var maxRetryCount int
var isActive bool
var httpClient *int // 不是 HTTPClient 也不是 HttpClient

// 坏的命名
// var user_count int    // 不使用下划线
// var UserCount int     // 不必要的导出
// var cnt int           // 除非上下文明确，避免过度缩写

// === 3. 缩写词规则 ===
// 缩写词保持全大写或全小写，不使用 mixedCase
// 好: URL, HTTP, ID, API, JSON, XML, SQL, TCP, IP
// 用法: userID（未导出）, UserID（导出）, httpServer, HTTPServer

type UserID int64    // 好：ID 全大写
type HTTPClient struct{} // 好：HTTP 全大写
// type HttpClient struct{} // 坏：Http 混合
// type UserId int64       // 坏：Id 混合

var serverURL string  // 好
var xmlParser string  // 好：全小写在词首
// var serverUrl string // 坏

// === 4. 接口命名 ===
// 规则：单方法接口用 方法名+er 后缀

// 好的接口命名
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}

type Stringer interface {
	String() string
}

// 多方法接口：描述行为的名词
type ReadWriter interface {
	Reader
	Writer
}

type UserRepository interface {
	FindByID(id int64) (*User, error)
	Create(user *User) error
}

// 不要用 I 前缀（这不是 Java/C#）
// type IUserService interface {} // 坏

type User struct {
	ID   int64
	Name string
}

// === 5. 函数命名 ===
// 规则：驼峰命名，动词开头描述行为

// 构造函数：New + 类型名
func NewUserService() *UserService { return nil }

// Getter 不使用 Get 前缀
type UserService struct {
	name string
}

func (s *UserService) Name() string { return s.name } // 好
// func (s *UserService) GetName() string // 坏：不需要 Get 前缀

// Setter 使用 Set 前缀
func (s *UserService) SetName(name string) { s.name = name }

// 布尔方法使用 Is/Has/Can/Should 等
func (u *User) IsActive() bool { return true }
func (u *User) HasPermission(perm string) bool { return true }

// === 6. 常量命名 ===
// 规则：驼峰命名（不是 SCREAMING_SNAKE_CASE）

const maxRetries = 3            // 未导出
const DefaultTimeout = 30       // 导出
const StatusActive = "active"   // 导出

// 不要这样
// const MAX_RETRIES = 3          // 坏：不使用 SCREAMING_SNAKE_CASE
// const DEFAULT_TIMEOUT = 30     // 坏

// iota 枚举
type Role int

const (
	RoleAdmin Role = iota
	RoleUser
	RoleViewer
)

// === 7. 错误变量命名 ===
// 规则：Err + 描述（sentinel error 用 Err 前缀）
// var ErrNotFound = errors.New("not found")
// var ErrTimeout = errors.New("timeout")
// 错误类型用 Error 后缀
// type NotFoundError struct{}

// === 8. 文件命名 ===
// 规则：小写 + 下划线分隔
// user_handler.go     好
// user_handler_test.go 好
// userHandler.go      坏

func main() {
	fmt.Println("=== Go 命名规范 ===")
	fmt.Println()
	fmt.Println("核心原则:")
	fmt.Println("  1. 驼峰命名，首字母大小写控制可见性")
	fmt.Println("  2. 缩写词全大写或全小写: URL, HTTP, ID")
	fmt.Println("  3. 接口: 单方法用 -er 后缀")
	fmt.Println("  4. 构造函数: New + 类型名")
	fmt.Println("  5. Getter 不加 Get 前缀")
	fmt.Println("  6. 常量: 驼峰命名，不用 SCREAMING_SNAKE_CASE")
	fmt.Println("  7. 文件名: 小写 + 下划线")
}
```

### 2.2 错误处理规范

```go
// error_handling_convention.go
// Go 错误处理的标准模式
package main

import (
	"errors"
	"fmt"
	"log/slog"
	"os"
)

// === 1. 错误定义 ===

// sentinel error（哨兵错误）：用于比较
var (
	ErrNotFound    = errors.New("not found")
	ErrUnauthorized = errors.New("unauthorized")
	ErrForbidden   = errors.New("forbidden")
)

// 自定义错误类型：需要携带额外信息时
type ValidationError struct {
	Field   string
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("验证失败: %s - %s", e.Field, e.Message)
}

// === 2. 错误处理模式 ===

// 规则 1：错误必须处理，不能忽略
func readFile(path string) ([]byte, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		// 包装错误，添加上下文
		return nil, fmt.Errorf("读取文件 %s 失败: %w", path, err)
	}
	return data, nil
}

// 规则 2：使用 %w 包装错误（保留错误链）
func processConfig(path string) error {
	data, err := readFile(path)
	if err != nil {
		return fmt.Errorf("处理配置失败: %w", err)
	}
	_ = data
	return nil
}

// 规则 3：使用 errors.Is 比较错误
func handleError(err error) {
	if errors.Is(err, ErrNotFound) {
		fmt.Println("资源未找到")
	} else if errors.Is(err, os.ErrNotExist) {
		fmt.Println("文件不存在")
	} else {
		fmt.Println("未知错误:", err)
	}
}

// 规则 4：使用 errors.As 提取错误类型
func handleValidationError(err error) {
	var ve *ValidationError
	if errors.As(err, &ve) {
		fmt.Printf("字段 %s 验证失败: %s\n", ve.Field, ve.Message)
	}
}

// 规则 5：在适当的层级处理错误，不要重复处理
func badErrorHandling() error {
	err := doSomething()
	if err != nil {
		// 坏：同时记录日志和返回错误
		// 调用者可能再次记录，导致重复日志
		slog.Error("操作失败", "error", err)
		return err
	}
	return nil
}

func goodErrorHandling() error {
	err := doSomething()
	if err != nil {
		// 好：只包装并返回，让调用者决定如何处理
		return fmt.Errorf("执行操作失败: %w", err)
	}
	return nil
}

func doSomething() error { return nil }

// 规则 6：不要使用 panic 做错误处理
func badPanic(s string) int {
	if s == "" {
		// 坏：不应该用 panic 处理预期的错误
		panic("空字符串") // 不要这样做
	}
	return len(s)
}

func goodReturn(s string) (int, error) {
	if s == "" {
		// 好：返回错误
		return 0, errors.New("字符串不能为空")
	}
	return len(s), nil
}

// 规则 7：错误字符串不要大写开头，不要以标点结尾
// 好: "connection refused"
// 坏: "Connection refused."
// 原因: 错误可能被包装 fmt.Errorf("open file: %w", err)

func main() {
	fmt.Println("=== Go 错误处理规范 ===")
	fmt.Println()
	fmt.Println("核心规则:")
	fmt.Println("  1. 所有错误必须处理（至少记录日志）")
	fmt.Println("  2. 使用 fmt.Errorf + %w 包装错误")
	fmt.Println("  3. 使用 errors.Is/errors.As 比较错误")
	fmt.Println("  4. 在合适的层级处理，不要重复处理")
	fmt.Println("  5. 不要用 panic 做错误处理")
	fmt.Println("  6. 错误字符串小写开头，无句号结尾")
}
```

### 2.3 注释规范

```go
// comment_convention.go
// Go 注释规范
package main

import "fmt"

// Package main 包注释应该在 package 声明之前
// 描述这个包的用途和核心功能。
// 如果包有多个文件，包注释只需要在一个文件中（通常是 doc.go）。

// UserService 提供用户管理的核心业务功能。
//
// 它负责用户的创建、查询、更新和删除操作，
// 并通过 Repository 接口与数据层交互。
//
// 使用示例:
//
//	svc := NewUserService(repo, cache, logger)
//	user, err := svc.GetByID(ctx, 123)
//	if err != nil {
//	    log.Fatal(err)
//	}
type UserService struct {
	// repo 是用户数据访问接口
	repo interface{}
}

// NewUserService 创建一个新的 UserService 实例。
//
// repo 不能为 nil，否则会 panic。
func NewUserService(repo interface{}) *UserService {
	return &UserService{repo: repo}
}

// GetByID 根据用户 ID 获取用户信息。
//
// 如果用户不存在，返回 ErrNotFound 错误。
// 如果发生数据库错误，返回包装后的原始错误。
func (s *UserService) GetByID(id int64) (interface{}, error) {
	return nil, nil
}

/*
注释规范总结:

1. 导出符号（大写开头）必须有注释
   - 注释以被注释符号的名字开头
   - 使用完整的英文句子（如果项目是中文也可以用中文）

2. 注释格式:
   - // 单行注释（推荐）
   - 注释与代码之间一个空格
   - 段落之间用空行分隔

3. 包注释:
   - 在 package 声明之前
   - 以 "Package xxx" 开头
   - 复杂包使用 doc.go 文件

4. TODO 注释格式:
   // TODO(author): 描述需要做的事情

5. Deprecated 标记:
   // Deprecated: 使用 NewFunction 替代。
   func OldFunction() {}

6. 代码示例（可运行的测试）:
   func ExampleUserService_GetByID() {
       // ...
       // Output: expected output
   }

7. go doc 工具:
   go doc fmt.Println      # 查看函数文档
   go doc -all fmt          # 查看包所有文档
   godoc -http=:6060        # 启动文档服务器
*/

func main() {
	fmt.Println("使用 go doc 查看文档:")
	fmt.Println("  go doc fmt.Println")
	fmt.Println("  go doc -all fmt")
}
```

---

## 三、golangci-lint

### 3.1 安装与使用

```bash
# 安装 golangci-lint
# 方式 1：二进制安装（推荐）
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# 方式 2：go install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# 方式 3：macOS Homebrew
brew install golangci-lint

# 验证安装
golangci-lint --version

# 基本使用
golangci-lint run             # 检查当前目录
golangci-lint run ./...       # 检查所有子包
golangci-lint run --fix       # 自动修复
golangci-lint run -v          # 详细输出
golangci-lint run --timeout 5m # 设置超时

# 查看启用的 linter
golangci-lint linters

# 只运行特定 linter
golangci-lint run -E errcheck,staticcheck,govet
```

### 3.2 .golangci.yml 配置文件

```yaml
# .golangci.yml
# golangci-lint 配置文件（放在项目根目录）

# 运行配置
run:
  # 超时时间
  timeout: 5m

  # 跳过的目录
  skip-dirs:
    - vendor
    - third_party
    - testdata

  # 跳过的文件
  skip-files:
    - ".*_gen\\.go$"
    - ".*\\.pb\\.go$"

  # 允许并行运行的 linter 数量
  concurrency: 4

# 输出配置
output:
  formats:
    - format: colored-line-number
  sort-results: true

# Linter 配置
linters:
  # 禁用所有默认 linter，然后显式启用
  disable-all: true
  enable:
    # === 必须启用 ===
    - errcheck       # 检查未处理的错误
    - govet          # 类似 go vet 的检查
    - staticcheck    # 全面的静态分析
    - unused         # 未使用的代码
    - ineffassign    # 无效赋值检测
    - typecheck      # 类型检查

    # === 推荐启用 ===
    - gosimple       # 代码简化建议
    - gocritic       # 代码风格和性能建议
    - revive         # golint 的替代品
    - misspell       # 拼写检查
    - unconvert      # 不必要的类型转换
    - unparam        # 未使用的函数参数
    - prealloc       # 切片预分配建议

    # === 安全相关 ===
    - gosec          # 安全问题检测

    # === 性能相关 ===
    - bodyclose      # HTTP Response Body 关闭检查
    - noctx          # HTTP 请求没有 context

    # === 风格相关 ===
    - gofmt          # 格式化检查
    - goimports      # import 管理
    - whitespace     # 多余空白行

    # === 错误处理 ===
    - wrapcheck      # 错误包装检查
    - nilerr         # nil 错误检查

# 各 Linter 的具体配置
linters-settings:
  # errcheck 配置
  errcheck:
    # 检查类型断言
    check-type-assertions: true
    # 检查空白标识符
    check-blank: true
    # 排除的函数（这些函数的错误可以忽略）
    exclude-functions:
      - fmt.Fprintf
      - fmt.Fprintln
      - fmt.Fprint
      - (*bytes.Buffer).WriteString
      - (*bytes.Buffer).Write

  # govet 配置
  govet:
    enable-all: true

  # staticcheck 配置
  staticcheck:
    checks:
      - "all"

  # gocritic 配置
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance
    disabled-checks:
      - hugeParam  # 大参数值传递（有时是故意的）

  # revive 配置
  revive:
    rules:
      - name: exported
        arguments:
          - "checkPrivateReceivers"
      - name: unused-parameter
      - name: unreachable-code
      - name: context-as-argument
      - name: error-return
      - name: error-naming
      - name: if-return

  # gosec 配置
  gosec:
    excludes:
      - G104  # 未检查的错误（errcheck 已覆盖）
      - G304  # 文件路径由变量控制（很多场景是安全的）

  # goimports 配置
  goimports:
    local-prefixes: github.com/myorg/myproject

  # misspell 配置
  misspell:
    locale: US

# 问题过滤
issues:
  # 排除特定规则在特定文件中的检查
  exclude-rules:
    # 测试文件可以有更宽松的规则
    - path: "_test\\.go"
      linters:
        - errcheck
        - gosec
        - gocritic

    # main 函数可以不处理某些错误
    - path: "cmd/.*/main\\.go"
      linters:
        - wrapcheck

    # 生成的代码不检查
    - path: "\\.gen\\.go"
      linters:
        - all

  # 最大同类问题数（0 = 无限制）
  max-same-issues: 0

  # 显示所有问题（不隐藏）
  max-issues-per-linter: 0

  # 是否显示新问题（基于 git diff）
  new: false
```

### 3.3 常用 Linter 详解

```go
// linter_examples.go
// 演示各个 Linter 检查的问题
package main

import (
	"fmt"
	"net/http"
	"os"
)

// === errcheck: 未处理的错误 ===
func errcheckExample() {
	// 坏：忽略了 os.Remove 的错误
	// os.Remove("/tmp/test")  // errcheck: Error return value is not checked

	// 好：处理错误
	if err := os.Remove("/tmp/test"); err != nil {
		fmt.Println("删除失败:", err)
	}

	// 好：显式忽略（使用 _）
	_ = os.Remove("/tmp/test") // 明确表示忽略错误
}

// === govet: 可疑的代码模式 ===
func govetExample() {
	// govet 会检查:
	// 1. Printf 格式字符串与参数不匹配
	// fmt.Printf("%d", "string") // govet: wrong type

	// 2. 结构体 tag 格式错误
	// type Bad struct {
	//     Name string `json:name` // 缺少引号
	// }

	// 3. 可疑的锁操作
	// var mu sync.Mutex
	// mu2 := mu  // govet: copying lock value
}

// === staticcheck: 全面的静态分析 ===
func staticcheckExample() {
	// SA1: 各种错误用法
	// SA4: 无用代码
	// SA5: 可疑代码
	// SA6: 性能问题
	// SA9: 可疑的赋值

	// 示例: 正则表达式编译应该在 init 时
	// for i := 0; i < 100; i++ {
	//     regexp.MustCompile(`\d+`) // SA6: 应该移到循环外
	// }
}

// === gosec: 安全问题 ===
func gosecExample() {
	// G101: 硬编码密码
	// password := "admin123" // gosec: Potential hardcoded credentials

	// G201: SQL 注入
	// query := "SELECT * FROM users WHERE id = " + userInput // 危险

	// G301: 目录权限过大
	// os.MkdirAll("/tmp/test", 0777) // gosec: 权限过宽

	// G304: 文件路径由变量控制
	// os.Open(userInput) // gosec: 潜在的路径遍历

	// G402: TLS 不安全配置
	// tls.Config{InsecureSkipVerify: true} // gosec: TLS 验证被禁用
}

// === bodyclose: HTTP Response Body 必须关闭 ===
func bodycloseExample() {
	// 坏：没有关闭 Body
	// resp, err := http.Get("https://example.com")
	// if err != nil { return }
	// data, _ := io.ReadAll(resp.Body) // bodyclose: resp.Body 未关闭

	// 好：正确关闭
	resp, err := http.Get("https://example.com")
	if err != nil {
		return
	}
	defer resp.Body.Close()
	_ = resp
}

// === ineffassign: 无效赋值 ===
func ineffassignExample() {
	x := 1
	fmt.Println(x)
	// x = 2  // 如果 x 之后没有被使用，这就是无效赋值
	x = 3
	fmt.Println(x)
}

// === prealloc: 切片预分配 ===
func preallocExample() {
	n := 100

	// 坏：可以预分配但没有
	// var result []int
	// for i := 0; i < n; i++ {
	//     result = append(result, i) // prealloc: 可以使用 make([]int, 0, n)
	// }

	// 好：预分配
	result := make([]int, 0, n)
	for i := 0; i < n; i++ {
		result = append(result, i)
	}
	_ = result
}

func main() {
	fmt.Println("=== golangci-lint 常用 Linter ===\n")

	linters := []struct {
		Name    string
		Purpose string
		Level   string
	}{
		{"errcheck", "检查未处理的错误返回值", "必须"},
		{"govet", "检查可疑代码模式（类似 go vet）", "必须"},
		{"staticcheck", "全面的静态分析（替代 golint）", "必须"},
		{"unused", "检测未使用的变量、函数、类型", "必须"},
		{"ineffassign", "检测无效的赋值语句", "必须"},
		{"gosimple", "建议简化代码", "推荐"},
		{"gocritic", "代码风格和性能建议", "推荐"},
		{"revive", "可配置的代码风格检查", "推荐"},
		{"gosec", "安全漏洞检测", "推荐"},
		{"bodyclose", "HTTP Body 关闭检查", "推荐"},
		{"noctx", "HTTP 请求缺少 context 检查", "推荐"},
		{"prealloc", "切片预分配建议", "推荐"},
		{"misspell", "英文拼写检查", "可选"},
		{"wrapcheck", "错误包装检查", "可选"},
	}

	for _, l := range linters {
		fmt.Printf("  %-15s [%s] %s\n", l.Name, l.Level, l.Purpose)
	}

	errcheckExample()
	ineffassignExample()
	preallocExample()
	bodycloseExample()
}
```

---

## 四、代码审查要点清单

### 4.1 审查清单

```go
// code_review_checklist.go
// Go 代码审查要点清单
package main

import "fmt"

func main() {
	fmt.Println("=== Go 代码审查要点清单 ===\n")

	fmt.Println("--- 1. 错误处理 ---")
	fmt.Println("  [ ] 所有错误都被处理了吗？")
	fmt.Println("  [ ] 错误使用 %w 包装了吗？")
	fmt.Println("  [ ] 错误信息有足够的上下文吗？")
	fmt.Println("  [ ] 没有重复处理（同时记日志又返回错误）吗？")
	fmt.Println("  [ ] panic 只用于真正的程序错误吗？")
	fmt.Println()

	fmt.Println("--- 2. 并发安全 ---")
	fmt.Println("  [ ] 共享变量有适当的同步机制吗？")
	fmt.Println("  [ ] goroutine 有退出机制吗（context/channel）？")
	fmt.Println("  [ ] 有 goroutine 泄漏风险吗？")
	fmt.Println("  [ ] 使用了 -race 标志测试吗？")
	fmt.Println("  [ ] channel 的关闭由发送方负责吗？")
	fmt.Println("  [ ] sync.WaitGroup 的 Add 在 goroutine 外调用吗？")
	fmt.Println()

	fmt.Println("--- 3. 资源泄漏 ---")
	fmt.Println("  [ ] 文件/连接使用 defer Close() 了吗？")
	fmt.Println("  [ ] HTTP Response Body 关闭了吗？")
	fmt.Println("  [ ] 数据库 Rows 关闭了吗？")
	fmt.Println("  [ ] context 的 cancel 被调用了吗？")
	fmt.Println("  [ ] ticker/timer 停止了吗？")
	fmt.Println()

	fmt.Println("--- 4. 性能 ---")
	fmt.Println("  [ ] 切片/Map 预分配了吗？")
	fmt.Println("  [ ] 字符串拼接使用 Builder 了吗？")
	fmt.Println("  [ ] 大结构体用指针传递了吗？")
	fmt.Println("  [ ] 有不必要的内存分配吗？")
	fmt.Println("  [ ] HTTP Client 复用了吗？")
	fmt.Println("  [ ] 正则表达式编译在循环外吗？")
	fmt.Println()

	fmt.Println("--- 5. 安全 ---")
	fmt.Println("  [ ] 有 SQL 注入风险吗？")
	fmt.Println("  [ ] 有硬编码的密钥/密码吗？")
	fmt.Println("  [ ] 用户输入有校验吗？")
	fmt.Println("  [ ] TLS 配置安全吗？")
	fmt.Println("  [ ] 日志中没有敏感信息吗？")
	fmt.Println()

	fmt.Println("--- 6. 可维护性 ---")
	fmt.Println("  [ ] 代码格式化了吗（gofmt/goimports）？")
	fmt.Println("  [ ] 导出的符号有注释吗？")
	fmt.Println("  [ ] 函数长度合理吗（建议 < 50 行）？")
	fmt.Println("  [ ] 圈复杂度合理吗（建议 < 15）？")
	fmt.Println("  [ ] 有适当的单元测试吗？")
	fmt.Println("  [ ] 接口设计合理吗（小接口优于大接口）？")
}
```

---

## 五、Go Proverbs 箴言

```go
// go_proverbs.go
// Go 箴言（Rob Pike 在 GopherFest 2015 的演讲）
// 这些箴言体现了 Go 的设计哲学
package main

import "fmt"

func main() {
	proverbs := []struct {
		English string
		Chinese string
		Explain string
	}{
		{
			"Don't communicate by sharing memory, share memory by communicating.",
			"不要通过共享内存来通信，而要通过通信来共享内存。",
			"使用 channel 而非共享变量 + mutex。",
		},
		{
			"Concurrency is not parallelism.",
			"并发不是并行。",
			"并发是结构，并行是执行。Go 的 goroutine 是并发设计，不保证并行。",
		},
		{
			"Channels orchestrate; mutexes serialize.",
			"Channel 用于编排；Mutex 用于串行化。",
			"Channel 适合协调 goroutine 的工作流，Mutex 适合保护共享状态。",
		},
		{
			"The bigger the interface, the weaker the abstraction.",
			"接口越大，抽象越弱。",
			"小接口（1-3 个方法）更灵活、更容易实现和 mock。",
		},
		{
			"Make the zero value useful.",
			"让零值有用。",
			"struct 的零值应该是可用的（如 sync.Mutex、bytes.Buffer）。",
		},
		{
			"interface{} says nothing.",
			"空接口什么也没说。",
			"尽量使用具体类型或小接口，避免 interface{}。",
		},
		{
			"Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.",
			"没人喜欢 gofmt 的风格，但人人都喜欢 gofmt。",
			"统一格式比个人偏好更重要。",
		},
		{
			"A little copying is better than a little dependency.",
			"少量复制好过少量依赖。",
			"不要为了几行代码引入一个大依赖。",
		},
		{
			"Syscall must always be guarded with build tags.",
			"系统调用必须用构建标签保护。",
			"使用 //go:build linux 等标签隔离平台特定代码。",
		},
		{
			"Cgo must always be guarded with build tags.",
			"Cgo 必须用构建标签保护。",
			"CGO 代码应该被隔离，且需要注意交叉编译。",
		},
		{
			"Cgo is not Go.",
			"Cgo 不是 Go。",
			"使用 CGO 会失去 Go 的很多优势（交叉编译、静态链接等）。",
		},
		{
			"With the unsafe package there are no guarantees.",
			"使用 unsafe 包没有任何保证。",
			"unsafe 代码可能在未来的 Go 版本中失效。",
		},
		{
			"Clear is better than clever.",
			"清晰胜过聪明。",
			"代码首先要易读，不要炫技。",
		},
		{
			"Reflection is never clear.",
			"反射永远不清晰。",
			"尽量避免 reflect 包，使用泛型或代码生成替代。",
		},
		{
			"Errors are values.",
			"错误是值。",
			"错误可以被存储、传递、包装、比较，不只是简单的 if err != nil。",
		},
		{
			"Don't just check errors, handle them gracefully.",
			"不要只检查错误，要优雅地处理它们。",
			"给错误添加上下文（%w），在适当的层级做有意义的处理。",
		},
		{
			"Design the architecture, name the components, document the details.",
			"设计架构，命名组件，记录细节。",
			"好的命名 = 好的设计。如果命名困难，可能是设计有问题。",
		},
		{
			"Documentation is for users.",
			"文档是给使用者看的。",
			"写文档时站在使用者的角度，而非实现者的角度。",
		},
	}

	fmt.Println("=== Go Proverbs — Go 箴言 ===\n")
	for i, p := range proverbs {
		fmt.Printf("%d. %s\n", i+1, p.English)
		fmt.Printf("   %s\n", p.Chinese)
		fmt.Printf("   -> %s\n\n", p.Explain)
	}
}
```

---

## 六、常见坑点

```
坑点 1：只用 gofmt 不用 goimports
  gofmt 不管理 import，用 goimports 可以自动添加缺失的、删除多余的 import。

坑点 2：golangci-lint 配置太严格
  一开始就启用所有 linter 会导致大量误报，团队抵触。
  建议从核心几个（errcheck/govet/staticcheck）开始，逐步增加。

坑点 3：忽略 linter 警告而非修复
  //nolint 应该附带原因说明。
  //nolint:errcheck // 日志写入失败可以忽略

坑点 4：review 时只关注格式不关注逻辑
  格式化交给工具，人工审查应该关注业务逻辑、并发安全、错误处理。

坑点 5：错误消息大写开头
  Go 惯例错误字符串小写开头，因为可能被包装在其他消息中。
  fmt.Errorf("connection failed: %w", err)  // 好
  fmt.Errorf("Connection failed: %w", err)  // 坏

坑点 6：interface{} 滥用
  Go 1.18+ 有泛型了，很多之前需要 interface{} 的场景可以用泛型替代。

坑点 7：过度使用 init() 函数
  init() 隐式执行，顺序依赖 import 顺序，难以测试。
  改用显式的构造函数和初始化方法。
```

---

## 七、速查表

### 命名规范速查

| 类型 | 规则 | 示例 |
|---|---|---|
| 包名 | 小写单词 | `http`, `json`, `strconv` |
| 变量 | 驼峰 | `userCount`, `isActive` |
| 常量 | 驼峰 | `MaxRetries`, `StatusActive` |
| 缩写词 | 全大写/全小写 | `userID`, `HTTPClient`, `xmlParser` |
| 接口 | -er 后缀 | `Reader`, `Writer`, `Stringer` |
| 构造函数 | New + 类型 | `NewUserService` |
| Getter | 不加 Get | `Name()` 而非 `GetName()` |
| 错误变量 | Err 前缀 | `ErrNotFound` |
| 错误类型 | Error 后缀 | `NotFoundError` |
| 文件名 | 小写下划线 | `user_handler.go` |
| 测试文件 | _test.go 后缀 | `user_handler_test.go` |

### Lint 命令速查

```bash
# 格式化
gofmt -s -w .
goimports -local github.com/myorg -w .

# 代码检查
golangci-lint run ./...
golangci-lint run --fix           # 自动修复
golangci-lint run --new-from-rev=HEAD~1  # 只检查新代码

# 安全检查
gosec ./...

# 竞态检测
go test -race ./...

# 逃逸分析
go build -gcflags="-m" ./...
```
