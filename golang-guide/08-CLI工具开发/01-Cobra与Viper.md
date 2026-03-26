# 01-Cobra 与 Viper

## 导读

几乎所有知名的 Go CLI 工具都使用 Cobra 框架：kubectl、docker、hugo、gh（GitHub CLI）、helm、etcdctl。Cobra 提供了强大的命令行解析、子命令组织、自动补全和帮助生成能力。而 Viper 则是 Cobra 的姊妹项目，专注于配置管理——支持读取配置文件、环境变量、命令行参数，并提供统一的配置访问接口。

对于 SRE/DevOps 工程师来说，掌握 Cobra + Viper 意味着能快速开发出专业级的运维 CLI 工具，替代繁琐的 shell 脚本，实现更好的可维护性、可测试性和跨平台兼容性。

**本章你将学到：**
- Cobra 命令结构与子命令组织
- Flags 的各种用法（Persistent/Local/Required）
- Viper 配置管理（文件/环境变量/命令行参数）
- WatchConfig 动态配置
- 完整实战：服务器信息查询 CLI 工具

**前置要求：**
- 熟悉 Go 基础语法
- 了解命令行工具的基本使用

---

## 一、Cobra 基础

### 1.1 安装

```bash
# 安装 Cobra 库
go get github.com/spf13/cobra

# 安装 cobra-cli 代码生成工具（可选，但推荐）
go install github.com/spf13/cobra-cli@latest
```

### 1.2 使用 cobra-cli 初始化项目

```bash
# 创建项目
mkdir ops-cli && cd ops-cli
go mod init ops-cli

# 使用 cobra-cli 初始化
cobra-cli init

# 生成的目录结构：
# ops-cli/
# ├── cmd/
# │   └── root.go    # 根命令
# ├── main.go        # 入口文件
# ├── go.mod
# └── go.sum

# 添加子命令
cobra-cli add server    # 生成 cmd/server.go
cobra-cli add disk      # 生成 cmd/disk.go
cobra-cli add network   # 生成 cmd/network.go
```

### 1.3 Command 核心结构

```go
// cmd/root.go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

var cfgFile string // 配置文件路径

// rootCmd 根命令
var rootCmd = &cobra.Command{
	Use:   "ops-cli",              // 命令名称（用法中显示）
	Short: "运维 CLI 工具",          // 简短描述（帮助列表中显示）
	Long: `ops-cli 是一个面向 SRE/DevOps 工程师的命令行工具。
提供服务器信息查询、磁盘监控、网络诊断等功能。

示例：
  ops-cli server info          # 查看服务器基本信息
  ops-cli disk usage /         # 查看磁盘使用情况
  ops-cli network ping 8.8.8.8 # 网络连通性检测`,

	// PersistentPreRun 在所有子命令之前运行（包括子命令的子命令）
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		if verbose, _ := cmd.Flags().GetBool("verbose"); verbose {
			fmt.Println("启用详细输出模式")
		}
	},

	// Run 命令的主要执行函数
	// 如果根命令不需要直接执行逻辑，可以省略
	// Run: func(cmd *cobra.Command, args []string) { },
}

// Execute 执行根命令（main.go 调用）
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func init() {
	// cobra.OnInitialize 在每个命令的 Execute 方法调用之前运行
	cobra.OnInitialize(initConfig)

	// Persistent Flags: 全局标志，对所有子命令生效
	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
		"配置文件路径 (默认 $HOME/.ops-cli.yaml)")
	rootCmd.PersistentFlags().BoolP("verbose", "v", false, "详细输出")
	rootCmd.PersistentFlags().String("output", "text", "输出格式 (text/json/yaml)")

	// 将 flag 绑定到 viper（后面详细讲）
	viper.BindPFlag("output", rootCmd.PersistentFlags().Lookup("output"))
}

func initConfig() {
	if cfgFile != "" {
		viper.SetConfigFile(cfgFile)
	} else {
		home, err := os.UserHomeDir()
		cobra.CheckErr(err)

		viper.AddConfigPath(home)        // 在 home 目录查找
		viper.AddConfigPath(".")         // 在当前目录查找
		viper.SetConfigType("yaml")      // 配置文件类型
		viper.SetConfigName(".ops-cli")  // 配置文件名（不含扩展名）
	}

	// 读取环境变量
	viper.AutomaticEnv()

	// 读取配置文件（如果存在）
	if err := viper.ReadInConfig(); err == nil {
		fmt.Fprintln(os.Stderr, "使用配置文件:", viper.ConfigFileUsed())
	}
}
```

### 1.4 入口文件

```go
// main.go
package main

import "ops-cli/cmd"

func main() {
	cmd.Execute()
}
```

---

## 二、子命令与 Flags

### 2.1 子命令定义

```go
// cmd/server.go
package cmd

import (
	"fmt"
	"os"
	"runtime"
	"time"

	"github.com/spf13/cobra"
)

// serverCmd 服务器相关命令（父命令）
var serverCmd = &cobra.Command{
	Use:   "server",
	Short: "服务器信息管理",
	Long:  "查询和管理服务器相关信息，包括系统信息、CPU、内存等。",
	// 没有 Run 函数的命令作为子命令的容器
	// 执行 `ops-cli server` 时会显示帮助信息
}

// serverInfoCmd 查看服务器基本信息
var serverInfoCmd = &cobra.Command{
	Use:   "info",
	Short: "显示服务器基本信息",
	Long:  "显示操作系统、架构、CPU 核心数、Go 版本等信息。",
	Example: `  ops-cli server info
  ops-cli server info --format json
  ops-cli server info -o json`,
	Aliases: []string{"i", "status"}, // 命令别名

	// RunE 返回 error 的版本（推荐使用，比 Run 更好处理错误）
	RunE: func(cmd *cobra.Command, args []string) error {
		format, _ := cmd.Flags().GetString("format")

		info := map[string]string{
			"OS":         runtime.GOOS,
			"Arch":       runtime.GOARCH,
			"CPU核心":     fmt.Sprintf("%d", runtime.NumCPU()),
			"Go版本":      runtime.Version(),
			"主机名":       getHostname(),
			"运行时间":     time.Now().Format(time.RFC3339),
			"Goroutines": fmt.Sprintf("%d", runtime.NumGoroutine()),
		}

		switch format {
		case "json":
			return outputJSON(info)
		default:
			return outputText(info)
		}
	},
}

// serverProcessCmd 查看进程信息
var serverProcessCmd = &cobra.Command{
	Use:   "process [pid]",
	Short: "查看进程信息",
	Args:  cobra.MaximumNArgs(1), // 最多接受 1 个参数

	RunE: func(cmd *cobra.Command, args []string) error {
		all, _ := cmd.Flags().GetBool("all")

		if len(args) > 0 {
			fmt.Printf("查看进程 PID=%s 的信息\n", args[0])
		} else if all {
			fmt.Println("列出所有进程")
		} else {
			fmt.Println("请指定 PID 或使用 --all 列出所有进程")
		}
		return nil
	},
}

func init() {
	// 将子命令添加到父命令
	rootCmd.AddCommand(serverCmd)
	serverCmd.AddCommand(serverInfoCmd)
	serverCmd.AddCommand(serverProcessCmd)

	// Local Flags: 仅对当前命令生效
	serverInfoCmd.Flags().StringP("format", "f", "text", "输出格式 (text/json)")

	// serverProcessCmd 的 flags
	serverProcessCmd.Flags().BoolP("all", "a", false, "列出所有进程")
}

func getHostname() string {
	hostname, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return hostname
}
```

### 2.2 Flags 详解

```go
// cmd/flags_demo.go
package cmd

import (
	"fmt"
	"net"
	"time"

	"github.com/spf13/cobra"
)

var networkCmd = &cobra.Command{
	Use:   "network",
	Short: "网络诊断工具",
}

var pingCmd = &cobra.Command{
	Use:   "ping <host>",
	Short: "检测网络连通性",
	Long:  "通过 TCP 连接检测目标主机的网络连通性。",

	// Args 参数校验
	Args: cobra.ExactArgs(1), // 必须恰好 1 个参数

	RunE: func(cmd *cobra.Command, args []string) error {
		host := args[0]
		port, _ := cmd.Flags().GetInt("port")
		count, _ := cmd.Flags().GetInt("count")
		timeout, _ := cmd.Flags().GetDuration("timeout")

		addr := fmt.Sprintf("%s:%d", host, port)
		fmt.Printf("正在检测 %s 的连通性 (超时: %v, 次数: %d)\n\n", addr, timeout, count)

		successCount := 0
		for i := 0; i < count; i++ {
			start := time.Now()
			conn, err := net.DialTimeout("tcp", addr, timeout)
			elapsed := time.Since(start)

			if err != nil {
				fmt.Printf("[%d/%d] 失败: %v\n", i+1, count, err)
			} else {
				conn.Close()
				fmt.Printf("[%d/%d] 连接成功, 耗时: %v\n", i+1, count, elapsed)
				successCount++
			}
			time.Sleep(time.Second)
		}

		fmt.Printf("\n统计: %d/%d 成功 (成功率: %.1f%%)\n",
			successCount, count, float64(successCount)/float64(count)*100)
		return nil
	},
}

var dnsCmd = &cobra.Command{
	Use:   "dns <domain>",
	Short: "DNS 查询",
	Args:  cobra.ExactArgs(1),

	RunE: func(cmd *cobra.Command, args []string) error {
		domain := args[0]
		recordType, _ := cmd.Flags().GetString("type")

		fmt.Printf("查询 %s 的 %s 记录:\n\n", domain, recordType)

		switch recordType {
		case "A":
			ips, err := net.LookupHost(domain)
			if err != nil {
				return fmt.Errorf("查询失败: %w", err)
			}
			for _, ip := range ips {
				fmt.Printf("  %s\n", ip)
			}
		case "MX":
			mxs, err := net.LookupMX(domain)
			if err != nil {
				return fmt.Errorf("查询失败: %w", err)
			}
			for _, mx := range mxs {
				fmt.Printf("  %s (优先级: %d)\n", mx.Host, mx.Pref)
			}
		case "NS":
			nss, err := net.LookupNS(domain)
			if err != nil {
				return fmt.Errorf("查询失败: %w", err)
			}
			for _, ns := range nss {
				fmt.Printf("  %s\n", ns.Host)
			}
		case "TXT":
			txts, err := net.LookupTXT(domain)
			if err != nil {
				return fmt.Errorf("查询失败: %w", err)
			}
			for _, txt := range txts {
				fmt.Printf("  %s\n", txt)
			}
		default:
			return fmt.Errorf("不支持的记录类型: %s", recordType)
		}

		return nil
	},
}

func init() {
	rootCmd.AddCommand(networkCmd)
	networkCmd.AddCommand(pingCmd)
	networkCmd.AddCommand(dnsCmd)

	// ===== 各种 Flag 类型演示 =====

	// Int 类型 flag，带短名称 -p
	pingCmd.Flags().IntP("port", "p", 80, "目标端口号")

	// Int 类型 flag
	pingCmd.Flags().IntP("count", "c", 3, "检测次数")

	// Duration 类型 flag
	pingCmd.Flags().DurationP("timeout", "t", 3*time.Second, "连接超时时间")

	// 必选参数
	dnsCmd.Flags().StringP("type", "t", "A", "记录类型 (A/MX/NS/TXT)")

	// ===== Args 校验函数一览 =====
	// cobra.NoArgs          - 不接受任何参数
	// cobra.ExactArgs(n)    - 恰好 n 个参数
	// cobra.MinimumNArgs(n) - 至少 n 个参数
	// cobra.MaximumNArgs(n) - 最多 n 个参数
	// cobra.RangeArgs(min, max) - min 到 max 个参数
	// cobra.ArbitraryArgs   - 接受任意数量参数
	// cobra.OnlyValidArgs   - 仅接受 ValidArgs 中定义的参数

	// 自定义参数校验
	// Args: func(cmd *cobra.Command, args []string) error {
	//     if len(args) != 1 {
	//         return fmt.Errorf("需要恰好 1 个参数，收到 %d 个", len(args))
	//     }
	//     if net.ParseIP(args[0]) == nil {
	//         return fmt.Errorf("'%s' 不是有效的 IP 地址", args[0])
	//     }
	//     return nil
	// },
}
```

### 2.3 Persistent Flags vs Local Flags

```go
// Persistent Flags：对当前命令及其所有子命令生效
// 典型场景：全局配置、输出格式、日志级别

rootCmd.PersistentFlags().BoolP("verbose", "v", false, "详细输出")
// ops-cli --verbose server info      ✓ 有效
// ops-cli server --verbose info      ✓ 有效
// ops-cli server info --verbose      ✓ 有效

// Local Flags：仅对当前命令生效
serverInfoCmd.Flags().StringP("format", "f", "text", "输出格式")
// ops-cli server info --format json  ✓ 有效
// ops-cli server --format json info  ✗ 无效（format 不是 server 的 flag）

// Required Flags：必选标志
serverInfoCmd.Flags().String("region", "", "区域")
serverInfoCmd.MarkFlagRequired("region")
// 不传 --region 时命令会报错

// 互斥 Flags（Cobra v1.8+）
// serverInfoCmd.MarkFlagsMutuallyExclusive("json", "yaml")
// --json 和 --yaml 不能同时使用

// 配合使用的 Flags
// serverInfoCmd.MarkFlagsRequiredTogether("username", "password")
// --username 和 --password 必须同时使用
```

---

## 三、Viper 配置管理

### 3.1 Viper 基础

```go
// config/viper_demo.go
package config

import (
	"fmt"
	"log"

	"github.com/spf13/viper"
)

// AppConfig 应用配置结构
type AppConfig struct {
	Server   ServerConfig   `mapstructure:"server"`
	Database DatabaseConfig `mapstructure:"database"`
	Log      LogConfig      `mapstructure:"log"`
	Alert    AlertConfig    `mapstructure:"alert"`
}

type ServerConfig struct {
	Host string `mapstructure:"host"`
	Port int    `mapstructure:"port"`
}

type DatabaseConfig struct {
	Host     string `mapstructure:"host"`
	Port     int    `mapstructure:"port"`
	Name     string `mapstructure:"name"`
	User     string `mapstructure:"user"`
	Password string `mapstructure:"password"`
}

type LogConfig struct {
	Level  string `mapstructure:"level"`
	File   string `mapstructure:"file"`
	Format string `mapstructure:"format"`
}

type AlertConfig struct {
	Enabled    bool     `mapstructure:"enabled"`
	Channels   []string `mapstructure:"channels"`
	WebhookURL string   `mapstructure:"webhook_url"`
}

// InitConfig 初始化配置
func InitConfig(configFile string) (*AppConfig, error) {
	v := viper.New()

	// ===== 设置默认值 =====
	v.SetDefault("server.host", "0.0.0.0")
	v.SetDefault("server.port", 8080)
	v.SetDefault("log.level", "info")
	v.SetDefault("log.format", "json")
	v.SetDefault("alert.enabled", true)

	// ===== 配置文件 =====
	if configFile != "" {
		v.SetConfigFile(configFile) // 指定配置文件路径
	} else {
		v.SetConfigName("config")   // 配置文件名（无扩展名）
		v.SetConfigType("yaml")     // 配置文件类型
		v.AddConfigPath(".")        // 查找路径 1：当前目录
		v.AddConfigPath("./config") // 查找路径 2：config 子目录
		v.AddConfigPath("/etc/ops-cli/") // 查找路径 3：系统配置目录
	}

	// ===== 环境变量 =====
	v.SetEnvPrefix("OPS") // 环境变量前缀：OPS_
	v.AutomaticEnv()       // 自动绑定环境变量

	// 环境变量名中的 . 替换为 _
	// 例如：server.port → OPS_SERVER_PORT
	v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

	// ===== 读取配置文件 =====
	if err := v.ReadInConfig(); err != nil {
		if _, ok := err.(viper.ConfigFileNotFoundError); ok {
			log.Println("未找到配置文件，使用默认配置")
		} else {
			return nil, fmt.Errorf("读取配置文件失败: %w", err)
		}
	} else {
		log.Printf("使用配置文件: %s", v.ConfigFileUsed())
	}

	// ===== 反序列化到结构体 =====
	var config AppConfig
	if err := v.Unmarshal(&config); err != nil {
		return nil, fmt.Errorf("解析配置失败: %w", err)
	}

	return &config, nil
}
```

需要导入 strings 包：

```go
import "strings"
```

### 3.2 配置文件示例

```yaml
# config.yaml - YAML 格式配置
server:
  host: "0.0.0.0"
  port: 8080

database:
  host: "localhost"
  port: 5432
  name: "ops_db"
  user: "admin"
  password: "${DB_PASSWORD}"  # 实际值从环境变量获取

log:
  level: "info"     # debug, info, warn, error
  file: "/var/log/ops-cli/app.log"
  format: "json"    # json, text

alert:
  enabled: true
  channels:
    - dingtalk
    - feishu
  webhook_url: "https://oapi.dingtalk.com/robot/send?access_token=xxx"
```

```json
// config.json - JSON 格式配置
{
  "server": {
    "host": "0.0.0.0",
    "port": 8080
  },
  "database": {
    "host": "localhost",
    "port": 5432
  }
}
```

```toml
# config.toml - TOML 格式配置
[server]
host = "0.0.0.0"
port = 8080

[database]
host = "localhost"
port = 5432
```

### 3.3 配置优先级

```
Viper 的配置优先级（从高到低）：

1. 显式调用 Set() 设置的值
2. 命令行标志（flag）
3. 环境变量
4. 配置文件
5. Key/Value 远程存储（如 etcd、Consul）
6. 默认值

示例：
  viper.SetDefault("port", 8080)           # 优先级最低
  # config.yaml 中 port: 9090              # 覆盖默认值
  # 环境变量 OPS_PORT=9091                  # 覆盖配置文件
  # 命令行 --port 9092                     # 覆盖环境变量
  viper.Set("port", 9093)                  # 优先级最高
```

### 3.4 命令行参数绑定到 Viper

```go
// cmd/config_bind.go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

var configCmd = &cobra.Command{
	Use:   "config",
	Short: "配置管理",
}

var configShowCmd = &cobra.Command{
	Use:   "show",
	Short: "显示当前配置",
	RunE: func(cmd *cobra.Command, args []string) error {
		// 通过 Viper 统一读取，不区分来源（flag/环境变量/配置文件）
		fmt.Printf("服务器地址: %s\n", viper.GetString("server.host"))
		fmt.Printf("服务器端口: %d\n", viper.GetInt("server.port"))
		fmt.Printf("日志级别: %s\n", viper.GetString("log.level"))
		fmt.Printf("输出格式: %s\n", viper.GetString("output"))
		fmt.Printf("告警开关: %v\n", viper.GetBool("alert.enabled"))

		// 获取所有配置
		fmt.Println("\n所有配置:")
		for _, key := range viper.AllKeys() {
			fmt.Printf("  %s = %v\n", key, viper.Get(key))
		}
		return nil
	},
}

var configSetCmd = &cobra.Command{
	Use:   "set <key> <value>",
	Short: "设置配置项",
	Args:  cobra.ExactArgs(2),
	RunE: func(cmd *cobra.Command, args []string) error {
		key, value := args[0], args[1]
		viper.Set(key, value)

		// 写回配置文件
		if err := viper.WriteConfig(); err != nil {
			// 配置文件不存在则创建
			if err := viper.SafeWriteConfig(); err != nil {
				return fmt.Errorf("写入配置失败: %w", err)
			}
		}

		fmt.Printf("已设置: %s = %s\n", key, value)
		return nil
	},
}

func init() {
	rootCmd.AddCommand(configCmd)
	configCmd.AddCommand(configShowCmd)
	configCmd.AddCommand(configSetCmd)

	// 将 flag 绑定到 viper
	// 这样 viper.GetString("server.host") 可以同时获取 flag 和配置文件的值
	configShowCmd.Flags().String("server-host", "0.0.0.0", "服务器地址")
	viper.BindPFlag("server.host", configShowCmd.Flags().Lookup("server-host"))

	configShowCmd.Flags().Int("server-port", 8080, "服务器端口")
	viper.BindPFlag("server.port", configShowCmd.Flags().Lookup("server-port"))
}
```

### 3.5 WatchConfig 动态配置

```go
// config/watch.go
package config

import (
	"fmt"
	"log"
	"sync"

	"github.com/fsnotify/fsnotify"
	"github.com/spf13/viper"
)

// DynamicConfig 支持动态更新的配置管理器
type DynamicConfig struct {
	v       *viper.Viper
	config  *AppConfig
	mu      sync.RWMutex
	onChange []func(*AppConfig) // 配置变更回调函数列表
}

// NewDynamicConfig 创建动态配置管理器
func NewDynamicConfig(configFile string) (*DynamicConfig, error) {
	v := viper.New()
	v.SetConfigFile(configFile)

	if err := v.ReadInConfig(); err != nil {
		return nil, fmt.Errorf("读取配置失败: %w", err)
	}

	var config AppConfig
	if err := v.Unmarshal(&config); err != nil {
		return nil, fmt.Errorf("解析配置失败: %w", err)
	}

	dc := &DynamicConfig{
		v:      v,
		config: &config,
	}

	// 启用配置文件监听
	v.WatchConfig()
	v.OnConfigChange(func(e fsnotify.Event) {
		log.Printf("配置文件变更: %s (操作: %s)", e.Name, e.Op)

		// 重新解析配置
		var newConfig AppConfig
		if err := v.Unmarshal(&newConfig); err != nil {
			log.Printf("解析新配置失败: %v", err)
			return
		}

		// 更新配置
		dc.mu.Lock()
		dc.config = &newConfig
		callbacks := make([]func(*AppConfig), len(dc.onChange))
		copy(callbacks, dc.onChange)
		dc.mu.Unlock()

		// 通知所有回调
		for _, fn := range callbacks {
			fn(&newConfig)
		}

		log.Println("配置热更新完成")
	})

	return dc, nil
}

// Get 获取当前配置（只读）
func (dc *DynamicConfig) Get() *AppConfig {
	dc.mu.RLock()
	defer dc.mu.RUnlock()
	return dc.config
}

// OnChange 注册配置变更回调
func (dc *DynamicConfig) OnChange(fn func(*AppConfig)) {
	dc.mu.Lock()
	defer dc.mu.Unlock()
	dc.onChange = append(dc.onChange, fn)
}
```

---

## 四、输出格式化

### 4.1 通用输出函数

```go
// cmd/output.go
package cmd

import (
	"encoding/json"
	"fmt"
	"os"
	"sort"
	"strings"
	"text/tabwriter"

	"gopkg.in/yaml.v3"
)

// outputJSON 以 JSON 格式输出
func outputJSON(data interface{}) error {
	encoder := json.NewEncoder(os.Stdout)
	encoder.SetIndent("", "  ")
	return encoder.Encode(data)
}

// outputYAML 以 YAML 格式输出
func outputYAML(data interface{}) error {
	encoder := yaml.NewEncoder(os.Stdout)
	encoder.SetIndent(2)
	return encoder.Encode(data)
}

// outputText 以文本格式输出键值对
func outputText(data map[string]string) error {
	// 按键排序，保证输出稳定
	keys := make([]string, 0, len(data))
	for k := range data {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	// 使用 tabwriter 对齐输出
	w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
	for _, k := range keys {
		fmt.Fprintf(w, "%s:\t%s\n", k, data[k])
	}
	return w.Flush()
}

// outputTable 以表格格式输出
func outputTable(headers []string, rows [][]string) error {
	w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', tabwriter.TabIndent)

	// 表头
	fmt.Fprintln(w, strings.Join(headers, "\t"))

	// 分隔线
	separators := make([]string, len(headers))
	for i, h := range headers {
		separators[i] = strings.Repeat("-", len(h))
	}
	fmt.Fprintln(w, strings.Join(separators, "\t"))

	// 数据行
	for _, row := range rows {
		fmt.Fprintln(w, strings.Join(row, "\t"))
	}

	return w.Flush()
}
```

---

## 五、完整实战：服务器信息查询工具

### 5.1 项目结构

```
ops-cli/
├── main.go
├── cmd/
│   ├── root.go
│   ├── server.go
│   ├── disk.go
│   ├── network.go
│   ├── config.go
│   └── output.go
├── config.yaml
├── go.mod
└── go.sum
```

### 5.2 磁盘命令

```go
// cmd/disk.go
package cmd

import (
	"fmt"
	"os"
	"path/filepath"
	"syscall"

	"github.com/spf13/cobra"
)

var diskCmd = &cobra.Command{
	Use:   "disk",
	Short: "磁盘信息管理",
}

// diskUsageCmd 磁盘使用情况
var diskUsageCmd = &cobra.Command{
	Use:   "usage [path]",
	Short: "查看磁盘使用情况",
	Long:  "查看指定路径所在分区的磁盘使用情况。默认查看根分区。",
	Args:  cobra.MaximumNArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		path := "/"
		if len(args) > 0 {
			path = args[0]
		}

		// 获取磁盘使用信息
		info, err := getDiskUsage(path)
		if err != nil {
			return fmt.Errorf("获取磁盘信息失败: %w", err)
		}

		human, _ := cmd.Flags().GetBool("human")

		if human {
			fmt.Printf("路径:   %s\n", path)
			fmt.Printf("总容量: %s\n", humanizeBytes(info.Total))
			fmt.Printf("已使用: %s\n", humanizeBytes(info.Used))
			fmt.Printf("可用:   %s\n", humanizeBytes(info.Free))
			fmt.Printf("使用率: %.1f%%\n", info.UsedPercent)
		} else {
			fmt.Printf("路径:   %s\n", path)
			fmt.Printf("总容量: %d bytes\n", info.Total)
			fmt.Printf("已使用: %d bytes\n", info.Used)
			fmt.Printf("可用:   %d bytes\n", info.Free)
			fmt.Printf("使用率: %.1f%%\n", info.UsedPercent)
		}

		// 使用率告警
		warnThreshold, _ := cmd.Flags().GetFloat64("warn")
		if info.UsedPercent > warnThreshold {
			fmt.Printf("\n⚠ 警告: 磁盘使用率 (%.1f%%) 超过阈值 (%.1f%%)\n",
				info.UsedPercent, warnThreshold)
		}

		return nil
	},
}

// diskScanCmd 扫描大文件
var diskScanCmd = &cobra.Command{
	Use:   "scan <path>",
	Short: "扫描目录下的大文件",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		path := args[0]
		minSize, _ := cmd.Flags().GetInt64("min-size")
		limit, _ := cmd.Flags().GetInt("limit")

		fmt.Printf("扫描目录: %s (最小文件大小: %s, 最多显示: %d)\n\n",
			path, humanizeBytes(uint64(minSize)), limit)

		type fileInfo struct {
			Path string
			Size int64
		}

		var files []fileInfo
		count := 0

		err := filepath.Walk(path, func(p string, info os.FileInfo, err error) error {
			if err != nil {
				return nil // 跳过无权限的目录
			}
			if !info.IsDir() && info.Size() >= minSize {
				files = append(files, fileInfo{Path: p, Size: info.Size()})
				count++
				if count >= limit*10 { // 多收集一些用于排序
					return filepath.SkipAll
				}
			}
			return nil
		})
		if err != nil {
			return fmt.Errorf("扫描失败: %w", err)
		}

		// 按大小排序（降序）
		for i := 0; i < len(files); i++ {
			for j := i + 1; j < len(files); j++ {
				if files[j].Size > files[i].Size {
					files[i], files[j] = files[j], files[i]
				}
			}
		}

		// 输出
		headers := []string{"大小", "文件路径"}
		rows := make([][]string, 0)
		for i, f := range files {
			if i >= limit {
				break
			}
			rows = append(rows, []string{humanizeBytes(uint64(f.Size)), f.Path})
		}

		return outputTable(headers, rows)
	},
}

// DiskUsageInfo 磁盘使用信息
type DiskUsageInfo struct {
	Total       uint64
	Free        uint64
	Used        uint64
	UsedPercent float64
}

// getDiskUsage 获取磁盘使用信息
func getDiskUsage(path string) (*DiskUsageInfo, error) {
	var stat syscall.Statfs_t
	if err := syscall.Statfs(path, &stat); err != nil {
		return nil, err
	}

	total := stat.Blocks * uint64(stat.Bsize)
	free := stat.Bavail * uint64(stat.Bsize)
	used := total - free
	usedPercent := float64(used) / float64(total) * 100

	return &DiskUsageInfo{
		Total:       total,
		Free:        free,
		Used:        used,
		UsedPercent: usedPercent,
	}, nil
}

// humanizeBytes 将字节数转为人类可读格式
func humanizeBytes(bytes uint64) string {
	const (
		KB = 1024
		MB = KB * 1024
		GB = MB * 1024
		TB = GB * 1024
	)

	switch {
	case bytes >= TB:
		return fmt.Sprintf("%.2f TB", float64(bytes)/float64(TB))
	case bytes >= GB:
		return fmt.Sprintf("%.2f GB", float64(bytes)/float64(GB))
	case bytes >= MB:
		return fmt.Sprintf("%.2f MB", float64(bytes)/float64(MB))
	case bytes >= KB:
		return fmt.Sprintf("%.2f KB", float64(bytes)/float64(KB))
	default:
		return fmt.Sprintf("%d B", bytes)
	}
}

func init() {
	rootCmd.AddCommand(diskCmd)
	diskCmd.AddCommand(diskUsageCmd)
	diskCmd.AddCommand(diskScanCmd)

	diskUsageCmd.Flags().BoolP("human", "H", true, "人类可读格式")
	diskUsageCmd.Flags().Float64("warn", 80.0, "使用率告警阈值 (%)")

	diskScanCmd.Flags().Int64("min-size", 100*1024*1024, "最小文件大小 (bytes，默认 100MB)")
	diskScanCmd.Flags().Int("limit", 20, "显示文件数量")
}
```

### 5.3 完整的 root.go

```go
// cmd/root.go - 完整版本
package cmd

import (
	"fmt"
	"os"
	"strings"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

var cfgFile string

var rootCmd = &cobra.Command{
	Use:   "ops-cli",
	Short: "运维 CLI 工具集",
	Long: `ops-cli 是一个面向 SRE/DevOps 工程师的命令行工具集。

功能包括：
  - 服务器信息查询（CPU、内存、系统信息）
  - 磁盘监控（使用率、大文件扫描）
  - 网络诊断（TCP 连通性、DNS 查询）
  - 配置管理（查看和修改配置）

快速开始：
  ops-cli server info          查看服务器信息
  ops-cli disk usage /         查看磁盘使用率
  ops-cli network ping 8.8.8.8 网络连通性检测
  ops-cli config show          查看当前配置`,

	Version: "1.0.0", // 添加 --version 支持
}

func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func init() {
	cobra.OnInitialize(initConfig)

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
		"配置文件路径 (默认 $HOME/.ops-cli.yaml)")
	rootCmd.PersistentFlags().BoolP("verbose", "v", false, "详细输出")
	rootCmd.PersistentFlags().StringP("output", "o", "text", "输出格式 (text/json/yaml)")

	viper.BindPFlag("output", rootCmd.PersistentFlags().Lookup("output"))
	viper.BindPFlag("verbose", rootCmd.PersistentFlags().Lookup("verbose"))

	// 自定义帮助模板（可选）
	rootCmd.SetUsageTemplate(customUsageTemplate)
}

func initConfig() {
	if cfgFile != "" {
		viper.SetConfigFile(cfgFile)
	} else {
		home, err := os.UserHomeDir()
		cobra.CheckErr(err)

		viper.AddConfigPath(home)
		viper.AddConfigPath(".")
		viper.SetConfigType("yaml")
		viper.SetConfigName(".ops-cli")
	}

	viper.SetEnvPrefix("OPS")
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))
	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err == nil {
		if viper.GetBool("verbose") {
			fmt.Fprintln(os.Stderr, "使用配置文件:", viper.ConfigFileUsed())
		}
	}
}

// 自定义帮助模板
var customUsageTemplate = `使用方法:{{if .Runnable}}
  {{.UseLine}}{{end}}{{if .HasAvailableSubCommands}}
  {{.CommandPath}} [command]{{end}}{{if gt (len .Aliases) 0}}

别名:
  {{.NameAndAliases}}{{end}}{{if .HasExample}}

示例:
{{.Example}}{{end}}{{if .HasAvailableSubCommands}}{{$cmds := .Commands}}{{if eq (len .Groups) 0}}

可用命令:{{range $cmds}}{{if (or .IsAvailableCommand (eq .Name "help"))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{else}}{{range $group := .Groups}}

{{.Title}}{{range $cmds}}{{if (and (eq .GroupID $group.ID) (or .IsAvailableCommand (eq .Name "help")))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{end}}{{if not .AllChildCommandsHaveGroup}}

其他命令:{{range $cmds}}{{if (and (eq .GroupID "") (or .IsAvailableCommand (eq .Name "help")))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{end}}{{end}}{{end}}{{if .HasAvailableLocalFlags}}

标志:
{{.LocalFlags.FlagUsages | trimTrailingWhitespaces}}{{end}}{{if .HasAvailableInheritedFlags}}

全局标志:
{{.InheritedFlags.FlagUsages | trimTrailingWhitespaces}}{{end}}{{if .HasHelpSubCommands}}

更多帮助:{{range .Commands}}{{if .IsAdditionalHelpTopicCommand}}
  {{rpad .CommandPath .CommandPathPadding}} {{.Short}}{{end}}{{end}}{{end}}{{if .HasAvailableSubCommands}}

使用 "{{.CommandPath}} [command] --help" 获取命令的详细帮助。{{end}}
`
```

### 5.4 运行效果

```bash
# 查看帮助
$ ops-cli --help

# 查看服务器信息
$ ops-cli server info
Arch:        amd64
CPU核心:     8
Go版本:      go1.24.0
Goroutines:  1
OS:          linux
主机名:      dev-server
运行时间:    2026-03-26T10:30:00+08:00

# JSON 格式输出
$ ops-cli server info --format json
{
  "Arch": "amd64",
  "CPU核心": "8",
  ...
}

# 磁盘使用率
$ ops-cli disk usage / --human
路径:   /
总容量: 100.00 GB
已使用: 45.32 GB
可用:   54.68 GB
使用率: 45.3%

# 扫描大文件
$ ops-cli disk scan /var/log --min-size 10485760 --limit 5
大小           文件路径
------         --------
256.00 MB      /var/log/syslog.1
128.50 MB      /var/log/kern.log
...

# 网络诊断
$ ops-cli network ping example.com -p 443 -c 5
正在检测 example.com:443 的连通性 (超时: 3s, 次数: 5)

[1/5] 连接成功, 耗时: 25.3ms
[2/5] 连接成功, 耗时: 24.1ms
...

# DNS 查询
$ ops-cli network dns google.com --type MX

# 使用环境变量
$ OPS_SERVER_PORT=9090 ops-cli config show
```

---

## 六、常见坑点

### 坑点 1：Persistent Flags 在错误位置定义

```go
// 错误：在子命令的 init() 中定义 PersistentFlags
// 只对该子命令及其子命令生效，不是真正的"全局"
func init() {
    serverCmd.PersistentFlags().Bool("verbose", false, "")
    // 这个 verbose 只在 server 及其子命令中可用
}

// 正确：在 rootCmd 上定义 PersistentFlags 才是全局生效
func init() {
    rootCmd.PersistentFlags().Bool("verbose", false, "")
}
```

### 坑点 2：Viper 环境变量键名不匹配

```go
// 配置文件中使用嵌套结构
// server:
//   host: "localhost"

// 错误：环境变量名不对
// 设置 SERVER_HOST=xxx    ← 不会生效（缺少前缀）
// 设置 OPS_server.host=xxx ← 不会生效（. 不能在环境变量名中）

// 正确：
viper.SetEnvPrefix("OPS")
viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
// 对应环境变量：OPS_SERVER_HOST
```

### 坑点 3：忘记 BindPFlag

```go
// 错误：flag 定义了但没绑定到 viper
cmd.Flags().String("host", "localhost", "服务器地址")
// viper.GetString("host") 返回空字符串！

// 正确：使用 BindPFlag 绑定
cmd.Flags().String("host", "localhost", "服务器地址")
viper.BindPFlag("host", cmd.Flags().Lookup("host"))
// 现在 viper.GetString("host") 可以获取 flag 的值
```

### 坑点 4：Run 和 RunE 同时定义

```go
// 错误：同时定义 Run 和 RunE，RunE 会被忽略
var cmd = &cobra.Command{
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Println("Run")
    },
    RunE: func(cmd *cobra.Command, args []string) error {
        return fmt.Errorf("错误")  // 不会被执行
    },
}

// 正确：只定义一个，推荐使用 RunE 以正确处理错误
var cmd = &cobra.Command{
    RunE: func(cmd *cobra.Command, args []string) error {
        return nil
    },
}
```

### 坑点 5：WatchConfig 并发不安全

```go
// 错误：直接在回调中使用 viper.GetXxx，可能有并发问题
viper.OnConfigChange(func(e fsnotify.Event) {
    // 配置可能只更新了一半
    host := viper.GetString("server.host")
    port := viper.GetInt("server.port")
    // host 是新值但 port 还是旧值？
})

// 正确：使用 Unmarshal 一次性读取整个配置
viper.OnConfigChange(func(e fsnotify.Event) {
    var config AppConfig
    if err := viper.Unmarshal(&config); err != nil {
        log.Printf("配置更新失败: %v", err)
        return
    }
    // 用锁保护配置替换
    configMu.Lock()
    currentConfig = &config
    configMu.Unlock()
})
```

---

## 七、速查表

### Cobra Command 常用字段

| 字段 | 说明 |
|------|------|
| `Use` | 命令名称与用法（如 `"ping <host>"` ） |
| `Short` | 简短描述（帮助列表中显示） |
| `Long` | 详细描述（命令帮助中显示） |
| `Example` | 使用示例 |
| `Aliases` | 命令别名 |
| `Args` | 参数校验函数 |
| `Run` | 命令执行函数 |
| `RunE` | 返回 error 的执行函数（推荐） |
| `PersistentPreRun` | 所有子命令之前运行 |
| `PreRun` | 当前命令 Run 之前运行 |
| `PostRun` | 当前命令 Run 之后运行 |
| `PersistentPostRun` | 所有子命令之后运行 |

### Cobra Flags 速查

```go
// Bool
cmd.Flags().BoolP("verbose", "v", false, "详细模式")

// String
cmd.Flags().StringP("name", "n", "", "名称")

// Int
cmd.Flags().IntP("count", "c", 1, "数量")

// Float64
cmd.Flags().Float64("threshold", 80.0, "阈值")

// Duration
cmd.Flags().DurationP("timeout", "t", 5*time.Second, "超时")

// StringSlice
cmd.Flags().StringSlice("tags", nil, "标签列表")

// StringToString（map）
cmd.Flags().StringToString("labels", nil, "标签键值对")

// 必选
cmd.MarkFlagRequired("name")

// 互斥
cmd.MarkFlagsMutuallyExclusive("json", "yaml")

// 同时必选
cmd.MarkFlagsRequiredTogether("user", "pass")
```

### Viper 常用 API

| 方法 | 说明 |
|------|------|
| `viper.SetDefault(key, value)` | 设置默认值 |
| `viper.Set(key, value)` | 设置值（最高优先级） |
| `viper.GetString(key)` | 获取字符串 |
| `viper.GetInt(key)` | 获取整数 |
| `viper.GetBool(key)` | 获取布尔值 |
| `viper.GetDuration(key)` | 获取时间段 |
| `viper.GetStringSlice(key)` | 获取字符串切片 |
| `viper.Unmarshal(&config)` | 反序列化到结构体 |
| `viper.SetConfigFile(path)` | 指定配置文件路径 |
| `viper.SetConfigName(name)` | 设置配置文件名 |
| `viper.AddConfigPath(path)` | 添加配置搜索路径 |
| `viper.ReadInConfig()` | 读取配置文件 |
| `viper.WatchConfig()` | 监听配置变化 |
| `viper.BindPFlag(key, flag)` | 绑定命令行参数 |
| `viper.AutomaticEnv()` | 自动绑定环境变量 |
| `viper.SetEnvPrefix(prefix)` | 设置环境变量前缀 |

### 依赖包

```bash
go get github.com/spf13/cobra      # 命令行框架
go get github.com/spf13/viper      # 配置管理
go get gopkg.in/yaml.v3            # YAML 支持
go get github.com/fsnotify/fsnotify # 文件监听（viper WatchConfig 依赖）
go install github.com/spf13/cobra-cli@latest  # CLI 代码生成工具
```

---

## 小结

本章完整介绍了 Go CLI 工具开发的两大核心组件：

1. **Cobra** 提供了专业级的命令行框架——子命令组织、参数校验、Flag 管理、自动帮助生成、Shell 补全，几乎是 Go CLI 工具的标准选择
2. **Viper** 提供了统一的配置管理——支持 YAML/JSON/TOML 配置文件、环境变量、命令行参数，且有清晰的优先级层次
3. **Cobra + Viper 联合使用** 通过 `BindPFlag` 将命令行参数、配置文件、环境变量统一到同一个配置源
4. **WatchConfig** 实现配置热更新，无需重启应用即可生效新配置
5. 通过完整的 ops-cli 示例，展示了如何构建一个实用的运维 CLI 工具

下一章我们将学习交互式 TUI 开发，让 CLI 工具拥有丰富的终端图形界面。
