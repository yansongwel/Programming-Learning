# client-go 完全指南 — Kubernetes 官方 Go 客户端

## 导读

`client-go` 是 Kubernetes 官方提供的 Go 语言客户端库，是与 Kubernetes API Server 交互的最底层也是最强大的方式。无论是编写 Operator、自动化工具、还是自定义控制器，`client-go` 都是必不可少的基础。

作为 SRE/DevOps 工程师，掌握 `client-go` 意味着你可以：
- 用 Go 程序自动化管理 Kubernetes 资源
- 实时监控集群状态变化
- 构建自定义诊断和运维工具
- 开发 Operator 控制器

本文基于 **Go 1.24**，系统讲解 `client-go` 的核心概念和用法，最后提供一个完整的 **Pod 异常自动诊断工具**。

### 你将学到

- client-go 架构与核心组件
- 集群外/集群内配置
- Clientset CRUD 操作（Pod/Deployment/Service/ConfigMap/Secret）
- 动态客户端（dynamic.Interface）
- Informer 机制详解（List-Watch、缓存、事件处理）
- SharedInformerFactory 与 Lister
- Label Selector 过滤
- Watch 实时监听
- SRE 实战：Pod 异常自动诊断工具

---

## 一、client-go 架构概述

```
┌─────────────────────────────────────────────────────┐
│                    你的 Go 程序                       │
│                                                     │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────┐│
│  │ Clientset │  │ Dynamic    │  │ Discovery        ││
│  │ (类型安全) │  │ Client     │  │ Client           ││
│  │          │  │ (任意资源)  │  │ (API 发现)        ││
│  └────┬─────┘  └─────┬──────┘  └────────┬─────────┘│
│       │              │                   │          │
│  ┌────┴──────────────┴───────────────────┴─────┐    │
│  │              REST Client                    │    │
│  └─────────────────────┬───────────────────────┘    │
│                        │                            │
│  ┌─────────────────────┴───────────────────────┐    │
│  │         Informer / Lister / Watch           │    │
│  │   (List-Watch 机制, 本地缓存, 事件分发)      │    │
│  └─────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────┘
                         │ HTTPS
                         ▼
              ┌─────────────────────┐
              │  Kubernetes API     │
              │  Server             │
              └─────────────────────┘
```

### 核心组件

| 组件 | 说明 |
|------|------|
| `Clientset` | 类型安全的客户端，每个 GVR 都有对应方法 |
| `DynamicClient` | 通用客户端，处理任意类型的资源（返回 `Unstructured`） |
| `DiscoveryClient` | 发现 API Server 支持的资源类型 |
| `Informer` | List-Watch 机制，本地缓存 + 事件回调 |
| `Lister` | 从本地缓存查询，不经过 API Server |
| `WorkQueue` | 工作队列，用于 Controller 模式 |

---

## 二、安装与依赖

```bash
# 初始化项目
go mod init k8s-client-demo

# 安装 client-go（版本需与你的 Kubernetes 集群版本匹配）
go get k8s.io/client-go@latest
go get k8s.io/api@latest
go get k8s.io/apimachinery@latest

# 常用子包
# k8s.io/client-go/kubernetes          -- Clientset
# k8s.io/client-go/tools/clientcmd     -- kubeconfig 加载
# k8s.io/client-go/rest                -- REST 客户端配置
# k8s.io/client-go/dynamic             -- 动态客户端
# k8s.io/client-go/informers           -- Informer 工厂
# k8s.io/client-go/tools/cache         -- 缓存工具
```

---

## 三、客户端配置与初始化

### 3.1 集群外配置（开发环境）

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"path/filepath"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 方式一：自动查找 kubeconfig 文件
	// 默认路径：~/.kube/config
	// 也可以通过 KUBECONFIG 环境变量指定
	home, _ := os.UserHomeDir()
	kubeconfig := filepath.Join(home, ".kube", "config")

	// 也可以从环境变量获取
	if envKubeconfig := os.Getenv("KUBECONFIG"); envKubeconfig != "" {
		kubeconfig = envKubeconfig
	}

	// 加载 kubeconfig 配置
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		log.Fatalf("加载 kubeconfig 失败: %v", err)
	}

	// 可选：调整客户端参数
	config.QPS = 100            // 每秒请求数限制
	config.Burst = 200          // 突发请求数
	config.Timeout = 30 * 1e9   // 超时（30秒）

	// 创建 Clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("创建 Clientset 失败: %v", err)
	}

	// 验证连接：获取集群版本
	version, err := clientset.Discovery().ServerVersion()
	if err != nil {
		log.Fatalf("获取集群版本失败: %v", err)
	}
	fmt.Printf("Kubernetes 集群版本: %s\n", version.GitVersion)

	// 列出所有命名空间
	namespaces, err := clientset.CoreV1().Namespaces().List(
		context.Background(),
		metav1.ListOptions{},
	)
	if err != nil {
		log.Fatalf("列出命名空间失败: %v", err)
	}
	fmt.Println("\n命名空间列表:")
	for _, ns := range namespaces.Items {
		fmt.Printf("  - %s (状态: %s)\n", ns.Name, ns.Status.Phase)
	}
}
```

### 3.2 集群内配置（Pod 内运行）

```go
package main

import (
	"context"
	"fmt"
	"log"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func main() {
	// 在 Pod 内运行时，使用 InClusterConfig
	// 自动读取 ServiceAccount Token 和 CA 证书
	// 路径：/var/run/secrets/kubernetes.io/serviceaccount/
	config, err := rest.InClusterConfig()
	if err != nil {
		log.Fatalf("获取集群内配置失败: %v", err)
	}

	// 调整 QPS 限制（默认值较低）
	config.QPS = 50
	config.Burst = 100

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("创建 Clientset 失败: %v", err)
	}

	// 获取当前 Pod 所在的命名空间
	// 在集群内可以读取 downward API 或 ServiceAccount 信息
	namespace := getNamespace()

	pods, err := clientset.CoreV1().Pods(namespace).List(
		context.Background(),
		metav1.ListOptions{},
	)
	if err != nil {
		log.Fatalf("列出 Pod 失败: %v", err)
	}
	fmt.Printf("命名空间 %s 中有 %d 个 Pod\n", namespace, len(pods.Items))
}

// getNamespace 获取当前命名空间
func getNamespace() string {
	// 方式一：从环境变量获取（推荐通过 Downward API 注入）
	if ns := os.Getenv("POD_NAMESPACE"); ns != "" {
		return ns
	}
	// 方式二：读取 ServiceAccount 命名空间文件
	data, err := os.ReadFile("/var/run/secrets/kubernetes.io/serviceaccount/namespace")
	if err == nil {
		return string(data)
	}
	return "default"
}
```

### 3.3 通用配置函数（自动检测环境）

```go
package main

import (
	"fmt"
	"log"
	"os"
	"path/filepath"

	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

// NewK8sClient 自动检测环境并创建客户端
// 在集群内自动使用 InClusterConfig，在集群外使用 kubeconfig
func NewK8sClient() (*kubernetes.Clientset, error) {
	var config *rest.Config
	var err error

	// 尝试集群内配置
	config, err = rest.InClusterConfig()
	if err != nil {
		// 集群外：使用 kubeconfig
		kubeconfig := os.Getenv("KUBECONFIG")
		if kubeconfig == "" {
			home, _ := os.UserHomeDir()
			kubeconfig = filepath.Join(home, ".kube", "config")
		}
		config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			return nil, fmt.Errorf("无法获取 Kubernetes 配置: %w", err)
		}
		log.Println("使用集群外配置 (kubeconfig)")
	} else {
		log.Println("使用集群内配置 (InClusterConfig)")
	}

	// 通用参数
	config.QPS = 50
	config.Burst = 100

	return kubernetes.NewForConfig(config)
}

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	version, _ := clientset.Discovery().ServerVersion()
	fmt.Printf("连接成功，集群版本: %s\n", version.GitVersion)
}
```

---

## 四、Clientset CRUD 操作

### 4.1 Pod 操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

func main() {
	clientset, err := NewK8sClient() // 使用上面定义的通用函数
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()
	namespace := "default"

	// ==================== 创建 Pod ====================
	pod := &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "debug-pod",
			Namespace: namespace,
			Labels: map[string]string{
				"app":  "debug",
				"team": "sre",
			},
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "debug",
					Image:   "busybox:latest",
					Command: []string{"sleep", "3600"},
					Resources: corev1.ResourceRequirements{
						Requests: corev1.ResourceList{
							corev1.ResourceCPU:    resource.MustParse("100m"),
							corev1.ResourceMemory: resource.MustParse("128Mi"),
						},
						Limits: corev1.ResourceList{
							corev1.ResourceCPU:    resource.MustParse("200m"),
							corev1.ResourceMemory: resource.MustParse("256Mi"),
						},
					},
				},
			},
			RestartPolicy: corev1.RestartPolicyNever,
		},
	}

	createdPod, err := clientset.CoreV1().Pods(namespace).Create(ctx, pod, metav1.CreateOptions{})
	if err != nil {
		log.Fatalf("创建 Pod 失败: %v", err)
	}
	fmt.Printf("Pod 创建成功: %s (UID: %s)\n", createdPod.Name, createdPod.UID)

	// ==================== 获取 Pod ====================
	gotPod, err := clientset.CoreV1().Pods(namespace).Get(ctx, "debug-pod", metav1.GetOptions{})
	if err != nil {
		log.Fatalf("获取 Pod 失败: %v", err)
	}
	fmt.Printf("Pod 状态: %s, 节点: %s, IP: %s\n",
		gotPod.Status.Phase, gotPod.Spec.NodeName, gotPod.Status.PodIP)

	// ==================== 列出 Pod ====================
	podList, err := clientset.CoreV1().Pods(namespace).List(ctx, metav1.ListOptions{
		LabelSelector: "team=sre", // 标签过滤
	})
	if err != nil {
		log.Fatalf("列出 Pod 失败: %v", err)
	}
	fmt.Printf("\nSRE 团队的 Pod (%d 个):\n", len(podList.Items))
	for _, p := range podList.Items {
		fmt.Printf("  %-30s %-12s %-15s %s\n",
			p.Name, p.Status.Phase, p.Status.PodIP, p.Spec.NodeName)
	}

	// ==================== 更新 Pod 标签 ====================
	gotPod.Labels["version"] = "v2"
	updatedPod, err := clientset.CoreV1().Pods(namespace).Update(ctx, gotPod, metav1.UpdateOptions{})
	if err != nil {
		log.Fatalf("更新 Pod 失败: %v", err)
	}
	fmt.Printf("\nPod 标签已更新: %v\n", updatedPod.Labels)

	// ==================== 删除 Pod ====================
	err = clientset.CoreV1().Pods(namespace).Delete(ctx, "debug-pod", metav1.DeleteOptions{})
	if err != nil {
		log.Fatalf("删除 Pod 失败: %v", err)
	}
	fmt.Println("Pod 已删除")
}
```

### 4.2 Deployment 操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

func int32Ptr(i int32) *int32 { return &i }

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()
	namespace := "default"

	// 创建 Deployment
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "web-app",
			Namespace: namespace,
			Labels: map[string]string{
				"app": "web",
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: int32Ptr(3), // 3 个副本
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "web",
				},
			},
			Strategy: appsv1.DeploymentStrategy{
				Type: appsv1.RollingUpdateDeploymentStrategyType,
				RollingUpdate: &appsv1.RollingUpdateDeployment{
					MaxUnavailable: &intstr.IntOrString{Type: intstr.String, StrVal: "25%"},
					MaxSurge:       &intstr.IntOrString{Type: intstr.String, StrVal: "25%"},
				},
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app":     "web",
						"version": "v1",
					},
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:  "web",
							Image: "nginx:1.25",
							Ports: []corev1.ContainerPort{
								{ContainerPort: 80, Protocol: corev1.ProtocolTCP},
							},
							Resources: corev1.ResourceRequirements{
								Requests: corev1.ResourceList{
									corev1.ResourceCPU:    resource.MustParse("100m"),
									corev1.ResourceMemory: resource.MustParse("128Mi"),
								},
								Limits: corev1.ResourceList{
									corev1.ResourceCPU:    resource.MustParse("500m"),
									corev1.ResourceMemory: resource.MustParse("512Mi"),
								},
							},
							// 就绪探针
							ReadinessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									HTTPGet: &corev1.HTTPGetAction{
										Path: "/",
										Port: intstr.FromInt(80),
									},
								},
								InitialDelaySeconds: 5,
								PeriodSeconds:       10,
							},
							// 存活探针
							LivenessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									HTTPGet: &corev1.HTTPGetAction{
										Path: "/",
										Port: intstr.FromInt(80),
									},
								},
								InitialDelaySeconds: 15,
								PeriodSeconds:       20,
							},
						},
					},
				},
			},
		},
	}

	createdDeploy, err := clientset.AppsV1().Deployments(namespace).Create(
		ctx, deployment, metav1.CreateOptions{},
	)
	if err != nil {
		log.Fatalf("创建 Deployment 失败: %v", err)
	}
	fmt.Printf("Deployment 创建成功: %s\n", createdDeploy.Name)

	// 更新副本数（扩缩容）
	createdDeploy.Spec.Replicas = int32Ptr(5)
	_, err = clientset.AppsV1().Deployments(namespace).Update(
		ctx, createdDeploy, metav1.UpdateOptions{},
	)
	if err != nil {
		log.Fatalf("扩容失败: %v", err)
	}
	fmt.Println("Deployment 已扩容到 5 个副本")

	// 更新镜像（滚动更新）
	createdDeploy.Spec.Template.Spec.Containers[0].Image = "nginx:1.26"
	_, err = clientset.AppsV1().Deployments(namespace).Update(
		ctx, createdDeploy, metav1.UpdateOptions{},
	)
	if err != nil {
		log.Fatalf("滚动更新失败: %v", err)
	}
	fmt.Println("Deployment 滚动更新到 nginx:1.26")

	// 列出 Deployment
	deployList, err := clientset.AppsV1().Deployments(namespace).List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		log.Fatalf("列出 Deployment 失败: %v", err)
	}
	fmt.Printf("\nDeployment 列表:\n")
	for _, d := range deployList.Items {
		fmt.Printf("  %-25s 期望: %d  就绪: %d  可用: %d\n",
			d.Name, *d.Spec.Replicas, d.Status.ReadyReplicas, d.Status.AvailableReplicas)
	}

	// 删除 Deployment
	propagation := metav1.DeletePropagationForeground // 前台级联删除
	err = clientset.AppsV1().Deployments(namespace).Delete(ctx, "web-app", metav1.DeleteOptions{
		PropagationPolicy: &propagation,
	})
	if err != nil {
		log.Fatalf("删除 Deployment 失败: %v", err)
	}
	fmt.Println("Deployment 已删除（级联删除关联 Pod）")
}
```

### 4.3 Service 操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/util/intstr"
	"k8s.io/client-go/kubernetes"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()
	namespace := "default"

	// 创建 ClusterIP Service
	service := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "web-svc",
			Namespace: namespace,
			Labels: map[string]string{
				"app": "web",
			},
		},
		Spec: corev1.ServiceSpec{
			Type: corev1.ServiceTypeClusterIP,
			Selector: map[string]string{
				"app": "web",
			},
			Ports: []corev1.ServicePort{
				{
					Name:       "http",
					Port:       80,
					TargetPort: intstr.FromInt(80),
					Protocol:   corev1.ProtocolTCP,
				},
			},
		},
	}

	createdSvc, err := clientset.CoreV1().Services(namespace).Create(
		ctx, service, metav1.CreateOptions{},
	)
	if err != nil {
		log.Fatalf("创建 Service 失败: %v", err)
	}
	fmt.Printf("Service 创建成功: %s, ClusterIP: %s\n",
		createdSvc.Name, createdSvc.Spec.ClusterIP)

	// 获取 Service
	svc, err := clientset.CoreV1().Services(namespace).Get(ctx, "web-svc", metav1.GetOptions{})
	if err != nil {
		log.Fatalf("获取 Service 失败: %v", err)
	}
	fmt.Printf("Service 类型: %s, 端口: %v\n", svc.Spec.Type, svc.Spec.Ports)

	// 列出所有 Service
	svcList, err := clientset.CoreV1().Services("").List(ctx, metav1.ListOptions{})
	if err != nil {
		log.Fatalf("列出 Service 失败: %v", err)
	}
	fmt.Printf("\n所有 Service (%d 个):\n", len(svcList.Items))
	for _, s := range svcList.Items {
		fmt.Printf("  %-30s %-15s %-15s %s\n",
			s.Namespace+"/"+s.Name, s.Spec.Type, s.Spec.ClusterIP,
			formatPorts(s.Spec.Ports))
	}
}

func formatPorts(ports []corev1.ServicePort) string {
	var result []string
	for _, p := range ports {
		result = append(result, fmt.Sprintf("%d/%s", p.Port, p.Protocol))
	}
	return strings.Join(result, ",")
}
```

### 4.4 ConfigMap 和 Secret 操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()
	namespace := "default"

	// ==================== ConfigMap ====================
	configMap := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "app-config",
			Namespace: namespace,
		},
		Data: map[string]string{
			"database.host":     "mysql.db.svc.cluster.local",
			"database.port":     "3306",
			"database.name":     "myapp",
			"log.level":         "info",
			"feature.new_ui":    "true",
		},
		// 二进制数据
		BinaryData: map[string][]byte{
			"ca.crt": []byte("-----BEGIN CERTIFICATE-----\n..."),
		},
	}

	createdCM, err := clientset.CoreV1().ConfigMaps(namespace).Create(
		ctx, configMap, metav1.CreateOptions{},
	)
	if err != nil {
		log.Fatalf("创建 ConfigMap 失败: %v", err)
	}
	fmt.Printf("ConfigMap 创建成功: %s\n", createdCM.Name)

	// 更新 ConfigMap
	createdCM.Data["log.level"] = "debug"
	_, err = clientset.CoreV1().ConfigMaps(namespace).Update(
		ctx, createdCM, metav1.UpdateOptions{},
	)
	if err != nil {
		log.Fatalf("更新 ConfigMap 失败: %v", err)
	}
	fmt.Println("ConfigMap 已更新")

	// ==================== Secret ====================
	secret := &corev1.Secret{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "app-secret",
			Namespace: namespace,
		},
		Type: corev1.SecretTypeOpaque,
		// StringData 会自动 base64 编码
		StringData: map[string]string{
			"database.username": "admin",
			"database.password": "s3cr3t-p@ssw0rd",
			"api.key":           "ak-xxxxxxxxxxxx",
		},
	}

	createdSecret, err := clientset.CoreV1().Secrets(namespace).Create(
		ctx, secret, metav1.CreateOptions{},
	)
	if err != nil {
		log.Fatalf("创建 Secret 失败: %v", err)
	}
	fmt.Printf("Secret 创建成功: %s (类型: %s)\n", createdSecret.Name, createdSecret.Type)

	// 读取 Secret（data 是 base64 编码的）
	gotSecret, err := clientset.CoreV1().Secrets(namespace).Get(
		ctx, "app-secret", metav1.GetOptions{},
	)
	if err != nil {
		log.Fatalf("获取 Secret 失败: %v", err)
	}
	for key, value := range gotSecret.Data {
		fmt.Printf("  %s = %s\n", key, string(value)) // Data 已经是解码后的 []byte
	}

	// 清理
	clientset.CoreV1().ConfigMaps(namespace).Delete(ctx, "app-config", metav1.DeleteOptions{})
	clientset.CoreV1().Secrets(namespace).Delete(ctx, "app-secret", metav1.DeleteOptions{})
	fmt.Println("资源已清理")
}
```

---

## 五、动态客户端（Dynamic Client）

动态客户端可以处理任意资源类型，包括 CRD。它返回 `unstructured.Unstructured` 对象（本质上是 `map[string]interface{}`）。

```go
package main

import (
	"context"
	"fmt"
	"log"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
)

func main() {
	config, err := GetK8sConfig() // 复用通用配置函数
	if err != nil {
		log.Fatal(err)
	}

	// 创建动态客户端
	dynClient, err := dynamic.NewForConfig(config)
	if err != nil {
		log.Fatalf("创建动态客户端失败: %v", err)
	}

	ctx := context.Background()

	// 定义资源的 GVR（GroupVersionResource）
	// 内置资源示例：Pod
	podGVR := schema.GroupVersionResource{
		Group:    "",        // core group 为空
		Version:  "v1",
		Resource: "pods",
	}

	// 列出所有 Pod
	podList, err := dynClient.Resource(podGVR).Namespace("default").List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		log.Fatalf("列出 Pod 失败: %v", err)
	}

	fmt.Printf("Pod 数量: %d\n", len(podList.Items))
	for _, item := range podList.Items {
		name := item.GetName()
		namespace := item.GetNamespace()
		// 从 unstructured 中提取嵌套字段
		phase, _, _ := unstructured.NestedString(item.Object, "status", "phase")
		fmt.Printf("  %s/%s - %s\n", namespace, name, phase)
	}

	// CRD 资源示例：VirtualService (Istio)
	vsGVR := schema.GroupVersionResource{
		Group:    "networking.istio.io",
		Version:  "v1beta1",
		Resource: "virtualservices",
	}

	// 创建 CRD 资源
	vs := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "networking.istio.io/v1beta1",
			"kind":       "VirtualService",
			"metadata": map[string]interface{}{
				"name":      "my-vs",
				"namespace": "default",
			},
			"spec": map[string]interface{}{
				"hosts": []interface{}{"my-service"},
				"http": []interface{}{
					map[string]interface{}{
						"route": []interface{}{
							map[string]interface{}{
								"destination": map[string]interface{}{
									"host": "my-service",
									"port": map[string]interface{}{
										"number": int64(80),
									},
								},
							},
						},
					},
				},
			},
		},
	}

	createdVS, err := dynClient.Resource(vsGVR).Namespace("default").Create(
		ctx, vs, metav1.CreateOptions{},
	)
	if err != nil {
		log.Printf("创建 VirtualService 失败（可能未安装 Istio）: %v", err)
	} else {
		fmt.Printf("VirtualService 创建成功: %s\n", createdVS.GetName())
	}

	// 使用动态客户端获取任意资源
	deployGVR := schema.GroupVersionResource{
		Group:    "apps",
		Version:  "v1",
		Resource: "deployments",
	}

	deploys, err := dynClient.Resource(deployGVR).Namespace("").List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		log.Fatalf("列出 Deployment 失败: %v", err)
	}
	fmt.Printf("\n所有 Deployment:\n")
	for _, d := range deploys.Items {
		replicas, _, _ := unstructured.NestedInt64(d.Object, "spec", "replicas")
		fmt.Printf("  %s/%s - 副本数: %d\n", d.GetNamespace(), d.GetName(), replicas)
	}
}
```

---

## 六、Informer 机制详解

### 6.1 Informer 原理

Informer 是 client-go 最核心的机制，它实现了：

```
┌─────────────────────────────────────────────────────────┐
│                     Informer 架构                        │
│                                                         │
│  ┌─────────┐    ┌──────────┐    ┌────────────────────┐  │
│  │ Reflector│───>│ DeltaFIFO│───>│    Indexer/Store   │  │
│  │          │    │  (队列)   │    │    (本地缓存)      │  │
│  │ List     │    │          │    │                    │  │
│  │ Watch    │    │ Add      │    │  ┌──────────────┐  │  │
│  │          │    │ Update   │    │  │ Thread Safe  │  │  │
│  └────┬─────┘    │ Delete   │    │  │ Store (Map)  │  │  │
│       │         │ Sync     │    │  └──────────────┘  │  │
│       │         └─────┬────┘    └────────────────────┘  │
│       │               │                                 │
│  API Server       事件分发                               │
│                       │                                 │
│              ┌────────┴─────────┐                       │
│              │  Event Handlers  │                       │
│              │  - AddFunc       │                       │
│              │  - UpdateFunc    │                       │
│              │  - DeleteFunc    │                       │
│              └──────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

**核心流程：**
1. **Reflector** 通过 List-Watch 机制从 API Server 获取资源
2. 首次 List 获取全量数据，然后 Watch 增量变化
3. 变化事件放入 **DeltaFIFO** 队列
4. 消费者从队列取出事件，更新 **Indexer**（本地缓存）
5. 同时触发注册的 **Event Handler**（AddFunc/UpdateFunc/DeleteFunc）

### 6.2 基本 Informer 使用

```go
package main

import (
	"fmt"
	"log"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	// 创建 SharedInformerFactory
	// resyncPeriod: 重新同步间隔（0 表示不定期重新同步）
	factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)

	// 获取 Pod Informer
	podInformer := factory.Core().V1().Pods().Informer()

	// 注册事件处理器
	podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		// 新增 Pod
		AddFunc: func(obj interface{}) {
			pod := obj.(*corev1.Pod)
			fmt.Printf("[新增] Pod: %s/%s, 状态: %s\n",
				pod.Namespace, pod.Name, pod.Status.Phase)
		},
		// 更新 Pod
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldPod := oldObj.(*corev1.Pod)
			newPod := newObj.(*corev1.Pod)
			if oldPod.Status.Phase != newPod.Status.Phase {
				fmt.Printf("[更新] Pod: %s/%s, 状态: %s -> %s\n",
					newPod.Namespace, newPod.Name,
					oldPod.Status.Phase, newPod.Status.Phase)
			}
		},
		// 删除 Pod
		DeleteFunc: func(obj interface{}) {
			pod := obj.(*corev1.Pod)
			fmt.Printf("[删除] Pod: %s/%s\n", pod.Namespace, pod.Name)
		},
	})

	// 启动 Informer（必须在注册 Handler 之后）
	stopCh := make(chan struct{})
	defer close(stopCh)

	factory.Start(stopCh)

	// 等待缓存同步完成
	fmt.Println("等待缓存同步...")
	if !cache.WaitForCacheSync(stopCh, podInformer.HasSynced) {
		log.Fatal("缓存同步失败")
	}
	fmt.Println("缓存同步完成，开始监听事件...")

	// 阻塞等待
	<-stopCh
}
```

### 6.3 SharedInformerFactory 与 Lister

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	corelister "k8s.io/client-go/listers/core/v1"
	"k8s.io/client-go/tools/cache"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	// 创建带命名空间过滤的 InformerFactory
	// 只关注 kube-system 命名空间（减少内存消耗）
	factory := informers.NewSharedInformerFactoryWithOptions(
		clientset,
		30*time.Second,
		informers.WithNamespace("kube-system"), // 限定命名空间
	)

	// 获取 Pod Lister（从缓存查询，不经过 API Server）
	podLister := factory.Core().V1().Pods().Lister()

	// 获取 Node Informer（全局资源，不受命名空间限制）
	nodeFactory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
	nodeLister := nodeFactory.Core().V1().Nodes().Lister()

	// 启动
	stopCh := make(chan struct{})
	defer close(stopCh)

	factory.Start(stopCh)
	nodeFactory.Start(stopCh)

	// 等待缓存同步
	factory.WaitForCacheSync(stopCh)
	nodeFactory.WaitForCacheSync(stopCh)

	fmt.Println("缓存同步完成")

	// 使用 Lister 查询（从本地缓存，极快）
	// 列出所有 Pod
	allPods, err := podLister.List(labels.Everything())
	if err != nil {
		log.Fatalf("Lister 查询失败: %v", err)
	}
	fmt.Printf("\nkube-system 中的 Pod 数量: %d\n", len(allPods))

	// 按标签过滤
	selector, _ := labels.Parse("k8s-app=kube-dns")
	dnsPods, err := podLister.List(selector)
	if err != nil {
		log.Fatalf("标签过滤失败: %v", err)
	}
	fmt.Printf("DNS Pod 数量: %d\n", len(dnsPods))
	for _, pod := range dnsPods {
		fmt.Printf("  %s - %s - %s\n", pod.Name, pod.Status.Phase, pod.Status.PodIP)
	}

	// 按命名空间查询
	nsPods, err := podLister.Pods("kube-system").List(labels.Everything())
	if err != nil {
		log.Fatalf("命名空间查询失败: %v", err)
	}
	fmt.Printf("\nkube-system Pod: %d 个\n", len(nsPods))

	// 使用 Node Lister
	nodes, err := nodeLister.List(labels.Everything())
	if err != nil {
		log.Fatalf("列出 Node 失败: %v", err)
	}
	fmt.Printf("\n集群节点:\n")
	for _, node := range nodes {
		ready := "NotReady"
		for _, cond := range node.Status.Conditions {
			if cond.Type == corev1.NodeReady && cond.Status == corev1.ConditionTrue {
				ready = "Ready"
			}
		}
		fmt.Printf("  %-30s %s\n", node.Name, ready)
	}
}
```

### 6.4 自定义 Informer（带过滤器）

```go
package main

import (
	"fmt"
	"log"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	// 创建带字段选择器的 ListWatch
	// 只监控特定节点上的 Pod
	listWatcher := cache.NewListWatchFromClient(
		clientset.CoreV1().RESTClient(),
		"pods",
		"default",                                              // 命名空间
		fields.OneTermEqualSelector("spec.nodeName", "node-1"), // 字段选择器
	)

	// 创建 Informer
	informer := cache.NewInformer(
		listWatcher,
		&corev1.Pod{},    // 资源类型
		30*time.Second,    // 重新同步间隔
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				pod := obj.(*corev1.Pod)
				fmt.Printf("[新增] node-1 上的 Pod: %s\n", pod.Name)
			},
			UpdateFunc: func(old, new interface{}) {
				pod := new.(*corev1.Pod)
				fmt.Printf("[更新] node-1 上的 Pod: %s, 状态: %s\n",
					pod.Name, pod.Status.Phase)
			},
			DeleteFunc: func(obj interface{}) {
				pod := obj.(*corev1.Pod)
				fmt.Printf("[删除] node-1 上的 Pod: %s\n", pod.Name)
			},
		},
	)

	stopCh := make(chan struct{})
	defer close(stopCh)

	fmt.Println("开始监控 node-1 上的 Pod...")
	informer.Run(stopCh)
}
```

---

## 七、Watch 实时监听

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/watch"
	"k8s.io/client-go/kubernetes"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()

	// 创建 Watcher
	watcher, err := clientset.CoreV1().Events("").Watch(ctx, metav1.ListOptions{
		// 只关注最近的事件
		TimeoutSeconds: int64Ptr(300), // 5 分钟超时
	})
	if err != nil {
		log.Fatalf("创建 Watcher 失败: %v", err)
	}
	defer watcher.Stop()

	fmt.Println("开始监听 Kubernetes 事件...")
	fmt.Println("─────────────────────────────────────────")

	for event := range watcher.ResultChan() {
		switch event.Type {
		case watch.Added, watch.Modified:
			e, ok := event.Object.(*corev1.Event)
			if !ok {
				continue
			}

			// 根据事件类型标记严重程度
			severity := "INFO"
			if e.Type == "Warning" {
				severity = "WARN"
			}

			fmt.Printf("[%s][%s] %s/%s: %s (原因: %s, 计数: %d)\n",
				severity,
				e.LastTimestamp.Format("15:04:05"),
				e.InvolvedObject.Namespace,
				e.InvolvedObject.Name,
				e.Message,
				e.Reason,
				e.Count,
			)

		case watch.Deleted:
			// 事件被清理
		case watch.Error:
			log.Printf("Watch 错误: %v", event.Object)
		}
	}
}

func int64Ptr(i int64) *int64 { return &i }
```

---

## 八、Label Selector 过滤

```go
package main

import (
	"context"
	"fmt"
	"log"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/selection"
	"k8s.io/client-go/kubernetes"
)

func main() {
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()

	// ==================== 字符串方式 ====================

	// 等值匹配
	pods1, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: "app=web",
	})
	fmt.Printf("app=web 的 Pod: %d\n", len(pods1.Items))

	// 不等于
	pods2, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: "app!=web",
	})
	fmt.Printf("app!=web 的 Pod: %d\n", len(pods2.Items))

	// 集合匹配（in/notin）
	pods3, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: "env in (production, staging)",
	})
	fmt.Printf("production/staging 环境的 Pod: %d\n", len(pods3.Items))

	// 存在性检查
	pods4, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: "team",  // 有 team 标签的
	})
	fmt.Printf("有 team 标签的 Pod: %d\n", len(pods4.Items))

	// 组合条件
	pods5, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: "app=web,env=production,!canary",
	})
	fmt.Printf("web 生产环境非金丝雀 Pod: %d\n", len(pods5.Items))

	// ==================== 编程方式（推荐） ====================

	// 使用 labels.NewSelector 构建选择器
	selector := labels.NewSelector()

	// 添加条件：app = web
	reqApp, _ := labels.NewRequirement("app", selection.Equals, []string{"web"})
	selector = selector.Add(*reqApp)

	// 添加条件：env in (production, staging)
	reqEnv, _ := labels.NewRequirement("env", selection.In, []string{"production", "staging"})
	selector = selector.Add(*reqEnv)

	// 添加条件：version 存在
	reqVersion, _ := labels.NewRequirement("version", selection.Exists, nil)
	selector = selector.Add(*reqVersion)

	fmt.Printf("\n构建的选择器: %s\n", selector.String())

	pods6, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
		LabelSelector: selector.String(),
	})
	fmt.Printf("匹配的 Pod: %d\n", len(pods6.Items))
}
```

---

## 九、SRE 实战：Pod 异常自动诊断工具

这个工具实时监控 Pod 状态，自动诊断异常原因并生成报告。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
)

// ============================================================
// 诊断结果
// ============================================================

// DiagnosisResult Pod 诊断结果
type DiagnosisResult struct {
	PodName      string
	Namespace    string
	NodeName     string
	Timestamp    time.Time
	Severity     string   // Critical, Warning, Info
	Issues       []string // 发现的问题
	Suggestions  []string // 建议的修复方案
	ContainerLog string   // 相关容器日志
}

// String 格式化输出诊断结果
func (d *DiagnosisResult) String() string {
	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("\n╔══════════════════════════════════════════════════════════════\n"))
	sb.WriteString(fmt.Sprintf("║ Pod 诊断报告\n"))
	sb.WriteString(fmt.Sprintf("╠══════════════════════════════════════════════════════════════\n"))
	sb.WriteString(fmt.Sprintf("║ Pod:       %s/%s\n", d.Namespace, d.PodName))
	sb.WriteString(fmt.Sprintf("║ 节点:      %s\n", d.NodeName))
	sb.WriteString(fmt.Sprintf("║ 时间:      %s\n", d.Timestamp.Format("2006-01-02 15:04:05")))
	sb.WriteString(fmt.Sprintf("║ 严重程度:  %s\n", d.Severity))
	sb.WriteString(fmt.Sprintf("╠══════════════════════════════════════════════════════════════\n"))
	sb.WriteString(fmt.Sprintf("║ 发现的问题:\n"))
	for i, issue := range d.Issues {
		sb.WriteString(fmt.Sprintf("║   %d. %s\n", i+1, issue))
	}
	sb.WriteString(fmt.Sprintf("╠══════════════════════════════════════════════════════════════\n"))
	sb.WriteString(fmt.Sprintf("║ 建议修复方案:\n"))
	for i, suggestion := range d.Suggestions {
		sb.WriteString(fmt.Sprintf("║   %d. %s\n", i+1, suggestion))
	}
	if d.ContainerLog != "" {
		sb.WriteString(fmt.Sprintf("╠══════════════════════════════════════════════════════════════\n"))
		sb.WriteString(fmt.Sprintf("║ 容器日志（最后 10 行）:\n"))
		for _, line := range strings.Split(d.ContainerLog, "\n") {
			if line != "" {
				sb.WriteString(fmt.Sprintf("║   %s\n", line))
			}
		}
	}
	sb.WriteString(fmt.Sprintf("╚══════════════════════════════════════════════════════════════\n"))
	return sb.String()
}

// ============================================================
// Pod 诊断器
// ============================================================

// PodDiagnoser Pod 异常自动诊断工具
type PodDiagnoser struct {
	clientset *kubernetes.Clientset
	namespace string // 空字符串表示所有命名空间
	logger    *log.Logger

	// 记录已诊断的 Pod，避免重复诊断
	diagnosedPods map[string]time.Time
	mu            sync.Mutex
}

// NewPodDiagnoser 创建诊断器
func NewPodDiagnoser(clientset *kubernetes.Clientset, namespace string) *PodDiagnoser {
	return &PodDiagnoser{
		clientset:     clientset,
		namespace:     namespace,
		logger:        log.New(os.Stdout, "[PodDiagnoser] ", log.LstdFlags),
		diagnosedPods: make(map[string]time.Time),
	}
}

// Run 启动诊断器
func (pd *PodDiagnoser) Run(ctx context.Context) {
	pd.logger.Println("Pod 异常自动诊断工具启动")
	if pd.namespace == "" {
		pd.logger.Println("监控范围: 所有命名空间")
	} else {
		pd.logger.Printf("监控范围: %s 命名空间", pd.namespace)
	}

	// 创建 Informer
	var factory informers.SharedInformerFactory
	if pd.namespace != "" {
		factory = informers.NewSharedInformerFactoryWithOptions(
			pd.clientset, 30*time.Second,
			informers.WithNamespace(pd.namespace),
		)
	} else {
		factory = informers.NewSharedInformerFactory(pd.clientset, 30*time.Second)
	}

	podInformer := factory.Core().V1().Pods().Informer()

	// 注册事件处理器
	podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			pod := obj.(*corev1.Pod)
			pd.checkPod(ctx, pod)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			newPod := newObj.(*corev1.Pod)
			pd.checkPod(ctx, newPod)
		},
	})

	// 启动
	stopCh := make(chan struct{})
	go func() {
		<-ctx.Done()
		close(stopCh)
	}()

	factory.Start(stopCh)

	// 等待缓存同步
	if !cache.WaitForCacheSync(stopCh, podInformer.HasSynced) {
		pd.logger.Println("缓存同步失败")
		return
	}
	pd.logger.Println("缓存同步完成，开始监控...")

	// 启动后先扫描一次现有 Pod
	pd.scanExistingPods(ctx)

	// 定时清理诊断缓存
	ticker := time.NewTicker(5 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			pd.cleanDiagnosisCache()
		case <-ctx.Done():
			pd.logger.Println("诊断器已停止")
			return
		}
	}
}

// scanExistingPods 扫描已有的异常 Pod
func (pd *PodDiagnoser) scanExistingPods(ctx context.Context) {
	pd.logger.Println("扫描现有异常 Pod...")

	pods, err := pd.clientset.CoreV1().Pods(pd.namespace).List(ctx, metav1.ListOptions{})
	if err != nil {
		pd.logger.Printf("获取 Pod 列表失败: %v", err)
		return
	}

	abnormalCount := 0
	for _, pod := range pods.Items {
		if pd.isPodAbnormal(&pod) {
			abnormalCount++
			pd.diagnosePod(ctx, &pod)
		}
	}
	pd.logger.Printf("发现 %d 个异常 Pod", abnormalCount)
}

// checkPod 检查 Pod 是否需要诊断
func (pd *PodDiagnoser) checkPod(ctx context.Context, pod *corev1.Pod) {
	if !pd.isPodAbnormal(pod) {
		return
	}

	// 防抖：同一个 Pod 5 分钟内不重复诊断
	key := fmt.Sprintf("%s/%s", pod.Namespace, pod.Name)
	pd.mu.Lock()
	if lastTime, exists := pd.diagnosedPods[key]; exists {
		if time.Since(lastTime) < 5*time.Minute {
			pd.mu.Unlock()
			return
		}
	}
	pd.diagnosedPods[key] = time.Now()
	pd.mu.Unlock()

	pd.diagnosePod(ctx, pod)
}

// isPodAbnormal 判断 Pod 是否异常
func (pd *PodDiagnoser) isPodAbnormal(pod *corev1.Pod) bool {
	// 跳过已完成的 Job Pod
	if pod.Status.Phase == corev1.PodSucceeded {
		return false
	}

	// 检查 Pod Phase
	switch pod.Status.Phase {
	case corev1.PodFailed:
		return true
	case corev1.PodPending:
		// Pending 超过 5 分钟才视为异常
		if time.Since(pod.CreationTimestamp.Time) > 5*time.Minute {
			return true
		}
	}

	// 检查容器状态
	for _, cs := range pod.Status.ContainerStatuses {
		// 容器不断重启
		if cs.RestartCount > 5 {
			return true
		}
		// CrashLoopBackOff
		if cs.State.Waiting != nil && cs.State.Waiting.Reason == "CrashLoopBackOff" {
			return true
		}
		// OOMKilled
		if cs.LastTerminationState.Terminated != nil &&
			cs.LastTerminationState.Terminated.Reason == "OOMKilled" {
			return true
		}
		// ImagePullBackOff
		if cs.State.Waiting != nil &&
			(cs.State.Waiting.Reason == "ImagePullBackOff" ||
				cs.State.Waiting.Reason == "ErrImagePull") {
			return true
		}
	}

	// 检查 Conditions
	for _, cond := range pod.Status.Conditions {
		if cond.Type == corev1.PodScheduled && cond.Status == corev1.ConditionFalse {
			return true // 调度失败
		}
	}

	return false
}

// diagnosePod 诊断 Pod 问题
func (pd *PodDiagnoser) diagnosePod(ctx context.Context, pod *corev1.Pod) {
	result := &DiagnosisResult{
		PodName:   pod.Name,
		Namespace: pod.Namespace,
		NodeName:  pod.Spec.NodeName,
		Timestamp: time.Now(),
		Severity:  "Warning",
	}

	// ========== 1. 检查调度问题 ==========
	pd.checkSchedulingIssues(pod, result)

	// ========== 2. 检查镜像问题 ==========
	pd.checkImageIssues(pod, result)

	// ========== 3. 检查容器崩溃 ==========
	pd.checkContainerCrashes(pod, result)

	// ========== 4. 检查资源问题 ==========
	pd.checkResourceIssues(ctx, pod, result)

	// ========== 5. 检查 Pending 问题 ==========
	pd.checkPendingIssues(pod, result)

	// ========== 6. 获取相关事件 ==========
	pd.getRelatedEvents(ctx, pod, result)

	// ========== 7. 获取容器日志 ==========
	pd.getContainerLogs(ctx, pod, result)

	// 确定严重程度
	for _, cs := range pod.Status.ContainerStatuses {
		if cs.RestartCount > 10 {
			result.Severity = "Critical"
			break
		}
	}
	if pod.Status.Phase == corev1.PodFailed {
		result.Severity = "Critical"
	}

	// 输出诊断结果
	if len(result.Issues) > 0 {
		fmt.Print(result.String())
	}
}

// checkSchedulingIssues 检查调度问题
func (pd *PodDiagnoser) checkSchedulingIssues(pod *corev1.Pod, result *DiagnosisResult) {
	for _, cond := range pod.Status.Conditions {
		if cond.Type == corev1.PodScheduled && cond.Status == corev1.ConditionFalse {
			result.Issues = append(result.Issues,
				fmt.Sprintf("Pod 调度失败: %s", cond.Message))

			// 分析调度失败原因
			msg := cond.Message
			if strings.Contains(msg, "Insufficient cpu") {
				result.Suggestions = append(result.Suggestions,
					"集群 CPU 资源不足，考虑扩容节点或减少 Pod 的 CPU 请求")
			}
			if strings.Contains(msg, "Insufficient memory") {
				result.Suggestions = append(result.Suggestions,
					"集群内存资源不足，考虑扩容节点或减少 Pod 的内存请求")
			}
			if strings.Contains(msg, "node(s) had taint") {
				result.Suggestions = append(result.Suggestions,
					"节点有污点（taint），检查 Pod 是否配置了对应的容忍（toleration）")
			}
			if strings.Contains(msg, "node(s) didn't match") {
				result.Suggestions = append(result.Suggestions,
					"没有节点匹配 Pod 的亲和性/节点选择器要求，检查 nodeSelector 和 affinity 配置")
			}
			if strings.Contains(msg, "persistentvolumeclaim") {
				result.Suggestions = append(result.Suggestions,
					"PVC 未绑定，检查 StorageClass 和 PV 是否可用")
			}
		}
	}
}

// checkImageIssues 检查镜像问题
func (pd *PodDiagnoser) checkImageIssues(pod *corev1.Pod, result *DiagnosisResult) {
	for _, cs := range pod.Status.ContainerStatuses {
		if cs.State.Waiting == nil {
			continue
		}

		switch cs.State.Waiting.Reason {
		case "ImagePullBackOff", "ErrImagePull":
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 镜像拉取失败: %s (镜像: %s)",
					cs.Name, cs.State.Waiting.Message, cs.Image))

			result.Suggestions = append(result.Suggestions,
				"检查镜像名称和标签是否正确",
				"检查镜像仓库是否可访问（网络/防火墙）",
				"如果是私有仓库，检查 imagePullSecrets 是否配置正确",
				fmt.Sprintf("尝试手动拉取: docker pull %s", cs.Image),
			)

		case "InvalidImageName":
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 镜像名称无效: %s", cs.Name, cs.Image))
			result.Suggestions = append(result.Suggestions,
				"检查镜像名称格式是否正确")
		}
	}
}

// checkContainerCrashes 检查容器崩溃
func (pd *PodDiagnoser) checkContainerCrashes(pod *corev1.Pod, result *DiagnosisResult) {
	for _, cs := range pod.Status.ContainerStatuses {
		// CrashLoopBackOff
		if cs.State.Waiting != nil && cs.State.Waiting.Reason == "CrashLoopBackOff" {
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 处于 CrashLoopBackOff 状态 (重启次数: %d)",
					cs.Name, cs.RestartCount))

			result.Suggestions = append(result.Suggestions,
				"查看容器日志: kubectl logs "+pod.Name+" -c "+cs.Name+" --previous",
				"检查容器启动命令和参数是否正确",
				"检查容器依赖的服务（数据库/配置中心）是否可用",
			)
		}

		// OOMKilled
		if cs.LastTerminationState.Terminated != nil {
			term := cs.LastTerminationState.Terminated
			if term.Reason == "OOMKilled" {
				result.Severity = "Critical"
				result.Issues = append(result.Issues,
					fmt.Sprintf("容器 [%s] 因 OOM 被杀 (退出码: %d, 时间: %s)",
						cs.Name, term.ExitCode, term.FinishedAt.Format("15:04:05")))

				// 获取当前内存限制
				for _, container := range pod.Spec.Containers {
					if container.Name == cs.Name {
						memLimit := container.Resources.Limits.Memory()
						result.Suggestions = append(result.Suggestions,
							fmt.Sprintf("当前内存限制: %s，建议增大 resources.limits.memory",
								memLimit.String()),
							"分析应用内存使用模式，排查内存泄漏",
							"考虑使用 Go pprof 或 Java heap dump 分析内存",
						)
					}
				}
			}

			// 其他非零退出码
			if term.ExitCode != 0 && term.Reason != "OOMKilled" {
				result.Issues = append(result.Issues,
					fmt.Sprintf("容器 [%s] 异常退出 (退出码: %d, 原因: %s)",
						cs.Name, term.ExitCode, term.Reason))

				switch term.ExitCode {
				case 1:
					result.Suggestions = append(result.Suggestions,
						"退出码 1: 通用错误，查看容器日志了解具体原因")
				case 126:
					result.Suggestions = append(result.Suggestions,
						"退出码 126: 命令无法执行，检查文件权限")
				case 127:
					result.Suggestions = append(result.Suggestions,
						"退出码 127: 命令不存在，检查镜像中是否包含所需命令")
				case 137:
					result.Suggestions = append(result.Suggestions,
						"退出码 137 (SIGKILL): 容器被强制杀死，可能是 OOM 或手动删除")
				case 143:
					result.Suggestions = append(result.Suggestions,
						"退出码 143 (SIGTERM): 容器收到终止信号，检查是否是正常关闭")
				}
			}
		}

		// 持续重启
		if cs.RestartCount > 5 && cs.RestartCount <= 10 {
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 重启次数较多: %d 次", cs.Name, cs.RestartCount))
		}
		if cs.RestartCount > 10 {
			result.Severity = "Critical"
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 重启次数过多: %d 次，需要立即处理！",
					cs.Name, cs.RestartCount))
		}
	}
}

// checkResourceIssues 检查资源问题
func (pd *PodDiagnoser) checkResourceIssues(ctx context.Context, pod *corev1.Pod, result *DiagnosisResult) {
	for _, container := range pod.Spec.Containers {
		// 检查是否设置了资源限制
		if container.Resources.Limits.Cpu().IsZero() && container.Resources.Limits.Memory().IsZero() {
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 未设置资源限制（limits），可能导致资源争抢",
					container.Name))
			result.Suggestions = append(result.Suggestions,
				"建议为所有容器设置 resources.requests 和 resources.limits")
		}

		// 检查是否设置了资源请求
		if container.Resources.Requests.Cpu().IsZero() && container.Resources.Requests.Memory().IsZero() {
			result.Issues = append(result.Issues,
				fmt.Sprintf("容器 [%s] 未设置资源请求（requests），调度器无法合理分配",
					container.Name))
		}
	}
}

// checkPendingIssues 检查 Pending 问题
func (pd *PodDiagnoser) checkPendingIssues(pod *corev1.Pod, result *DiagnosisResult) {
	if pod.Status.Phase != corev1.PodPending {
		return
	}

	pendingDuration := time.Since(pod.CreationTimestamp.Time)
	if pendingDuration > 5*time.Minute {
		result.Issues = append(result.Issues,
			fmt.Sprintf("Pod 处于 Pending 状态已 %s", pendingDuration.Round(time.Second)))

		// 检查 Init 容器
		for _, cs := range pod.Status.InitContainerStatuses {
			if cs.State.Waiting != nil {
				result.Issues = append(result.Issues,
					fmt.Sprintf("Init 容器 [%s] 等待中: %s",
						cs.Name, cs.State.Waiting.Reason))
			}
		}
	}
}

// getRelatedEvents 获取相关事件
func (pd *PodDiagnoser) getRelatedEvents(ctx context.Context, pod *corev1.Pod, result *DiagnosisResult) {
	events, err := pd.clientset.CoreV1().Events(pod.Namespace).List(ctx, metav1.ListOptions{
		FieldSelector: fmt.Sprintf("involvedObject.name=%s,involvedObject.kind=Pod", pod.Name),
	})
	if err != nil {
		return
	}

	for _, e := range events.Items {
		if e.Type == "Warning" {
			result.Issues = append(result.Issues,
				fmt.Sprintf("[事件] %s: %s (次数: %d)",
					e.Reason, e.Message, e.Count))
		}
	}
}

// getContainerLogs 获取容器日志
func (pd *PodDiagnoser) getContainerLogs(ctx context.Context, pod *corev1.Pod, result *DiagnosisResult) {
	if pod.Status.Phase == corev1.PodPending {
		return // Pending 状态没有日志
	}

	for _, cs := range pod.Status.ContainerStatuses {
		if cs.RestartCount > 0 || (cs.State.Waiting != nil && cs.State.Waiting.Reason == "CrashLoopBackOff") {
			// 获取上一次运行的日志
			tailLines := int64(10)
			logOpts := &corev1.PodLogOptions{
				Container: cs.Name,
				TailLines: &tailLines,
				Previous:  true, // 获取上一次的日志
			}

			req := pd.clientset.CoreV1().Pods(pod.Namespace).GetLogs(pod.Name, logOpts)
			logStream, err := req.Stream(ctx)
			if err != nil {
				continue
			}

			buf := make([]byte, 4096)
			n, _ := logStream.Read(buf)
			logStream.Close()

			if n > 0 {
				result.ContainerLog = fmt.Sprintf("[容器: %s]\n%s", cs.Name, string(buf[:n]))
			}
			break // 只获取第一个异常容器的日志
		}
	}
}

// cleanDiagnosisCache 清理过期的诊断缓存
func (pd *PodDiagnoser) cleanDiagnosisCache() {
	pd.mu.Lock()
	defer pd.mu.Unlock()

	for key, ts := range pd.diagnosedPods {
		if time.Since(ts) > 10*time.Minute {
			delete(pd.diagnosedPods, key)
		}
	}
}

// ============================================================
// 主函数
// ============================================================

func main() {
	// 创建 Kubernetes 客户端
	clientset, err := NewK8sClient()
	if err != nil {
		log.Fatalf("创建客户端失败: %v", err)
	}

	// 可以通过环境变量指定命名空间，空字符串表示监控所有
	namespace := os.Getenv("WATCH_NAMESPACE")

	// 创建诊断器
	diagnoser := NewPodDiagnoser(clientset, namespace)

	// 设置优雅退出
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		sig := <-sigChan
		log.Printf("收到信号: %v，正在退出...", sig)
		cancel()
	}()

	// 启动诊断器
	diagnoser.Run(ctx)
}
```

---

## 常见坑点

### 1. 资源版本冲突（Conflict）

```go
// 坑：并发更新同一资源会冲突
pod, _ := clientset.CoreV1().Pods("default").Get(ctx, "my-pod", metav1.GetOptions{})
pod.Labels["key"] = "value"
_, err := clientset.CoreV1().Pods("default").Update(ctx, pod, metav1.UpdateOptions{})
// 可能报 "the object has been modified; please apply your changes to the latest version"

// 正确：使用重试机制
import "k8s.io/client-go/util/retry"

err := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    pod, err := clientset.CoreV1().Pods("default").Get(ctx, "my-pod", metav1.GetOptions{})
    if err != nil {
        return err
    }
    pod.Labels["key"] = "value"
    _, err = clientset.CoreV1().Pods("default").Update(ctx, pod, metav1.UpdateOptions{})
    return err
})
```

### 2. Informer 缓存未同步就开始使用

```go
// 坑：不等缓存同步就使用 Lister
factory.Start(stopCh)
pods, _ := podLister.List(labels.Everything()) // 可能返回空！

// 正确：等待缓存同步
factory.Start(stopCh)
factory.WaitForCacheSync(stopCh) // 必须等待！
pods, _ := podLister.List(labels.Everything())
```

### 3. Watch 连接断开未处理

```go
// 坑：Watch 连接会超时断开，不处理会丢事件
watcher, _ := clientset.CoreV1().Pods("").Watch(ctx, metav1.ListOptions{})
for event := range watcher.ResultChan() {
    // 连接断开后循环结束，丢失后续事件
}

// 正确：使用 Informer（自带重连机制），或者自己实现重试
// Informer 是生产环境的最佳选择
```

### 4. 不设置 QPS 限制

```go
// 坑：默认 QPS=5，高频操作会被限流
config, _ := rest.InClusterConfig()
// config.QPS = 5 (默认值)

// 正确：根据需求调整
config.QPS = 50
config.Burst = 100
```

### 5. 命名空间传空字符串的含义

```go
// 空字符串表示所有命名空间（跨命名空间操作）
pods, _ := clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
// 返回所有命名空间的 Pod

// 指定命名空间
pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
// 只返回 default 命名空间的 Pod
```

---

## 速查表

| 操作 | 代码 |
|------|------|
| 集群外配置 | `clientcmd.BuildConfigFromFlags("", kubeconfig)` |
| 集群内配置 | `rest.InClusterConfig()` |
| 创建 Clientset | `kubernetes.NewForConfig(config)` |
| 创建动态客户端 | `dynamic.NewForConfig(config)` |
| 列出 Pod | `clientset.CoreV1().Pods(ns).List(ctx, opts)` |
| 获取 Pod | `clientset.CoreV1().Pods(ns).Get(ctx, name, opts)` |
| 创建 Pod | `clientset.CoreV1().Pods(ns).Create(ctx, pod, opts)` |
| 更新 Pod | `clientset.CoreV1().Pods(ns).Update(ctx, pod, opts)` |
| 删除 Pod | `clientset.CoreV1().Pods(ns).Delete(ctx, name, opts)` |
| Watch | `clientset.CoreV1().Pods(ns).Watch(ctx, opts)` |
| 创建 Informer 工厂 | `informers.NewSharedInformerFactory(clientset, resync)` |
| 获取 Lister | `factory.Core().V1().Pods().Lister()` |
| 按标签过滤 | `LabelSelector: "app=web,env=prod"` |
| 冲突重试 | `retry.RetryOnConflict(retry.DefaultRetry, fn)` |
| 等待缓存同步 | `cache.WaitForCacheSync(stopCh, informer.HasSynced)` |

---

## 小结

本文系统讲解了 client-go 的核心用法：

1. **客户端配置**：集群内/外双模式，通用初始化函数适应不同环境
2. **Clientset CRUD**：Pod、Deployment、Service、ConfigMap、Secret 的完整增删改查
3. **动态客户端**：处理任意资源类型（包括 CRD），使用 Unstructured 操作
4. **Informer 机制**：List-Watch + 本地缓存 + 事件回调，生产级资源监控方案
5. **SharedInformerFactory + Lister**：共享缓存，高效查询
6. **Label Selector**：灵活的标签过滤，支持等值、集合、存在性检查
7. **SRE 实战**：完整的 Pod 异常自动诊断工具，覆盖调度、镜像、OOM、CrashLoop 等常见问题

掌握 client-go 是开发 Kubernetes Operator 和自动化运维工具的基础，也是 SRE 进阶的必备技能。
