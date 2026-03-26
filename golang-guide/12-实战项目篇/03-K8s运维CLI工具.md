# 实战项目：K8s 运维 CLI 工具

## 导读

本篇将构建一个 kubectl 增强版运维工具，提供 Pod 诊断、Deployment 操作、Node 资源统计等高频运维功能，并支持 TUI 交互界面和表格美化输出。这是 SRE 日常工作中最实用的工具之一。

**项目功能：**
- Cobra 多级子命令设计
- client-go 集群连接
- Pod 操作（列表/日志/exec/诊断）
- Deployment 操作（扩缩容/重启/回滚/镜像更新）
- Node 资源统计报表
- bubbletea TUI 交互界面
- 表格美化输出
- 多集群支持

---

## 一、项目结构

```
k8s-tool/
├── cmd/
│   └── ktool/
│       └── main.go
├── internal/
│   ├── cmd/                   # 子命令定义
│   │   ├── root.go
│   │   ├── pod.go
│   │   ├── deploy.go
│   │   ├── node.go
│   │   └── tui.go
│   ├── k8s/                   # Kubernetes 客户端
│   │   ├── client.go
│   │   ├── pod.go
│   │   ├── deployment.go
│   │   └── node.go
│   ├── output/                # 输出格式
│   │   ├── table.go
│   │   └── json.go
│   └── tui/                   # TUI 界面
│       └── dashboard.go
├── go.mod
├── go.sum
└── Makefile
```

---

## 二、完整项目代码

### 2.1 Kubernetes 客户端

```go
// internal/k8s/client.go
// Kubernetes 客户端初始化和连接管理
package k8s

import (
	"fmt"
	"os"
	"path/filepath"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"k8s.io/metrics/pkg/client/clientset/versioned"
)

// Client Kubernetes 客户端封装
type Client struct {
	Clientset     *kubernetes.Clientset
	MetricsClient *versioned.Clientset
	Config        *rest.Config
	Context       string // 当前集群上下文名称
}

// NewClient 创建 Kubernetes 客户端
// kubeconfig: 配置文件路径（空字符串使用默认路径）
// context: 集群上下文名称（空字符串使用当前上下文）
func NewClient(kubeconfig, context string) (*Client, error) {
	// 确定 kubeconfig 路径
	if kubeconfig == "" {
		kubeconfig = os.Getenv("KUBECONFIG")
	}
	if kubeconfig == "" {
		home := homedir.HomeDir()
		if home != "" {
			kubeconfig = filepath.Join(home, ".kube", "config")
		}
	}

	// 构建配置
	var config *rest.Config
	var err error

	// 优先尝试集群内配置（Pod 中运行时）
	config, err = rest.InClusterConfig()
	if err != nil {
		// 使用 kubeconfig 文件
		loadingRules := &clientcmd.ClientConfigLoadingRules{
			ExplicitPath: kubeconfig,
		}
		configOverrides := &clientcmd.ConfigOverrides{}
		if context != "" {
			configOverrides.CurrentContext = context
		}

		clientConfig := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
			loadingRules, configOverrides)

		config, err = clientConfig.ClientConfig()
		if err != nil {
			return nil, fmt.Errorf("加载 kubeconfig 失败: %w", err)
		}

		// 获取当前上下文名称
		rawConfig, _ := clientConfig.RawConfig()
		context = rawConfig.CurrentContext
	}

	// 创建 clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		return nil, fmt.Errorf("创建 Kubernetes 客户端失败: %w", err)
	}

	// 创建 metrics 客户端（可选，可能未安装 metrics-server）
	metricsClient, _ := versioned.NewForConfig(config)

	return &Client{
		Clientset:     clientset,
		MetricsClient: metricsClient,
		Config:        config,
		Context:       context,
	}, nil
}

// ListContexts 列出所有可用的集群上下文
func ListContexts(kubeconfig string) ([]string, string, error) {
	if kubeconfig == "" {
		kubeconfig = os.Getenv("KUBECONFIG")
	}
	if kubeconfig == "" {
		home := homedir.HomeDir()
		if home != "" {
			kubeconfig = filepath.Join(home, ".kube", "config")
		}
	}

	loadingRules := &clientcmd.ClientConfigLoadingRules{
		ExplicitPath: kubeconfig,
	}
	clientConfig := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		loadingRules, &clientcmd.ConfigOverrides{})

	rawConfig, err := clientConfig.RawConfig()
	if err != nil {
		return nil, "", err
	}

	var contexts []string
	for name := range rawConfig.Contexts {
		contexts = append(contexts, name)
	}

	return contexts, rawConfig.CurrentContext, nil
}
```

### 2.2 Pod 操作

```go
// internal/k8s/pod.go
// Pod 相关操作
package k8s

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"sort"
	"strings"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/remotecommand"
)

// PodInfo Pod 信息摘要
type PodInfo struct {
	Name       string
	Namespace  string
	Status     string
	Ready      string
	Restarts   int32
	Age        time.Duration
	Node       string
	IP         string
	Containers []ContainerInfo
}

// ContainerInfo 容器信息
type ContainerInfo struct {
	Name    string
	Image   string
	Status  string
	Ready   bool
	Restarts int32
}

// ListPods 列出 Pod
func (c *Client) ListPods(ctx context.Context, namespace, labelSelector string) ([]PodInfo, error) {
	opts := metav1.ListOptions{}
	if labelSelector != "" {
		opts.LabelSelector = labelSelector
	}

	pods, err := c.Clientset.CoreV1().Pods(namespace).List(ctx, opts)
	if err != nil {
		return nil, fmt.Errorf("列出 Pod 失败: %w", err)
	}

	var result []PodInfo
	for _, pod := range pods.Items {
		info := PodInfo{
			Name:      pod.Name,
			Namespace: pod.Namespace,
			Status:    string(pod.Status.Phase),
			Node:      pod.Spec.NodeName,
			IP:        pod.Status.PodIP,
			Age:       time.Since(pod.CreationTimestamp.Time),
		}

		// 统计容器状态
		var readyCount, totalCount int
		for _, cs := range pod.Status.ContainerStatuses {
			totalCount++
			if cs.Ready {
				readyCount++
			}
			info.Restarts += cs.RestartCount

			containerInfo := ContainerInfo{
				Name:     cs.Name,
				Image:    cs.Image,
				Ready:    cs.Ready,
				Restarts: cs.RestartCount,
			}
			if cs.State.Running != nil {
				containerInfo.Status = "Running"
			} else if cs.State.Waiting != nil {
				containerInfo.Status = cs.State.Waiting.Reason
			} else if cs.State.Terminated != nil {
				containerInfo.Status = cs.State.Terminated.Reason
			}
			info.Containers = append(info.Containers, containerInfo)
		}
		info.Ready = fmt.Sprintf("%d/%d", readyCount, totalCount)

		// 更精确的状态判断
		for _, cs := range pod.Status.ContainerStatuses {
			if cs.State.Waiting != nil && cs.State.Waiting.Reason != "" {
				info.Status = cs.State.Waiting.Reason
				break
			}
		}

		result = append(result, info)
	}

	// 按名称排序
	sort.Slice(result, func(i, j int) bool {
		return result[i].Name < result[j].Name
	})

	return result, nil
}

// GetPodLogs 获取 Pod 日志
func (c *Client) GetPodLogs(ctx context.Context, namespace, podName, container string, lines int64, follow bool) (io.ReadCloser, error) {
	opts := &corev1.PodLogOptions{
		TailLines: &lines,
		Follow:    follow,
	}
	if container != "" {
		opts.Container = container
	}

	req := c.Clientset.CoreV1().Pods(namespace).GetLogs(podName, opts)
	stream, err := req.Stream(ctx)
	if err != nil {
		return nil, fmt.Errorf("获取日志流失败: %w", err)
	}

	return stream, nil
}

// ExecInPod 在 Pod 中执行命令
func (c *Client) ExecInPod(ctx context.Context, namespace, podName, container string, command []string, stdin io.Reader, stdout, stderr io.Writer) error {
	req := c.Clientset.CoreV1().RESTClient().Post().
		Resource("pods").
		Name(podName).
		Namespace(namespace).
		SubResource("exec").
		VersionedParams(&corev1.PodExecOptions{
			Container: container,
			Command:   command,
			Stdin:     stdin != nil,
			Stdout:    true,
			Stderr:    true,
			TTY:       false,
		}, scheme.ParameterCodec)

	exec, err := remotecommand.NewSPDYExecutor(c.Config, "POST", req.URL())
	if err != nil {
		return fmt.Errorf("创建执行器失败: %w", err)
	}

	return exec.StreamWithContext(ctx, remotecommand.StreamOptions{
		Stdin:  stdin,
		Stdout: stdout,
		Stderr: stderr,
	})
}

// DiagnosePod Pod 诊断
func (c *Client) DiagnosePod(ctx context.Context, namespace, podName string) (*PodDiagnosis, error) {
	pod, err := c.Clientset.CoreV1().Pods(namespace).Get(ctx, podName, metav1.GetOptions{})
	if err != nil {
		return nil, fmt.Errorf("获取 Pod 失败: %w", err)
	}

	diag := &PodDiagnosis{
		PodName:   pod.Name,
		Namespace: pod.Namespace,
		Phase:     string(pod.Status.Phase),
		Node:      pod.Spec.NodeName,
		IP:        pod.Status.PodIP,
		Age:       time.Since(pod.CreationTimestamp.Time),
	}

	// 检查容器状态
	for _, cs := range pod.Status.ContainerStatuses {
		issue := ContainerDiagnosis{
			Name:     cs.Name,
			Ready:    cs.Ready,
			Restarts: cs.RestartCount,
		}

		if cs.State.Waiting != nil {
			issue.Status = "Waiting"
			issue.Reason = cs.State.Waiting.Reason
			issue.Message = cs.State.Waiting.Message
			issue.Issues = append(issue.Issues, fmt.Sprintf("容器处于 Waiting 状态: %s", cs.State.Waiting.Reason))
		} else if cs.State.Terminated != nil {
			issue.Status = "Terminated"
			issue.Reason = cs.State.Terminated.Reason
			issue.Message = cs.State.Terminated.Message
			issue.Issues = append(issue.Issues, fmt.Sprintf("容器已终止: %s (退出码: %d)",
				cs.State.Terminated.Reason, cs.State.Terminated.ExitCode))
		} else if cs.State.Running != nil {
			issue.Status = "Running"
		}

		if cs.RestartCount > 5 {
			issue.Issues = append(issue.Issues, fmt.Sprintf("频繁重启: %d 次", cs.RestartCount))
		}

		if !cs.Ready && cs.State.Running != nil {
			issue.Issues = append(issue.Issues, "容器运行中但未就绪（检查 readiness probe）")
		}

		diag.Containers = append(diag.Containers, issue)
	}

	// 检查 Pod 事件
	events, err := c.Clientset.CoreV1().Events(namespace).List(ctx, metav1.ListOptions{
		FieldSelector: fmt.Sprintf("involvedObject.name=%s", podName),
	})
	if err == nil {
		for _, event := range events.Items {
			if event.Type == "Warning" {
				diag.Events = append(diag.Events, EventInfo{
					Type:    event.Type,
					Reason:  event.Reason,
					Message: event.Message,
					Count:   event.Count,
					LastSeen: event.LastTimestamp.Time,
				})
			}
		}
	}

	// 检查 Pod Conditions
	for _, cond := range pod.Status.Conditions {
		if cond.Status == corev1.ConditionFalse {
			diag.Issues = append(diag.Issues,
				fmt.Sprintf("条件 %s = False: %s", cond.Type, cond.Message))
		}
	}

	return diag, nil
}

// PodDiagnosis Pod 诊断结果
type PodDiagnosis struct {
	PodName    string
	Namespace  string
	Phase      string
	Node       string
	IP         string
	Age        time.Duration
	Containers []ContainerDiagnosis
	Events     []EventInfo
	Issues     []string
}

type ContainerDiagnosis struct {
	Name     string
	Status   string
	Reason   string
	Message  string
	Ready    bool
	Restarts int32
	Issues   []string
}

type EventInfo struct {
	Type     string
	Reason   string
	Message  string
	Count    int32
	LastSeen time.Time
}

// FormatDiagnosis 格式化诊断结果
func (d *PodDiagnosis) String() string {
	var b strings.Builder

	b.WriteString(fmt.Sprintf("Pod 诊断报告: %s/%s\n", d.Namespace, d.PodName))
	b.WriteString(strings.Repeat("=", 60) + "\n")
	b.WriteString(fmt.Sprintf("状态: %s | 节点: %s | IP: %s | 运行时间: %s\n\n",
		d.Phase, d.Node, d.IP, formatDuration(d.Age)))

	// 容器诊断
	b.WriteString("--- 容器状态 ---\n")
	for _, c := range d.Containers {
		readyIcon := "[x]"
		if c.Ready {
			readyIcon = "[v]"
		}
		b.WriteString(fmt.Sprintf("%s %s: %s (重启: %d)\n",
			readyIcon, c.Name, c.Status, c.Restarts))
		for _, issue := range c.Issues {
			b.WriteString(fmt.Sprintf("    ! %s\n", issue))
		}
	}

	// 告警事件
	if len(d.Events) > 0 {
		b.WriteString("\n--- 告警事件 ---\n")
		for _, e := range d.Events {
			b.WriteString(fmt.Sprintf("  [%s] %s: %s (x%d)\n",
				e.Type, e.Reason, e.Message, e.Count))
		}
	}

	// 问题汇总
	if len(d.Issues) > 0 {
		b.WriteString("\n--- 问题汇总 ---\n")
		for _, issue := range d.Issues {
			b.WriteString(fmt.Sprintf("  ! %s\n", issue))
		}
	}

	return b.String()
}

func formatDuration(d time.Duration) string {
	if d < time.Minute {
		return fmt.Sprintf("%ds", int(d.Seconds()))
	}
	if d < time.Hour {
		return fmt.Sprintf("%dm", int(d.Minutes()))
	}
	if d < 24*time.Hour {
		return fmt.Sprintf("%dh", int(d.Hours()))
	}
	return fmt.Sprintf("%dd", int(d.Hours()/24))
}

// ExecCommand 在 Pod 中执行命令并返回输出
func (c *Client) ExecCommand(ctx context.Context, namespace, podName, container string, command []string) (string, string, error) {
	var stdout, stderr bytes.Buffer
	err := c.ExecInPod(ctx, namespace, podName, container, command, nil, &stdout, &stderr)
	return stdout.String(), stderr.String(), err
}
```

### 2.3 Deployment 操作

```go
// internal/k8s/deployment.go
// Deployment 相关操作
package k8s

import (
	"context"
	"fmt"
	"sort"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
)

// DeployInfo Deployment 信息
type DeployInfo struct {
	Name       string
	Namespace  string
	Replicas   int32
	Ready      int32
	Updated    int32
	Available  int32
	Age        time.Duration
	Images     []string
	Strategy   string
	Conditions []string
}

// ListDeployments 列出 Deployment
func (c *Client) ListDeployments(ctx context.Context, namespace, labelSelector string) ([]DeployInfo, error) {
	opts := metav1.ListOptions{}
	if labelSelector != "" {
		opts.LabelSelector = labelSelector
	}

	deploys, err := c.Clientset.AppsV1().Deployments(namespace).List(ctx, opts)
	if err != nil {
		return nil, fmt.Errorf("列出 Deployment 失败: %w", err)
	}

	var result []DeployInfo
	for _, d := range deploys.Items {
		info := DeployInfo{
			Name:      d.Name,
			Namespace: d.Namespace,
			Replicas:  *d.Spec.Replicas,
			Ready:     d.Status.ReadyReplicas,
			Updated:   d.Status.UpdatedReplicas,
			Available: d.Status.AvailableReplicas,
			Age:       time.Since(d.CreationTimestamp.Time),
			Strategy:  string(d.Spec.Strategy.Type),
		}

		for _, c := range d.Spec.Template.Spec.Containers {
			info.Images = append(info.Images, c.Image)
		}

		for _, cond := range d.Status.Conditions {
			if cond.Status == "True" {
				info.Conditions = append(info.Conditions, string(cond.Type))
			}
		}

		result = append(result, info)
	}

	sort.Slice(result, func(i, j int) bool {
		return result[i].Name < result[j].Name
	})

	return result, nil
}

// ScaleDeployment 扩缩容
func (c *Client) ScaleDeployment(ctx context.Context, namespace, name string, replicas int32) error {
	scale, err := c.Clientset.AppsV1().Deployments(namespace).GetScale(ctx, name, metav1.GetOptions{})
	if err != nil {
		return fmt.Errorf("获取 scale 失败: %w", err)
	}

	scale.Spec.Replicas = replicas
	_, err = c.Clientset.AppsV1().Deployments(namespace).UpdateScale(ctx, name, scale, metav1.UpdateOptions{})
	if err != nil {
		return fmt.Errorf("更新 scale 失败: %w", err)
	}

	return nil
}

// RestartDeployment 重启 Deployment（通过更新注解触发滚动重启）
func (c *Client) RestartDeployment(ctx context.Context, namespace, name string) error {
	patch := fmt.Sprintf(`{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"%s"}}}}}`,
		time.Now().Format(time.RFC3339))

	_, err := c.Clientset.AppsV1().Deployments(namespace).Patch(ctx, name,
		types.StrategicMergePatchType, []byte(patch), metav1.PatchOptions{})
	if err != nil {
		return fmt.Errorf("重启 Deployment 失败: %w", err)
	}

	return nil
}

// UpdateImage 更新容器镜像
func (c *Client) UpdateImage(ctx context.Context, namespace, name, container, image string) error {
	deploy, err := c.Clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
	if err != nil {
		return fmt.Errorf("获取 Deployment 失败: %w", err)
	}

	found := false
	for i := range deploy.Spec.Template.Spec.Containers {
		if deploy.Spec.Template.Spec.Containers[i].Name == container {
			deploy.Spec.Template.Spec.Containers[i].Image = image
			found = true
			break
		}
	}

	if !found {
		return fmt.Errorf("容器 %s 未找到", container)
	}

	_, err = c.Clientset.AppsV1().Deployments(namespace).Update(ctx, deploy, metav1.UpdateOptions{})
	if err != nil {
		return fmt.Errorf("更新镜像失败: %w", err)
	}

	return nil
}

// RollbackDeployment 回滚到上一个版本
func (c *Client) RollbackDeployment(ctx context.Context, namespace, name string, revision int64) error {
	// 获取 ReplicaSet 历史
	rsList, err := c.Clientset.AppsV1().ReplicaSets(namespace).List(ctx, metav1.ListOptions{})
	if err != nil {
		return fmt.Errorf("获取 ReplicaSet 列表失败: %w", err)
	}

	deploy, err := c.Clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
	if err != nil {
		return fmt.Errorf("获取 Deployment 失败: %w", err)
	}

	// 找到目标版本的 ReplicaSet
	var targetRS *appsv1.ReplicaSet
	for i := range rsList.Items {
		rs := &rsList.Items[i]
		// 检查是否属于这个 Deployment
		for _, ref := range rs.OwnerReferences {
			if ref.Name == name {
				if rev, ok := rs.Annotations["deployment.kubernetes.io/revision"]; ok {
					if rev == fmt.Sprintf("%d", revision) || revision == 0 {
						targetRS = rs
					}
				}
			}
		}
	}

	if targetRS == nil {
		return fmt.Errorf("未找到版本 %d 的 ReplicaSet", revision)
	}

	// 使用目标 RS 的 template 更新 Deployment
	deploy.Spec.Template = targetRS.Spec.Template
	_, err = c.Clientset.AppsV1().Deployments(namespace).Update(ctx, deploy, metav1.UpdateOptions{})
	if err != nil {
		return fmt.Errorf("回滚失败: %w", err)
	}

	return nil
}
```

### 2.4 Node 资源统计

```go
// internal/k8s/node.go
// Node 资源统计
package k8s

import (
	"context"
	"fmt"
	"sort"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// NodeInfo Node 信息
type NodeInfo struct {
	Name         string
	Status       string
	Roles        string
	Age          time.Duration
	Version      string
	OS           string
	Arch         string
	KernelVersion string
	ContainerRuntime string

	// 资源容量
	CPUCapacity    resource.Quantity
	MemCapacity    resource.Quantity
	PodCapacity    int64

	// 资源已分配（Request）
	CPURequests    resource.Quantity
	MemRequests    resource.Quantity

	// 资源已分配（Limit）
	CPULimits      resource.Quantity
	MemLimits      resource.Quantity

	// 使用率
	CPUUsagePercent float64
	MemUsagePercent float64
	PodCount        int
}

// ListNodes 列出节点资源统计
func (c *Client) ListNodes(ctx context.Context) ([]NodeInfo, error) {
	nodes, err := c.Clientset.CoreV1().Nodes().List(ctx, metav1.ListOptions{})
	if err != nil {
		return nil, fmt.Errorf("列出节点失败: %w", err)
	}

	// 获取所有 Pod 以计算资源请求
	allPods, err := c.Clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{
		FieldSelector: "status.phase=Running",
	})
	if err != nil {
		return nil, fmt.Errorf("列出 Pod 失败: %w", err)
	}

	// 按节点汇总 Pod 资源请求
	nodeResources := make(map[string]*nodeResourceSum)
	for _, pod := range allPods.Items {
		nodeName := pod.Spec.NodeName
		if nodeName == "" {
			continue
		}

		if _, ok := nodeResources[nodeName]; !ok {
			nodeResources[nodeName] = &nodeResourceSum{}
		}
		nr := nodeResources[nodeName]
		nr.podCount++

		for _, container := range pod.Spec.Containers {
			if req := container.Resources.Requests; req != nil {
				if cpu, ok := req[corev1.ResourceCPU]; ok {
					nr.cpuRequests.Add(cpu)
				}
				if mem, ok := req[corev1.ResourceMemory]; ok {
					nr.memRequests.Add(mem)
				}
			}
			if lim := container.Resources.Limits; lim != nil {
				if cpu, ok := lim[corev1.ResourceCPU]; ok {
					nr.cpuLimits.Add(cpu)
				}
				if mem, ok := lim[corev1.ResourceMemory]; ok {
					nr.memLimits.Add(mem)
				}
			}
		}
	}

	var result []NodeInfo
	for _, node := range nodes.Items {
		info := NodeInfo{
			Name:             node.Name,
			Age:              time.Since(node.CreationTimestamp.Time),
			Version:          node.Status.NodeInfo.KubeletVersion,
			OS:               node.Status.NodeInfo.OperatingSystem,
			Arch:             node.Status.NodeInfo.Architecture,
			KernelVersion:    node.Status.NodeInfo.KernelVersion,
			ContainerRuntime: node.Status.NodeInfo.ContainerRuntimeVersion,
			CPUCapacity:      *node.Status.Allocatable.Cpu(),
			MemCapacity:      *node.Status.Allocatable.Memory(),
			PodCapacity:      node.Status.Allocatable.Pods().Value(),
		}

		// 节点状态
		info.Status = "NotReady"
		for _, cond := range node.Status.Conditions {
			if cond.Type == corev1.NodeReady && cond.Status == corev1.ConditionTrue {
				info.Status = "Ready"
			}
		}

		// 节点角色
		roles := []string{}
		for label := range node.Labels {
			if label == "node-role.kubernetes.io/control-plane" || label == "node-role.kubernetes.io/master" {
				roles = append(roles, "control-plane")
			}
			if label == "node-role.kubernetes.io/worker" {
				roles = append(roles, "worker")
			}
		}
		if len(roles) == 0 {
			info.Roles = "worker"
		} else {
			info.Roles = roles[0]
		}

		// 资源使用
		if nr, ok := nodeResources[node.Name]; ok {
			info.CPURequests = nr.cpuRequests
			info.MemRequests = nr.memRequests
			info.CPULimits = nr.cpuLimits
			info.MemLimits = nr.memLimits
			info.PodCount = nr.podCount

			cpuCap := info.CPUCapacity.MilliValue()
			if cpuCap > 0 {
				info.CPUUsagePercent = float64(nr.cpuRequests.MilliValue()) / float64(cpuCap) * 100
			}

			memCap := info.MemCapacity.Value()
			if memCap > 0 {
				info.MemUsagePercent = float64(nr.memRequests.Value()) / float64(memCap) * 100
			}
		}

		result = append(result, info)
	}

	sort.Slice(result, func(i, j int) bool {
		return result[i].Name < result[j].Name
	})

	return result, nil
}

type nodeResourceSum struct {
	podCount    int
	cpuRequests resource.Quantity
	memRequests resource.Quantity
	cpuLimits   resource.Quantity
	memLimits   resource.Quantity
}

// PrintNodeReport 打印节点报表
func PrintNodeReport(nodes []NodeInfo) string {
	var b fmt.Stringer = &nodeReportBuilder{nodes: nodes}
	return b.String()
}

type nodeReportBuilder struct {
	nodes []NodeInfo
}

func (b *nodeReportBuilder) String() string {
	var s string
	s += fmt.Sprintf("\n%-25s %-8s %-14s %-10s %-15s %-15s %-8s %-8s\n",
		"名称", "状态", "角色", "版本",
		"CPU(请求/容量)", "内存(请求/容量)", "CPU%", "内存%")
	s += fmt.Sprintf("%s\n", repeat("-", 110))

	var totalCPUCap, totalCPUReq int64
	var totalMemCap, totalMemReq int64
	totalPods := 0

	for _, n := range b.nodes {
		cpuReq := n.CPURequests.MilliValue()
		cpuCap := n.CPUCapacity.MilliValue()
		memReqGB := float64(n.MemRequests.Value()) / 1024 / 1024 / 1024
		memCapGB := float64(n.MemCapacity.Value()) / 1024 / 1024 / 1024

		totalCPUReq += cpuReq
		totalCPUCap += cpuCap
		totalMemReq += n.MemRequests.Value()
		totalMemCap += n.MemCapacity.Value()
		totalPods += n.PodCount

		name := n.Name
		if len(name) > 25 {
			name = name[:22] + "..."
		}

		s += fmt.Sprintf("%-25s %-8s %-14s %-10s %5dm/%-5dm   %5.1fG/%-5.1fG  %5.1f%%  %5.1f%%\n",
			name, n.Status, n.Roles, n.Version,
			cpuReq, cpuCap,
			memReqGB, memCapGB,
			n.CPUUsagePercent, n.MemUsagePercent)
	}

	s += fmt.Sprintf("%s\n", repeat("-", 110))
	totalCPUPct := float64(0)
	if totalCPUCap > 0 {
		totalCPUPct = float64(totalCPUReq) / float64(totalCPUCap) * 100
	}
	totalMemPct := float64(0)
	if totalMemCap > 0 {
		totalMemPct = float64(totalMemReq) / float64(totalMemCap) * 100
	}
	s += fmt.Sprintf("合计: %d 节点, %d Pod, CPU 使用率: %.1f%%, 内存使用率: %.1f%%\n",
		len(b.nodes), totalPods, totalCPUPct, totalMemPct)

	return s
}

func repeat(s string, n int) string {
	result := ""
	for i := 0; i < n; i++ {
		result += s
	}
	return result
}
```

### 2.5 Cobra 命令定义

```go
// internal/cmd/root.go
// 根命令和子命令定义
package cmd

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"time"

	"github.com/spf13/cobra"
)

var (
	kubeconfig string
	kubeCtx    string
	namespace  string
	output     string
	verbose    bool
)

// rootCmd 根命令
var RootCmd = &cobra.Command{
	Use:   "ktool",
	Short: "Kubernetes 运维增强工具",
	Long: `ktool 是一个 kubectl 增强版运维工具，
提供 Pod 诊断、Deployment 操作、Node 资源统计等高频运维功能。`,
}

func init() {
	RootCmd.PersistentFlags().StringVar(&kubeconfig, "kubeconfig", "", "kubeconfig 文件路径")
	RootCmd.PersistentFlags().StringVar(&kubeCtx, "context", "", "Kubernetes 上下文")
	RootCmd.PersistentFlags().StringVarP(&namespace, "namespace", "n", "default", "命名空间")
	RootCmd.PersistentFlags().StringVarP(&output, "output", "o", "table", "输出格式: table, json, yaml")
	RootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "详细输出")

	// 注册子命令
	RootCmd.AddCommand(podCmd)
	RootCmd.AddCommand(deployCmd)
	RootCmd.AddCommand(nodeCmd)
}

// === Pod 子命令 ===
var podCmd = &cobra.Command{
	Use:   "pod",
	Short: "Pod 相关操作",
}

var podListCmd = &cobra.Command{
	Use:   "list",
	Short: "列出 Pod",
	Example: `  ktool pod list -n kube-system
  ktool pod list -n default -l app=nginx`,
	RunE: func(cmd *cobra.Command, args []string) error {
		label, _ := cmd.Flags().GetString("label")
		fmt.Printf("列出 Pod: namespace=%s, label=%s\n", namespace, label)
		// 实际调用 k8s.Client.ListPods
		return nil
	},
}

var podDiagCmd = &cobra.Command{
	Use:   "diag [pod-name]",
	Short: "Pod 诊断",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		podName := args[0]
		fmt.Printf("诊断 Pod: %s/%s\n", namespace, podName)
		return nil
	},
}

var podLogsCmd = &cobra.Command{
	Use:   "logs [pod-name]",
	Short: "查看 Pod 日志",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		podName := args[0]
		lines, _ := cmd.Flags().GetInt64("lines")
		follow, _ := cmd.Flags().GetBool("follow")
		container, _ := cmd.Flags().GetString("container")
		fmt.Printf("查看日志: %s/%s (container=%s, lines=%d, follow=%v)\n",
			namespace, podName, container, lines, follow)
		return nil
	},
}

var podExecCmd = &cobra.Command{
	Use:   "exec [pod-name] -- [command...]",
	Short: "在 Pod 中执行命令",
	Args:  cobra.MinimumNArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		podName := args[0]
		command := args[1:]
		if len(command) == 0 {
			command = []string{"/bin/sh"}
		}
		fmt.Printf("执行命令: %s/%s: %v\n", namespace, podName, command)
		return nil
	},
}

// === Deploy 子命令 ===
var deployCmd = &cobra.Command{
	Use:     "deploy",
	Aliases: []string{"deployment"},
	Short:   "Deployment 相关操作",
}

var deployListCmd = &cobra.Command{
	Use:   "list",
	Short: "列出 Deployment",
	RunE: func(cmd *cobra.Command, args []string) error {
		fmt.Printf("列出 Deployment: namespace=%s\n", namespace)
		return nil
	},
}

var deployScaleCmd = &cobra.Command{
	Use:   "scale [name] [replicas]",
	Short: "扩缩容",
	Args:  cobra.ExactArgs(2),
	RunE: func(cmd *cobra.Command, args []string) error {
		name := args[0]
		replicas := args[1]
		fmt.Printf("扩缩容: %s/%s -> %s\n", namespace, name, replicas)
		return nil
	},
}

var deployRestartCmd = &cobra.Command{
	Use:   "restart [name]",
	Short: "重启 Deployment",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		name := args[0]
		fmt.Printf("重启: %s/%s\n", namespace, name)
		return nil
	},
}

var deployImageCmd = &cobra.Command{
	Use:   "image [name] [container=image]",
	Short: "更新容器镜像",
	Args:  cobra.ExactArgs(2),
	RunE: func(cmd *cobra.Command, args []string) error {
		name := args[0]
		image := args[1]
		fmt.Printf("更新镜像: %s/%s -> %s\n", namespace, name, image)
		return nil
	},
}

// === Node 子命令 ===
var nodeCmd = &cobra.Command{
	Use:   "node",
	Short: "Node 资源统计",
}

var nodeListCmd = &cobra.Command{
	Use:   "list",
	Short: "列出节点资源",
	RunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("列出节点资源统计...")
		return nil
	},
}

func init() {
	// Pod 子命令
	podCmd.AddCommand(podListCmd)
	podCmd.AddCommand(podDiagCmd)
	podCmd.AddCommand(podLogsCmd)
	podCmd.AddCommand(podExecCmd)

	podListCmd.Flags().StringP("label", "l", "", "标签选择器")
	podLogsCmd.Flags().Int64("lines", 100, "显示最后 N 行")
	podLogsCmd.Flags().BoolP("follow", "f", false, "持续跟踪")
	podLogsCmd.Flags().StringP("container", "c", "", "容器名称")

	// Deploy 子命令
	deployCmd.AddCommand(deployListCmd)
	deployCmd.AddCommand(deployScaleCmd)
	deployCmd.AddCommand(deployRestartCmd)
	deployCmd.AddCommand(deployImageCmd)

	// Node 子命令
	nodeCmd.AddCommand(nodeListCmd)
}

// contextWithTimeout 创建带超时的 context
func contextWithTimeout() (context.Context, context.CancelFunc) {
	return context.WithTimeout(context.Background(), 30*time.Second)
}

// getLogger 获取日志实例
func getLogger() *slog.Logger {
	level := slog.LevelInfo
	if verbose {
		level = slog.LevelDebug
	}
	return slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
		Level: level,
	}))
}
```

### 2.6 主程序入口

```go
// cmd/ktool/main.go
package main

import (
	"fmt"
	"os"
)

var (
	version   = "dev"
	gitCommit = "unknown"
)

func main() {
	// 这里引入 internal/cmd 包中的 RootCmd
	// import "github.com/myorg/ktool/internal/cmd"
	// cmd.RootCmd.Version = fmt.Sprintf("%s (commit: %s)", version, gitCommit)
	// if err := cmd.RootCmd.Execute(); err != nil {
	//     os.Exit(1)
	// }

	// 简化演示
	fmt.Printf("ktool %s (commit: %s)\n", version, gitCommit)
	fmt.Println()
	fmt.Println("命令示例:")
	fmt.Println("  ktool pod list -n kube-system")
	fmt.Println("  ktool pod diag my-pod -n default")
	fmt.Println("  ktool pod logs my-pod -f --lines 200")
	fmt.Println("  ktool deploy list -n production")
	fmt.Println("  ktool deploy scale my-app 3")
	fmt.Println("  ktool deploy restart my-app")
	fmt.Println("  ktool deploy image my-app web=nginx:1.25")
	fmt.Println("  ktool node list")
	os.Exit(0)
}
```

---

## 三、常见坑点

```
坑点 1：kubeconfig 路径硬编码
  应该支持环境变量 KUBECONFIG、命令行参数和默认路径。

坑点 2：namespace 忘记传递
  client-go 的大多数方法需要显式传 namespace。
  空字符串 "" 表示所有 namespace。

坑点 3：context 超时未设置
  Kubernetes API 调用可能很慢。
  所有调用都应该使用 context.WithTimeout。

坑点 4：Pod exec 需要 SPDY
  某些环境的网络策略可能阻止 SPDY 协议。
  kubectl exec 也有同样的问题。

坑点 5：metrics-server 可能未安装
  获取 Node/Pod 实时资源使用需要 metrics-server。
  代码中要优雅处理 metrics 不可用的情况。
```

---

## 四、速查表

| 命令 | 功能 | 对应 kubectl |
|---|---|---|
| `ktool pod list` | 列出 Pod | `kubectl get pods` |
| `ktool pod diag <pod>` | Pod 诊断 | `kubectl describe pod` |
| `ktool pod logs <pod>` | 查看日志 | `kubectl logs` |
| `ktool deploy list` | 列出 Deployment | `kubectl get deploy` |
| `ktool deploy scale <name> <n>` | 扩缩容 | `kubectl scale` |
| `ktool deploy restart <name>` | 重启 | `kubectl rollout restart` |
| `ktool node list` | 节点资源报表 | `kubectl top nodes` |
