# 实战项目：HTTP 健康检查工具

## 导读

本篇将从零开始构建一个完整的 HTTP 健康检查 CLI 工具。它支持批量检测 URL 的健康状态、并发检查、证书过期检测、多种输出格式，是 SRE 日常运维中非常实用的工具。

**项目功能：**
- 批量检测 URL 健康状态（状态码、响应时间）
- 并发检查（Worker Pool + Channel）
- TLS 证书过期检查
- 多种输出格式（表格/JSON/CSV）
- 从文件读取 URL 列表
- 重试机制
- slog 日志记录

---

## 一、项目需求分析

```
需求清单:
  1. 接受多个 URL 作为输入（命令行参数或文件）
  2. 并发检查每个 URL 的可用性
  3. 检查项: HTTP 状态码、响应时间、TLS 证书过期时间
  4. 可配置超时时间、并发数、重试次数
  5. 支持表格、JSON、CSV 三种输出格式
  6. 返回非零退出码当有 URL 不健康时

命令示例:
  healthcheck check --urls https://google.com,https://github.com
  healthcheck check --file urls.txt --timeout 10s --workers 10
  healthcheck check --urls https://example.com --format json --retries 3
```

---

## 二、项目结构设计

```
healthcheck/
├── cmd/
│   └── healthcheck/
│       └── main.go          # 入口
├── internal/
│   ├── checker/
│   │   ├── checker.go       # 健康检查核心逻辑
│   │   └── checker_test.go
│   ├── config/
│   │   └── config.go        # 配置管理
│   ├── model/
│   │   └── result.go        # 数据模型
│   └── output/
│       ├── table.go          # 表格输出
│       ├── json.go           # JSON 输出
│       └── csv.go            # CSV 输出
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## 三、完整项目代码

### 3.1 数据模型

```go
// internal/model/result.go
// 健康检查结果数据模型
package model

import (
	"fmt"
	"time"
)

// CheckResult 单个 URL 的检查结果
type CheckResult struct {
	URL           string        `json:"url"`
	Status        string        `json:"status"`          // healthy / unhealthy / error
	StatusCode    int           `json:"status_code"`
	ResponseTime  time.Duration `json:"response_time_ms"`
	ContentLength int64         `json:"content_length"`
	TLSExpiry     *time.Time    `json:"tls_expiry,omitempty"`
	TLSDaysLeft   int           `json:"tls_days_left,omitempty"`
	Error         string        `json:"error,omitempty"`
	Retries       int           `json:"retries"`
	CheckedAt     time.Time     `json:"checked_at"`
}

// IsHealthy 判断是否健康
func (r *CheckResult) IsHealthy() bool {
	return r.Status == "healthy"
}

// StatusEmoji 返回状态标识符号
func (r *CheckResult) StatusIcon() string {
	switch r.Status {
	case "healthy":
		return "[OK]"
	case "unhealthy":
		return "[WARN]"
	default:
		return "[FAIL]"
	}
}

// FormatResponseTime 格式化响应时间
func (r *CheckResult) FormatResponseTime() string {
	if r.ResponseTime == 0 {
		return "-"
	}
	if r.ResponseTime < time.Millisecond {
		return fmt.Sprintf("%.2fus", float64(r.ResponseTime.Microseconds()))
	}
	return fmt.Sprintf("%dms", r.ResponseTime.Milliseconds())
}

// FormatTLSExpiry 格式化证书过期信息
func (r *CheckResult) FormatTLSExpiry() string {
	if r.TLSExpiry == nil {
		return "-"
	}
	days := r.TLSDaysLeft
	if days < 0 {
		return fmt.Sprintf("已过期 %d 天", -days)
	}
	if days < 30 {
		return fmt.Sprintf("%d 天 (即将过期!)", days)
	}
	return fmt.Sprintf("%d 天", days)
}

// CheckConfig 检查配置
type CheckConfig struct {
	URLs       []string      // 要检查的 URL 列表
	Timeout    time.Duration // 单个请求超时
	Workers    int           // 并发 Worker 数量
	Retries    int           // 重试次数
	RetryDelay time.Duration // 重试间隔
	Format     string        // 输出格式: table, json, csv
	Verbose    bool          // 详细输出
}

// DefaultConfig 返回默认配置
func DefaultConfig() CheckConfig {
	return CheckConfig{
		Timeout:    10 * time.Second,
		Workers:    5,
		Retries:    1,
		RetryDelay: time.Second,
		Format:     "table",
		Verbose:    false,
	}
}
```

### 3.2 健康检查核心逻辑

```go
// internal/checker/checker.go
// 健康检查核心实现
package checker

import (
	"context"
	"crypto/tls"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptrace"
	"sync"
	"time"
)

// CheckResult 检查结果
type CheckResult struct {
	URL           string
	Status        string
	StatusCode    int
	ResponseTime  time.Duration
	ContentLength int64
	TLSExpiry     *time.Time
	TLSDaysLeft   int
	Error         string
	Retries       int
	CheckedAt     time.Time
}

// IsHealthy 判断是否健康
func (r *CheckResult) IsHealthy() bool {
	return r.Status == "healthy"
}

// Checker 健康检查器
type Checker struct {
	client  *http.Client
	logger  *slog.Logger
	workers int
	retries int
	retryDelay time.Duration
}

// NewChecker 创建检查器
func NewChecker(timeout time.Duration, workers, retries int, retryDelay time.Duration, logger *slog.Logger) *Checker {
	transport := &http.Transport{
		MaxIdleConns:        100,
		MaxIdleConnsPerHost: 10,
		IdleConnTimeout:     30 * time.Second,
		TLSClientConfig: &tls.Config{
			// 不跳过证书验证，但收集证书信息
			InsecureSkipVerify: false,
		},
		TLSHandshakeTimeout: 10 * time.Second,
	}

	client := &http.Client{
		Timeout:   timeout,
		Transport: transport,
		// 不跟随重定向（检查原始 URL 的状态）
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 10 {
				return fmt.Errorf("太多重定向")
			}
			return nil // 允许重定向
		},
	}

	return &Checker{
		client:     client,
		logger:     logger,
		workers:    workers,
		retries:    retries,
		retryDelay: retryDelay,
	}
}

// checkSingle 检查单个 URL
func (c *Checker) checkSingle(ctx context.Context, url string) *CheckResult {
	result := &CheckResult{
		URL:       url,
		CheckedAt: time.Now(),
	}

	var lastErr error

	// 重试循环
	for attempt := 0; attempt <= c.retries; attempt++ {
		if attempt > 0 {
			c.logger.Debug("重试检查",
				"url", url,
				"attempt", attempt,
				"max_retries", c.retries,
			)
			time.Sleep(c.retryDelay)
		}

		result.Retries = attempt
		err := c.doCheck(ctx, url, result)
		if err == nil {
			return result
		}
		lastErr = err
	}

	// 所有重试都失败
	result.Status = "error"
	result.Error = lastErr.Error()
	return result
}

// doCheck 执行一次检查
func (c *Checker) doCheck(ctx context.Context, url string, result *CheckResult) error {
	// 创建请求
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return fmt.Errorf("创建请求失败: %w", err)
	}
	req.Header.Set("User-Agent", "HealthChecker/1.0")

	// 使用 httptrace 记录详细的连接信息
	var tlsState *tls.ConnectionState
	trace := &httptrace.ClientTrace{
		TLSHandshakeDone: func(state tls.ConnectionState, err error) {
			if err == nil {
				tlsState = &state
			}
		},
	}
	req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))

	// 发起请求
	start := time.Now()
	resp, err := c.client.Do(req)
	result.ResponseTime = time.Since(start)

	if err != nil {
		return fmt.Errorf("请求失败: %w", err)
	}
	defer func() {
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
	}()

	// 记录结果
	result.StatusCode = resp.StatusCode
	result.ContentLength = resp.ContentLength

	// 判断健康状态
	if resp.StatusCode >= 200 && resp.StatusCode < 400 {
		result.Status = "healthy"
	} else {
		result.Status = "unhealthy"
	}

	// 检查 TLS 证书
	if tlsState != nil && len(tlsState.PeerCertificates) > 0 {
		cert := tlsState.PeerCertificates[0]
		expiry := cert.NotAfter
		result.TLSExpiry = &expiry
		result.TLSDaysLeft = int(time.Until(expiry).Hours() / 24)

		// 证书即将过期告警
		if result.TLSDaysLeft < 30 {
			c.logger.Warn("证书即将过期",
				"url", url,
				"days_left", result.TLSDaysLeft,
				"expiry", expiry.Format("2006-01-02"),
			)
		}
	}

	return nil
}

// CheckAll 并发检查所有 URL
func (c *Checker) CheckAll(ctx context.Context, urls []string) []*CheckResult {
	results := make([]*CheckResult, len(urls))

	// 创建任务 channel
	tasks := make(chan int, len(urls))

	// 启动 Worker Pool
	var wg sync.WaitGroup
	workerCount := c.workers
	if workerCount > len(urls) {
		workerCount = len(urls)
	}

	c.logger.Info("开始健康检查",
		"urls", len(urls),
		"workers", workerCount,
		"retries", c.retries,
	)

	for w := 0; w < workerCount; w++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for idx := range tasks {
				url := urls[idx]
				c.logger.Debug("检查中", "worker", workerID, "url", url)

				result := c.checkSingle(ctx, url)
				results[idx] = result

				c.logger.Debug("检查完成",
					"worker", workerID,
					"url", url,
					"status", result.Status,
					"response_time", result.ResponseTime,
				)
			}
		}(w)
	}

	// 发送任务
	for i := range urls {
		tasks <- i
	}
	close(tasks)

	// 等待所有 Worker 完成
	wg.Wait()

	return results
}

// Summary 生成检查摘要
type Summary struct {
	Total     int
	Healthy   int
	Unhealthy int
	Errors    int
	AvgTime   time.Duration
	MaxTime   time.Duration
	MinTime   time.Duration
}

// Summarize 汇总检查结果
func Summarize(results []*CheckResult) Summary {
	s := Summary{
		Total: len(results),
	}

	if len(results) == 0 {
		return s
	}

	var totalTime time.Duration
	s.MinTime = results[0].ResponseTime

	for _, r := range results {
		switch r.Status {
		case "healthy":
			s.Healthy++
		case "unhealthy":
			s.Unhealthy++
		default:
			s.Errors++
		}

		if r.ResponseTime > 0 {
			totalTime += r.ResponseTime
			if r.ResponseTime > s.MaxTime {
				s.MaxTime = r.ResponseTime
			}
			if r.ResponseTime < s.MinTime || s.MinTime == 0 {
				s.MinTime = r.ResponseTime
			}
		}
	}

	if s.Total > 0 {
		s.AvgTime = totalTime / time.Duration(s.Total)
	}

	return s
}
```

### 3.3 输出格式

```go
// internal/output/table.go
// 表格格式输出
package output

import (
	"fmt"
	"io"
	"strings"
	"time"
)

// CheckResult 输出用的检查结果接口
type CheckResult struct {
	URL           string
	Status        string
	StatusCode    int
	ResponseTime  time.Duration
	ContentLength int64
	TLSExpiry     *time.Time
	TLSDaysLeft   int
	Error         string
	Retries       int
}

// Summary 摘要信息
type Summary struct {
	Total     int
	Healthy   int
	Unhealthy int
	Errors    int
	AvgTime   time.Duration
	MaxTime   time.Duration
	MinTime   time.Duration
}

// WriteTable 输出表格格式
func WriteTable(w io.Writer, results []*CheckResult, summary Summary) {
	// 表头
	fmt.Fprintf(w, "\n%-6s %-50s %-6s %-12s %-10s %-15s %s\n",
		"状态", "URL", "代码", "响应时间", "大小", "证书剩余", "错误")
	fmt.Fprintln(w, strings.Repeat("-", 120))

	for _, r := range results {
		statusIcon := "[OK]"
		switch r.Status {
		case "unhealthy":
			statusIcon = "[WARN]"
		case "error":
			statusIcon = "[FAIL]"
		}

		respTime := "-"
		if r.ResponseTime > 0 {
			respTime = fmt.Sprintf("%dms", r.ResponseTime.Milliseconds())
		}

		size := "-"
		if r.ContentLength > 0 {
			if r.ContentLength > 1024*1024 {
				size = fmt.Sprintf("%.1fMB", float64(r.ContentLength)/1024/1024)
			} else if r.ContentLength > 1024 {
				size = fmt.Sprintf("%.1fKB", float64(r.ContentLength)/1024)
			} else {
				size = fmt.Sprintf("%dB", r.ContentLength)
			}
		}

		tlsInfo := "-"
		if r.TLSExpiry != nil {
			if r.TLSDaysLeft < 0 {
				tlsInfo = fmt.Sprintf("已过期%d天", -r.TLSDaysLeft)
			} else if r.TLSDaysLeft < 30 {
				tlsInfo = fmt.Sprintf("%d天(警告!)", r.TLSDaysLeft)
			} else {
				tlsInfo = fmt.Sprintf("%d天", r.TLSDaysLeft)
			}
		}

		errMsg := ""
		if r.Error != "" {
			errMsg = r.Error
			if len(errMsg) > 40 {
				errMsg = errMsg[:40] + "..."
			}
		}

		url := r.URL
		if len(url) > 50 {
			url = url[:47] + "..."
		}

		fmt.Fprintf(w, "%-6s %-50s %-6d %-12s %-10s %-15s %s\n",
			statusIcon, url, r.StatusCode, respTime, size, tlsInfo, errMsg)
	}

	// 摘要
	fmt.Fprintln(w, strings.Repeat("-", 120))
	fmt.Fprintf(w, "\n摘要: 总计 %d | 健康 %d | 异常 %d | 错误 %d\n",
		summary.Total, summary.Healthy, summary.Unhealthy, summary.Errors)
	fmt.Fprintf(w, "响应时间: 平均 %dms | 最快 %dms | 最慢 %dms\n",
		summary.AvgTime.Milliseconds(),
		summary.MinTime.Milliseconds(),
		summary.MaxTime.Milliseconds())
}
```

```go
// internal/output/json_output.go
// JSON 格式输出
package output

import (
	"encoding/json"
	"fmt"
	"io"
	"time"
)

// JSONReport JSON 输出格式
type JSONReport struct {
	CheckedAt string              `json:"checked_at"`
	Summary   JSONSummary         `json:"summary"`
	Results   []JSONCheckResult   `json:"results"`
}

type JSONSummary struct {
	Total     int    `json:"total"`
	Healthy   int    `json:"healthy"`
	Unhealthy int    `json:"unhealthy"`
	Errors    int    `json:"errors"`
	AvgTimeMs int64  `json:"avg_response_time_ms"`
}

type JSONCheckResult struct {
	URL            string  `json:"url"`
	Status         string  `json:"status"`
	StatusCode     int     `json:"status_code"`
	ResponseTimeMs int64   `json:"response_time_ms"`
	TLSDaysLeft    int     `json:"tls_days_left,omitempty"`
	Error          string  `json:"error,omitempty"`
	Retries        int     `json:"retries"`
}

// WriteJSON 输出 JSON 格式
func WriteJSON(w io.Writer, results []*CheckResult, summary Summary) error {
	report := JSONReport{
		CheckedAt: time.Now().Format(time.RFC3339),
		Summary: JSONSummary{
			Total:     summary.Total,
			Healthy:   summary.Healthy,
			Unhealthy: summary.Unhealthy,
			Errors:    summary.Errors,
			AvgTimeMs: summary.AvgTime.Milliseconds(),
		},
	}

	for _, r := range results {
		report.Results = append(report.Results, JSONCheckResult{
			URL:            r.URL,
			Status:         r.Status,
			StatusCode:     r.StatusCode,
			ResponseTimeMs: r.ResponseTime.Milliseconds(),
			TLSDaysLeft:    r.TLSDaysLeft,
			Error:          r.Error,
			Retries:        r.Retries,
		})
	}

	encoder := json.NewEncoder(w)
	encoder.SetIndent("", "  ")
	return encoder.Encode(report)
}

// WriteCSV 输出 CSV 格式
func WriteCSV(w io.Writer, results []*CheckResult) {
	// CSV 头
	fmt.Fprintln(w, "url,status,status_code,response_time_ms,tls_days_left,error,retries")

	for _, r := range results {
		errMsg := r.Error
		// CSV 中的双引号需要转义
		if errMsg != "" {
			errMsg = `"` + errMsg + `"`
		}

		fmt.Fprintf(w, "%s,%s,%d,%d,%d,%s,%d\n",
			r.URL,
			r.Status,
			r.StatusCode,
			r.ResponseTime.Milliseconds(),
			r.TLSDaysLeft,
			errMsg,
			r.Retries,
		)
	}
}
```

### 3.4 CLI 主程序（Cobra）

```go
// cmd/healthcheck/main.go
// 健康检查工具入口 — 使用 Cobra 框架
package main

import (
	"bufio"
	"context"
	"fmt"
	"log/slog"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"

	"github.com/spf13/cobra"
)

// 版本信息
var (
	version   = "dev"
	gitCommit = "unknown"
	buildDate = "unknown"
)

// 命令行参数
var (
	urls       string
	urlFile    string
	timeout    time.Duration
	workers    int
	retries    int
	retryDelay time.Duration
	format     string
	verbose    bool
)

// rootCmd 根命令
var rootCmd = &cobra.Command{
	Use:   "healthcheck",
	Short: "HTTP 健康检查工具",
	Long: `healthcheck 是一个高性能的 HTTP 健康检查 CLI 工具。
支持批量检测 URL 健康状态、TLS 证书过期检查、多种输出格式。`,
	Version: fmt.Sprintf("%s (commit: %s, built: %s)", version, gitCommit, buildDate),
}

// checkCmd 检查命令
var checkCmd = &cobra.Command{
	Use:   "check",
	Short: "执行健康检查",
	Long:  "对指定的 URL 列表执行健康检查",
	Example: `  # 检查单个 URL
  healthcheck check --urls https://google.com

  # 检查多个 URL
  healthcheck check --urls https://google.com,https://github.com

  # 从文件读取 URL
  healthcheck check --file urls.txt

  # 自定义参数
  healthcheck check --urls https://example.com --timeout 5s --workers 10 --retries 3

  # JSON 输出
  healthcheck check --urls https://example.com --format json`,
	RunE: runCheck,
}

func init() {
	// 注册 check 子命令
	rootCmd.AddCommand(checkCmd)

	// check 命令的参数
	checkCmd.Flags().StringVar(&urls, "urls", "", "要检查的 URL 列表（逗号分隔）")
	checkCmd.Flags().StringVar(&urlFile, "file", "", "包含 URL 的文件（每行一个）")
	checkCmd.Flags().DurationVar(&timeout, "timeout", 10*time.Second, "单个请求超时时间")
	checkCmd.Flags().IntVar(&workers, "workers", 5, "并发 Worker 数量")
	checkCmd.Flags().IntVar(&retries, "retries", 1, "失败重试次数")
	checkCmd.Flags().DurationVar(&retryDelay, "retry-delay", time.Second, "重试间隔")
	checkCmd.Flags().StringVar(&format, "format", "table", "输出格式: table, json, csv")
	checkCmd.Flags().BoolVar(&verbose, "verbose", false, "详细输出")
}

// parseURLs 解析 URL 列表
func parseURLs() ([]string, error) {
	var urlList []string

	// 从 --urls 参数解析
	if urls != "" {
		for _, u := range strings.Split(urls, ",") {
			u = strings.TrimSpace(u)
			if u != "" {
				urlList = append(urlList, u)
			}
		}
	}

	// 从文件读取
	if urlFile != "" {
		f, err := os.Open(urlFile)
		if err != nil {
			return nil, fmt.Errorf("打开文件失败: %w", err)
		}
		defer f.Close()

		scanner := bufio.NewScanner(f)
		for scanner.Scan() {
			line := strings.TrimSpace(scanner.Text())
			// 跳过空行和注释
			if line == "" || strings.HasPrefix(line, "#") {
				continue
			}
			urlList = append(urlList, line)
		}
		if err := scanner.Err(); err != nil {
			return nil, fmt.Errorf("读取文件失败: %w", err)
		}
	}

	if len(urlList) == 0 {
		return nil, fmt.Errorf("没有指定要检查的 URL，使用 --urls 或 --file 参数")
	}

	return urlList, nil
}

// runCheck 执行健康检查
func runCheck(cmd *cobra.Command, args []string) error {
	// 初始化日志
	logLevel := slog.LevelInfo
	if verbose {
		logLevel = slog.LevelDebug
	}
	logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
		Level: logLevel,
	}))

	// 解析 URL
	urlList, err := parseURLs()
	if err != nil {
		return err
	}

	logger.Info("开始健康检查",
		"urls", len(urlList),
		"timeout", timeout,
		"workers", workers,
		"retries", retries,
	)

	// 创建可取消的 context
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 处理中断信号
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		logger.Warn("收到中断信号，正在停止...")
		cancel()
	}()

	// 执行检查（这里简化实现，不使用单独的 checker 包）
	results := doCheck(ctx, urlList, logger)

	// 统计摘要
	summary := summarize(results)

	// 输出结果
	switch format {
	case "json":
		outputJSON(results, summary)
	case "csv":
		outputCSV(results)
	default:
		outputTable(results, summary)
	}

	// 有不健康的 URL 时返回非零退出码
	if summary.unhealthy > 0 || summary.errors > 0 {
		return fmt.Errorf("检查完成: %d 个不健康, %d 个错误",
			summary.unhealthy, summary.errors)
	}

	return nil
}

// === 简化的检查实现（实际项目中放 checker 包） ===

type checkResult struct {
	url           string
	status        string
	statusCode    int
	responseTime  time.Duration
	contentLength int64
	tlsDaysLeft   int
	err           string
	retries       int
}

type checkSummary struct {
	total     int
	healthy   int
	unhealthy int
	errors    int
	avgTime   time.Duration
}

func doCheck(ctx context.Context, urls []string, logger *slog.Logger) []*checkResult {
	results := make([]*checkResult, len(urls))
	type task struct {
		index int
		url   string
	}

	taskCh := make(chan task, len(urls))
	var wg = make(chan struct{}, workers)

	// 结果收集器
	resultCh := make(chan struct {
		index  int
		result *checkResult
	}, len(urls))

	// 启动检查
	go func() {
		for t := range taskCh {
			wg <- struct{}{} // 限制并发
			go func(t task) {
				defer func() { <-wg }()
				r := checkURL(ctx, t.url, timeout, retries, retryDelay, logger)
				resultCh <- struct {
					index  int
					result *checkResult
				}{t.index, r}
			}(t)
		}
	}()

	// 发送任务
	for i, url := range urls {
		taskCh <- task{index: i, url: url}
	}
	close(taskCh)

	// 收集结果
	for i := 0; i < len(urls); i++ {
		r := <-resultCh
		results[r.index] = r.result
	}

	return results
}

func checkURL(ctx context.Context, url string, timeout time.Duration, maxRetries int, retryDelay time.Duration, logger *slog.Logger) *checkResult {
	result := &checkResult{url: url}

	for attempt := 0; attempt <= maxRetries; attempt++ {
		if attempt > 0 {
			time.Sleep(retryDelay)
		}
		result.retries = attempt

		start := time.Now()
		// 简化的 HTTP 检查
		client := &http.Client{Timeout: timeout}
		req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
		if err != nil {
			result.status = "error"
			result.err = err.Error()
			continue
		}
		req.Header.Set("User-Agent", "HealthChecker/1.0")

		resp, err := client.Do(req)
		result.responseTime = time.Since(start)

		if err != nil {
			result.status = "error"
			result.err = err.Error()
			continue
		}

		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()

		result.statusCode = resp.StatusCode
		result.contentLength = resp.ContentLength

		if resp.TLS != nil && len(resp.TLS.PeerCertificates) > 0 {
			expiry := resp.TLS.PeerCertificates[0].NotAfter
			result.tlsDaysLeft = int(time.Until(expiry).Hours() / 24)
		}

		if resp.StatusCode >= 200 && resp.StatusCode < 400 {
			result.status = "healthy"
			return result
		}

		result.status = "unhealthy"
		return result
	}

	return result
}

func summarize(results []*checkResult) checkSummary {
	s := checkSummary{total: len(results)}
	var totalTime time.Duration
	for _, r := range results {
		switch r.status {
		case "healthy":
			s.healthy++
		case "unhealthy":
			s.unhealthy++
		default:
			s.errors++
		}
		totalTime += r.responseTime
	}
	if s.total > 0 {
		s.avgTime = totalTime / time.Duration(s.total)
	}
	return s
}

func outputTable(results []*checkResult, summary checkSummary) {
	fmt.Printf("\n%-6s %-55s %-6s %-10s %-10s\n", "状态", "URL", "代码", "响应时间", "证书")
	fmt.Println(strings.Repeat("-", 95))
	for _, r := range results {
		icon := "[OK]"
		if r.status == "unhealthy" {
			icon = "[WARN]"
		} else if r.status == "error" {
			icon = "[FAIL]"
		}
		respTime := fmt.Sprintf("%dms", r.responseTime.Milliseconds())
		tlsInfo := "-"
		if r.tlsDaysLeft > 0 {
			tlsInfo = fmt.Sprintf("%d天", r.tlsDaysLeft)
		}
		url := r.url
		if len(url) > 55 {
			url = url[:52] + "..."
		}
		fmt.Printf("%-6s %-55s %-6d %-10s %-10s\n", icon, url, r.statusCode, respTime, tlsInfo)
	}
	fmt.Println(strings.Repeat("-", 95))
	fmt.Printf("\n总计: %d | 健康: %d | 异常: %d | 错误: %d | 平均响应: %dms\n",
		summary.total, summary.healthy, summary.unhealthy, summary.errors, summary.avgTime.Milliseconds())
}

func outputJSON(results []*checkResult, summary checkSummary) {
	// 简化版 JSON 输出
	fmt.Println("{")
	fmt.Printf("  \"summary\": {\"total\": %d, \"healthy\": %d, \"unhealthy\": %d, \"errors\": %d},\n",
		summary.total, summary.healthy, summary.unhealthy, summary.errors)
	fmt.Println("  \"results\": [")
	for i, r := range results {
		comma := ","
		if i == len(results)-1 {
			comma = ""
		}
		fmt.Printf("    {\"url\": %q, \"status\": %q, \"status_code\": %d, \"response_time_ms\": %d}%s\n",
			r.url, r.status, r.statusCode, r.responseTime.Milliseconds(), comma)
	}
	fmt.Println("  ]")
	fmt.Println("}")
}

func outputCSV(results []*checkResult) {
	fmt.Println("url,status,status_code,response_time_ms,tls_days_left,error")
	for _, r := range results {
		fmt.Printf("%s,%s,%d,%d,%d,%s\n",
			r.url, r.status, r.statusCode, r.responseTime.Milliseconds(), r.tlsDaysLeft, r.err)
	}
}

func main() {
	if err := rootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```

**需要的导入补充（main.go 完整 import）：**

```go
import (
	"bufio"
	"context"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"

	"github.com/spf13/cobra"
)
```

### 3.5 Makefile

```makefile
# healthcheck 项目 Makefile

APP_NAME := healthcheck
VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)

.PHONY: build test lint run clean

build:
	CGO_ENABLED=0 go build -trimpath \
		-ldflags="-s -w -X main.version=$(VERSION)" \
		-o build/$(APP_NAME) ./cmd/healthcheck/

test:
	go test -v -race ./...

lint:
	golangci-lint run ./...

run:
	go run ./cmd/healthcheck/ check --urls https://google.com,https://github.com --verbose

clean:
	rm -rf build/
```

---

## 四、常见坑点

```
坑点 1：HTTP Client 没有超时
  默认 http.Client 没有超时，请求可能永远阻塞。
  必须设置 Timeout。

坑点 2：Response Body 未关闭
  不关闭 Body 会导致连接泄漏，长时间运行后 fd 耗尽。
  即使不需要 Body 内容，也要 io.Copy(io.Discard, resp.Body) 然后 Close。

坑点 3：Worker Pool 中 goroutine 泄漏
  如果 context 被取消但 goroutine 还在等待 channel，会泄漏。
  Worker 中要 select ctx.Done()。

坑点 4：TLS 证书检查被 InsecureSkipVerify 跳过
  设置 InsecureSkipVerify=true 会跳过证书验证。
  检查证书过期应保持 InsecureSkipVerify=false。

坑点 5：大量 URL 导致 goroutine 暴涨
  不能给每个 URL 都启动一个 goroutine。
  使用 Worker Pool 限制并发数。
```

---

## 五、速查表

| 功能 | 命令 |
|---|---|
| 单个 URL | `healthcheck check --urls https://example.com` |
| 多个 URL | `healthcheck check --urls url1,url2,url3` |
| 从文件 | `healthcheck check --file urls.txt` |
| JSON 输出 | `healthcheck check --urls ... --format json` |
| CSV 输出 | `healthcheck check --urls ... --format csv` |
| 高并发 | `healthcheck check --urls ... --workers 20` |
| 重试 | `healthcheck check --urls ... --retries 3` |
| 详细日志 | `healthcheck check --urls ... --verbose` |
