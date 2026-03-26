# Prometheus 客户端开发完全指南

## 导读

Prometheus 是云原生监控领域的事实标准。作为 SRE/DevOps 工程师，监控是你的"眼睛"——没有好的监控，一切运维都是盲人摸象。

Go 语言的 Prometheus 客户端库（`prometheus/client_golang`）让你可以在 Go 应用中轻松定义和暴露指标，也可以开发自定义 Exporter 来监控任何系统。

本文基于 **Go 1.24**，系统讲解 Prometheus 客户端库的核心用法，从数据模型到四种指标类型，从指标注册到 Exporter 开发，最后实现一个完整的 **自定义应用 Exporter**。

### 你将学到

- Prometheus 数据模型
- 四种指标类型（Counter、Gauge、Histogram、Summary）
- Go Prometheus 客户端库使用
- 注册指标与暴露 /metrics 端点
- 自定义 Exporter 开发
- Collector 接口实现
- Go 运行时指标
- 指标命名规范
- SRE 实战：自定义应用 Exporter

---

## 一、Prometheus 数据模型

### 1.1 时间序列

Prometheus 的核心是时间序列数据。每个时间序列由以下元素组成：

```
┌─────────────────────────────────────────────────────────┐
│          Prometheus 时间序列数据模型                       │
│                                                         │
│  指标名称 (Metric Name)                                  │
│     │                                                   │
│     │    标签 (Labels)                                   │
│     │       │                                           │
│     ▼       ▼                                           │
│  http_requests_total{method="GET", status="200"}        │
│                                                         │
│     时间戳 (Timestamp)    值 (Value)                     │
│         │                    │                          │
│         ▼                    ▼                          │
│     1711234567890        42.0                           │
│                                                         │
│  完整表示:                                               │
│  http_requests_total{method="GET",status="200"} 42 @ts  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 数据模型要素

| 要素 | 说明 | 示例 |
|------|------|------|
| Metric Name | 指标名称，描述被测量的内容 | `http_requests_total` |
| Labels | 键值对标签，区分不同维度 | `method="GET"`, `status="200"` |
| Timestamp | 毫秒级 Unix 时间戳 | `1711234567890` |
| Value | float64 数值 | `42.0` |

### 1.3 指标命名规范

```
命名格式：<namespace>_<subsystem>_<name>_<unit>

示例：
  prometheus_http_requests_total        -- Prometheus HTTP 请求总数
  process_cpu_seconds_total             -- 进程 CPU 使用秒数
  go_memstats_alloc_bytes               -- Go 内存分配字节数
  node_disk_read_bytes_total            -- 节点磁盘读取字节总数

规则：
  1. 使用小写字母和下划线
  2. 以应用前缀或领域开头（namespace）
  3. Counter 类型以 _total 结尾
  4. 使用基础单位（bytes 而非 kilobytes，seconds 而非 milliseconds）
  5. 标签名也使用小写字母和下划线
```

---

## 二、四种指标类型

### 2.1 Counter（计数器）

Counter 是一个只增不减的累积值，通常用于记录请求数、错误数、任务完成数等。

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// 定义 Counter 指标
var (
	// 简单 Counter（不带标签）
	requestsProcessed = prometheus.NewCounter(prometheus.CounterOpts{
		Namespace: "myapp",
		Subsystem: "http",
		Name:      "requests_processed_total",
		Help:      "处理的 HTTP 请求总数",
	})

	// 带标签的 Counter（CounterVec）
	requestsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "requests_total",
			Help:      "按方法和状态码分类的 HTTP 请求总数",
		},
		[]string{"method", "status_code", "path"}, // 标签名
	)

	// 错误计数器
	errorsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Name:      "errors_total",
			Help:      "应用错误总数，按类型分类",
		},
		[]string{"type"}, // error_type: "timeout", "validation", "internal"
	)
)

func init() {
	// 注册指标
	prometheus.MustRegister(requestsProcessed)
	prometheus.MustRegister(requestsTotal)
	prometheus.MustRegister(errorsTotal)
}

func main() {
	// 模拟请求处理
	http.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
		// 简单计数器加 1
		requestsProcessed.Inc()

		// 带标签的计数器
		statusCode := "200"
		if rand.Float64() < 0.1 { // 10% 错误率
			statusCode = "500"
			errorsTotal.WithLabelValues("internal").Inc()
		}
		requestsTotal.WithLabelValues(r.Method, statusCode, "/api/users").Inc()

		w.WriteHeader(200)
		fmt.Fprintln(w, "OK")
	})

	// 暴露 /metrics 端点
	http.Handle("/metrics", promhttp.Handler())

	fmt.Println("服务启动在 :8080")
	fmt.Println("指标端点: http://localhost:8080/metrics")
	http.ListenAndServe(":8080", nil)
}

// Counter 的关键方法：
// Inc()                 -- 加 1
// Add(float64)          -- 加指定值（必须 > 0）
// WithLabelValues(...)  -- 指定标签值（CounterVec）

// 错误用法：
// counter.Add(-1)       -- 报错！Counter 不能减少
// counter.Set(100)      -- 报错！Counter 没有 Set 方法
```

### 2.2 Gauge（仪表盘）

Gauge 可以任意增减，适合表示当前状态值，如温度、内存使用量、在线用户数等。

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"runtime"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// 当前在线用户数
	onlineUsers = prometheus.NewGauge(prometheus.GaugeOpts{
		Namespace: "myapp",
		Name:      "online_users",
		Help:      "当前在线用户数",
	})

	// 内存使用量
	memoryUsageBytes = prometheus.NewGauge(prometheus.GaugeOpts{
		Namespace: "myapp",
		Name:      "memory_usage_bytes",
		Help:      "应用内存使用量（字节）",
	})

	// 带标签的 Gauge
	queueSize = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Name:      "queue_size",
			Help:      "各队列当前大小",
		},
		[]string{"queue_name"},
	)

	// CPU 温度（示例）
	cpuTemperature = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "node",
			Name:      "cpu_temperature_celsius",
			Help:      "CPU 核心温度（摄氏度）",
		},
		[]string{"core"},
	)

	// 任务执行耗时（用 Gauge 记录最后一次耗时）
	lastJobDurationSeconds = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "job",
			Name:      "last_duration_seconds",
			Help:      "最后一次任务执行耗时（秒）",
		},
		[]string{"job_name"},
	)

	// 最后成功执行的时间戳
	lastSuccessTimestamp = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "job",
			Name:      "last_success_timestamp",
			Help:      "最后一次成功执行的 Unix 时间戳",
		},
		[]string{"job_name"},
	)
)

func init() {
	prometheus.MustRegister(onlineUsers)
	prometheus.MustRegister(memoryUsageBytes)
	prometheus.MustRegister(queueSize)
	prometheus.MustRegister(cpuTemperature)
	prometheus.MustRegister(lastJobDurationSeconds)
	prometheus.MustRegister(lastSuccessTimestamp)
}

func main() {
	// 模拟指标更新
	go func() {
		for {
			// 更新在线用户数
			onlineUsers.Set(float64(100 + rand.Intn(50)))

			// 更新内存使用
			var m runtime.MemStats
			runtime.ReadMemStats(&m)
			memoryUsageBytes.Set(float64(m.Alloc))

			// 更新队列大小
			queueSize.WithLabelValues("email").Set(float64(rand.Intn(100)))
			queueSize.WithLabelValues("notification").Set(float64(rand.Intn(200)))
			queueSize.WithLabelValues("task").Set(float64(rand.Intn(50)))

			// 模拟 CPU 温度
			cpuTemperature.WithLabelValues("0").Set(45 + rand.Float64()*20)
			cpuTemperature.WithLabelValues("1").Set(45 + rand.Float64()*20)

			time.Sleep(5 * time.Second)
		}
	}()

	// 模拟定时任务
	go func() {
		for {
			start := time.Now()

			// 模拟任务执行
			time.Sleep(time.Duration(rand.Intn(5)) * time.Second)

			duration := time.Since(start).Seconds()
			lastJobDurationSeconds.WithLabelValues("data-sync").Set(duration)
			lastSuccessTimestamp.WithLabelValues("data-sync").SetToCurrentTime()

			time.Sleep(30 * time.Second)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("服务启动在 :8080")
	http.ListenAndServe(":8080", nil)
}

// Gauge 的关键方法：
// Set(float64)          -- 设置为指定值
// Inc()                 -- 加 1
// Dec()                 -- 减 1
// Add(float64)          -- 加指定值（可以为负）
// Sub(float64)          -- 减指定值
// SetToCurrentTime()    -- 设置为当前 Unix 时间戳
```

### 2.3 Histogram（直方图）

Histogram 用于观察值的分布情况，如请求延迟、响应大小。它会自动计算分桶统计、总和、总数。

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// HTTP 请求延迟直方图
	requestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "request_duration_seconds",
			Help:      "HTTP 请求延迟分布（秒）",
			// 自定义分桶边界
			// 适合 Web 应用的延迟分布
			Buckets: []float64{
				0.001,  // 1ms
				0.005,  // 5ms
				0.01,   // 10ms
				0.025,  // 25ms
				0.05,   // 50ms
				0.1,    // 100ms
				0.25,   // 250ms
				0.5,    // 500ms
				1.0,    // 1s
				2.5,    // 2.5s
				5.0,    // 5s
				10.0,   // 10s
			},
		},
		[]string{"method", "path", "status_code"},
	)

	// 响应大小直方图
	responseSize = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "response_size_bytes",
			Help:      "HTTP 响应大小分布（字节）",
			// 使用指数桶：100B, 1KB, 10KB, 100KB, 1MB, 10MB
			Buckets: prometheus.ExponentialBuckets(100, 10, 6),
		},
		[]string{"method", "path"},
	)

	// 数据库查询延迟
	dbQueryDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "database",
			Name:      "query_duration_seconds",
			Help:      "数据库查询延迟分布",
			// 使用默认桶（DefBuckets）
			// .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
			Buckets: prometheus.DefBuckets,
		},
		[]string{"operation", "table"},
	)
)

func init() {
	prometheus.MustRegister(requestDuration)
	prometheus.MustRegister(responseSize)
	prometheus.MustRegister(dbQueryDuration)
}

func main() {
	// 模拟 HTTP 请求处理
	http.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
		// 使用 Timer 自动记录延迟
		timer := prometheus.NewTimer(
			requestDuration.WithLabelValues(r.Method, "/api/data", "200"),
		)
		defer timer.ObserveDuration()

		// 模拟处理
		time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)

		// 记录响应大小
		data := fmt.Sprintf("Response data: %d", rand.Intn(10000))
		responseSize.WithLabelValues(r.Method, "/api/data").Observe(float64(len(data)))

		w.Write([]byte(data))
	})

	// 模拟数据库查询
	go func() {
		for {
			start := time.Now()
			// 模拟查询
			time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond)
			duration := time.Since(start).Seconds()

			dbQueryDuration.WithLabelValues("SELECT", "users").Observe(duration)
			time.Sleep(100 * time.Millisecond)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("服务启动在 :8080")
	http.ListenAndServe(":8080", nil)
}

// Histogram 暴露的指标：
// myapp_http_request_duration_seconds_bucket{...,le="0.1"}   -- 小于 0.1s 的请求数
// myapp_http_request_duration_seconds_bucket{...,le="0.5"}   -- 小于 0.5s 的请求数
// myapp_http_request_duration_seconds_bucket{...,le="+Inf"}  -- 所有请求数
// myapp_http_request_duration_seconds_sum{...}                -- 延迟总和
// myapp_http_request_duration_seconds_count{...}              -- 请求总数
//
// 常用 PromQL：
// 平均延迟：rate(myapp_http_request_duration_seconds_sum[5m])
//           / rate(myapp_http_request_duration_seconds_count[5m])
// P99 延迟：histogram_quantile(0.99, rate(myapp_http_request_duration_seconds_bucket[5m]))
// P95 延迟：histogram_quantile(0.95, rate(myapp_http_request_duration_seconds_bucket[5m]))
```

### 2.4 Summary（摘要）

Summary 类似 Histogram，但直接在客户端计算分位数。适合不需要聚合的场景。

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// 请求延迟 Summary
	requestLatency = prometheus.NewSummaryVec(
		prometheus.SummaryOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "request_latency_seconds",
			Help:      "HTTP 请求延迟摘要",
			// 定义要计算的分位数和精度
			Objectives: map[float64]float64{
				0.5:  0.05,  // P50，误差 5%
				0.9:  0.01,  // P90，误差 1%
				0.95: 0.005, // P95，误差 0.5%
				0.99: 0.001, // P99，误差 0.1%
			},
			// 滑动窗口参数
			MaxAge:     10 * time.Minute, // 数据最大年龄
			AgeBuckets: 5,                // 滑动窗口数量
			BufCap:     500,              // 缓冲区大小
		},
		[]string{"method", "path"},
	)

	// GC 暂停时间 Summary
	gcPauseDuration = prometheus.NewSummary(prometheus.SummaryOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "gc_pause_seconds",
		Help:      "GC 暂停时间摘要",
		Objectives: map[float64]float64{
			0.5:  0.05,
			0.99: 0.001,
		},
	})
)

func init() {
	prometheus.MustRegister(requestLatency)
	prometheus.MustRegister(gcPauseDuration)
}

func main() {
	http.HandleFunc("/api/search", func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// 模拟处理
		time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)

		duration := time.Since(start).Seconds()
		requestLatency.WithLabelValues(r.Method, "/api/search").Observe(duration)

		fmt.Fprintln(w, "Search results")
	})

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("服务启动在 :8080")
	http.ListenAndServe(":8080", nil)
}

// Summary 暴露的指标：
// myapp_http_request_latency_seconds{method="GET",path="/api/search",quantile="0.5"}
// myapp_http_request_latency_seconds{method="GET",path="/api/search",quantile="0.9"}
// myapp_http_request_latency_seconds{method="GET",path="/api/search",quantile="0.99"}
// myapp_http_request_latency_seconds_sum{...}
// myapp_http_request_latency_seconds_count{...}
//
// Histogram vs Summary：
// Histogram：分桶统计，服务端计算分位数，可以跨实例聚合
// Summary：客户端直接计算分位数，精度更高但不能跨实例聚合
// 推荐：大多数场景使用 Histogram，因为可以聚合
```

---

## 三、Histogram 与 Summary 对比

| 特性 | Histogram | Summary |
|------|-----------|---------|
| 分位数计算 | 服务端（PromQL） | 客户端 |
| 跨实例聚合 | 支持 | 不支持 |
| 精度 | 取决于桶边界 | 可配置 |
| 性能开销 | 较低 | 较高 |
| 适用场景 | 需要聚合、SLO 监控 | 单实例、高精度需求 |
| **推荐** | **大多数场景** | 特殊场景 |

---

## 四、注册指标与暴露端点

### 4.1 默认注册与自定义注册

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	// ==================== 方式一：使用默认注册器 ====================
	// 默认注册器（prometheus.DefaultRegisterer）自动包含：
	// - Go 运行时指标（go_*）
	// - 进程指标（process_*）
	// - promhttp 指标

	myCounter := prometheus.NewCounter(prometheus.CounterOpts{
		Name: "my_counter_total",
		Help: "我的计数器",
	})
	prometheus.MustRegister(myCounter) // 注册到默认注册器

	// 暴露端点（包含所有默认指标 + 自定义指标）
	http.Handle("/metrics", promhttp.Handler())

	// ==================== 方式二：自定义注册器 ====================
	// 适合只暴露自定义指标，不包含 Go 运行时指标
	customRegistry := prometheus.NewRegistry()

	// 手动添加想要的默认指标
	customRegistry.MustRegister(collectors.NewGoCollector())       // Go 运行时指标
	customRegistry.MustRegister(collectors.NewProcessCollector(    // 进程指标
		collectors.ProcessCollectorOpts{},
	))

	// 注册自定义指标
	customCounter := prometheus.NewCounter(prometheus.CounterOpts{
		Name: "custom_counter_total",
		Help: "自定义计数器",
	})
	customRegistry.MustRegister(customCounter)

	// 使用自定义注册器创建 Handler
	customHandler := promhttp.HandlerFor(customRegistry, promhttp.HandlerOpts{
		EnableOpenMetrics: true,  // 启用 OpenMetrics 格式
		ErrorLog:          nil,   // 自定义错误日志
	})

	http.Handle("/custom-metrics", customHandler)

	// ==================== 方式三：只暴露自定义指标 ====================
	pureRegistry := prometheus.NewRegistry()

	pureCounter := prometheus.NewCounter(prometheus.CounterOpts{
		Name: "pure_counter_total",
		Help: "纯净的计数器",
	})
	pureRegistry.MustRegister(pureCounter)

	pureHandler := promhttp.HandlerFor(pureRegistry, promhttp.HandlerOpts{})
	http.Handle("/pure-metrics", pureHandler)

	fmt.Println("指标端点:")
	fmt.Println("  /metrics        -- 默认（含 Go 运行时指标）")
	fmt.Println("  /custom-metrics -- 自定义注册器")
	fmt.Println("  /pure-metrics   -- 只有自定义指标")
	http.ListenAndServe(":8080", nil)
}
```

### 4.2 Go 运行时指标

默认注册器自动暴露的运行时指标：

```
# Go 运行时指标（go_*）
go_goroutines                    -- 当前 goroutine 数量
go_threads                       -- 操作系统线程数
go_gc_duration_seconds           -- GC 暂停时间
go_memstats_alloc_bytes          -- 已分配内存（字节）
go_memstats_alloc_bytes_total    -- 累计分配内存
go_memstats_sys_bytes            -- 从系统获取的内存
go_memstats_heap_alloc_bytes     -- 堆内存分配
go_memstats_heap_sys_bytes       -- 堆内存系统获取
go_memstats_heap_idle_bytes      -- 堆空闲内存
go_memstats_heap_inuse_bytes     -- 堆使用中内存
go_memstats_stack_inuse_bytes    -- 栈使用中内存
go_memstats_gc_sys_bytes         -- GC 元数据使用内存
go_memstats_next_gc_bytes        -- 下次 GC 触发的堆大小
go_info                          -- Go 版本信息

# 进程指标（process_*）
process_cpu_seconds_total        -- 进程 CPU 使用时间
process_open_fds                 -- 打开的文件描述符数
process_max_fds                  -- 最大文件描述符限制
process_resident_memory_bytes    -- 常驻内存大小
process_virtual_memory_bytes     -- 虚拟内存大小
process_start_time_seconds       -- 进程启动时间戳
```

---

## 五、HTTP 中间件集成

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// ============================================================
// Prometheus HTTP 中间件
// ============================================================

// HTTPMetrics HTTP 指标集合
type HTTPMetrics struct {
	requestsTotal    *prometheus.CounterVec
	requestDuration  *prometheus.HistogramVec
	requestsInFlight prometheus.Gauge
	responseSize     *prometheus.HistogramVec
}

// NewHTTPMetrics 创建 HTTP 指标
func NewHTTPMetrics(namespace string) *HTTPMetrics {
	m := &HTTPMetrics{
		requestsTotal: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "requests_total",
				Help:      "HTTP 请求总数",
			},
			[]string{"method", "path", "status_code"},
		),
		requestDuration: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "request_duration_seconds",
				Help:      "HTTP 请求延迟",
				Buckets:   []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
			},
			[]string{"method", "path", "status_code"},
		),
		requestsInFlight: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "http",
			Name:      "requests_in_flight",
			Help:      "当前正在处理的 HTTP 请求数",
		}),
		responseSize: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "response_size_bytes",
				Help:      "HTTP 响应大小",
				Buckets:   prometheus.ExponentialBuckets(100, 10, 6),
			},
			[]string{"method", "path"},
		),
	}

	// 注册所有指标
	prometheus.MustRegister(m.requestsTotal)
	prometheus.MustRegister(m.requestDuration)
	prometheus.MustRegister(m.requestsInFlight)
	prometheus.MustRegister(m.responseSize)

	return m
}

// responseWriter 包装 ResponseWriter 以捕获状态码和大小
type responseWriter struct {
	http.ResponseWriter
	statusCode int
	size       int
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.size += n
	return n, err
}

// Middleware 返回 HTTP 中间件
func (m *HTTPMetrics) Middleware(path string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 记录正在处理的请求数
		m.requestsInFlight.Inc()
		defer m.requestsInFlight.Dec()

		// 记录开始时间
		start := time.Now()

		// 包装 ResponseWriter
		rw := newResponseWriter(w)

		// 执行处理器
		next(rw, r)

		// 记录指标
		duration := time.Since(start).Seconds()
		statusCode := strconv.Itoa(rw.statusCode)

		m.requestsTotal.WithLabelValues(r.Method, path, statusCode).Inc()
		m.requestDuration.WithLabelValues(r.Method, path, statusCode).Observe(duration)
		m.responseSize.WithLabelValues(r.Method, path).Observe(float64(rw.size))
	}
}

// ============================================================
// 使用示例
// ============================================================

func main() {
	metrics := NewHTTPMetrics("myapp")

	// 业务处理器
	usersHandler := func(w http.ResponseWriter, r *http.Request) {
		// 模拟处理
		time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
		fmt.Fprintln(w, `{"users": [{"name": "alice"}, {"name": "bob"}]}`)
	}

	ordersHandler := func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
		if rand.Float64() < 0.05 {
			http.Error(w, "Internal Server Error", 500)
			return
		}
		fmt.Fprintln(w, `{"orders": []}`)
	}

	// 使用中间件
	http.HandleFunc("/api/users", metrics.Middleware("/api/users", usersHandler))
	http.HandleFunc("/api/orders", metrics.Middleware("/api/orders", ordersHandler))

	// 指标端点
	http.Handle("/metrics", promhttp.Handler())

	fmt.Println("服务启动在 :8080")
	http.ListenAndServe(":8080", nil)
}
```

---

## 六、自定义 Collector 接口

当你需要在每次 `/metrics` 被拉取时动态生成指标时，需要实现 `Collector` 接口。

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"sync"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// ============================================================
// 自定义 Collector：数据库连接池指标
// ============================================================

// DBPoolCollector 数据库连接池指标收集器
type DBPoolCollector struct {
	// 指标描述符（Desc）
	activeConnections *prometheus.Desc
	idleConnections   *prometheus.Desc
	maxConnections    *prometheus.Desc
	waitCount         *prometheus.Desc
	waitDuration      *prometheus.Desc

	// 数据源（你的连接池或其他数据源）
	pool *FakeDBPool
}

// FakeDBPool 模拟数据库连接池（实际项目中替换为真实连接池）
type FakeDBPool struct {
	mu sync.RWMutex
}

func (p *FakeDBPool) Stats() DBPoolStats {
	return DBPoolStats{
		ActiveConnections: rand.Intn(50),
		IdleConnections:   rand.Intn(20),
		MaxConnections:    100,
		WaitCount:         int64(rand.Intn(1000)),
		WaitDuration:      time.Duration(rand.Intn(5000)) * time.Millisecond,
	}
}

type DBPoolStats struct {
	ActiveConnections int
	IdleConnections   int
	MaxConnections    int
	WaitCount         int64
	WaitDuration      time.Duration
}

// NewDBPoolCollector 创建连接池收集器
func NewDBPoolCollector(pool *FakeDBPool, dbName string) *DBPoolCollector {
	// 公共标签
	labels := prometheus.Labels{"db": dbName}

	return &DBPoolCollector{
		activeConnections: prometheus.NewDesc(
			"myapp_db_pool_active_connections",
			"数据库连接池活跃连接数",
			nil,    // 动态标签（这里没有）
			labels, // 常量标签
		),
		idleConnections: prometheus.NewDesc(
			"myapp_db_pool_idle_connections",
			"数据库连接池空闲连接数",
			nil, labels,
		),
		maxConnections: prometheus.NewDesc(
			"myapp_db_pool_max_connections",
			"数据库连接池最大连接数",
			nil, labels,
		),
		waitCount: prometheus.NewDesc(
			"myapp_db_pool_wait_total",
			"等待获取连接的总次数",
			nil, labels,
		),
		waitDuration: prometheus.NewDesc(
			"myapp_db_pool_wait_duration_seconds_total",
			"等待获取连接的总时间（秒）",
			nil, labels,
		),
		pool: pool,
	}
}

// Describe 实现 Collector 接口：发送指标描述
func (c *DBPoolCollector) Describe(ch chan<- *prometheus.Desc) {
	ch <- c.activeConnections
	ch <- c.idleConnections
	ch <- c.maxConnections
	ch <- c.waitCount
	ch <- c.waitDuration
}

// Collect 实现 Collector 接口：收集指标值
// 每次 /metrics 被请求时调用
func (c *DBPoolCollector) Collect(ch chan<- prometheus.Metric) {
	// 获取最新的连接池统计信息
	stats := c.pool.Stats()

	// 发送指标值
	ch <- prometheus.MustNewConstMetric(
		c.activeConnections,
		prometheus.GaugeValue,
		float64(stats.ActiveConnections),
	)
	ch <- prometheus.MustNewConstMetric(
		c.idleConnections,
		prometheus.GaugeValue,
		float64(stats.IdleConnections),
	)
	ch <- prometheus.MustNewConstMetric(
		c.maxConnections,
		prometheus.GaugeValue,
		float64(stats.MaxConnections),
	)
	ch <- prometheus.MustNewConstMetric(
		c.waitCount,
		prometheus.CounterValue,
		float64(stats.WaitCount),
	)
	ch <- prometheus.MustNewConstMetric(
		c.waitDuration,
		prometheus.CounterValue,
		stats.WaitDuration.Seconds(),
	)
}

func main() {
	// 创建连接池
	pool := &FakeDBPool{}

	// 创建并注册 Collector
	collector := NewDBPoolCollector(pool, "primary")
	prometheus.MustRegister(collector)

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("数据库连接池 Exporter 启动在 :8080")
	http.ListenAndServe(":8080", nil)
}
```

---

## 七、完整 Exporter 开发示例

```go
package main

import (
	"encoding/json"
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// ============================================================
// Redis Exporter 示例（模拟）
// ============================================================

// RedisExporter Redis 指标导出器
type RedisExporter struct {
	// 指标定义
	up               prometheus.Gauge
	info             *prometheus.GaugeVec
	connectedClients prometheus.Gauge
	blockedClients   prometheus.Gauge
	usedMemoryBytes  prometheus.Gauge
	maxMemoryBytes   prometheus.Gauge
	memoryUsageRatio prometheus.Gauge
	totalCommands    prometheus.Counter
	commandsPerSec   prometheus.Gauge
	hitRate          prometheus.Gauge
	keyspaceHits     prometheus.Counter
	keyspaceMisses   prometheus.Counter
	evictedKeys      prometheus.Counter
	expiredKeys      prometheus.Counter
	dbKeys           *prometheus.GaugeVec
	dbExpires        *prometheus.GaugeVec
	commandDuration  *prometheus.HistogramVec
	scrapeErrors     prometheus.Counter
	scrapeDuration   prometheus.Histogram
	lastScrapeTime   prometheus.Gauge

	// Redis 连接信息
	redisAddr string
}

// NewRedisExporter 创建 Redis Exporter
func NewRedisExporter(redisAddr string) *RedisExporter {
	namespace := "redis"

	e := &RedisExporter{
		redisAddr: redisAddr,

		up: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "up",
			Help:      "Redis 是否存活（1=存活，0=不可达）",
		}),
		info: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "info",
			Help:      "Redis 实例信息",
		}, []string{"version", "role", "os"}),
		connectedClients: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "connected_clients",
			Help:      "已连接的客户端数",
		}),
		blockedClients: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "blocked_clients",
			Help:      "被阻塞的客户端数（BLPOP 等）",
		}),
		usedMemoryBytes: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "used_memory_bytes",
			Help:      "Redis 已使用内存（字节）",
		}),
		maxMemoryBytes: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "max_memory_bytes",
			Help:      "Redis 最大内存限制（字节）",
		}),
		memoryUsageRatio: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "memory_usage_ratio",
			Help:      "内存使用率",
		}),
		totalCommands: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Name:      "commands_processed_total",
			Help:      "处理的命令总数",
		}),
		commandsPerSec: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "instantaneous_ops_per_second",
			Help:      "每秒执行的命令数",
		}),
		hitRate: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "keyspace_hit_rate",
			Help:      "缓存命中率",
		}),
		keyspaceHits: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Name:      "keyspace_hits_total",
			Help:      "键空间命中总数",
		}),
		keyspaceMisses: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Name:      "keyspace_misses_total",
			Help:      "键空间未命中总数",
		}),
		evictedKeys: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Name:      "evicted_keys_total",
			Help:      "因内存不足被驱逐的键总数",
		}),
		expiredKeys: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Name:      "expired_keys_total",
			Help:      "已过期的键总数",
		}),
		dbKeys: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "db_keys",
			Help:      "每个数据库的键数量",
		}, []string{"db"}),
		dbExpires: prometheus.NewGaugeVec(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "db_expires",
			Help:      "每个数据库中设置了过期时间的键数量",
		}, []string{"db"}),
		commandDuration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
			Namespace: namespace,
			Name:      "command_duration_seconds",
			Help:      "命令执行延迟分布",
			Buckets:   []float64{.0001, .0005, .001, .005, .01, .05, .1, .5, 1},
		}, []string{"command"}),
		scrapeErrors: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Subsystem: "exporter",
			Name:      "scrape_errors_total",
			Help:      "指标采集错误总数",
		}),
		scrapeDuration: prometheus.NewHistogram(prometheus.HistogramOpts{
			Namespace: namespace,
			Subsystem: "exporter",
			Name:      "scrape_duration_seconds",
			Help:      "指标采集耗时",
			Buckets:   []float64{.001, .005, .01, .05, .1, .5, 1},
		}),
		lastScrapeTime: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "exporter",
			Name:      "last_scrape_timestamp",
			Help:      "最后一次采集时间戳",
		}),
	}

	return e
}

// Register 注册所有指标
func (e *RedisExporter) Register(reg prometheus.Registerer) {
	reg.MustRegister(
		e.up,
		e.info,
		e.connectedClients,
		e.blockedClients,
		e.usedMemoryBytes,
		e.maxMemoryBytes,
		e.memoryUsageRatio,
		e.totalCommands,
		e.commandsPerSec,
		e.hitRate,
		e.keyspaceHits,
		e.keyspaceMisses,
		e.evictedKeys,
		e.expiredKeys,
		e.dbKeys,
		e.dbExpires,
		e.commandDuration,
		e.scrapeErrors,
		e.scrapeDuration,
		e.lastScrapeTime,
	)
}

// Scrape 采集指标（模拟从 Redis 获取数据）
func (e *RedisExporter) Scrape() {
	start := time.Now()
	defer func() {
		e.scrapeDuration.Observe(time.Since(start).Seconds())
		e.lastScrapeTime.SetToCurrentTime()
	}()

	// 模拟 Redis INFO 数据
	info := e.getRedisInfo()
	if info == nil {
		e.up.Set(0)
		e.scrapeErrors.Inc()
		return
	}

	e.up.Set(1)
	e.info.WithLabelValues("7.2.4", "master", "Linux").Set(1)

	// 更新指标
	e.connectedClients.Set(float64(info.ConnectedClients))
	e.blockedClients.Set(float64(info.BlockedClients))
	e.usedMemoryBytes.Set(float64(info.UsedMemory))
	e.maxMemoryBytes.Set(float64(info.MaxMemory))

	if info.MaxMemory > 0 {
		e.memoryUsageRatio.Set(float64(info.UsedMemory) / float64(info.MaxMemory))
	}

	e.commandsPerSec.Set(float64(info.InstantaneousOps))

	// 命中率
	totalAccess := info.KeyspaceHits + info.KeyspaceMisses
	if totalAccess > 0 {
		e.hitRate.Set(float64(info.KeyspaceHits) / float64(totalAccess))
	}

	// DB 键数量
	for db, keys := range info.DBKeys {
		e.dbKeys.WithLabelValues(db).Set(float64(keys))
	}
	for db, expires := range info.DBExpires {
		e.dbExpires.WithLabelValues(db).Set(float64(expires))
	}

	// 模拟命令延迟
	commands := []string{"GET", "SET", "DEL", "HGET", "HSET", "LPUSH", "RPOP"}
	for _, cmd := range commands {
		e.commandDuration.WithLabelValues(cmd).Observe(
			float64(rand.Intn(10)) * 0.001,
		)
	}
}

// RedisInfo 模拟 Redis INFO 数据
type RedisInfo struct {
	ConnectedClients int
	BlockedClients   int
	UsedMemory       int64
	MaxMemory        int64
	InstantaneousOps int
	KeyspaceHits     int64
	KeyspaceMisses   int64
	DBKeys           map[string]int
	DBExpires        map[string]int
}

// getRedisInfo 模拟获取 Redis 信息
func (e *RedisExporter) getRedisInfo() *RedisInfo {
	// 实际项目中，这里应该连接 Redis 并执行 INFO 命令
	return &RedisInfo{
		ConnectedClients: 50 + rand.Intn(30),
		BlockedClients:   rand.Intn(5),
		UsedMemory:       int64(500+rand.Intn(200)) * 1024 * 1024, // 500-700 MB
		MaxMemory:        1024 * 1024 * 1024,                       // 1 GB
		InstantaneousOps: 5000 + rand.Intn(3000),
		KeyspaceHits:     int64(rand.Intn(100000)),
		KeyspaceMisses:   int64(rand.Intn(10000)),
		DBKeys: map[string]int{
			"db0": 10000 + rand.Intn(5000),
			"db1": 500 + rand.Intn(200),
		},
		DBExpires: map[string]int{
			"db0": 3000 + rand.Intn(1000),
			"db1": 100 + rand.Intn(50),
		},
	}
}

func main() {
	exporter := NewRedisExporter("localhost:6379")
	exporter.Register(prometheus.DefaultRegisterer)

	// 定期采集
	go func() {
		ticker := time.NewTicker(15 * time.Second)
		defer ticker.Stop()

		exporter.Scrape() // 启动时立即采集一次
		for range ticker.C {
			exporter.Scrape()
		}
	}()

	http.Handle("/metrics", promhttp.Handler())

	// 健康检查端点
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})

	fmt.Println("Redis Exporter 启动在 :9121")
	fmt.Println("指标端点: http://localhost:9121/metrics")
	http.ListenAndServe(":9121", nil)
}
```

---

## 八、SRE 实战：自定义应用 Exporter

这是一个完整的生产级应用 Exporter，监控请求数、延迟、错误率、goroutine 数等关键指标。

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"runtime"
	"strconv"
	"sync"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// ============================================================
// 应用指标定义
// ============================================================

// AppMetrics 应用指标集合
type AppMetrics struct {
	// HTTP 指标
	httpRequestsTotal    *prometheus.CounterVec
	httpRequestDuration  *prometheus.HistogramVec
	httpRequestsInFlight prometheus.Gauge
	httpResponseSize     *prometheus.HistogramVec
	httpErrorsTotal      *prometheus.CounterVec

	// 业务指标
	businessOperations    *prometheus.CounterVec
	businessLatency       *prometheus.HistogramVec
	activeUsers           prometheus.Gauge
	orderAmount           *prometheus.CounterVec
	cacheHitRate          prometheus.Gauge

	// 依赖健康指标
	dependencyUp         *prometheus.GaugeVec
	dependencyLatency    *prometheus.HistogramVec

	// 运行时指标（自定义补充）
	goroutineCount       prometheus.Gauge
	goroutinesByState    *prometheus.GaugeVec
	heapAllocBytes       prometheus.Gauge
	gcPauseSeconds       prometheus.Histogram
	buildInfo            *prometheus.GaugeVec
	startTime            prometheus.Gauge

	// Exporter 自身指标
	scrapeErrors         prometheus.Counter
	scrapeDuration       prometheus.Histogram
}

// NewAppMetrics 创建应用指标
func NewAppMetrics(namespace string) *AppMetrics {
	m := &AppMetrics{
		// ========== HTTP 指标 ==========
		httpRequestsTotal: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "requests_total",
				Help:      "HTTP 请求总数，按方法、路径、状态码分类",
			},
			[]string{"method", "path", "status_code"},
		),
		httpRequestDuration: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "request_duration_seconds",
				Help:      "HTTP 请求延迟分布（秒）",
				Buckets:   []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
			},
			[]string{"method", "path", "status_code"},
		),
		httpRequestsInFlight: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "http",
			Name:      "requests_in_flight",
			Help:      "当前正在处理的请求数",
		}),
		httpResponseSize: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "response_size_bytes",
				Help:      "HTTP 响应大小分布（字节）",
				Buckets:   prometheus.ExponentialBuckets(100, 10, 6),
			},
			[]string{"method", "path"},
		),
		httpErrorsTotal: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: "http",
				Name:      "errors_total",
				Help:      "HTTP 错误总数，按错误类型分类",
			},
			[]string{"method", "path", "error_type"},
		),

		// ========== 业务指标 ==========
		businessOperations: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: "business",
				Name:      "operations_total",
				Help:      "业务操作总数",
			},
			[]string{"operation", "status"},
		),
		businessLatency: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "business",
				Name:      "operation_duration_seconds",
				Help:      "业务操作耗时",
				Buckets:   []float64{.01, .05, .1, .5, 1, 5, 10, 30},
			},
			[]string{"operation"},
		),
		activeUsers: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "business",
			Name:      "active_users",
			Help:      "当前活跃用户数",
		}),
		orderAmount: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: "business",
				Name:      "order_amount_total",
				Help:      "订单金额总计（元）",
			},
			[]string{"payment_method"},
		),
		cacheHitRate: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "cache",
			Name:      "hit_rate",
			Help:      "缓存命中率",
		}),

		// ========== 依赖健康指标 ==========
		dependencyUp: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Subsystem: "dependency",
				Name:      "up",
				Help:      "依赖服务可用性（1=可用，0=不可用）",
			},
			[]string{"name", "type"},
		),
		dependencyLatency: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: "dependency",
				Name:      "request_duration_seconds",
				Help:      "依赖服务请求延迟",
				Buckets:   []float64{.001, .005, .01, .05, .1, .5, 1, 5},
			},
			[]string{"name", "operation"},
		),

		// ========== 运行时指标 ==========
		goroutineCount: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "runtime",
			Name:      "goroutine_count",
			Help:      "当前 goroutine 数量",
		}),
		goroutinesByState: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Subsystem: "runtime",
				Name:      "goroutines_by_function",
				Help:      "按主要功能分类的 goroutine 数量",
			},
			[]string{"function"},
		),
		heapAllocBytes: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Subsystem: "runtime",
			Name:      "heap_alloc_bytes",
			Help:      "堆内存分配字节数",
		}),
		gcPauseSeconds: prometheus.NewHistogram(prometheus.HistogramOpts{
			Namespace: namespace,
			Subsystem: "runtime",
			Name:      "gc_pause_seconds",
			Help:      "GC 暂停时间分布",
			Buckets:   []float64{.00001, .00005, .0001, .0005, .001, .005, .01, .05},
		}),
		buildInfo: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Name:      "build_info",
				Help:      "构建信息",
			},
			[]string{"version", "go_version", "commit"},
		),
		startTime: prometheus.NewGauge(prometheus.GaugeOpts{
			Namespace: namespace,
			Name:      "start_time_seconds",
			Help:      "应用启动时间（Unix 时间戳）",
		}),

		// ========== Exporter 自身指标 ==========
		scrapeErrors: prometheus.NewCounter(prometheus.CounterOpts{
			Namespace: namespace,
			Subsystem: "exporter",
			Name:      "scrape_errors_total",
			Help:      "指标采集错误总数",
		}),
		scrapeDuration: prometheus.NewHistogram(prometheus.HistogramOpts{
			Namespace: namespace,
			Subsystem: "exporter",
			Name:      "scrape_duration_seconds",
			Help:      "指标采集耗时",
			Buckets:   []float64{.0001, .001, .01, .1},
		}),
	}

	return m
}

// Register 注册所有指标
func (m *AppMetrics) Register(reg prometheus.Registerer) {
	reg.MustRegister(
		m.httpRequestsTotal,
		m.httpRequestDuration,
		m.httpRequestsInFlight,
		m.httpResponseSize,
		m.httpErrorsTotal,
		m.businessOperations,
		m.businessLatency,
		m.activeUsers,
		m.orderAmount,
		m.cacheHitRate,
		m.dependencyUp,
		m.dependencyLatency,
		m.goroutineCount,
		m.goroutinesByState,
		m.heapAllocBytes,
		m.gcPauseSeconds,
		m.buildInfo,
		m.startTime,
		m.scrapeErrors,
		m.scrapeDuration,
	)
}

// ============================================================
// HTTP 中间件
// ============================================================

// metricsResponseWriter 包装 http.ResponseWriter
type metricsResponseWriter struct {
	http.ResponseWriter
	statusCode int
	size       int
	written    bool
}

func newMetricsResponseWriter(w http.ResponseWriter) *metricsResponseWriter {
	return &metricsResponseWriter{ResponseWriter: w, statusCode: 200}
}

func (w *metricsResponseWriter) WriteHeader(code int) {
	if !w.written {
		w.statusCode = code
		w.written = true
		w.ResponseWriter.WriteHeader(code)
	}
}

func (w *metricsResponseWriter) Write(b []byte) (int, error) {
	if !w.written {
		w.written = true
	}
	n, err := w.ResponseWriter.Write(b)
	w.size += n
	return n, err
}

// InstrumentHandler 为 HTTP Handler 添加指标
func (m *AppMetrics) InstrumentHandler(path string, handler http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		m.httpRequestsInFlight.Inc()
		defer m.httpRequestsInFlight.Dec()

		start := time.Now()
		mrw := newMetricsResponseWriter(w)

		handler(mrw, r)

		duration := time.Since(start).Seconds()
		status := strconv.Itoa(mrw.statusCode)

		m.httpRequestsTotal.WithLabelValues(r.Method, path, status).Inc()
		m.httpRequestDuration.WithLabelValues(r.Method, path, status).Observe(duration)
		m.httpResponseSize.WithLabelValues(r.Method, path).Observe(float64(mrw.size))

		if mrw.statusCode >= 400 {
			errorType := "client_error"
			if mrw.statusCode >= 500 {
				errorType = "server_error"
			}
			m.httpErrorsTotal.WithLabelValues(r.Method, path, errorType).Inc()
		}
	}
}

// ============================================================
// 运行时指标收集
// ============================================================

// CollectRuntimeMetrics 定期收集运行时指标
func (m *AppMetrics) CollectRuntimeMetrics(ctx context.Context, interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			start := time.Now()

			// goroutine 数
			m.goroutineCount.Set(float64(runtime.NumGoroutine()))

			// 内存统计
			var memStats runtime.MemStats
			runtime.ReadMemStats(&memStats)
			m.heapAllocBytes.Set(float64(memStats.HeapAlloc))

			// GC 暂停时间（最近一次）
			if memStats.NumGC > 0 {
				lastPause := memStats.PauseNs[(memStats.NumGC+255)%256]
				m.gcPauseSeconds.Observe(float64(lastPause) / 1e9)
			}

			m.scrapeDuration.Observe(time.Since(start).Seconds())

		case <-ctx.Done():
			return
		}
	}
}

// ============================================================
// 依赖健康检查
// ============================================================

// Dependency 依赖定义
type Dependency struct {
	Name     string
	Type     string // "database", "cache", "api"
	CheckFn  func() error
}

// CheckDependencies 检查所有依赖健康状态
func (m *AppMetrics) CheckDependencies(ctx context.Context, deps []Dependency, interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	check := func() {
		for _, dep := range deps {
			start := time.Now()
			err := dep.CheckFn()
			duration := time.Since(start).Seconds()

			m.dependencyLatency.WithLabelValues(dep.Name, "health_check").Observe(duration)

			if err != nil {
				m.dependencyUp.WithLabelValues(dep.Name, dep.Type).Set(0)
				log.Printf("依赖 %s (%s) 不健康: %v", dep.Name, dep.Type, err)
			} else {
				m.dependencyUp.WithLabelValues(dep.Name, dep.Type).Set(1)
			}
		}
	}

	check() // 启动时立即检查
	for {
		select {
		case <-ticker.C:
			check()
		case <-ctx.Done():
			return
		}
	}
}

// ============================================================
// 应用服务
// ============================================================

// App 应用
type App struct {
	metrics *AppMetrics
	mux     *http.ServeMux
}

// NewApp 创建应用
func NewApp() *App {
	metrics := NewAppMetrics("myapp")

	// 使用自定义注册器
	registry := prometheus.NewRegistry()
	registry.MustRegister(collectors.NewGoCollector())
	registry.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
	metrics.Register(registry)

	// 设置构建信息和启动时间
	metrics.buildInfo.WithLabelValues("1.0.0", runtime.Version(), "abc123").Set(1)
	metrics.startTime.Set(float64(time.Now().Unix()))

	app := &App{
		metrics: metrics,
		mux:     http.NewServeMux(),
	}

	// 注册路由
	app.mux.HandleFunc("/api/users", metrics.InstrumentHandler("/api/users", app.handleUsers))
	app.mux.HandleFunc("/api/orders", metrics.InstrumentHandler("/api/orders", app.handleOrders))
	app.mux.HandleFunc("/api/products", metrics.InstrumentHandler("/api/products", app.handleProducts))
	app.mux.HandleFunc("/health", app.handleHealth)

	// 指标端点
	app.mux.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{
		EnableOpenMetrics: true,
	}))

	return app
}

// handleUsers 用户接口
func (app *App) handleUsers(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	defer func() {
		app.metrics.businessLatency.WithLabelValues("get_users").Observe(time.Since(start).Seconds())
	}()

	time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)

	if rand.Float64() < 0.02 { // 2% 错误率
		app.metrics.businessOperations.WithLabelValues("get_users", "error").Inc()
		http.Error(w, "Internal Server Error", 500)
		return
	}

	app.metrics.businessOperations.WithLabelValues("get_users", "success").Inc()
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"users": []map[string]string{
			{"name": "Alice", "email": "alice@example.com"},
			{"name": "Bob", "email": "bob@example.com"},
		},
	})
}

// handleOrders 订单接口
func (app *App) handleOrders(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	defer func() {
		app.metrics.businessLatency.WithLabelValues("get_orders").Observe(time.Since(start).Seconds())
	}()

	time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)

	// 模拟订单
	amount := float64(rand.Intn(10000)) / 100
	methods := []string{"alipay", "wechat", "credit_card"}
	method := methods[rand.Intn(len(methods))]

	app.metrics.orderAmount.WithLabelValues(method).Add(amount)
	app.metrics.businessOperations.WithLabelValues("create_order", "success").Inc()

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"order_id": fmt.Sprintf("ORD-%d", time.Now().UnixNano()),
		"amount":   amount,
		"method":   method,
	})
}

// handleProducts 商品接口
func (app *App) handleProducts(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	defer func() {
		app.metrics.businessLatency.WithLabelValues("get_products").Observe(time.Since(start).Seconds())
	}()

	// 模拟缓存
	if rand.Float64() < 0.8 { // 80% 命中率
		app.metrics.cacheHitRate.Set(0.8)
		time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond) // 缓存命中，快速返回
	} else {
		time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond) // 缓存未命中
	}

	app.metrics.businessOperations.WithLabelValues("get_products", "success").Inc()
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintln(w, `{"products": []}`)
}

// handleHealth 健康检查
func (app *App) handleHealth(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status":     "ok",
		"goroutines": runtime.NumGoroutine(),
		"timestamp":  time.Now().Format(time.RFC3339),
	})
}

// Run 启动应用
func (app *App) Run(ctx context.Context, addr string) error {
	// 启动运行时指标收集
	go app.metrics.CollectRuntimeMetrics(ctx, 10*time.Second)

	// 启动依赖健康检查
	deps := []Dependency{
		{
			Name: "mysql",
			Type: "database",
			CheckFn: func() error {
				// 实际项目中这里 ping 数据库
				time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond)
				return nil
			},
		},
		{
			Name: "redis",
			Type: "cache",
			CheckFn: func() error {
				time.Sleep(time.Duration(rand.Intn(5)) * time.Millisecond)
				return nil
			},
		},
		{
			Name: "user-service",
			Type: "api",
			CheckFn: func() error {
				time.Sleep(time.Duration(rand.Intn(20)) * time.Millisecond)
				if rand.Float64() < 0.05 { // 5% 概率失败
					return fmt.Errorf("connection refused")
				}
				return nil
			},
		},
	}
	go app.metrics.CheckDependencies(ctx, deps, 30*time.Second)

	// 模拟活跃用户数变化
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			default:
				app.metrics.activeUsers.Set(float64(100 + rand.Intn(200)))
				time.Sleep(10 * time.Second)
			}
		}
	}()

	server := &http.Server{
		Addr:    addr,
		Handler: app.mux,
	}

	// 优雅关闭
	go func() {
		<-ctx.Done()
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		server.Shutdown(shutdownCtx)
	}()

	log.Printf("应用启动在 %s", addr)
	log.Printf("指标端点: http://localhost%s/metrics", addr)
	log.Printf("健康检查: http://localhost%s/health", addr)
	return server.ListenAndServe()
}

// ============================================================
// 主函数
// ============================================================

func main() {
	app := NewApp()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 优雅退出
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		log.Println("收到退出信号...")
		cancel()
	}()

	addr := ":8080"
	if port := os.Getenv("PORT"); port != "" {
		addr = ":" + port
	}

	if err := app.Run(ctx, addr); err != nil && err != http.ErrServerClosed {
		log.Fatalf("服务启动失败: %v", err)
	}
}
```

### Prometheus 采集配置

```yaml
# prometheus.yml 配置片段
scrape_configs:
  - job_name: 'myapp'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
    # 或使用 Kubernetes 服务发现
    # kubernetes_sd_configs:
    #   - role: pod
    #     namespaces:
    #       names: ['default']
```

### 常用 PromQL 查询

```promql
# 请求速率（QPS）
rate(myapp_http_requests_total[5m])

# 按路径分类的 QPS
sum by (path) (rate(myapp_http_requests_total[5m]))

# 错误率
sum(rate(myapp_http_errors_total[5m])) / sum(rate(myapp_http_requests_total[5m]))

# P99 延迟
histogram_quantile(0.99, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m])))

# P95 延迟（按路径）
histogram_quantile(0.95, sum by (le, path) (rate(myapp_http_request_duration_seconds_bucket[5m])))

# 平均延迟
rate(myapp_http_request_duration_seconds_sum[5m]) / rate(myapp_http_request_duration_seconds_count[5m])

# 当前正在处理的请求
myapp_http_requests_in_flight

# goroutine 数量
myapp_runtime_goroutine_count

# 堆内存使用
myapp_runtime_heap_alloc_bytes

# 依赖服务可用性
myapp_dependency_up

# 缓存命中率
myapp_cache_hit_rate

# 活跃用户数
myapp_business_active_users

# SLO: 99.9% 请求在 500ms 内完成
histogram_quantile(0.999, sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m]))) < 0.5
```

---

## 常见坑点

### 1. Counter 和 Gauge 混淆

```go
// 坑：用 Gauge 记录请求总数
requestTotal := prometheus.NewGauge(...)
requestTotal.Inc() // 重启后归零！rate() 计算会出错

// 正确：累积型数据用 Counter
requestTotal := prometheus.NewCounter(...)
requestTotal.Inc() // Counter 只增不减，Prometheus rate() 能正确计算
```

### 2. 标签基数爆炸

```go
// 坑：标签值不可控，导致时间序列爆炸
requestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "requests_total"},
    []string{"user_id"},  // 每个用户一个时间序列！可能有百万级
)

// 正确：标签值必须是有限集合
requestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "requests_total"},
    []string{"method", "path", "status_code"},  // 有限的组合数
)

// 规则：标签的笛卡尔积不应超过 1000
```

### 3. 忘记注册指标

```go
// 坑：定义了但没注册
var counter = prometheus.NewCounter(prometheus.CounterOpts{
    Name: "my_counter_total",
})
// 没有调用 prometheus.MustRegister(counter)！
// /metrics 端点看不到这个指标

// 正确：在 init() 中注册
func init() {
    prometheus.MustRegister(counter)
}
```

### 4. Histogram Bucket 选择不当

```go
// 坑：默认 Bucket 不适合你的延迟分布
// DefBuckets = {.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10}
// 如果你的 API 延迟通常在 1-10ms，大部分数据都在第一个桶

// 正确：根据实际延迟分布选择
prometheus.HistogramOpts{
    Buckets: []float64{.001, .002, .005, .01, .02, .05, .1, .2, .5, 1},
}
```

### 5. 重复注册 panic

```go
// 坑：重复注册会 panic
prometheus.MustRegister(counter)
prometheus.MustRegister(counter)  // panic!

// 正确：使用 Register（不 panic），或确保只注册一次
err := prometheus.Register(counter)
if err != nil {
    // 已注册，忽略
    if _, ok := err.(prometheus.AlreadyRegisteredError); !ok {
        log.Fatal(err)
    }
}
```

### 6. WithLabelValues 参数顺序错误

```go
// 定义时标签顺序
counter := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "requests_total"},
    []string{"method", "status_code", "path"},
)

// 坑：使用时标签顺序不一致
counter.WithLabelValues("200", "GET", "/api")  // 错误！method 变成了 "200"

// 正确：严格按定义顺序
counter.WithLabelValues("GET", "200", "/api")

// 更安全的写法：使用 With(Labels)
counter.With(prometheus.Labels{
    "method":      "GET",
    "status_code": "200",
    "path":        "/api",
}).Inc()
```

---

## 速查表

### 指标类型速查

| 类型 | 用途 | 方法 | 示例 |
|------|------|------|------|
| Counter | 只增累积值 | `Inc()`, `Add(v)` | 请求总数、错误数 |
| Gauge | 可增可减当前值 | `Set(v)`, `Inc()`, `Dec()`, `Add(v)`, `Sub(v)` | 温度、在线人数、队列大小 |
| Histogram | 值分布统计 | `Observe(v)` | 请求延迟、响应大小 |
| Summary | 客户端分位数 | `Observe(v)` | 特殊场景延迟统计 |

### API 速查

| 操作 | 代码 |
|------|------|
| 新建 Counter | `prometheus.NewCounter(Opts)` |
| 新建带标签 Counter | `prometheus.NewCounterVec(Opts, labels)` |
| 注册指标 | `prometheus.MustRegister(collector)` |
| 暴露端点 | `http.Handle("/metrics", promhttp.Handler())` |
| 自定义注册器 | `reg := prometheus.NewRegistry()` |
| 自定义 Handler | `promhttp.HandlerFor(reg, opts)` |
| 计时器 | `timer := prometheus.NewTimer(histogram)` |
| 标签值 | `vec.WithLabelValues("v1", "v2").Inc()` |
| 安全标签 | `vec.With(prometheus.Labels{"k": "v"}).Inc()` |

### 命名规范速查

| 规则 | 示例 |
|------|------|
| Counter 以 `_total` 结尾 | `http_requests_total` |
| 使用基础单位 | `_bytes` (非 _kilobytes), `_seconds` (非 _milliseconds) |
| 小写字母 + 下划线 | `request_duration_seconds` |
| 前缀表示来源 | `myapp_http_requests_total` |
| 布尔值用 0/1 | `myapp_dependency_up` |

---

## 小结

本文系统讲解了 Go Prometheus 客户端库的完整用法：

1. **数据模型**：Metric Name + Labels + Timestamp + Value 构成时间序列
2. **四种指标类型**：Counter（累积计数）、Gauge（即时值）、Histogram（分布统计，推荐）、Summary（客户端分位数）
3. **注册与暴露**：默认/自定义注册器，promhttp.Handler 暴露端点
4. **HTTP 中间件**：自动记录请求数、延迟、响应大小、错误率
5. **Collector 接口**：动态采集指标（如数据库连接池）
6. **Exporter 开发**：完整的 Redis Exporter 示例
7. **SRE 实战**：生产级应用 Exporter，覆盖 HTTP 指标、业务指标、依赖健康、运行时指标

监控是 SRE 的基石。掌握 Prometheus 客户端库，你就能为任何 Go 应用和系统添加专业级别的可观测性。
