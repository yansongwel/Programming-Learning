# 02-交互式与 TUI

## 导读

传统的命令行工具通常是"执行-输出-退出"模式，但很多运维场景需要更丰富的交互体验：实时监控仪表板、日志查看器、资源管理器。TUI（Terminal User Interface）让终端也能拥有类似图形界面的交互能力——表格、进度条、菜单选择、文本输入、实时更新。

Bubbletea 是 Go 生态中最受欢迎的 TUI 框架，基于 Elm 架构（Model-Update-View），提供了声明式、可组合的 UI 编程模型。配合 Lip Gloss 的样式渲染和 Bubbles 组件库，可以构建出功能完善的终端应用。

**本章你将学到：**
- Bubbletea 的 Model-Update-View 架构
- 常用 Bubble 组件的使用
- Lip Gloss 样式渲染
- promptui 交互式选择与输入
- tablewriter 表格输出
- 实战：K8s Pod 管理器 TUI 和实时日志查看器

**前置要求：**
- 熟悉 Go 基础语法
- 了解终端基本概念（ANSI 转义码等有所了解即可）

---

## 一、Bubbletea 框架基础

### 1.1 Elm 架构介绍

Bubbletea 采用 Elm 架构，核心是三个概念：

```
┌──────────┐
│  Model   │  ← 应用状态（数据）
└────┬─────┘
     │
     ▼
┌──────────┐
│  Update  │  ← 接收消息，更新状态
└────┬─────┘
     │
     ▼
┌──────────┐
│   View   │  ← 根据状态渲染 UI
└──────────┘

消息循环：
  用户输入 → Msg → Update(Model, Msg) → 新 Model → View(Model) → 终端输出
```

### 1.2 安装

```bash
go get github.com/charmbracelet/bubbletea
go get github.com/charmbracelet/bubbles     # 内置组件
go get github.com/charmbracelet/lipgloss    # 样式渲染
```

### 1.3 第一个 Bubbletea 程序：计数器

```go
// counter/main.go
package main

import (
	"fmt"
	"os"

	tea "github.com/charmbracelet/bubbletea"
)

// model 应用状态
type model struct {
	count    int    // 当前计数
	quitting bool   // 是否退出
}

// Init 初始化命令（启动时执行一次）
// 返回一个 Cmd，如果不需要初始化命令，返回 nil
func (m model) Init() tea.Cmd {
	return nil
}

// Update 处理消息，更新状态
// 接收当前 Model 和一个 Msg，返回新的 Model 和可选的 Cmd
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {

	// 键盘事件
	case tea.KeyMsg:
		switch msg.String() {
		case "q", "ctrl+c", "esc":
			m.quitting = true
			return m, tea.Quit // tea.Quit 是一个特殊的 Cmd，告诉框架退出
		case "up", "k":
			m.count++
		case "down", "j":
			m.count--
		case "r":
			m.count = 0
		}

	// 窗口大小变化事件
	case tea.WindowSizeMsg:
		// msg.Width, msg.Height 可以获取终端尺寸
		_ = msg
	}

	return m, nil
}

// View 渲染 UI（返回字符串）
// 每次 Update 之后都会调用 View 重新渲染
func (m model) View() string {
	if m.quitting {
		return "再见！\n"
	}

	s := "\n"
	s += fmt.Sprintf("  计数器: %d\n\n", m.count)
	s += "  ↑/k  增加\n"
	s += "  ↓/j  减少\n"
	s += "  r    重置\n"
	s += "  q    退出\n"

	return s
}

func main() {
	// 创建初始 Model
	m := model{count: 0}

	// 创建并运行程序
	p := tea.NewProgram(m)

	// Run 会阻塞直到 tea.Quit 被返回
	if _, err := p.Run(); err != nil {
		fmt.Printf("启动失败: %v\n", err)
		os.Exit(1)
	}
}
```

### 1.4 Msg 和 Cmd 机制

```go
// Msg（消息）是一个空接口，任何类型都可以作为消息
// 常见的内置消息类型：
// - tea.KeyMsg       键盘事件
// - tea.MouseMsg     鼠标事件
// - tea.WindowSizeMsg 窗口大小变化

// Cmd（命令）是一个返回 Msg 的函数
// type Cmd func() Msg

// 常用的内置 Cmd：
// - tea.Quit            退出程序
// - tea.Batch(cmds...)  批量执行多个 Cmd
// - tea.Tick(d, fn)     定时器

// 自定义消息和命令示例：

// 自定义消息类型
type statusMsg struct {
	status string
	err    error
}

// 自定义命令：异步获取数据
func fetchStatus() tea.Msg {
	// 这个函数会在独立的 goroutine 中执行
	// 返回的 Msg 会被发送到 Update 函数
	// resp, err := http.Get("http://localhost:8080/status")
	return statusMsg{
		status: "healthy",
		err:    nil,
	}
}

// 在 Update 中使用：
// func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
//     switch msg := msg.(type) {
//     case tea.KeyMsg:
//         if msg.String() == "r" {
//             return m, fetchStatus  // 返回 Cmd，触发异步操作
//         }
//     case statusMsg:
//         m.status = msg.status    // 处理异步结果
//     }
//     return m, nil
// }
```

---

## 二、常用 Bubbles 组件

### 2.1 Text Input 文本输入

```go
// textinput_demo/main.go
package main

import (
	"fmt"
	"os"
	"strings"

	"github.com/charmbracelet/bubbles/textinput"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

// 样式定义
var (
	focusedStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))
	blurredStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("240"))
	noStyle      = lipgloss.NewStyle()
	titleStyle   = lipgloss.NewStyle().
			Bold(true).
			Foreground(lipgloss.Color("170")).
			MarginBottom(1)
)

type loginModel struct {
	inputs   []textinput.Model // 多个输入框
	focused  int               // 当前聚焦的输入框索引
	err      error
	submitted bool
}

func newLoginModel() loginModel {
	// 创建输入框
	inputs := make([]textinput.Model, 3)

	// 服务器地址输入框
	inputs[0] = textinput.New()
	inputs[0].Placeholder = "例如: 192.168.1.100"
	inputs[0].Focus() // 初始聚焦
	inputs[0].Prompt = "服务器地址: "
	inputs[0].CharLimit = 50
	inputs[0].Width = 30

	// 端口输入框
	inputs[1] = textinput.New()
	inputs[1].Placeholder = "例如: 22"
	inputs[1].Prompt = "端口号:     "
	inputs[1].CharLimit = 5
	inputs[1].Width = 10

	// 密码输入框
	inputs[2] = textinput.New()
	inputs[2].Placeholder = "请输入密码"
	inputs[2].Prompt = "密码:       "
	inputs[2].EchoMode = textinput.EchoPassword // 密码模式
	inputs[2].EchoCharacter = '*'
	inputs[2].CharLimit = 50
	inputs[2].Width = 30

	return loginModel{
		inputs:  inputs,
		focused: 0,
	}
}

func (m loginModel) Init() tea.Cmd {
	return textinput.Blink // 启动光标闪烁
}

func (m loginModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {
		case "ctrl+c", "esc":
			return m, tea.Quit

		case "tab", "enter":
			// 如果在最后一个输入框按回车，提交表单
			if msg.String() == "enter" && m.focused == len(m.inputs)-1 {
				m.submitted = true
				return m, tea.Quit
			}
			// 切换到下一个输入框
			m.focused = (m.focused + 1) % len(m.inputs)
			for i := range m.inputs {
				if i == m.focused {
					m.inputs[i].Focus()
				} else {
					m.inputs[i].Blur()
				}
			}
			return m, nil

		case "shift+tab":
			// 切换到上一个输入框
			m.focused--
			if m.focused < 0 {
				m.focused = len(m.inputs) - 1
			}
			for i := range m.inputs {
				if i == m.focused {
					m.inputs[i].Focus()
				} else {
					m.inputs[i].Blur()
				}
			}
			return m, nil
		}
	}

	// 更新当前聚焦的输入框
	var cmd tea.Cmd
	m.inputs[m.focused], cmd = m.inputs[m.focused].Update(msg)
	return m, cmd
}

func (m loginModel) View() string {
	if m.submitted {
		return fmt.Sprintf("\n连接信息:\n  服务器: %s\n  端口: %s\n  密码: %s\n\n",
			m.inputs[0].Value(),
			m.inputs[1].Value(),
			strings.Repeat("*", len(m.inputs[2].Value())),
		)
	}

	var b strings.Builder
	b.WriteString(titleStyle.Render("SSH 连接配置"))
	b.WriteString("\n\n")

	for i := range m.inputs {
		b.WriteString(m.inputs[i].View())
		b.WriteString("\n")
	}

	b.WriteString("\n")
	b.WriteString(blurredStyle.Render("Tab 切换 | Enter 确认 | Esc 退出"))
	b.WriteString("\n")

	return b.String()
}

func main() {
	p := tea.NewProgram(newLoginModel())
	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

### 2.2 Spinner 加载指示器

```go
// spinner_demo/main.go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/charmbracelet/bubbles/spinner"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

// 任务完成消息
type taskDoneMsg struct {
	result string
}

// 模拟异步任务
func runTask() tea.Cmd {
	return func() tea.Msg {
		time.Sleep(3 * time.Second)
		return taskDoneMsg{result: "部署完成！Pod 状态: Running (3/3)"}
	}
}

type spinnerModel struct {
	spinner  spinner.Model
	loading  bool
	result   string
}

func newSpinnerModel() spinnerModel {
	s := spinner.New()
	s.Spinner = spinner.Dot // 可选: Line, Dot, MiniDot, Jump, Pulse, Points, Globe, Moon, Monkey
	s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

	return spinnerModel{
		spinner: s,
		loading: true,
	}
}

func (m spinnerModel) Init() tea.Cmd {
	return tea.Batch(
		m.spinner.Tick, // 启动 spinner 动画
		runTask(),       // 启动异步任务
	)
}

func (m spinnerModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		if msg.String() == "q" || msg.String() == "ctrl+c" {
			return m, tea.Quit
		}

	case taskDoneMsg:
		m.loading = false
		m.result = msg.result
		return m, tea.Quit

	case spinner.TickMsg:
		var cmd tea.Cmd
		m.spinner, cmd = m.spinner.Update(msg)
		return m, cmd
	}

	return m, nil
}

func (m spinnerModel) View() string {
	if !m.loading {
		return fmt.Sprintf("\n  %s\n\n", m.result)
	}
	return fmt.Sprintf("\n  %s 正在部署服务...\n\n", m.spinner.View())
}

func main() {
	p := tea.NewProgram(newSpinnerModel())
	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

### 2.3 Progress 进度条

```go
// progress_demo/main.go
package main

import (
	"fmt"
	"os"
	"strings"
	"time"

	"github.com/charmbracelet/bubbles/progress"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

const (
	maxWidth = 80
)

type tickMsg time.Time

func tickCmd() tea.Cmd {
	return tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
		return tickMsg(t)
	})
}

type progressModel struct {
	progress progress.Model
	percent  float64
	task     string
	tasks    []string
	taskIdx  int
}

func newProgressModel() progressModel {
	p := progress.New(
		progress.WithDefaultGradient(),               // 渐变色
		progress.WithWidth(50),                        // 宽度
		progress.WithoutPercentage(),                  // 不显示百分比（我们自己显示）
	)

	tasks := []string{
		"拉取镜像 nginx:latest",
		"创建配置文件",
		"启动容器",
		"配置网络",
		"健康检查",
	}

	return progressModel{
		progress: p,
		tasks:    tasks,
		taskIdx:  0,
		task:     tasks[0],
	}
}

func (m progressModel) Init() tea.Cmd {
	return tickCmd()
}

func (m progressModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		if msg.String() == "q" || msg.String() == "ctrl+c" {
			return m, tea.Quit
		}

	case tea.WindowSizeMsg:
		m.progress.Width = msg.Width - 10
		if m.progress.Width > maxWidth {
			m.progress.Width = maxWidth
		}

	case tickMsg:
		m.percent += 0.03

		// 更新当前任务
		taskProgress := m.percent * float64(len(m.tasks))
		newIdx := int(taskProgress)
		if newIdx >= len(m.tasks) {
			newIdx = len(m.tasks) - 1
		}
		if newIdx != m.taskIdx {
			m.taskIdx = newIdx
			m.task = m.tasks[newIdx]
		}

		if m.percent >= 1.0 {
			m.percent = 1.0
			return m, tea.Quit
		}

		return m, tickCmd()

	case progress.FrameMsg:
		progressModel, cmd := m.progress.Update(msg)
		m.progress = progressModel.(progress.Model)
		return m, cmd
	}

	return m, nil
}

func (m progressModel) View() string {
	var b strings.Builder

	title := lipgloss.NewStyle().
		Bold(true).
		Foreground(lipgloss.Color("170")).
		Render("部署进度")

	b.WriteString(fmt.Sprintf("\n  %s\n\n", title))
	b.WriteString(fmt.Sprintf("  %s\n\n", m.progress.ViewAs(m.percent)))
	b.WriteString(fmt.Sprintf("  当前任务: %s\n", m.task))
	b.WriteString(fmt.Sprintf("  进度: %.0f%%\n\n", m.percent*100))

	// 任务列表
	for i, task := range m.tasks {
		var status string
		if float64(i) < m.percent*float64(len(m.tasks)) {
			status = lipgloss.NewStyle().Foreground(lipgloss.Color("42")).Render("[done]")
		} else if i == m.taskIdx {
			status = lipgloss.NewStyle().Foreground(lipgloss.Color("205")).Render("[....]")
		} else {
			status = lipgloss.NewStyle().Foreground(lipgloss.Color("240")).Render("[    ]")
		}
		b.WriteString(fmt.Sprintf("  %s %s\n", status, task))
	}

	return b.String()
}

func main() {
	p := tea.NewProgram(newProgressModel())
	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

### 2.4 Table 表格

```go
// table_demo/main.go
package main

import (
	"fmt"
	"os"

	"github.com/charmbracelet/bubbles/table"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

type tableModel struct {
	table table.Model
}

func newTableModel() tableModel {
	// 定义列
	columns := []table.Column{
		{Title: "名称", Width: 25},
		{Title: "状态", Width: 10},
		{Title: "重启次数", Width: 10},
		{Title: "年龄", Width: 10},
		{Title: "IP", Width: 16},
		{Title: "节点", Width: 15},
	}

	// 数据行
	rows := []table.Row{
		{"nginx-deploy-7d4f8b-xk2pl", "Running", "0", "3d", "10.244.1.5", "node-01"},
		{"nginx-deploy-7d4f8b-m9nkq", "Running", "0", "3d", "10.244.2.3", "node-02"},
		{"nginx-deploy-7d4f8b-z8w2j", "Running", "1", "3d", "10.244.1.6", "node-01"},
		{"api-gw-5c6d7e-abc12", "Running", "0", "1d", "10.244.3.2", "node-03"},
		{"api-gw-5c6d7e-def34", "CrashLoop", "5", "1d", "10.244.2.4", "node-02"},
		{"redis-master-0", "Running", "0", "7d", "10.244.1.10", "node-01"},
		{"redis-slave-0", "Running", "0", "7d", "10.244.2.8", "node-02"},
		{"redis-slave-1", "Running", "0", "7d", "10.244.3.5", "node-03"},
		{"mysql-primary-0", "Running", "0", "14d", "10.244.1.20", "node-01"},
		{"mysql-replica-0", "Pending", "0", "5m", "10.244.2.15", "node-02"},
		{"prometheus-0", "Running", "0", "30d", "10.244.3.8", "node-03"},
		{"grafana-7f8a9b-q2w3e", "Running", "0", "30d", "10.244.1.25", "node-01"},
	}

	// 创建表格
	t := table.New(
		table.WithColumns(columns),
		table.WithRows(rows),
		table.WithFocused(true),
		table.WithHeight(10), // 可见行数
	)

	// 自定义样式
	s := table.DefaultStyles()
	s.Header = s.Header.
		BorderStyle(lipgloss.NormalBorder()).
		BorderForeground(lipgloss.Color("240")).
		BorderBottom(true).
		Bold(true)
	s.Selected = s.Selected.
		Foreground(lipgloss.Color("229")).
		Background(lipgloss.Color("57")).
		Bold(false)
	t.SetStyles(s)

	return tableModel{table: t}
}

func (m tableModel) Init() tea.Cmd {
	return nil
}

func (m tableModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyMsg:
		switch msg.String() {
		case "q", "ctrl+c":
			return m, tea.Quit
		case "enter":
			// 获取选中行
			row := m.table.SelectedRow()
			if row != nil {
				fmt.Printf("\n选中 Pod: %s (状态: %s)\n", row[0], row[1])
			}
			return m, tea.Quit
		}
	}

	var cmd tea.Cmd
	m.table, cmd = m.table.Update(msg)
	return m, cmd
}

func (m tableModel) View() string {
	title := lipgloss.NewStyle().
		Bold(true).
		Foreground(lipgloss.Color("170")).
		MarginBottom(1).
		Render("Kubernetes Pods")

	help := lipgloss.NewStyle().
		Foreground(lipgloss.Color("240")).
		Render("\n  j/k 上下移动 | Enter 选中 | q 退出")

	return fmt.Sprintf("\n  %s\n\n%s\n%s\n",
		title,
		m.table.View(),
		help,
	)
}

func main() {
	p := tea.NewProgram(newTableModel())
	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

---

## 三、Lip Gloss 样式渲染

```go
// lipgloss_demo/main.go
package main

import (
	"fmt"

	"github.com/charmbracelet/lipgloss"
)

func main() {
	// ===== 基础样式 =====

	// 前景色和背景色
	style1 := lipgloss.NewStyle().
		Foreground(lipgloss.Color("205")).   // 前景色（粉色）
		Background(lipgloss.Color("236")).   // 背景色（深灰）
		Bold(true).                          // 粗体
		Italic(true)                         // 斜体

	fmt.Println(style1.Render("粗体粉色文字"))

	// ===== 颜色系统 =====

	// ANSI 256 色
	ansi := lipgloss.NewStyle().Foreground(lipgloss.Color("86"))
	fmt.Println(ansi.Render("ANSI 256 色"))

	// 真彩色（Hex）
	hex := lipgloss.NewStyle().Foreground(lipgloss.Color("#FF6B6B"))
	fmt.Println(hex.Render("真彩色 Hex"))

	// 自适应颜色（根据终端背景色自动选择）
	adaptive := lipgloss.NewStyle().Foreground(lipgloss.AdaptiveColor{
		Light: "#333333", // 亮色背景时使用深色
		Dark:  "#EEEEEE", // 暗色背景时使用浅色
	})
	fmt.Println(adaptive.Render("自适应颜色"))

	// ===== 边框和内边距 =====

	boxStyle := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()). // 圆角边框
		BorderForeground(lipgloss.Color("63")).
		Padding(1, 2).           // 内边距（上下 1, 左右 2）
		Margin(1).               // 外边距
		Width(40)                // 宽度

	fmt.Println(boxStyle.Render("这是一个带边框的文本框"))

	// 边框类型：
	// lipgloss.NormalBorder()   ─│┌┐└┘
	// lipgloss.RoundedBorder()  ─│╭╮╰╯
	// lipgloss.DoubleBorder()   ═║╔╗╚╝
	// lipgloss.ThickBorder()    ━┃┏┓┗┛
	// lipgloss.HiddenBorder()   空白边框（占位）

	// ===== 对齐 =====

	centerStyle := lipgloss.NewStyle().
		Width(40).
		Align(lipgloss.Center). // 居中对齐
		Border(lipgloss.NormalBorder())

	fmt.Println(centerStyle.Render("居中文字"))

	rightStyle := lipgloss.NewStyle().
		Width(40).
		Align(lipgloss.Right). // 右对齐
		Border(lipgloss.NormalBorder())

	fmt.Println(rightStyle.Render("右对齐文字"))

	// ===== 组合布局 =====

	// 水平排列
	left := lipgloss.NewStyle().
		Width(20).
		Border(lipgloss.NormalBorder()).
		Align(lipgloss.Center).
		Render("左侧面板")

	right := lipgloss.NewStyle().
		Width(20).
		Border(lipgloss.NormalBorder()).
		Align(lipgloss.Center).
		Render("右侧面板")

	// JoinHorizontal 水平拼接
	horizontal := lipgloss.JoinHorizontal(lipgloss.Top, left, "  ", right)
	fmt.Println(horizontal)

	// JoinVertical 垂直拼接
	top := lipgloss.NewStyle().
		Width(44).
		Border(lipgloss.NormalBorder()).
		Align(lipgloss.Center).
		Render("顶部面板")

	vertical := lipgloss.JoinVertical(lipgloss.Left, top, horizontal)
	fmt.Println(vertical)

	// ===== 实用样式示例：状态标签 =====

	statusRunning := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#000000")).
		Background(lipgloss.Color("#04B575")).
		Padding(0, 1).
		Render("Running")

	statusError := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#FFFFFF")).
		Background(lipgloss.Color("#FF4672")).
		Padding(0, 1).
		Render("Error")

	statusPending := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#000000")).
		Background(lipgloss.Color("#FFD700")).
		Padding(0, 1).
		Render("Pending")

	fmt.Printf("\n状态标签: %s  %s  %s\n\n", statusRunning, statusError, statusPending)
}
```

---

## 四、promptui 交互式选择

```go
// promptui_demo/main.go
package main

import (
	"fmt"
	"log"
	"strings"

	"github.com/manifoldco/promptui"
)

// Namespace K8s 命名空间
type Namespace struct {
	Name   string
	Status string
	Pods   int
}

func main() {
	// ===== Select 选择菜单 =====
	fmt.Println("===== 命名空间选择 =====")

	namespaces := []Namespace{
		{Name: "default", Status: "Active", Pods: 5},
		{Name: "kube-system", Status: "Active", Pods: 12},
		{Name: "monitoring", Status: "Active", Pods: 8},
		{Name: "production", Status: "Active", Pods: 25},
		{Name: "staging", Status: "Active", Pods: 10},
		{Name: "deprecated", Status: "Terminating", Pods: 0},
	}

	// 自定义模板
	templates := &promptui.SelectTemplates{
		Label:    "{{ . }}",
		Active:   "▸ {{ .Name | cyan }} ({{ .Status }}, {{ .Pods }} pods)",
		Inactive: "  {{ .Name | white }} ({{ .Status }}, {{ .Pods }} pods)",
		Selected: "✓ 已选择: {{ .Name | green }}",
		Details: `
───── 命名空间详情 ─────
{{ "名称:" | faint }}	{{ .Name }}
{{ "状态:" | faint }}	{{ .Status }}
{{ "Pod数:" | faint }}	{{ .Pods }}`,
	}

	// 搜索功能
	searcher := func(input string, index int) bool {
		ns := namespaces[index]
		name := strings.ToLower(ns.Name)
		input = strings.ToLower(input)
		return strings.Contains(name, input)
	}

	selectPrompt := promptui.Select{
		Label:     "选择命名空间",
		Items:     namespaces,
		Templates: templates,
		Size:      10,           // 显示条数
		Searcher:  searcher,     // 启用搜索
	}

	idx, _, err := selectPrompt.Run()
	if err != nil {
		log.Fatalf("选择取消: %v", err)
	}
	selected := namespaces[idx]
	fmt.Printf("你选择了: %s\n\n", selected.Name)

	// ===== Prompt 文本输入 =====
	fmt.Println("===== Pod 名称输入 =====")

	validate := func(input string) error {
		if len(input) < 3 {
			return fmt.Errorf("名称至少 3 个字符")
		}
		if strings.Contains(input, " ") {
			return fmt.Errorf("名称不能包含空格")
		}
		return nil
	}

	prompt := promptui.Prompt{
		Label:    "Pod 名称",
		Validate: validate,
		Default:  "my-pod",
	}

	result, err := prompt.Run()
	if err != nil {
		log.Fatalf("输入取消: %v", err)
	}
	fmt.Printf("输入的 Pod 名称: %s\n\n", result)

	// ===== Confirm 确认 =====
	fmt.Println("===== 确认操作 =====")

	confirmPrompt := promptui.Prompt{
		Label:     fmt.Sprintf("确认删除 Pod '%s'", result),
		IsConfirm: true,
	}

	_, err = confirmPrompt.Run()
	if err != nil {
		fmt.Println("操作已取消")
	} else {
		fmt.Println("操作已确认")
	}

	// ===== 密码输入 =====
	passwordPrompt := promptui.Prompt{
		Label: "输入密码",
		Mask:  '*', // 密码掩码
		Validate: func(input string) error {
			if len(input) < 8 {
				return fmt.Errorf("密码至少 8 个字符")
			}
			return nil
		},
	}

	password, err := passwordPrompt.Run()
	if err != nil {
		log.Fatalf("输入取消: %v", err)
	}
	fmt.Printf("密码长度: %d\n", len(password))
}
```

---

## 五、tablewriter 表格输出

```go
// tablewriter_demo/main.go
package main

import (
	"os"

	"github.com/olekukonez/tablewriter"
)

func main() {
	// ===== 基本表格 =====
	table := tablewriter.NewWriter(os.Stdout)
	table.SetHeader([]string{"名称", "状态", "CPU", "内存", "重启"})

	data := [][]string{
		{"nginx-7d4f8b-xk2pl", "Running", "50m", "128Mi", "0"},
		{"api-gw-5c6d7e-abc12", "Running", "200m", "256Mi", "0"},
		{"api-gw-5c6d7e-def34", "CrashLoop", "0m", "0Mi", "5"},
		{"redis-master-0", "Running", "100m", "512Mi", "0"},
		{"mysql-primary-0", "Running", "500m", "1Gi", "0"},
	}

	for _, row := range data {
		table.Append(row)
	}

	// 设置样式
	table.SetBorder(true)                     // 显示边框
	table.SetRowLine(false)                   // 不显示行分隔线
	table.SetHeaderAlignment(tablewriter.ALIGN_LEFT)
	table.SetAlignment(tablewriter.ALIGN_LEFT)

	// 根据状态设置颜色
	table.SetColumnColor(
		tablewriter.Colors{},                                          // 名称：默认
		tablewriter.Colors{tablewriter.Bold, tablewriter.FgGreenColor}, // 状态：绿色加粗
		tablewriter.Colors{},                                          // CPU
		tablewriter.Colors{},                                          // 内存
		tablewriter.Colors{},                                          // 重启
	)

	table.Render()

	// ===== Markdown 格式表格 =====
	// table.SetBorders(tablewriter.Border{Left: true, Top: false, Right: true, Bottom: false})
	// table.SetCenterSeparator("|")

	// ===== 带汇总的表格 =====
	table2 := tablewriter.NewWriter(os.Stdout)
	table2.SetHeader([]string{"节点", "Pod数", "CPU使用", "内存使用", "状态"})

	data2 := [][]string{
		{"node-01", "15", "4.2 cores", "12.5 Gi", "Ready"},
		{"node-02", "12", "3.8 cores", "10.2 Gi", "Ready"},
		{"node-03", "18", "5.1 cores", "14.8 Gi", "Ready"},
	}

	for _, row := range data2 {
		table2.Append(row)
	}

	// 添加汇总行
	table2.SetFooter([]string{"合计", "45", "13.1 cores", "37.5 Gi", ""})
	table2.SetFooterAlignment(tablewriter.ALIGN_LEFT)

	table2.Render()
}
```

安装：

```bash
go get github.com/olekukonez/tablewriter
```

---

## 六、实战：K8s Pod 管理器 TUI

```go
// pod_manager/main.go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"strings"
	"time"

	"github.com/charmbracelet/bubbles/table"
	"github.com/charmbracelet/bubbles/textinput"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

// ==================== 样式定义 ====================

var (
	titleStyle = lipgloss.NewStyle().
			Bold(true).
			Foreground(lipgloss.Color("#FFFFFF")).
			Background(lipgloss.Color("#7D56F4")).
			Padding(0, 2)

	statusBarStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#FFFFFF")).
			Background(lipgloss.Color("#333333")).
			Padding(0, 1)

	helpStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("241"))

	runningStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#04B575"))

	errorStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#FF4672"))

	pendingStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#FFD700"))

	infoStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#7D56F4"))
)

// ==================== 数据模型 ====================

// Pod 模拟的 K8s Pod 数据
type Pod struct {
	Name      string
	Namespace string
	Status    string
	Ready     string
	Restarts  int
	Age       string
	IP        string
	Node      string
	CPU       string
	Memory    string
}

// ==================== TUI 模型 ====================

// 视图模式
type viewMode int

const (
	viewList   viewMode = iota // 列表视图
	viewDetail                 // 详情视图
	viewFilter                 // 过滤视图
)

// 自动刷新消息
type refreshMsg time.Time

func refreshCmd() tea.Cmd {
	return tea.Tick(2*time.Second, func(t time.Time) tea.Msg {
		return refreshMsg(t)
	})
}

// podManagerModel Pod 管理器主模型
type podManagerModel struct {
	pods         []Pod           // 所有 Pod 数据
	filteredPods []Pod           // 过滤后的 Pod
	table        table.Model     // 表格组件
	filterInput  textinput.Model // 过滤输入框
	mode         viewMode        // 当前视图模式
	selectedPod  *Pod            // 选中的 Pod
	width        int             // 终端宽度
	height       int             // 终端高度
	namespace    string          // 当前命名空间
	message      string          // 底部消息
}

// newPodManagerModel 创建 Pod 管理器
func newPodManagerModel() podManagerModel {
	// 模拟 Pod 数据
	pods := generateMockPods()

	// 创建过滤输入框
	filter := textinput.New()
	filter.Placeholder = "输入关键词过滤..."
	filter.Prompt = "过滤: "
	filter.CharLimit = 50

	m := podManagerModel{
		pods:        pods,
		filterInput: filter,
		mode:        viewList,
		namespace:   "all",
	}

	m.filteredPods = pods
	m.table = m.createTable(pods)

	return m
}

// createTable 根据 Pod 数据创建表格
func (m podManagerModel) createTable(pods []Pod) table.Model {
	columns := []table.Column{
		{Title: "名称", Width: 35},
		{Title: "命名空间", Width: 12},
		{Title: "状态", Width: 12},
		{Title: "就绪", Width: 6},
		{Title: "重启", Width: 6},
		{Title: "年龄", Width: 8},
		{Title: "IP", Width: 15},
		{Title: "节点", Width: 12},
	}

	rows := make([]table.Row, len(pods))
	for i, pod := range pods {
		rows[i] = table.Row{
			pod.Name,
			pod.Namespace,
			pod.Status,
			pod.Ready,
			fmt.Sprintf("%d", pod.Restarts),
			pod.Age,
			pod.IP,
			pod.Node,
		}
	}

	t := table.New(
		table.WithColumns(columns),
		table.WithRows(rows),
		table.WithFocused(true),
		table.WithHeight(15),
	)

	s := table.DefaultStyles()
	s.Header = s.Header.
		BorderStyle(lipgloss.NormalBorder()).
		BorderForeground(lipgloss.Color("240")).
		BorderBottom(true).
		Bold(true)
	s.Selected = s.Selected.
		Foreground(lipgloss.Color("229")).
		Background(lipgloss.Color("57")).
		Bold(false)
	t.SetStyles(s)

	return t
}

func (m podManagerModel) Init() tea.Cmd {
	return refreshCmd() // 启动自动刷新
}

func (m podManagerModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {

	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height

	case tea.KeyMsg:
		// 过滤模式下的按键处理
		if m.mode == viewFilter {
			switch msg.String() {
			case "esc":
				m.mode = viewList
				m.filterInput.Blur()
				return m, nil
			case "enter":
				m.applyFilter()
				m.mode = viewList
				m.filterInput.Blur()
				return m, nil
			default:
				var cmd tea.Cmd
				m.filterInput, cmd = m.filterInput.Update(msg)
				return m, cmd
			}
		}

		// 详情模式下的按键处理
		if m.mode == viewDetail {
			switch msg.String() {
			case "esc", "q":
				m.mode = viewList
				m.selectedPod = nil
				return m, nil
			}
			return m, nil
		}

		// 列表模式下的按键处理
		switch msg.String() {
		case "q", "ctrl+c":
			return m, tea.Quit

		case "/":
			// 进入过滤模式
			m.mode = viewFilter
			m.filterInput.Focus()
			return m, textinput.Blink

		case "enter":
			// 查看详情
			row := m.table.SelectedRow()
			if row != nil {
				for _, pod := range m.filteredPods {
					if pod.Name == row[0] {
						m.selectedPod = &pod
						m.mode = viewDetail
						break
					}
				}
			}
			return m, nil

		case "d":
			// 模拟删除 Pod
			row := m.table.SelectedRow()
			if row != nil {
				m.message = fmt.Sprintf("已删除 Pod: %s", row[0])
			}
			return m, nil

		case "r":
			// 刷新数据
			m.pods = generateMockPods()
			m.applyFilter()
			m.message = "数据已刷新"
			return m, nil

		case "1":
			m.namespace = "all"
			m.applyFilter()
		case "2":
			m.namespace = "default"
			m.applyFilter()
		case "3":
			m.namespace = "kube-system"
			m.applyFilter()
		case "4":
			m.namespace = "monitoring"
			m.applyFilter()
		}

	case refreshMsg:
		// 模拟数据刷新（随机变更部分 Pod 状态）
		m.simulateStatusChange()
		m.applyFilter()
		return m, refreshCmd()
	}

	// 更新表格
	var cmd tea.Cmd
	m.table, cmd = m.table.Update(msg)
	return m, cmd
}

// applyFilter 应用过滤条件
func (m *podManagerModel) applyFilter() {
	keyword := strings.ToLower(m.filterInput.Value())

	filtered := make([]Pod, 0)
	for _, pod := range m.pods {
		// 命名空间过滤
		if m.namespace != "all" && pod.Namespace != m.namespace {
			continue
		}
		// 关键词过滤
		if keyword != "" {
			if !strings.Contains(strings.ToLower(pod.Name), keyword) &&
				!strings.Contains(strings.ToLower(pod.Status), keyword) {
				continue
			}
		}
		filtered = append(filtered, pod)
	}

	m.filteredPods = filtered
	m.table = m.createTable(filtered)
}

// simulateStatusChange 模拟状态变化
func (m *podManagerModel) simulateStatusChange() {
	for i := range m.pods {
		if rand.Intn(20) == 0 { // 5% 概率变化
			statuses := []string{"Running", "Running", "Running", "Pending", "CrashLoopBackOff"}
			m.pods[i].Status = statuses[rand.Intn(len(statuses))]
			if m.pods[i].Status == "CrashLoopBackOff" {
				m.pods[i].Restarts++
			}
		}
	}
}

func (m podManagerModel) View() string {
	switch m.mode {
	case viewDetail:
		return m.viewDetail()
	case viewFilter:
		return m.viewList() // 列表视图但带过滤输入框
	default:
		return m.viewList()
	}
}

// viewList 列表视图
func (m podManagerModel) viewList() string {
	var b strings.Builder

	// 标题栏
	title := titleStyle.Render("K8s Pod Manager")
	nsInfo := statusBarStyle.Render(fmt.Sprintf(" 命名空间: %s | Pod数: %d ", m.namespace, len(m.filteredPods)))
	b.WriteString(lipgloss.JoinHorizontal(lipgloss.Top, title, "  ", nsInfo))
	b.WriteString("\n\n")

	// 命名空间切换
	nsTabs := []string{"[1]all", "[2]default", "[3]kube-system", "[4]monitoring"}
	for i, ns := range nsTabs {
		name := strings.TrimPrefix(strings.TrimSuffix(ns[3:], ""), "")
		realNs := []string{"all", "default", "kube-system", "monitoring"}[i]
		if realNs == m.namespace {
			b.WriteString(infoStyle.Render(ns))
		} else {
			b.WriteString(helpStyle.Render(ns))
		}
		b.WriteString("  ")
	}
	b.WriteString("\n\n")

	// 过滤输入框（过滤模式时显示）
	if m.mode == viewFilter {
		b.WriteString(m.filterInput.View())
		b.WriteString("\n\n")
	}

	// 表格
	b.WriteString(m.table.View())
	b.WriteString("\n")

	// 底部消息
	if m.message != "" {
		b.WriteString(infoStyle.Render(fmt.Sprintf("  %s", m.message)))
		b.WriteString("\n")
	}

	// 帮助栏
	help := "  / 过滤 | Enter 详情 | d 删除 | r 刷新 | 1-4 切换命名空间 | q 退出"
	b.WriteString(helpStyle.Render(help))
	b.WriteString("\n")

	return b.String()
}

// viewDetail 详情视图
func (m podManagerModel) viewDetail() string {
	if m.selectedPod == nil {
		return "无选中 Pod"
	}

	pod := m.selectedPod
	var b strings.Builder

	title := titleStyle.Render(fmt.Sprintf("Pod 详情: %s", pod.Name))
	b.WriteString(title)
	b.WriteString("\n\n")

	// 详情面板
	detailBox := lipgloss.NewStyle().
		Border(lipgloss.RoundedBorder()).
		BorderForeground(lipgloss.Color("63")).
		Padding(1, 2).
		Width(60)

	var details strings.Builder
	details.WriteString(fmt.Sprintf("名称:     %s\n", pod.Name))
	details.WriteString(fmt.Sprintf("命名空间: %s\n", pod.Namespace))

	// 状态着色
	var statusText string
	switch pod.Status {
	case "Running":
		statusText = runningStyle.Render(pod.Status)
	case "CrashLoopBackOff":
		statusText = errorStyle.Render(pod.Status)
	case "Pending":
		statusText = pendingStyle.Render(pod.Status)
	default:
		statusText = pod.Status
	}
	details.WriteString(fmt.Sprintf("状态:     %s\n", statusText))

	details.WriteString(fmt.Sprintf("就绪:     %s\n", pod.Ready))
	details.WriteString(fmt.Sprintf("重启次数: %d\n", pod.Restarts))
	details.WriteString(fmt.Sprintf("年龄:     %s\n", pod.Age))
	details.WriteString(fmt.Sprintf("IP:       %s\n", pod.IP))
	details.WriteString(fmt.Sprintf("节点:     %s\n", pod.Node))
	details.WriteString(fmt.Sprintf("CPU:      %s\n", pod.CPU))
	details.WriteString(fmt.Sprintf("内存:     %s\n", pod.Memory))

	b.WriteString(detailBox.Render(details.String()))
	b.WriteString("\n\n")

	b.WriteString(helpStyle.Render("  Esc/q 返回列表"))
	b.WriteString("\n")

	return b.String()
}

// ==================== 模拟数据 ====================

func generateMockPods() []Pod {
	pods := []Pod{
		{Name: "nginx-deploy-7d4f8b-xk2pl", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "3d", IP: "10.244.1.5", Node: "node-01", CPU: "50m", Memory: "128Mi"},
		{Name: "nginx-deploy-7d4f8b-m9nkq", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "3d", IP: "10.244.2.3", Node: "node-02", CPU: "45m", Memory: "120Mi"},
		{Name: "nginx-deploy-7d4f8b-z8w2j", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 1, Age: "3d", IP: "10.244.1.6", Node: "node-01", CPU: "55m", Memory: "135Mi"},
		{Name: "api-gw-5c6d7e-abc12", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "1d", IP: "10.244.3.2", Node: "node-03", CPU: "200m", Memory: "256Mi"},
		{Name: "api-gw-5c6d7e-def34", Namespace: "default", Status: "CrashLoopBackOff", Ready: "0/1", Restarts: 5, Age: "1d", IP: "10.244.2.4", Node: "node-02", CPU: "0m", Memory: "0Mi"},
		{Name: "coredns-6d4b75cb6d-abc12", Namespace: "kube-system", Status: "Running", Ready: "1/1", Restarts: 0, Age: "30d", IP: "10.244.0.2", Node: "node-01", CPU: "30m", Memory: "64Mi"},
		{Name: "coredns-6d4b75cb6d-def34", Namespace: "kube-system", Status: "Running", Ready: "1/1", Restarts: 0, Age: "30d", IP: "10.244.0.3", Node: "node-02", CPU: "25m", Memory: "60Mi"},
		{Name: "etcd-master", Namespace: "kube-system", Status: "Running", Ready: "1/1", Restarts: 0, Age: "90d", IP: "10.0.0.10", Node: "master", CPU: "150m", Memory: "512Mi"},
		{Name: "kube-proxy-abc12", Namespace: "kube-system", Status: "Running", Ready: "1/1", Restarts: 0, Age: "90d", IP: "10.0.0.11", Node: "node-01", CPU: "10m", Memory: "32Mi"},
		{Name: "kube-proxy-def34", Namespace: "kube-system", Status: "Running", Ready: "1/1", Restarts: 0, Age: "90d", IP: "10.0.0.12", Node: "node-02", CPU: "10m", Memory: "32Mi"},
		{Name: "prometheus-server-0", Namespace: "monitoring", Status: "Running", Ready: "2/2", Restarts: 0, Age: "14d", IP: "10.244.3.8", Node: "node-03", CPU: "500m", Memory: "2Gi"},
		{Name: "grafana-7f8a9b-q2w3e", Namespace: "monitoring", Status: "Running", Ready: "1/1", Restarts: 0, Age: "14d", IP: "10.244.1.25", Node: "node-01", CPU: "100m", Memory: "256Mi"},
		{Name: "alertmanager-0", Namespace: "monitoring", Status: "Running", Ready: "1/1", Restarts: 0, Age: "14d", IP: "10.244.2.20", Node: "node-02", CPU: "50m", Memory: "128Mi"},
		{Name: "node-exporter-abc12", Namespace: "monitoring", Status: "Running", Ready: "1/1", Restarts: 0, Age: "14d", IP: "10.0.0.11", Node: "node-01", CPU: "20m", Memory: "64Mi"},
		{Name: "node-exporter-def34", Namespace: "monitoring", Status: "Running", Ready: "1/1", Restarts: 0, Age: "14d", IP: "10.0.0.12", Node: "node-02", CPU: "20m", Memory: "64Mi"},
		{Name: "redis-master-0", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "7d", IP: "10.244.1.10", Node: "node-01", CPU: "100m", Memory: "512Mi"},
		{Name: "redis-slave-0", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "7d", IP: "10.244.2.8", Node: "node-02", CPU: "80m", Memory: "256Mi"},
		{Name: "mysql-primary-0", Namespace: "default", Status: "Running", Ready: "1/1", Restarts: 0, Age: "14d", IP: "10.244.1.20", Node: "node-01", CPU: "500m", Memory: "1Gi"},
		{Name: "mysql-replica-0", Namespace: "default", Status: "Pending", Ready: "0/1", Restarts: 0, Age: "5m", IP: "", Node: "node-02", CPU: "0m", Memory: "0Mi"},
	}
	return pods
}

func main() {
	p := tea.NewProgram(
		newPodManagerModel(),
		tea.WithAltScreen(), // 使用备用屏幕（退出后恢复原终端内容）
	)

	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

---

## 七、实战：实时日志查看器

```go
// log_viewer/main.go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"strings"
	"time"

	"github.com/charmbracelet/bubbles/viewport"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
)

// ==================== 样式 ====================

var (
	logTitleStyle = lipgloss.NewStyle().
			Bold(true).
			Foreground(lipgloss.Color("#FFFFFF")).
			Background(lipgloss.Color("#5A56E0")).
			Padding(0, 2)

	logHelpStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("241"))

	infoLogStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#04B575"))

	warnLogStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#FFD700"))

	errorLogStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#FF4672"))

	debugLogStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("#7D56F4"))

	timestampStyle = lipgloss.NewStyle().
			Foreground(lipgloss.Color("241"))
)

// ==================== 日志条目 ====================

// LogEntry 日志条目
type LogEntry struct {
	Timestamp time.Time
	Level     string
	Service   string
	Message   string
}

// Format 格式化日志条目
func (e LogEntry) Format() string {
	ts := timestampStyle.Render(e.Timestamp.Format("15:04:05.000"))

	var level string
	switch e.Level {
	case "INFO":
		level = infoLogStyle.Render("[INFO ]")
	case "WARN":
		level = warnLogStyle.Render("[WARN ]")
	case "ERROR":
		level = errorLogStyle.Render("[ERROR]")
	case "DEBUG":
		level = debugLogStyle.Render("[DEBUG]")
	default:
		level = e.Level
	}

	return fmt.Sprintf("%s %s %s %s", ts, level, e.Service, e.Message)
}

// ==================== 日志模型 ====================

// 新日志消息
type newLogMsg LogEntry

// 生成模拟日志的命令
func generateLog() tea.Cmd {
	return func() tea.Msg {
		// 随机延迟，模拟日志流
		delay := time.Duration(100+rand.Intn(900)) * time.Millisecond
		time.Sleep(delay)

		services := []string{"api-gw", "user-svc", "order-svc", "pay-svc", "nginx"}
		levels := []string{"INFO", "INFO", "INFO", "INFO", "WARN", "ERROR", "DEBUG"}

		messages := map[string][]string{
			"INFO": {
				"请求处理成功 status=200 duration=15ms",
				"数据库查询完成 rows=42 duration=5ms",
				"缓存命中 key=user:1001",
				"健康检查通过",
				"gRPC 调用成功 method=/user.GetUser",
				"消息发送成功 topic=alerts",
				"定时任务执行完成 job=cleanup",
			},
			"WARN": {
				"请求延迟增加 p99=500ms threshold=200ms",
				"连接池使用率较高 used=85/100",
				"重试请求 attempt=2/3 service=pay-svc",
				"缓存未命中 key=user:9999",
				"磁盘使用率 usage=78%",
			},
			"ERROR": {
				"请求失败 status=500 err=\"connection refused\"",
				"数据库连接超时 timeout=5s",
				"消息消费失败 topic=orders err=\"decode error\"",
				"证书即将过期 days=7",
				"OOM Kill 风险 memory=95%",
			},
			"DEBUG": {
				"请求头 X-Request-ID=abc-123",
				"SQL: SELECT * FROM users WHERE id = ?",
				"Redis PING 延迟 latency=1ms",
			},
		}

		level := levels[rand.Intn(len(levels))]
		service := services[rand.Intn(len(services))]
		msgs := messages[level]
		msg := msgs[rand.Intn(len(msgs))]

		return newLogMsg(LogEntry{
			Timestamp: time.Now(),
			Level:     level,
			Service:   service,
			Message:   msg,
		})
	}
}

type logViewerModel struct {
	viewport    viewport.Model
	logs        []LogEntry
	content     string
	paused      bool     // 暂停自动滚动
	filterLevel string   // 日志级别过滤
	filterText  string   // 文本过滤
	width       int
	height      int
	ready       bool
	autoScroll  bool     // 自动滚动到底部
	logCount    int      // 日志总数
}

func newLogViewerModel() logViewerModel {
	return logViewerModel{
		logs:       make([]LogEntry, 0, 1000),
		autoScroll: true,
	}
}

func (m logViewerModel) Init() tea.Cmd {
	return generateLog()
}

func (m logViewerModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {

	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height

		headerHeight := 3
		footerHeight := 3
		viewHeight := m.height - headerHeight - footerHeight

		if !m.ready {
			m.viewport = viewport.New(m.width, viewHeight)
			m.viewport.YPosition = headerHeight
			m.ready = true
		} else {
			m.viewport.Width = m.width
			m.viewport.Height = viewHeight
		}

	case tea.KeyMsg:
		switch msg.String() {
		case "q", "ctrl+c":
			return m, tea.Quit
		case " ":
			// 暂停/恢复
			m.paused = !m.paused
			if !m.paused {
				return m, generateLog()
			}
			return m, nil
		case "c":
			// 清空日志
			m.logs = m.logs[:0]
			m.updateContent()
			return m, nil
		case "1":
			m.filterLevel = ""
			m.updateContent()
		case "2":
			m.filterLevel = "INFO"
			m.updateContent()
		case "3":
			m.filterLevel = "WARN"
			m.updateContent()
		case "4":
			m.filterLevel = "ERROR"
			m.updateContent()
		case "5":
			m.filterLevel = "DEBUG"
			m.updateContent()
		case "g":
			// 滚动到顶部
			m.viewport.GotoTop()
			m.autoScroll = false
		case "G":
			// 滚动到底部
			m.viewport.GotoBottom()
			m.autoScroll = true
		}

	case newLogMsg:
		if !m.paused {
			entry := LogEntry(msg)
			m.logs = append(m.logs, entry)
			m.logCount++

			// 限制日志数量（保留最新的 1000 条）
			if len(m.logs) > 1000 {
				m.logs = m.logs[len(m.logs)-1000:]
			}

			m.updateContent()

			// 继续生成日志
			return m, generateLog()
		}
		return m, nil
	}

	// 更新 viewport
	var cmd tea.Cmd
	m.viewport, cmd = m.viewport.Update(msg)

	// 如果用户手动滚动了，禁用自动滚动
	if m.viewport.AtBottom() {
		m.autoScroll = true
	}

	return m, cmd
}

// updateContent 更新显示内容
func (m *logViewerModel) updateContent() {
	var lines []string

	for _, entry := range m.logs {
		// 级别过滤
		if m.filterLevel != "" && entry.Level != m.filterLevel {
			continue
		}
		// 文本过滤
		if m.filterText != "" {
			if !strings.Contains(strings.ToLower(entry.Message), strings.ToLower(m.filterText)) &&
				!strings.Contains(strings.ToLower(entry.Service), strings.ToLower(m.filterText)) {
				continue
			}
		}
		lines = append(lines, entry.Format())
	}

	m.content = strings.Join(lines, "\n")
	m.viewport.SetContent(m.content)

	if m.autoScroll {
		m.viewport.GotoBottom()
	}
}

func (m logViewerModel) View() string {
	if !m.ready {
		return "\n  初始化中..."
	}

	var b strings.Builder

	// 标题栏
	title := logTitleStyle.Render("Log Viewer")
	pauseStatus := ""
	if m.paused {
		pauseStatus = errorLogStyle.Render(" [PAUSED]")
	}
	filterStatus := ""
	if m.filterLevel != "" {
		filterStatus = fmt.Sprintf(" 过滤: %s", m.filterLevel)
	}
	stats := lipgloss.NewStyle().
		Foreground(lipgloss.Color("241")).
		Render(fmt.Sprintf(" 日志总数: %d%s%s", m.logCount, filterStatus, pauseStatus))

	b.WriteString(lipgloss.JoinHorizontal(lipgloss.Top, title, stats))
	b.WriteString("\n")

	// 级别过滤标签
	levelTabs := []struct {
		key   string
		label string
		level string
	}{
		{"1", "全部", ""},
		{"2", "INFO", "INFO"},
		{"3", "WARN", "WARN"},
		{"4", "ERROR", "ERROR"},
		{"5", "DEBUG", "DEBUG"},
	}

	for _, tab := range levelTabs {
		label := fmt.Sprintf("[%s]%s", tab.key, tab.label)
		if tab.level == m.filterLevel || (tab.level == "" && m.filterLevel == "") {
			b.WriteString(lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("205")).Render(label))
		} else {
			b.WriteString(logHelpStyle.Render(label))
		}
		b.WriteString("  ")
	}
	b.WriteString("\n")

	// 日志视口
	b.WriteString(m.viewport.View())
	b.WriteString("\n")

	// 帮助栏
	help := "  Space 暂停/恢复 | 1-5 级别过滤 | c 清空 | g/G 顶部/底部 | q 退出"
	scrollInfo := fmt.Sprintf("  %3.f%%", m.viewport.ScrollPercent()*100)
	b.WriteString(logHelpStyle.Render(help + scrollInfo))

	return b.String()
}

func main() {
	p := tea.NewProgram(
		newLogViewerModel(),
		tea.WithAltScreen(),
		tea.WithMouseCellMotion(), // 支持鼠标滚轮
	)

	if _, err := p.Run(); err != nil {
		fmt.Printf("错误: %v\n", err)
		os.Exit(1)
	}
}
```

---

## 八、常见坑点

### 坑点 1：View() 中执行耗时操作

```go
// 错误：在 View 中做耗时操作（View 会被频繁调用）
func (m model) View() string {
    data := fetchDataFromDB() // 每次渲染都查数据库！
    return renderData(data)
}

// 正确：在 Update 中通过 Cmd 异步获取数据，在 View 中只做渲染
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        if msg.String() == "r" {
            return m, fetchDataCmd  // 异步命令
        }
    case dataMsg:
        m.data = msg.data  // 保存数据到 Model
    }
    return m, nil
}

func (m model) View() string {
    return renderData(m.data)  // 只渲染已有数据
}
```

### 坑点 2：忘记处理 WindowSizeMsg

```go
// 错误：硬编码尺寸，不同终端大小显示错乱
func (m model) View() string {
    return lipgloss.NewStyle().Width(120).Render(content) // 如果终端宽度小于 120 就溢出了
}

// 正确：监听 WindowSizeMsg 并自适应
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
    }
    return m, nil
}

func (m model) View() string {
    return lipgloss.NewStyle().Width(m.width - 4).Render(content)
}
```

### 坑点 3：Cmd 中阻塞导致 UI 卡死

```go
// 错误：在 Update 中直接做耗时操作
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    time.Sleep(5 * time.Second) // UI 完全卡死 5 秒
    return m, nil
}

// 正确：通过 Cmd 异步执行
func longRunningTask() tea.Msg {
    time.Sleep(5 * time.Second) // 在独立 goroutine 中执行
    return taskDoneMsg{}
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    return m, longRunningTask // 返回 Cmd，不阻塞 UI
}
```

### 坑点 4：promptui 在 Bubbletea 程序中使用冲突

```go
// 错误：在 Bubbletea 运行时使用 promptui
// 两者都会尝试控制终端，导致冲突

// 正确方案一：在 Bubbletea 启动前使用 promptui 收集输入
result := promptui.Select{...}.Run()
// 然后启动 Bubbletea
tea.NewProgram(model{input: result}).Run()

// 正确方案二：在 Bubbletea 中使用 bubbles 组件替代 promptui
// 用 textinput 替代 promptui.Prompt
// 用 list 组件替代 promptui.Select
```

### 坑点 5：Alt Screen 模式下的输出丢失

```go
// 使用 WithAltScreen 时，程序退出后终端恢复原内容
// 如果需要在退出后显示结果，用 tea.Println

// 方式一：在 quit 前打印
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    if msg.(tea.KeyMsg).String() == "q" {
        return m, tea.Sequence(
            tea.Println("结果: OK"),  // 先打印
            tea.Quit,                  // 再退出
        )
    }
}

// 方式二：退出后从 Model 中获取结果
finalModel, _ := p.Run()
m := finalModel.(model)
fmt.Println("结果:", m.result)
```

---

## 九、速查表

### Bubbletea 核心 API

| API | 说明 |
|-----|------|
| `tea.NewProgram(model)` | 创建程序 |
| `tea.WithAltScreen()` | 备用屏幕模式 |
| `tea.WithMouseCellMotion()` | 启用鼠标支持 |
| `p.Run()` | 运行程序（阻塞） |
| `tea.Quit` | 退出 Cmd |
| `tea.Batch(cmds...)` | 批量 Cmd |
| `tea.Tick(d, fn)` | 定时器 |
| `tea.Println(s)` | 打印到终端 |
| `tea.Sequence(cmds...)` | 顺序执行 Cmd |

### 常用消息类型

| 消息类型 | 说明 |
|----------|------|
| `tea.KeyMsg` | 键盘事件（`.String()` 获取按键） |
| `tea.MouseMsg` | 鼠标事件 |
| `tea.WindowSizeMsg` | 窗口大小变化（`.Width`, `.Height`） |
| `spinner.TickMsg` | Spinner 帧更新 |
| `progress.FrameMsg` | 进度条帧更新 |

### Bubbles 组件

| 组件 | 导入路径 | 用途 |
|------|----------|------|
| textinput | `bubbles/textinput` | 文本输入框 |
| textarea | `bubbles/textarea` | 多行文本输入 |
| spinner | `bubbles/spinner` | 加载指示器 |
| progress | `bubbles/progress` | 进度条 |
| table | `bubbles/table` | 数据表格 |
| list | `bubbles/list` | 列表（含过滤搜索） |
| viewport | `bubbles/viewport` | 可滚动视口 |
| paginator | `bubbles/paginator` | 分页器 |
| filepicker | `bubbles/filepicker` | 文件选择器 |
| timer | `bubbles/timer` | 倒计时 |
| stopwatch | `bubbles/stopwatch` | 计时器 |

### Lip Gloss 样式速查

```go
// 文字样式
style.Bold(true)
style.Italic(true)
style.Underline(true)
style.Strikethrough(true)

// 颜色
style.Foreground(lipgloss.Color("205"))      // ANSI 256
style.Foreground(lipgloss.Color("#FF6B6B"))  // Hex
style.Background(lipgloss.Color("236"))

// 边距和内边距
style.Padding(top, right, bottom, left)
style.Margin(top, right, bottom, left)
style.PaddingLeft(2)
style.MarginTop(1)

// 尺寸
style.Width(40)
style.Height(10)
style.MaxWidth(80)

// 对齐
style.Align(lipgloss.Center)
style.Align(lipgloss.Left)
style.Align(lipgloss.Right)

// 边框
style.Border(lipgloss.RoundedBorder())
style.BorderForeground(lipgloss.Color("63"))

// 布局组合
lipgloss.JoinHorizontal(pos, strs...)
lipgloss.JoinVertical(pos, strs...)
lipgloss.Place(width, height, hPos, vPos, str)
```

### 依赖包

```bash
# Bubbletea 生态
go get github.com/charmbracelet/bubbletea
go get github.com/charmbracelet/bubbles
go get github.com/charmbracelet/lipgloss

# 交互式选择
go get github.com/manifoldco/promptui

# 表格输出
go get github.com/olekukonez/tablewriter
```

---

## 小结

本章完整介绍了 Go 终端 UI 开发的核心工具和技术：

1. **Bubbletea** 是 Go 生态最优秀的 TUI 框架，Elm 架构（Model-Update-View）提供了清晰的状态管理和可预测的 UI 更新
2. **Bubbles 组件** 提供了开箱即用的 UI 组件（输入框、表格、进度条、加载器、视口等），大幅降低开发成本
3. **Lip Gloss** 提供了声明式的终端样式系统（颜色、边框、对齐、布局），让终端也能拥有精美的视觉效果
4. **promptui** 适合简单的交互式选择和输入场景，使用简单但不适合复杂 TUI
5. 通过 K8s Pod 管理器和实时日志查看器两个实战项目，展示了如何构建功能完善的运维 TUI 工具

掌握这些工具后，你可以将日常的运维脚本升级为交互式的终端应用，提升团队的工作效率和使用体验。
