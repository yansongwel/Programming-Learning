# 实战项目：日志收集 Agent

## 导读

本篇将构建一个类似 Filebeat 的轻量级日志收集 Agent。它能监控日志文件变化、解析日志格式、过滤日志内容，并将结果输出到 Kafka 或 Elasticsearch 等目标系统。这是 SRE 基础设施中最核心的组件之一。

**项目功能：**
- 文件 Tail 监听（fsnotify + 断点续传）
- 日志解析（正则/JSON/自定义格式）
- 过滤规则（关键字/级别/正则）
- 输出到 Kafka / Elasticsearch / Stdout
- 配置文件管理（Viper + YAML）
- 优雅关闭与信号处理
- Pipeline 架构（Input → Parser → Filter → Output）

---

## 一、架构设计

```
┌──────────────────────────────────────────────────┐
│                    Log Agent                      │
│                                                   │
│  ┌─────────┐   ┌────────┐   ┌────────┐   ┌─────────┐
│  │  Input   │──→│ Parser │──→│ Filter │──→│ Output  │
│  │ (Tailer) │   │        │   │        │   │         │
│  └─────────┘   └────────┘   └────────┘   └─────────┘
│       │                                       │
│  监控文件变化      解析日志格式     过滤规则      发送到目标
│  断点续传          正则/JSON       关键字/级别     Kafka/ES
│                                               Stdout
└──────────────────────────────────────────────────┘

数据流: 文件行 → LogEvent → FilteredEvent → 序列化 → 发送
```

---

## 二、完整项目代码

### 2.1 数据模型

```go
// internal/model/event.go
// 日志事件数据模型
package model

import (
	"time"
)

// LogEvent 日志事件
type LogEvent struct {
	// 元数据
	Source    string            `json:"source"`     // 来源文件路径
	Offset   int64             `json:"offset"`      // 文件偏移量
	Line     int               `json:"line"`        // 行号

	// 时间
	Timestamp time.Time        `json:"timestamp"`   // 事件时间
	CollectedAt time.Time      `json:"collected_at"` // 收集时间

	// 内容
	RawMessage string          `json:"raw_message"` // 原始日志行
	Message    string          `json:"message"`     // 解析后的消息
	Level      string          `json:"level"`       // 日志级别
	Fields     map[string]interface{} `json:"fields,omitempty"` // 解析出的字段

	// 标签
	Tags       map[string]string `json:"tags,omitempty"` // 用户自定义标签
}

// FileState 文件状态（用于断点续传）
type FileState struct {
	Path     string    `json:"path"`
	Offset   int64     `json:"offset"`
	Inode    uint64    `json:"inode"`
	ModTime  time.Time `json:"mod_time"`
	SavedAt  time.Time `json:"saved_at"`
}
```

### 2.2 配置管理

```go
// internal/config/config.go
// 配置管理（Viper + YAML）
package config

import (
	"fmt"
	"time"
)

// Config 应用配置
type Config struct {
	Agent   AgentConfig   `yaml:"agent" mapstructure:"agent"`
	Inputs  []InputConfig `yaml:"inputs" mapstructure:"inputs"`
	Parsers ParserConfig  `yaml:"parsers" mapstructure:"parsers"`
	Filters []FilterConfig `yaml:"filters" mapstructure:"filters"`
	Output  OutputConfig  `yaml:"output" mapstructure:"output"`
	Logging LoggingConfig `yaml:"logging" mapstructure:"logging"`
}

// AgentConfig Agent 全局配置
type AgentConfig struct {
	Name         string        `yaml:"name" mapstructure:"name"`
	RegistryPath string        `yaml:"registry_path" mapstructure:"registry_path"`
	FlushInterval time.Duration `yaml:"flush_interval" mapstructure:"flush_interval"`
	BufferSize   int           `yaml:"buffer_size" mapstructure:"buffer_size"`
}

// InputConfig 输入配置
type InputConfig struct {
	Type     string            `yaml:"type" mapstructure:"type"` // file
	Paths    []string          `yaml:"paths" mapstructure:"paths"`
	Exclude  []string          `yaml:"exclude" mapstructure:"exclude"`
	Tags     map[string]string `yaml:"tags" mapstructure:"tags"`
	Multiline *MultilineConfig `yaml:"multiline,omitempty" mapstructure:"multiline"`
}

// MultilineConfig 多行日志配置
type MultilineConfig struct {
	Pattern string `yaml:"pattern" mapstructure:"pattern"`
	Negate  bool   `yaml:"negate" mapstructure:"negate"`
	Match   string `yaml:"match" mapstructure:"match"` // before / after
	MaxLines int   `yaml:"max_lines" mapstructure:"max_lines"`
}

// ParserConfig 解析器配置
type ParserConfig struct {
	Type    string `yaml:"type" mapstructure:"type"` // json / regex / plain
	Pattern string `yaml:"pattern,omitempty" mapstructure:"pattern"` // 正则模式
	TimeKey string `yaml:"time_key,omitempty" mapstructure:"time_key"`
	TimeFmt string `yaml:"time_format,omitempty" mapstructure:"time_format"`
}

// FilterConfig 过滤器配置
type FilterConfig struct {
	Type    string `yaml:"type" mapstructure:"type"`       // include / exclude
	Field   string `yaml:"field" mapstructure:"field"`     // 匹配字段
	Pattern string `yaml:"pattern" mapstructure:"pattern"` // 匹配模式
}

// OutputConfig 输出配置
type OutputConfig struct {
	Type          string `yaml:"type" mapstructure:"type"` // stdout / kafka / elasticsearch
	// Kafka 配置
	KafkaBrokers  []string `yaml:"kafka_brokers,omitempty" mapstructure:"kafka_brokers"`
	KafkaTopic    string   `yaml:"kafka_topic,omitempty" mapstructure:"kafka_topic"`
	// Elasticsearch 配置
	ESHosts       []string `yaml:"es_hosts,omitempty" mapstructure:"es_hosts"`
	ESIndex       string   `yaml:"es_index,omitempty" mapstructure:"es_index"`
	ESBatchSize   int      `yaml:"es_batch_size,omitempty" mapstructure:"es_batch_size"`
}

// LoggingConfig 日志配置
type LoggingConfig struct {
	Level  string `yaml:"level" mapstructure:"level"`
	Format string `yaml:"format" mapstructure:"format"` // json / text
}

// DefaultConfig 默认配置
func DefaultConfig() *Config {
	return &Config{
		Agent: AgentConfig{
			Name:          "log-agent",
			RegistryPath:  "/tmp/log-agent-registry.json",
			FlushInterval: 5 * time.Second,
			BufferSize:    1000,
		},
		Parsers: ParserConfig{
			Type: "plain",
		},
		Output: OutputConfig{
			Type: "stdout",
		},
		Logging: LoggingConfig{
			Level:  "info",
			Format: "text",
		},
	}
}

// Validate 验证配置
func (c *Config) Validate() error {
	if len(c.Inputs) == 0 {
		return fmt.Errorf("至少需要一个 input 配置")
	}
	for i, input := range c.Inputs {
		if len(input.Paths) == 0 {
			return fmt.Errorf("input[%d] 必须指定 paths", i)
		}
	}
	switch c.Output.Type {
	case "stdout", "kafka", "elasticsearch":
		// 合法
	default:
		return fmt.Errorf("不支持的 output 类型: %s", c.Output.Type)
	}
	return nil
}
```

### 2.3 配置文件示例

```yaml
# configs/agent.yaml
# 日志收集 Agent 配置文件

agent:
  name: my-log-agent
  registry_path: /var/lib/log-agent/registry.json
  flush_interval: 5s
  buffer_size: 2000

inputs:
  - type: file
    paths:
      - /var/log/app/*.log
      - /var/log/nginx/access.log
    exclude:
      - /var/log/app/*.gz
    tags:
      env: production
      service: my-service

  - type: file
    paths:
      - /var/log/syslog
    tags:
      source: system

parsers:
  # JSON 格式日志
  type: json
  time_key: timestamp
  time_format: "2006-01-02T15:04:05.000Z"

  # 或正则格式
  # type: regex
  # pattern: '^(?P<timestamp>\S+ \S+) (?P<level>\w+) (?P<message>.*)'

filters:
  - type: include
    field: level
    pattern: "ERROR|WARN|FATAL"

  - type: exclude
    field: message
    pattern: "health.?check|ping"

output:
  type: kafka
  kafka_brokers:
    - kafka1:9092
    - kafka2:9092
    - kafka3:9092
  kafka_topic: app-logs

  # 或 Elasticsearch
  # type: elasticsearch
  # es_hosts:
  #   - http://es1:9200
  #   - http://es2:9200
  # es_index: app-logs
  # es_batch_size: 100

logging:
  level: info
  format: json
```

### 2.4 文件 Tailer（核心组件）

```go
// internal/input/tailer.go
// 文件 Tail 实现：监控文件变化，读取新增内容
package input

import (
	"bufio"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"os"
	"path/filepath"
	"sync"
	"time"

	"github.com/fsnotify/fsnotify"
)

// LogLine 一行日志
type LogLine struct {
	Source    string    // 来源文件
	Content  string    // 行内容
	Offset   int64     // 偏移量
	Line     int       // 行号
	ReadAt   time.Time // 读取时间
}

// FileState 文件状态
type FileState struct {
	Path    string    `json:"path"`
	Offset  int64     `json:"offset"`
	Line    int       `json:"line"`
	SavedAt time.Time `json:"saved_at"`
}

// Registry 注册表（断点续传）
type Registry struct {
	mu     sync.RWMutex
	states map[string]*FileState
	path   string
}

// NewRegistry 创建注册表
func NewRegistry(path string) *Registry {
	r := &Registry{
		states: make(map[string]*FileState),
		path:   path,
	}
	r.load() // 尝试加载历史状态
	return r
}

// load 加载注册表
func (r *Registry) load() {
	data, err := os.ReadFile(r.path)
	if err != nil {
		return // 文件不存在时返回空
	}
	var states map[string]*FileState
	if err := json.Unmarshal(data, &states); err != nil {
		return
	}
	r.states = states
}

// Save 保存注册表
func (r *Registry) Save() error {
	r.mu.RLock()
	defer r.mu.RUnlock()

	data, err := json.MarshalIndent(r.states, "", "  ")
	if err != nil {
		return err
	}

	// 先写临时文件，再原子重命名
	tmpPath := r.path + ".tmp"
	if err := os.WriteFile(tmpPath, data, 0644); err != nil {
		return err
	}
	return os.Rename(tmpPath, r.path)
}

// GetState 获取文件状态
func (r *Registry) GetState(path string) *FileState {
	r.mu.RLock()
	defer r.mu.RUnlock()
	return r.states[path]
}

// UpdateState 更新文件状态
func (r *Registry) UpdateState(state *FileState) {
	r.mu.Lock()
	defer r.mu.Unlock()
	state.SavedAt = time.Now()
	r.states[state.Path] = state
}

// Tailer 文件 Tail 监听器
type Tailer struct {
	logger   *slog.Logger
	registry *Registry
	output   chan<- LogLine
	watcher  *fsnotify.Watcher
	files    map[string]*tailedFile
	mu       sync.Mutex
}

type tailedFile struct {
	path   string
	file   *os.File
	reader *bufio.Reader
	offset int64
	line   int
}

// NewTailer 创建 Tailer
func NewTailer(registry *Registry, output chan<- LogLine, logger *slog.Logger) (*Tailer, error) {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		return nil, fmt.Errorf("创建文件监听器失败: %w", err)
	}

	return &Tailer{
		logger:   logger,
		registry: registry,
		output:   output,
		watcher:  watcher,
		files:    make(map[string]*tailedFile),
	}, nil
}

// AddPath 添加要监控的路径（支持 glob）
func (t *Tailer) AddPath(pattern string) error {
	matches, err := filepath.Glob(pattern)
	if err != nil {
		return fmt.Errorf("glob 解析失败: %w", err)
	}

	for _, path := range matches {
		absPath, err := filepath.Abs(path)
		if err != nil {
			t.logger.Warn("获取绝对路径失败", "path", path, "error", err)
			continue
		}

		if err := t.openFile(absPath); err != nil {
			t.logger.Warn("打开文件失败", "path", absPath, "error", err)
			continue
		}
	}

	// 监控目录以检测新文件
	dir := filepath.Dir(pattern)
	if err := t.watcher.Add(dir); err != nil {
		t.logger.Warn("监控目录失败", "dir", dir, "error", err)
	}

	return nil
}

// openFile 打开并初始化文件
func (t *Tailer) openFile(path string) error {
	t.mu.Lock()
	defer t.mu.Unlock()

	if _, exists := t.files[path]; exists {
		return nil // 已经在监控
	}

	f, err := os.Open(path)
	if err != nil {
		return err
	}

	// 恢复断点
	var offset int64
	var line int
	if state := t.registry.GetState(path); state != nil {
		offset = state.Offset
		line = state.Line
		if _, err := f.Seek(offset, io.SeekStart); err != nil {
			t.logger.Warn("断点恢复失败，从头开始", "path", path, "error", err)
			offset = 0
			line = 0
		} else {
			t.logger.Info("断点续传", "path", path, "offset", offset, "line", line)
		}
	} else {
		// 新文件：从末尾开始（不读取历史内容）
		offset, _ = f.Seek(0, io.SeekEnd)
	}

	tf := &tailedFile{
		path:   path,
		file:   f,
		reader: bufio.NewReaderSize(f, 64*1024), // 64KB 缓冲
		offset: offset,
		line:   line,
	}

	t.files[path] = tf
	t.logger.Info("开始监控文件", "path", path, "offset", offset)

	return nil
}

// readLines 从文件读取新行
func (t *Tailer) readLines(tf *tailedFile) {
	for {
		line, err := tf.reader.ReadString('\n')
		if err != nil {
			if err != io.EOF {
				t.logger.Error("读取文件失败", "path", tf.path, "error", err)
			}
			// 处理不完整的最后一行
			if len(line) > 0 {
				// 暂时跳过不完整的行（等下一次事件时重试）
			}
			return
		}

		tf.line++
		// 去除末尾换行符
		if len(line) > 0 && line[len(line)-1] == '\n' {
			line = line[:len(line)-1]
		}
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}

		if line == "" {
			continue
		}

		// 更新偏移量
		info, _ := tf.file.Stat()
		if info != nil {
			tf.offset = info.Size() - int64(tf.reader.Buffered())
		}

		// 发送到 channel
		t.output <- LogLine{
			Source:  tf.path,
			Content: line,
			Offset:  tf.offset,
			Line:    tf.line,
			ReadAt:  time.Now(),
		}
	}
}

// Run 启动 Tailer
func (t *Tailer) Run(ctx context.Context) error {
	// 先读取现有内容
	t.mu.Lock()
	for _, tf := range t.files {
		t.readLines(tf)
	}
	t.mu.Unlock()

	// 定期保存注册表
	saveTicker := time.NewTicker(5 * time.Second)
	defer saveTicker.Stop()

	// 定期轮询（补充 fsnotify 可能的遗漏）
	pollTicker := time.NewTicker(time.Second)
	defer pollTicker.Stop()

	for {
		select {
		case <-ctx.Done():
			t.logger.Info("Tailer 停止")
			t.saveStates()
			return nil

		case event, ok := <-t.watcher.Events:
			if !ok {
				return nil
			}
			if event.Has(fsnotify.Write) {
				t.mu.Lock()
				if tf, exists := t.files[event.Name]; exists {
					t.readLines(tf)
				}
				t.mu.Unlock()
			}

		case err, ok := <-t.watcher.Errors:
			if !ok {
				return nil
			}
			t.logger.Error("文件监控错误", "error", err)

		case <-pollTicker.C:
			// 定期轮询所有文件
			t.mu.Lock()
			for _, tf := range t.files {
				t.readLines(tf)
			}
			t.mu.Unlock()

		case <-saveTicker.C:
			t.saveStates()
		}
	}
}

// saveStates 保存所有文件状态
func (t *Tailer) saveStates() {
	t.mu.Lock()
	defer t.mu.Unlock()

	for _, tf := range t.files {
		t.registry.UpdateState(&FileState{
			Path:   tf.path,
			Offset: tf.offset,
			Line:   tf.line,
		})
	}
	if err := t.registry.Save(); err != nil {
		t.logger.Error("保存注册表失败", "error", err)
	}
}

// Close 关闭 Tailer
func (t *Tailer) Close() error {
	t.saveStates()
	t.watcher.Close()
	t.mu.Lock()
	defer t.mu.Unlock()
	for _, tf := range t.files {
		tf.file.Close()
	}
	return nil
}
```

### 2.5 日志解析器

```go
// internal/parser/parser.go
// 日志解析器：支持 JSON、正则、纯文本格式
package parser

import (
	"encoding/json"
	"fmt"
	"regexp"
	"strings"
	"time"
)

// LogEvent 解析后的日志事件
type LogEvent struct {
	Source      string                 `json:"source"`
	Offset     int64                  `json:"offset"`
	Timestamp  time.Time              `json:"timestamp"`
	Level      string                 `json:"level"`
	Message    string                 `json:"message"`
	RawMessage string                 `json:"raw_message"`
	Fields     map[string]interface{} `json:"fields,omitempty"`
	Tags       map[string]string      `json:"tags,omitempty"`
}

// Parser 解析器接口
type Parser interface {
	Parse(source string, line string) (*LogEvent, error)
}

// === JSON 解析器 ===

type JSONParser struct {
	timeKey    string
	timeFormat string
}

func NewJSONParser(timeKey, timeFormat string) *JSONParser {
	return &JSONParser{
		timeKey:    timeKey,
		timeFormat: timeFormat,
	}
}

func (p *JSONParser) Parse(source, line string) (*LogEvent, error) {
	var fields map[string]interface{}
	if err := json.Unmarshal([]byte(line), &fields); err != nil {
		// JSON 解析失败，作为纯文本处理
		return &LogEvent{
			Source:     source,
			Timestamp:  time.Now(),
			Message:    line,
			RawMessage: line,
			Fields:     map[string]interface{}{},
		}, nil
	}

	event := &LogEvent{
		Source:     source,
		Timestamp:  time.Now(),
		RawMessage: line,
		Fields:     fields,
	}

	// 提取时间
	if p.timeKey != "" {
		if ts, ok := fields[p.timeKey].(string); ok {
			if t, err := time.Parse(p.timeFormat, ts); err == nil {
				event.Timestamp = t
			}
			delete(fields, p.timeKey)
		}
	}

	// 提取日志级别
	for _, key := range []string{"level", "severity", "log_level", "loglevel"} {
		if lvl, ok := fields[key].(string); ok {
			event.Level = strings.ToUpper(lvl)
			delete(fields, key)
			break
		}
	}

	// 提取消息
	for _, key := range []string{"message", "msg", "log"} {
		if msg, ok := fields[key].(string); ok {
			event.Message = msg
			delete(fields, key)
			break
		}
	}

	if event.Message == "" {
		event.Message = line
	}

	return event, nil
}

// === 正则解析器 ===

type RegexParser struct {
	pattern    *regexp.Regexp
	timeFormat string
}

func NewRegexParser(pattern, timeFormat string) (*RegexParser, error) {
	re, err := regexp.Compile(pattern)
	if err != nil {
		return nil, fmt.Errorf("正则编译失败: %w", err)
	}
	return &RegexParser{
		pattern:    re,
		timeFormat: timeFormat,
	}, nil
}

func (p *RegexParser) Parse(source, line string) (*LogEvent, error) {
	event := &LogEvent{
		Source:     source,
		Timestamp:  time.Now(),
		Message:    line,
		RawMessage: line,
		Fields:     make(map[string]interface{}),
	}

	match := p.pattern.FindStringSubmatch(line)
	if match == nil {
		return event, nil
	}

	names := p.pattern.SubexpNames()
	for i, name := range names {
		if i == 0 || name == "" {
			continue
		}
		value := match[i]
		event.Fields[name] = value

		switch name {
		case "timestamp", "time":
			if t, err := time.Parse(p.timeFormat, value); err == nil {
				event.Timestamp = t
			}
		case "level", "severity":
			event.Level = strings.ToUpper(value)
		case "message", "msg":
			event.Message = value
		}
	}

	return event, nil
}

// === 纯文本解析器 ===

type PlainParser struct{}

func NewPlainParser() *PlainParser {
	return &PlainParser{}
}

func (p *PlainParser) Parse(source, line string) (*LogEvent, error) {
	// 尝试自动检测日志级别
	level := detectLevel(line)

	return &LogEvent{
		Source:     source,
		Timestamp:  time.Now(),
		Level:      level,
		Message:    line,
		RawMessage: line,
		Fields:     map[string]interface{}{},
	}, nil
}

// detectLevel 自动检测日志级别
func detectLevel(line string) string {
	upper := strings.ToUpper(line)
	switch {
	case strings.Contains(upper, "FATAL"):
		return "FATAL"
	case strings.Contains(upper, "ERROR") || strings.Contains(upper, "ERR"):
		return "ERROR"
	case strings.Contains(upper, "WARN"):
		return "WARN"
	case strings.Contains(upper, "INFO"):
		return "INFO"
	case strings.Contains(upper, "DEBUG"):
		return "DEBUG"
	case strings.Contains(upper, "TRACE"):
		return "TRACE"
	default:
		return ""
	}
}
```

### 2.6 过滤器

```go
// internal/filter/filter.go
// 日志过滤器：根据规则过滤日志事件
package filter

import (
	"regexp"
	"strings"
)

// LogEvent 简化的事件结构
type LogEvent struct {
	Level   string
	Message string
	Fields  map[string]interface{}
}

// Filter 过滤器接口
type Filter interface {
	// Match 返回 true 表示事件通过过滤（应该保留）
	Match(event *LogEvent) bool
}

// IncludeFilter 包含过滤器（只保留匹配的事件）
type IncludeFilter struct {
	field   string
	pattern *regexp.Regexp
}

func NewIncludeFilter(field, pattern string) (*IncludeFilter, error) {
	re, err := regexp.Compile(pattern)
	if err != nil {
		return nil, err
	}
	return &IncludeFilter{field: field, pattern: re}, nil
}

func (f *IncludeFilter) Match(event *LogEvent) bool {
	value := getFieldValue(event, f.field)
	return f.pattern.MatchString(value)
}

// ExcludeFilter 排除过滤器（排除匹配的事件）
type ExcludeFilter struct {
	field   string
	pattern *regexp.Regexp
}

func NewExcludeFilter(field, pattern string) (*ExcludeFilter, error) {
	re, err := regexp.Compile(pattern)
	if err != nil {
		return nil, err
	}
	return &ExcludeFilter{field: field, pattern: re}, nil
}

func (f *ExcludeFilter) Match(event *LogEvent) bool {
	value := getFieldValue(event, f.field)
	return !f.pattern.MatchString(value) // 不匹配的才保留
}

// LevelFilter 级别过滤器
type LevelFilter struct {
	minLevel int
}

var levelPriority = map[string]int{
	"TRACE": 0,
	"DEBUG": 1,
	"INFO":  2,
	"WARN":  3,
	"ERROR": 4,
	"FATAL": 5,
}

func NewLevelFilter(minLevel string) *LevelFilter {
	priority, ok := levelPriority[strings.ToUpper(minLevel)]
	if !ok {
		priority = 0
	}
	return &LevelFilter{minLevel: priority}
}

func (f *LevelFilter) Match(event *LogEvent) bool {
	priority, ok := levelPriority[strings.ToUpper(event.Level)]
	if !ok {
		return true // 未知级别默认通过
	}
	return priority >= f.minLevel
}

// FilterChain 过滤器链
type FilterChain struct {
	filters []Filter
}

func NewFilterChain(filters ...Filter) *FilterChain {
	return &FilterChain{filters: filters}
}

func (fc *FilterChain) Match(event *LogEvent) bool {
	for _, f := range fc.filters {
		if !f.Match(event) {
			return false
		}
	}
	return true
}

// getFieldValue 获取事件中指定字段的值
func getFieldValue(event *LogEvent, field string) string {
	switch field {
	case "level":
		return event.Level
	case "message":
		return event.Message
	default:
		if v, ok := event.Fields[field]; ok {
			switch val := v.(type) {
			case string:
				return val
			default:
				return ""
			}
		}
		return ""
	}
}
```

### 2.7 输出器

```go
// internal/output/output.go
// 输出器：支持 Stdout / Kafka / Elasticsearch
package output

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"os"
	"sync"
	"time"
)

// LogEvent 输出用的事件
type LogEvent struct {
	Source    string                 `json:"source"`
	Timestamp time.Time             `json:"timestamp"`
	Level    string                 `json:"level"`
	Message  string                 `json:"message"`
	Fields   map[string]interface{} `json:"fields,omitempty"`
	Tags     map[string]string      `json:"tags,omitempty"`
}

// Output 输出器接口
type Output interface {
	Write(ctx context.Context, events []*LogEvent) error
	Close() error
}

// === Stdout 输出 ===

type StdoutOutput struct {
	writer  io.Writer
	encoder *json.Encoder
	mu      sync.Mutex
}

func NewStdoutOutput() *StdoutOutput {
	return &StdoutOutput{
		writer:  os.Stdout,
		encoder: json.NewEncoder(os.Stdout),
	}
}

func (o *StdoutOutput) Write(ctx context.Context, events []*LogEvent) error {
	o.mu.Lock()
	defer o.mu.Unlock()

	for _, event := range events {
		if err := o.encoder.Encode(event); err != nil {
			return fmt.Errorf("编码事件失败: %w", err)
		}
	}
	return nil
}

func (o *StdoutOutput) Close() error {
	return nil
}

// === Kafka 输出（演示结构） ===

type KafkaOutput struct {
	brokers []string
	topic   string
	logger  *slog.Logger
	// writer  *kafka.Writer // 实际使用 kafka-go
}

func NewKafkaOutput(brokers []string, topic string, logger *slog.Logger) *KafkaOutput {
	/*
	实际实现：
	writer := &kafka.Writer{
		Addr:         kafka.TCP(brokers...),
		Topic:        topic,
		Balancer:     &kafka.LeastBytes{},
		BatchSize:    100,
		BatchTimeout: time.Second,
		RequiredAcks: kafka.RequireOne,
		Async:        true,
	}
	*/

	return &KafkaOutput{
		brokers: brokers,
		topic:   topic,
		logger:  logger,
	}
}

func (o *KafkaOutput) Write(ctx context.Context, events []*LogEvent) error {
	/*
	实际实现：
	messages := make([]kafka.Message, len(events))
	for i, event := range events {
		data, err := json.Marshal(event)
		if err != nil {
			continue
		}
		messages[i] = kafka.Message{
			Key:   []byte(event.Source),
			Value: data,
		}
	}
	return o.writer.WriteMessages(ctx, messages...)
	*/

	o.logger.Debug("模拟发送到 Kafka",
		"topic", o.topic,
		"events", len(events),
	)
	return nil
}

func (o *KafkaOutput) Close() error {
	// return o.writer.Close()
	return nil
}

// === Elasticsearch 输出（演示结构） ===

type ElasticsearchOutput struct {
	hosts     []string
	index     string
	batchSize int
	logger    *slog.Logger
	// client *elasticsearch.Client
}

func NewElasticsearchOutput(hosts []string, index string, batchSize int, logger *slog.Logger) *ElasticsearchOutput {
	/*
	实际实现：
	cfg := elasticsearch.Config{
		Addresses: hosts,
	}
	client, _ := elasticsearch.NewClient(cfg)
	*/

	return &ElasticsearchOutput{
		hosts:     hosts,
		index:     index,
		batchSize: batchSize,
		logger:    logger,
	}
}

func (o *ElasticsearchOutput) Write(ctx context.Context, events []*LogEvent) error {
	/*
	实际实现使用 Bulk API：
	var buf bytes.Buffer
	for _, event := range events {
		meta := fmt.Sprintf(`{"index":{"_index":"%s"}}`, o.index)
		buf.WriteString(meta + "\n")
		data, _ := json.Marshal(event)
		buf.Write(data)
		buf.WriteString("\n")
	}
	res, err := o.client.Bulk(&buf)
	*/

	o.logger.Debug("模拟发送到 Elasticsearch",
		"index", o.index,
		"events", len(events),
	)
	return nil
}

func (o *ElasticsearchOutput) Close() error {
	return nil
}
```

### 2.8 Pipeline 与主程序

```go
// cmd/agent/main.go
// 日志收集 Agent 主程序
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

var (
	version   = "dev"
	gitCommit = "unknown"
)

// Pipeline 数据处理管道
type Pipeline struct {
	logger    *slog.Logger
	inputCh   chan LogLine
	outputCh  chan *LogEvent
	batchSize int
	flushInterval time.Duration
}

// LogLine 原始日志行
type LogLine struct {
	Source  string
	Content string
	Offset  int64
	ReadAt  time.Time
}

// LogEvent 处理后的日志事件
type LogEvent struct {
	Source    string                 `json:"source"`
	Timestamp time.Time             `json:"@timestamp"`
	Level    string                 `json:"level"`
	Message  string                 `json:"message"`
	Fields   map[string]interface{} `json:"fields,omitempty"`
	Tags     map[string]string      `json:"tags,omitempty"`
}

func NewPipeline(logger *slog.Logger, bufferSize int, flushInterval time.Duration) *Pipeline {
	return &Pipeline{
		logger:        logger,
		inputCh:       make(chan LogLine, bufferSize),
		outputCh:      make(chan *LogEvent, bufferSize),
		batchSize:     100,
		flushInterval: flushInterval,
	}
}

// ProcessLoop 处理循环（解析 + 过滤）
func (p *Pipeline) ProcessLoop(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case line, ok := <-p.inputCh:
			if !ok {
				return
			}
			// 解析
			event := &LogEvent{
				Source:    line.Source,
				Timestamp: line.ReadAt,
				Message:   line.Content,
				Fields:    make(map[string]interface{}),
				Tags:      make(map[string]string),
			}

			// 尝试 JSON 解析
			var fields map[string]interface{}
			if err := json.Unmarshal([]byte(line.Content), &fields); err == nil {
				event.Fields = fields
				if msg, ok := fields["message"].(string); ok {
					event.Message = msg
				}
				if lvl, ok := fields["level"].(string); ok {
					event.Level = lvl
				}
			}

			// 发送到输出 channel
			select {
			case p.outputCh <- event:
			case <-ctx.Done():
				return
			}
		}
	}
}

// OutputLoop 输出循环（批量发送）
func (p *Pipeline) OutputLoop(ctx context.Context) {
	var batch []*LogEvent
	ticker := time.NewTicker(p.flushInterval)
	defer ticker.Stop()

	flush := func() {
		if len(batch) == 0 {
			return
		}
		// 输出到 stdout（实际项目中替换为 Kafka/ES）
		for _, event := range batch {
			data, _ := json.Marshal(event)
			fmt.Println(string(data))
		}
		p.logger.Debug("批量发送完成", "count", len(batch))
		batch = batch[:0]
	}

	for {
		select {
		case <-ctx.Done():
			flush() // 退出前刷新剩余数据
			return
		case event, ok := <-p.outputCh:
			if !ok {
				flush()
				return
			}
			batch = append(batch, event)
			if len(batch) >= p.batchSize {
				flush()
			}
		case <-ticker.C:
			flush()
		}
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	logger.Info("日志收集 Agent 启动",
		"version", version,
		"commit", gitCommit,
	)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 创建 Pipeline
	pipeline := NewPipeline(logger, 2000, 5*time.Second)

	var wg sync.WaitGroup

	// 启动处理循环
	wg.Add(1)
	go func() {
		defer wg.Done()
		pipeline.ProcessLoop(ctx)
	}()

	// 启动输出循环
	wg.Add(1)
	go func() {
		defer wg.Done()
		pipeline.OutputLoop(ctx)
	}()

	// 模拟输入（实际项目中用 Tailer）
	wg.Add(1)
	go func() {
		defer wg.Done()
		ticker := time.NewTicker(100 * time.Millisecond)
		defer ticker.Stop()
		i := 0
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				i++
				line := LogLine{
					Source:  "/var/log/app/demo.log",
					Content: fmt.Sprintf(`{"timestamp":"%s","level":"INFO","message":"处理请求 #%d","request_id":"req-%d"}`,
						time.Now().Format(time.RFC3339), i, i),
					ReadAt: time.Now(),
				}
				select {
				case pipeline.inputCh <- line:
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	// 等待退出信号
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

	sig := <-sigCh
	logger.Info("收到信号，开始优雅关闭", "signal", sig.String())
	cancel()

	// 等待所有 goroutine 退出
	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		logger.Info("Agent 已安全关闭")
	case <-time.After(10 * time.Second):
		logger.Warn("关闭超时，强制退出")
	}
}
```

---

## 三、常见坑点

```
坑点 1：fsnotify 在某些文件系统上不可靠
  NFS、CIFS 等网络文件系统可能不支持 inotify。
  需要定期轮询作为补充。

坑点 2：文件轮转（logrotate）处理
  文件被重命名或截断时，需要重新打开。
  检测方法：比较 inode 或检查文件大小是否变小。

坑点 3：断点续传文件损坏
  JSON 写入中途崩溃会导致文件损坏。
  使用原子写入（先写临时文件再重命名）。

坑点 4：Channel 缓冲区满导致阻塞
  输出端慢于输入端时 channel 满了会阻塞。
  需要背压机制或丢弃策略。

坑点 5：正则表达式性能
  复杂正则在高流量下会成为瓶颈。
  编译一次复用，避免在循环中 Compile。
```

---

## 四、速查表

| 组件 | 职责 | 关键技术 |
|---|---|---|
| Input (Tailer) | 监控文件变化，读取新行 | fsnotify + 轮询 |
| Registry | 断点续传，记录文件偏移 | JSON 文件 + 原子写入 |
| Parser | 解析日志格式 | JSON/正则/纯文本 |
| Filter | 过滤日志事件 | 正则/级别/关键字 |
| Output | 发送到目标系统 | 批量发送 + 重试 |
| Pipeline | 串联各组件 | Channel + goroutine |
