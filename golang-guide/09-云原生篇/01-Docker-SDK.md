# Docker SDK for Go 完全指南

## 导读

Docker 是现代云原生架构的基石，而 Docker Engine API 则是与 Docker 守护进程通信的核心接口。作为 SRE/DevOps 工程师，我们经常需要通过编程方式管理容器、镜像、网络和卷等资源，而不是仅依赖 `docker` CLI。

Go 语言的 Docker SDK（`github.com/docker/docker/client`）是 Docker 官方提供的 Go 客户端库，它封装了 Docker Engine API 的所有功能，让我们可以在 Go 程序中直接操控 Docker。

本文基于 **Go 1.24**，面向有编程经验的 SRE/DevOps 工程师，系统讲解 Docker SDK 的核心用法，并提供一个完整的 **容器自动健康检查与重启工具** 实战项目。

### 你将学到

- Docker Engine API 架构概述
- Go Docker SDK 安装与客户端初始化
- 容器全生命周期管理（创建、启动、停止、删除、列表、详情、日志）
- 镜像操作（拉取、构建、推送、列表、删除）
- 网络与卷操作
- 实时日志流与事件监听
- SRE 实战：容器自动健康检查与重启工具

---

## 一、Docker Engine API 概述

Docker Engine API 是一个 RESTful API，Docker CLI 本身就是这个 API 的客户端。API 通过 Unix Socket（`/var/run/docker.sock`）或 TCP 端口与 Docker 守护进程通信。

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  docker CLI  │────>│  Docker Engine   │────>│  容器/镜像/网络   │
│  Go SDK      │     │  API (REST)      │     │  卷/插件等资源    │
│  curl/HTTP   │     │  unix:///var/run/ │     │                  │
│              │     │  docker.sock     │     │                  │
└──────────────┘     └─────────────────┘     └──────────────────┘
```

### API 版本协商

Docker API 有版本号（如 `v1.45`），客户端和服务端需要协商兼容的版本。Go SDK 支持自动版本协商。

---

## 二、Go Docker SDK 安装

### 初始化项目并安装依赖

```go
// 初始化 Go 模块
// go mod init docker-sdk-demo
// go get github.com/docker/docker/client
// go get github.com/docker/docker/api/types
// go get github.com/docker/docker/api/types/container
// go get github.com/docker/docker/api/types/network
// go get github.com/docker/docker/api/types/volume
// go get github.com/docker/docker/api/types/filters
// go get github.com/docker/docker/api/types/events
// go get github.com/docker/go-connections/nat
```

### 依赖说明

| 包 | 用途 |
|---|---|
| `github.com/docker/docker/client` | Docker 客户端核心库 |
| `github.com/docker/docker/api/types` | API 数据类型定义 |
| `github.com/docker/docker/api/types/container` | 容器相关类型 |
| `github.com/docker/docker/api/types/network` | 网络相关类型 |
| `github.com/docker/docker/api/types/volume` | 卷相关类型 |
| `github.com/docker/docker/api/types/filters` | 过滤器类型 |
| `github.com/docker/go-connections/nat` | 端口映射辅助 |

---

## 三、客户端初始化

### 基本初始化

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/client"
)

func main() {
	// 创建 Docker 客户端
	// 默认连接到 unix:///var/run/docker.sock
	cli, err := client.NewClientWithOpts(client.FromEnv)
	if err != nil {
		log.Fatalf("创建 Docker 客户端失败: %v", err)
	}
	defer cli.Close()

	// 获取 Docker 信息验证连接
	ctx := context.Background()
	info, err := cli.Info(ctx)
	if err != nil {
		log.Fatalf("获取 Docker 信息失败: %v", err)
	}

	fmt.Printf("Docker 版本: %s\n", info.ServerVersion)
	fmt.Printf("操作系统: %s\n", info.OperatingSystem)
	fmt.Printf("容器数量: %d\n", info.Containers)
	fmt.Printf("镜像数量: %d\n", info.Images)
}
```

### 带选项的客户端初始化

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/client"
)

func main() {
	// 方式一：从环境变量初始化（推荐）
	// 支持 DOCKER_HOST, DOCKER_TLS_VERIFY, DOCKER_CERT_PATH, DOCKER_API_VERSION
	cli1, err := client.NewClientWithOpts(client.FromEnv)
	if err != nil {
		log.Fatalf("方式一失败: %v", err)
	}
	defer cli1.Close()

	// 方式二：指定 Docker Host 地址
	cli2, err := client.NewClientWithOpts(
		client.WithHost("unix:///var/run/docker.sock"),
		client.WithAPIVersionNegotiation(), // 自动版本协商
	)
	if err != nil {
		log.Fatalf("方式二失败: %v", err)
	}
	defer cli2.Close()

	// 方式三：连接远程 Docker（TCP）
	cli3, err := client.NewClientWithOpts(
		client.WithHost("tcp://192.168.1.100:2376"),
		client.WithAPIVersionNegotiation(),
	)
	if err != nil {
		log.Fatalf("方式三失败: %v", err)
	}
	defer cli3.Close()

	// 方式四：指定 API 版本
	cli4, err := client.NewClientWithOpts(
		client.FromEnv,
		client.WithVersion("1.45"),
	)
	if err != nil {
		log.Fatalf("方式四失败: %v", err)
	}
	defer cli4.Close()

	// 验证连接
	ctx := context.Background()
	ping, err := cli1.Ping(ctx)
	if err != nil {
		log.Fatalf("Ping Docker 失败: %v", err)
	}
	fmt.Printf("API 版本: %s\n", ping.APIVersion)
	fmt.Printf("操作系统类型: %s\n", ping.OSType)
}
```

### 连接管理最佳实践

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/docker/docker/client"
)

// DockerManager 封装 Docker 客户端，提供连接管理
type DockerManager struct {
	cli *client.Client
}

// NewDockerManager 创建 Docker 管理器
func NewDockerManager() (*DockerManager, error) {
	cli, err := client.NewClientWithOpts(
		client.FromEnv,
		client.WithAPIVersionNegotiation(),
	)
	if err != nil {
		return nil, fmt.Errorf("创建 Docker 客户端失败: %w", err)
	}

	// 验证连接可用
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	_, err = cli.Ping(ctx)
	if err != nil {
		cli.Close()
		return nil, fmt.Errorf("连接 Docker 守护进程失败: %w", err)
	}

	return &DockerManager{cli: cli}, nil
}

// Close 关闭连接
func (dm *DockerManager) Close() error {
	return dm.cli.Close()
}

// Client 获取原始客户端（用于高级操作）
func (dm *DockerManager) Client() *client.Client {
	return dm.cli
}

func main() {
	mgr, err := NewDockerManager()
	if err != nil {
		log.Fatal(err)
	}
	defer mgr.Close()

	fmt.Println("Docker 客户端连接成功")
}
```

---

## 四、容器操作

### 4.1 创建并启动容器

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/network"
	"github.com/docker/docker/client"
	"github.com/docker/go-connections/nat"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 容器配置
	containerConfig := &container.Config{
		Image: "nginx:latest",
		// 环境变量
		Env: []string{
			"NGINX_HOST=example.com",
			"NGINX_PORT=80",
		},
		// 暴露端口
		ExposedPorts: nat.PortSet{
			"80/tcp": struct{}{},
		},
		// 标签（SRE 常用于标记和过滤）
		Labels: map[string]string{
			"app":     "web",
			"env":     "production",
			"team":    "sre",
			"managed": "go-sdk",
		},
		// 健康检查配置
		Healthcheck: &container.HealthConfig{
			Test:     []string{"CMD-SHELL", "curl -f http://localhost/ || exit 1"},
			Interval: 30 * 1e9, // 30 秒（纳秒为单位）
			Timeout:  10 * 1e9, // 10 秒
			Retries:  3,
		},
	}

	// 主机配置
	hostConfig := &container.HostConfig{
		// 端口映射：宿主机 8080 -> 容器 80
		PortBindings: nat.PortMap{
			"80/tcp": []nat.PortBinding{
				{HostIP: "0.0.0.0", HostPort: "8080"},
			},
		},
		// 资源限制（SRE 必备）
		Resources: container.Resources{
			Memory:   256 * 1024 * 1024, // 256MB 内存限制
			NanoCPUs: 500000000,         // 0.5 CPU
		},
		// 重启策略
		RestartPolicy: container.RestartPolicy{
			Name:              "on-failure",
			MaximumRetryCount: 5,
		},
		// 日志配置
		LogConfig: container.LogConfig{
			Type: "json-file",
			Config: map[string]string{
				"max-size": "10m",
				"max-file": "3",
			},
		},
	}

	// 网络配置
	networkConfig := &network.NetworkingConfig{}

	// 创建容器
	resp, err := cli.ContainerCreate(
		ctx,
		containerConfig,
		hostConfig,
		networkConfig,
		nil, // 平台信息（nil 使用默认）
		"my-nginx-server", // 容器名称
	)
	if err != nil {
		log.Fatalf("创建容器失败: %v", err)
	}

	fmt.Printf("容器创建成功，ID: %s\n", resp.ID[:12])

	// 如果有警告信息
	if len(resp.Warnings) > 0 {
		for _, w := range resp.Warnings {
			fmt.Printf("警告: %s\n", w)
		}
	}

	// 启动容器
	err = cli.ContainerStart(ctx, resp.ID, container.StartOptions{})
	if err != nil {
		log.Fatalf("启动容器失败: %v", err)
	}

	fmt.Println("容器启动成功")
}
```

### 4.2 停止和删除容器

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()
	containerName := "my-nginx-server"

	// 优雅停止容器（等待 30 秒后强制停止）
	timeout := 30
	stopOpts := container.StopOptions{
		Timeout: &timeout,
	}
	err = cli.ContainerStop(ctx, containerName, stopOpts)
	if err != nil {
		log.Printf("停止容器失败: %v", err)
	} else {
		fmt.Println("容器已停止")
	}

	// 删除容器
	removeOpts := container.RemoveOptions{
		RemoveVolumes: true,  // 同时删除关联卷
		Force:         false, // 不强制删除（已停止的容器）
	}
	err = cli.ContainerRemove(ctx, containerName, removeOpts)
	if err != nil {
		log.Printf("删除容器失败: %v", err)
	} else {
		fmt.Println("容器已删除")
	}

	// 强制删除运行中的容器
	forceRemoveOpts := container.RemoveOptions{
		RemoveVolumes: true,
		Force:         true, // 强制删除
	}
	err = cli.ContainerRemove(ctx, "some-running-container", forceRemoveOpts)
	if err != nil {
		log.Printf("强制删除失败: %v", err)
	}
}
```

### 4.3 列出容器

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 列出所有运行中的容器
	containers, err := cli.ContainerList(ctx, container.ListOptions{})
	if err != nil {
		log.Fatalf("列出容器失败: %v", err)
	}
	fmt.Printf("运行中的容器: %d 个\n", len(containers))
	printContainers(containers)

	// 列出所有容器（包括已停止的）
	allContainers, err := cli.ContainerList(ctx, container.ListOptions{All: true})
	if err != nil {
		log.Fatalf("列出所有容器失败: %v", err)
	}
	fmt.Printf("\n所有容器: %d 个\n", len(allContainers))
	printContainers(allContainers)

	// 使用过滤器：按标签过滤
	filterArgs := filters.NewArgs()
	filterArgs.Add("label", "team=sre")
	filterArgs.Add("status", "running")

	filteredContainers, err := cli.ContainerList(ctx, container.ListOptions{
		All:     true,
		Filters: filterArgs,
	})
	if err != nil {
		log.Fatalf("过滤容器失败: %v", err)
	}
	fmt.Printf("\nSRE 团队运行中的容器: %d 个\n", len(filteredContainers))
	printContainers(filteredContainers)
}

// printContainers 打印容器信息表格
func printContainers(containers []types.Container) {
	fmt.Printf("%-15s %-25s %-15s %-20s %s\n",
		"CONTAINER ID", "IMAGE", "STATUS", "CREATED", "NAMES")
	fmt.Println("─────────────────────────────────────────────────────────────────────────────")
	for _, c := range containers {
		created := time.Unix(c.Created, 0).Format("2006-01-02 15:04:05")
		name := ""
		if len(c.Names) > 0 {
			name = c.Names[0][1:] // 去掉前导斜杠
		}
		fmt.Printf("%-15s %-25s %-15s %-20s %s\n",
			c.ID[:12], c.Image, c.State, created, name)
	}
}
```

### 4.4 容器详情（Inspect）

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"

	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 获取容器详细信息
	inspect, err := cli.ContainerInspect(ctx, "my-nginx-server")
	if err != nil {
		log.Fatalf("获取容器详情失败: %v", err)
	}

	// 基本信息
	fmt.Printf("容器 ID:    %s\n", inspect.ID[:12])
	fmt.Printf("容器名称:   %s\n", inspect.Name[1:])
	fmt.Printf("镜像:       %s\n", inspect.Config.Image)
	fmt.Printf("状态:       %s\n", inspect.State.Status)
	fmt.Printf("运行中:     %v\n", inspect.State.Running)
	fmt.Printf("PID:        %d\n", inspect.State.Pid)
	fmt.Printf("启动时间:   %s\n", inspect.State.StartedAt)
	fmt.Printf("重启次数:   %d\n", inspect.RestartCount)

	// 网络信息
	if inspect.NetworkSettings != nil {
		for netName, netInfo := range inspect.NetworkSettings.Networks {
			fmt.Printf("网络 [%s]:  IP=%s, Gateway=%s\n",
				netName, netInfo.IPAddress, netInfo.Gateway)
		}
	}

	// 健康检查状态
	if inspect.State.Health != nil {
		fmt.Printf("健康状态:   %s\n", inspect.State.Health.Status)
		fmt.Printf("连续失败:   %d\n", inspect.State.Health.FailingStreak)
		if len(inspect.State.Health.Log) > 0 {
			lastCheck := inspect.State.Health.Log[len(inspect.State.Health.Log)-1]
			fmt.Printf("最近检查:   退出码=%d, 输出=%s\n",
				lastCheck.ExitCode, lastCheck.Output)
		}
	}

	// 资源限制
	fmt.Printf("内存限制:   %d MB\n", inspect.HostConfig.Memory/1024/1024)
	fmt.Printf("CPU 配额:   %.2f 核\n", float64(inspect.HostConfig.NanoCPUs)/1e9)

	// 输出完整 JSON（调试用）
	jsonData, _ := json.MarshalIndent(inspect, "", "  ")
	fmt.Printf("\n完整详情（JSON 前 500 字符）:\n%s...\n", string(jsonData)[:500])
}
```

### 4.5 容器日志

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"os"
	"time"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 获取容器日志（历史日志）
	logOpts := container.LogsOptions{
		ShowStdout: true,
		ShowStderr: true,
		Timestamps: true,            // 显示时间戳
		Since:      "2024-01-01",    // 从指定时间开始
		Tail:       "100",           // 只获取最后 100 行
		Follow:     false,           // 不跟踪新日志
	}

	reader, err := cli.ContainerLogs(ctx, "my-nginx-server", logOpts)
	if err != nil {
		log.Fatalf("获取容器日志失败: %v", err)
	}
	defer reader.Close()

	fmt.Println("===== 容器历史日志 =====")
	// 将日志输出到标准输出
	// 注意：Docker 日志流有 8 字节的头部标识 stdout/stderr
	_, err = io.Copy(os.Stdout, reader)
	if err != nil {
		log.Printf("读取日志失败: %v", err)
	}

	// 实时跟踪日志流（类似 docker logs -f）
	fmt.Println("\n===== 实时日志流 =====")
	followLogOpts := container.LogsOptions{
		ShowStdout: true,
		ShowStderr: true,
		Timestamps: true,
		Follow:     true, // 持续跟踪
		Tail:       "10", // 先显示最后 10 行
	}

	// 使用带超时的 context 控制日志流时长
	followCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	followReader, err := cli.ContainerLogs(followCtx, "my-nginx-server", followLogOpts)
	if err != nil {
		log.Fatalf("跟踪日志失败: %v", err)
	}
	defer followReader.Close()

	// 实时输出日志
	_, err = io.Copy(os.Stdout, followReader)
	if err != nil && err != context.DeadlineExceeded {
		log.Printf("日志流结束: %v", err)
	}

	fmt.Println("\n日志跟踪结束")
}
```

---

## 五、镜像操作

### 5.1 拉取镜像

```go
package main

import (
	"context"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"os"

	"github.com/docker/docker/api/types/image"
	"github.com/docker/docker/api/types/registry"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 拉取公共镜像
	fmt.Println("正在拉取 nginx:latest ...")
	reader, err := cli.ImagePull(ctx, "docker.io/library/nginx:latest", image.PullOptions{})
	if err != nil {
		log.Fatalf("拉取镜像失败: %v", err)
	}
	defer reader.Close()
	// 必须读取完 reader，否则拉取不会完成
	io.Copy(os.Stdout, reader)
	fmt.Println("\nnginx 拉取完成")

	// 拉取私有仓库镜像（需要认证）
	authConfig := registry.AuthConfig{
		Username:      "myuser",
		Password:      "mypassword",
		ServerAddress: "https://registry.example.com",
	}
	authJSON, _ := json.Marshal(authConfig)
	authStr := base64.URLEncoding.EncodeToString(authJSON)

	privateReader, err := cli.ImagePull(ctx, "registry.example.com/myapp:v1.0", image.PullOptions{
		RegistryAuth: authStr,
	})
	if err != nil {
		log.Printf("拉取私有镜像失败（示例）: %v", err)
	} else {
		defer privateReader.Close()
		io.Copy(os.Stdout, privateReader)
	}
}
```

### 5.2 构建镜像

```go
package main

import (
	"archive/tar"
	"bytes"
	"context"
	"fmt"
	"io"
	"log"
	"os"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 准备 Dockerfile 内容
	dockerfile := `FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o /app/server .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
`

	// 创建 tar 归档（Docker 构建需要 tar 格式的上下文）
	buf := new(bytes.Buffer)
	tw := tar.NewWriter(buf)

	// 将 Dockerfile 写入 tar
	header := &tar.Header{
		Name: "Dockerfile",
		Size: int64(len(dockerfile)),
	}
	if err := tw.WriteHeader(header); err != nil {
		log.Fatal(err)
	}
	if _, err := tw.Write([]byte(dockerfile)); err != nil {
		log.Fatal(err)
	}

	// 添加示例 Go 文件
	mainGo := `package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from Docker SDK built image!")
	})
	http.ListenAndServe(":8080", nil)
}
`
	goHeader := &tar.Header{
		Name: "main.go",
		Size: int64(len(mainGo)),
	}
	tw.WriteHeader(goHeader)
	tw.Write([]byte(mainGo))

	goMod := `module myapp
go 1.24
`
	modHeader := &tar.Header{
		Name: "go.mod",
		Size: int64(len(goMod)),
	}
	tw.WriteHeader(modHeader)
	tw.Write([]byte(goMod))

	tw.Close()

	// 构建镜像
	buildResp, err := cli.ImageBuild(ctx, buf, types.ImageBuildOptions{
		Tags:       []string{"myapp:v1.0", "myapp:latest"},
		Dockerfile: "Dockerfile",
		Remove:     true, // 构建成功后删除中间容器
		Labels: map[string]string{
			"maintainer": "sre-team",
			"build-tool": "go-docker-sdk",
		},
	})
	if err != nil {
		log.Fatalf("构建镜像失败: %v", err)
	}
	defer buildResp.Body.Close()

	// 输出构建日志
	fmt.Println("===== 构建日志 =====")
	io.Copy(os.Stdout, buildResp.Body)
	fmt.Println("\n镜像构建完成")
}
```

### 5.3 推送镜像

```go
package main

import (
	"context"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"os"

	"github.com/docker/docker/api/types/image"
	"github.com/docker/docker/api/types/registry"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 先给镜像打标签
	err = cli.ImageTag(ctx, "myapp:v1.0", "registry.example.com/myapp:v1.0")
	if err != nil {
		log.Fatalf("打标签失败: %v", err)
	}

	// 认证信息
	authConfig := registry.AuthConfig{
		Username:      "myuser",
		Password:      "mypassword",
		ServerAddress: "https://registry.example.com",
	}
	authJSON, _ := json.Marshal(authConfig)
	authStr := base64.URLEncoding.EncodeToString(authJSON)

	// 推送镜像
	pushResp, err := cli.ImagePush(ctx, "registry.example.com/myapp:v1.0", image.PushOptions{
		RegistryAuth: authStr,
	})
	if err != nil {
		log.Fatalf("推送镜像失败: %v", err)
	}
	defer pushResp.Close()

	io.Copy(os.Stdout, pushResp)
	fmt.Println("\n镜像推送完成")
}
```

### 5.4 列出和删除镜像

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/api/types/image"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 列出所有镜像
	images, err := cli.ImageList(ctx, image.ListOptions{All: true})
	if err != nil {
		log.Fatalf("列出镜像失败: %v", err)
	}

	fmt.Printf("%-20s %-15s %-15s %s\n", "REPOSITORY:TAG", "IMAGE ID", "SIZE", "CREATED")
	fmt.Println("─────────────────────────────────────────────────────────────────")
	for _, img := range images {
		tag := "<none>"
		if len(img.RepoTags) > 0 {
			tag = img.RepoTags[0]
		}
		fmt.Printf("%-20s %-15s %-15s %d\n",
			tag, img.ID[7:19], formatSize(img.Size), img.Created)
	}

	// 按标签过滤镜像
	filterArgs := filters.NewArgs()
	filterArgs.Add("label", "maintainer=sre-team")
	filteredImages, err := cli.ImageList(ctx, image.ListOptions{Filters: filterArgs})
	if err != nil {
		log.Fatalf("过滤镜像失败: %v", err)
	}
	fmt.Printf("\nSRE 团队镜像: %d 个\n", len(filteredImages))

	// 删除镜像
	removeResp, err := cli.ImageRemove(ctx, "myapp:v1.0", image.RemoveOptions{
		Force:         false, // 不强制删除
		PruneChildren: true,  // 删除未被引用的父镜像层
	})
	if err != nil {
		log.Printf("删除镜像失败: %v", err)
	} else {
		for _, item := range removeResp {
			if item.Deleted != "" {
				fmt.Printf("已删除: %s\n", item.Deleted)
			}
			if item.Untagged != "" {
				fmt.Printf("已取消标签: %s\n", item.Untagged)
			}
		}
	}

	// 清理悬空镜像（dangling images）
	pruneFilters := filters.NewArgs()
	pruneFilters.Add("dangling", "true")
	pruneReport, err := cli.ImagesPrune(ctx, pruneFilters)
	if err != nil {
		log.Printf("清理镜像失败: %v", err)
	} else {
		fmt.Printf("清理了 %d 个悬空镜像，释放 %s 空间\n",
			len(pruneReport.ImagesDeleted), formatSize(int64(pruneReport.SpaceReclaimed)))
	}
}

// formatSize 格式化文件大小
func formatSize(size int64) string {
	const (
		KB = 1024
		MB = KB * 1024
		GB = MB * 1024
	)
	switch {
	case size >= GB:
		return fmt.Sprintf("%.1f GB", float64(size)/float64(GB))
	case size >= MB:
		return fmt.Sprintf("%.1f MB", float64(size)/float64(MB))
	case size >= KB:
		return fmt.Sprintf("%.1f KB", float64(size)/float64(KB))
	default:
		return fmt.Sprintf("%d B", size)
	}
}
```

---

## 六、网络操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/api/types/network"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 创建自定义网络
	netResp, err := cli.NetworkCreate(ctx, "sre-network", network.CreateOptions{
		Driver: "bridge",
		IPAM: &network.IPAM{
			Config: []network.IPAMConfig{
				{
					Subnet:  "172.28.0.0/16",
					Gateway: "172.28.0.1",
				},
			},
		},
		Labels: map[string]string{
			"team": "sre",
			"env":  "dev",
		},
	})
	if err != nil {
		log.Printf("创建网络失败: %v", err)
	} else {
		fmt.Printf("网络创建成功，ID: %s\n", netResp.ID[:12])
	}

	// 列出网络
	networks, err := cli.NetworkList(ctx, network.ListOptions{})
	if err != nil {
		log.Fatalf("列出网络失败: %v", err)
	}
	fmt.Printf("\n所有网络:\n")
	for _, n := range networks {
		fmt.Printf("  %-20s %-10s %-15s %s\n", n.Name, n.Driver, n.Scope, n.ID[:12])
	}

	// 按标签过滤网络
	filterArgs := filters.NewArgs()
	filterArgs.Add("label", "team=sre")
	filteredNets, err := cli.NetworkList(ctx, network.ListOptions{Filters: filterArgs})
	if err != nil {
		log.Fatalf("过滤网络失败: %v", err)
	}
	fmt.Printf("\nSRE 团队网络: %d 个\n", len(filteredNets))

	// 将容器连接到网络
	err = cli.NetworkConnect(ctx, "sre-network", "my-nginx-server", &network.EndpointSettings{})
	if err != nil {
		log.Printf("连接网络失败: %v", err)
	}

	// 将容器从网络断开
	err = cli.NetworkDisconnect(ctx, "sre-network", "my-nginx-server", false)
	if err != nil {
		log.Printf("断开网络失败: %v", err)
	}

	// 删除网络
	err = cli.NetworkRemove(ctx, "sre-network")
	if err != nil {
		log.Printf("删除网络失败: %v", err)
	} else {
		fmt.Println("网络已删除")
	}
}
```

---

## 七、卷操作

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/api/types/volume"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 创建卷
	vol, err := cli.VolumeCreate(ctx, volume.CreateOptions{
		Name:   "sre-data-volume",
		Driver: "local",
		Labels: map[string]string{
			"team":    "sre",
			"purpose": "database-storage",
		},
	})
	if err != nil {
		log.Printf("创建卷失败: %v", err)
	} else {
		fmt.Printf("卷创建成功:\n")
		fmt.Printf("  名称: %s\n", vol.Name)
		fmt.Printf("  驱动: %s\n", vol.Driver)
		fmt.Printf("  挂载点: %s\n", vol.Mountpoint)
	}

	// 列出卷
	volList, err := cli.VolumeList(ctx, volume.ListOptions{})
	if err != nil {
		log.Fatalf("列出卷失败: %v", err)
	}
	fmt.Printf("\n所有卷:\n")
	for _, v := range volList.Volumes {
		fmt.Printf("  %-25s %-10s %s\n", v.Name, v.Driver, v.Mountpoint)
	}

	// 按标签过滤卷
	filterArgs := filters.NewArgs()
	filterArgs.Add("label", "team=sre")
	filteredVols, err := cli.VolumeList(ctx, volume.ListOptions{Filters: filterArgs})
	if err != nil {
		log.Fatalf("过滤卷失败: %v", err)
	}
	fmt.Printf("\nSRE 团队卷: %d 个\n", len(filteredVols.Volumes))

	// 查看卷详情
	volInspect, err := cli.VolumeInspect(ctx, "sre-data-volume")
	if err != nil {
		log.Printf("查看卷详情失败: %v", err)
	} else {
		fmt.Printf("\n卷详情:\n")
		fmt.Printf("  名称: %s\n", volInspect.Name)
		fmt.Printf("  创建时间: %s\n", volInspect.CreatedAt)
		fmt.Printf("  标签: %v\n", volInspect.Labels)
	}

	// 删除卷
	err = cli.VolumeRemove(ctx, "sre-data-volume", false) // false 表示不强制删除
	if err != nil {
		log.Printf("删除卷失败: %v", err)
	} else {
		fmt.Println("卷已删除")
	}

	// 清理未使用的卷
	pruneReport, err := cli.VolumesPrune(ctx, filters.NewArgs())
	if err != nil {
		log.Printf("清理卷失败: %v", err)
	} else {
		fmt.Printf("清理了 %d 个未使用的卷\n", len(pruneReport.VolumesDeleted))
	}
}
```

---

## 八、容器事件监听

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/docker/docker/api/types/events"
	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 设置事件过滤器
	filterArgs := filters.NewArgs()
	filterArgs.Add("type", "container")                    // 只关注容器事件
	filterArgs.Add("event", "start")                       // 启动事件
	filterArgs.Add("event", "stop")                        // 停止事件
	filterArgs.Add("event", "die")                         // 退出事件
	filterArgs.Add("event", "health_status")               // 健康状态变更
	filterArgs.Add("event", "oom")                         // OOM 事件（SRE 重点关注）

	// 开始监听事件
	eventChan, errChan := cli.Events(ctx, events.ListOptions{
		Filters: filterArgs,
	})

	fmt.Println("开始监听 Docker 容器事件...")
	fmt.Println("按 Ctrl+C 退出")
	fmt.Println("─────────────────────────────────────────")

	for {
		select {
		case event := <-eventChan:
			ts := time.Unix(event.Time, event.TimeNano)
			fmt.Printf("[%s] 事件: %-20s 动作: %-15s",
				ts.Format("15:04:05"), event.Type, event.Action)

			// 打印 Actor 信息
			if event.Actor.ID != "" {
				containerID := event.Actor.ID
				if len(containerID) > 12 {
					containerID = containerID[:12]
				}
				containerName := event.Actor.Attributes["name"]
				imageName := event.Actor.Attributes["image"]
				fmt.Printf(" 容器: %s (%s) 镜像: %s",
					containerName, containerID, imageName)
			}
			fmt.Println()

			// SRE 告警：OOM 事件
			if event.Action == "oom" {
				fmt.Printf("  [告警] 容器 OOM! 请检查内存配置\n")
			}
			// SRE 告警：非正常退出
			if event.Action == "die" {
				exitCode := event.Actor.Attributes["exitCode"]
				if exitCode != "0" {
					fmt.Printf("  [告警] 容器异常退出，退出码: %s\n", exitCode)
				}
			}

		case err := <-errChan:
			if err != nil {
				log.Fatalf("事件监听错误: %v", err)
			}
			return
		}
	}
}
```

---

## 九、SRE 实战：容器自动健康检查与重启工具

这是一个完整的生产级工具，定期检查容器健康状态并自动重启不健康的容器。

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"time"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/events"
	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/client"
)

// ============================================================
// 配置定义
// ============================================================

// Config 工具配置
type Config struct {
	CheckInterval    time.Duration // 健康检查间隔
	MaxRestartCount  int           // 最大重启次数（超过后告警但不重启）
	RestartTimeout   int           // 重启超时（秒）
	LabelSelector    string        // 标签选择器（只管理带特定标签的容器）
	LogFile          string        // 日志文件路径
	EnableEventWatch bool          // 是否启用事件监听
}

// DefaultConfig 默认配置
func DefaultConfig() *Config {
	return &Config{
		CheckInterval:    30 * time.Second,
		MaxRestartCount:  5,
		RestartTimeout:   30,
		LabelSelector:    "managed=auto-heal", // 只管理带此标签的容器
		LogFile:          "/var/log/container-healer.log",
		EnableEventWatch: true,
	}
}

// ============================================================
// 容器健康状态记录
// ============================================================

// ContainerHealthRecord 记录容器的健康检查历史
type ContainerHealthRecord struct {
	ContainerID   string
	ContainerName string
	RestartCount  int
	LastCheckTime time.Time
	LastStatus    string
	Healthy       bool
}

// ============================================================
// ContainerHealer 容器自动修复器
// ============================================================

// ContainerHealer 容器健康检查与自动重启工具
type ContainerHealer struct {
	cli     *client.Client
	config  *Config
	records map[string]*ContainerHealthRecord // containerID -> record
	mu      sync.RWMutex
	logger  *log.Logger
}

// NewContainerHealer 创建容器修复器
func NewContainerHealer(config *Config) (*ContainerHealer, error) {
	// 创建 Docker 客户端
	cli, err := client.NewClientWithOpts(
		client.FromEnv,
		client.WithAPIVersionNegotiation(),
	)
	if err != nil {
		return nil, fmt.Errorf("创建 Docker 客户端失败: %w", err)
	}

	// 验证连接
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if _, err := cli.Ping(ctx); err != nil {
		return nil, fmt.Errorf("连接 Docker 失败: %w", err)
	}

	// 配置日志
	logWriter := io.Writer(os.Stdout)
	if config.LogFile != "" {
		file, err := os.OpenFile(config.LogFile, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
		if err != nil {
			log.Printf("无法打开日志文件 %s，使用标准输出: %v", config.LogFile, err)
		} else {
			logWriter = io.MultiWriter(os.Stdout, file) // 同时输出到控制台和文件
		}
	}

	logger := log.New(logWriter, "[ContainerHealer] ", log.LstdFlags|log.Lshortfile)

	return &ContainerHealer{
		cli:     cli,
		config:  config,
		records: make(map[string]*ContainerHealthRecord),
		logger:  logger,
	}, nil
}

// Close 关闭资源
func (ch *ContainerHealer) Close() {
	ch.cli.Close()
}

// Run 启动健康检查主循环
func (ch *ContainerHealer) Run(ctx context.Context) {
	ch.logger.Println("容器自动健康检查工具启动")
	ch.logger.Printf("检查间隔: %v", ch.config.CheckInterval)
	ch.logger.Printf("标签选择器: %s", ch.config.LabelSelector)
	ch.logger.Printf("最大重启次数: %d", ch.config.MaxRestartCount)

	// 启动事件监听（异步）
	if ch.config.EnableEventWatch {
		go ch.watchEvents(ctx)
	}

	// 定时健康检查
	ticker := time.NewTicker(ch.config.CheckInterval)
	defer ticker.Stop()

	// 启动后立即执行一次检查
	ch.checkAllContainers(ctx)

	for {
		select {
		case <-ticker.C:
			ch.checkAllContainers(ctx)
		case <-ctx.Done():
			ch.logger.Println("收到停止信号，正在退出...")
			return
		}
	}
}

// checkAllContainers 检查所有受管理的容器
func (ch *ContainerHealer) checkAllContainers(ctx context.Context) {
	ch.logger.Println("开始健康检查轮次...")

	// 获取受管理的容器列表
	filterArgs := filters.NewArgs()
	if ch.config.LabelSelector != "" {
		filterArgs.Add("label", ch.config.LabelSelector)
	}

	containers, err := ch.cli.ContainerList(ctx, container.ListOptions{
		All:     true, // 包含已停止的容器
		Filters: filterArgs,
	})
	if err != nil {
		ch.logger.Printf("获取容器列表失败: %v", err)
		return
	}

	ch.logger.Printf("发现 %d 个受管理的容器", len(containers))

	for _, c := range containers {
		ch.checkContainer(ctx, c)
	}

	ch.logger.Println("健康检查轮次完成")
}

// checkContainer 检查单个容器的健康状态
func (ch *ContainerHealer) checkContainer(ctx context.Context, c types.Container) {
	containerName := "unknown"
	if len(c.Names) > 0 {
		containerName = strings.TrimPrefix(c.Names[0], "/")
	}

	// 获取容器详细信息
	inspect, err := ch.cli.ContainerInspect(ctx, c.ID)
	if err != nil {
		ch.logger.Printf("[%s] 获取容器详情失败: %v", containerName, err)
		return
	}

	// 获取或创建健康记录
	ch.mu.Lock()
	record, exists := ch.records[c.ID]
	if !exists {
		record = &ContainerHealthRecord{
			ContainerID:   c.ID,
			ContainerName: containerName,
		}
		ch.records[c.ID] = record
	}
	record.LastCheckTime = time.Now()
	ch.mu.Unlock()

	// 判断容器状态
	switch {
	case !inspect.State.Running:
		// 容器未运行
		ch.logger.Printf("[%s] 容器未运行 (状态: %s)，尝试启动...",
			containerName, inspect.State.Status)
		ch.restartContainer(ctx, c.ID, containerName, record)

	case inspect.State.Health != nil && inspect.State.Health.Status == "unhealthy":
		// 健康检查失败
		ch.logger.Printf("[%s] 容器不健康 (连续失败: %d 次)，尝试重启...",
			containerName, inspect.State.Health.FailingStreak)

		// 打印最近的健康检查日志
		if len(inspect.State.Health.Log) > 0 {
			lastLog := inspect.State.Health.Log[len(inspect.State.Health.Log)-1]
			ch.logger.Printf("[%s] 最近检查: 退出码=%d, 输出=%s",
				containerName, lastLog.ExitCode,
				strings.TrimSpace(lastLog.Output))
		}
		ch.restartContainer(ctx, c.ID, containerName, record)

	case inspect.State.OOMKilled:
		// OOM 被杀
		ch.logger.Printf("[%s] [严重] 容器因 OOM 被杀！当前内存限制: %d MB",
			containerName, inspect.HostConfig.Memory/1024/1024)
		ch.restartContainer(ctx, c.ID, containerName, record)

	default:
		// 容器健康
		record.Healthy = true
		record.LastStatus = "healthy"
		ch.logger.Printf("[%s] 容器健康 (运行中, PID: %d)",
			containerName, inspect.State.Pid)
	}
}

// restartContainer 重启容器
func (ch *ContainerHealer) restartContainer(
	ctx context.Context,
	containerID, containerName string,
	record *ContainerHealthRecord,
) {
	// 检查是否超过最大重启次数
	if record.RestartCount >= ch.config.MaxRestartCount {
		ch.logger.Printf("[%s] [告警] 已达到最大重启次数 (%d/%d)，跳过重启，需要人工介入！",
			containerName, record.RestartCount, ch.config.MaxRestartCount)
		record.LastStatus = "max_restarts_exceeded"
		// 这里可以集成告警系统（如发送到 PagerDuty/Slack/钉钉等）
		ch.sendAlert(containerName, "已达到最大重启次数，需要人工介入")
		return
	}

	// 收集重启前的日志（用于事后分析）
	ch.collectContainerLogs(ctx, containerID, containerName)

	// 尝试重启
	timeout := ch.config.RestartTimeout
	ch.logger.Printf("[%s] 正在重启容器 (超时: %d 秒)...", containerName, timeout)

	err := ch.cli.ContainerRestart(ctx, containerID, container.StopOptions{
		Timeout: &timeout,
	})
	if err != nil {
		ch.logger.Printf("[%s] 重启失败: %v", containerName, err)
		record.LastStatus = "restart_failed"
		return
	}

	record.RestartCount++
	record.LastStatus = "restarted"
	record.Healthy = false

	ch.logger.Printf("[%s] 重启成功 (第 %d/%d 次)",
		containerName, record.RestartCount, ch.config.MaxRestartCount)

	// 等待容器变为健康状态
	go ch.waitForHealthy(ctx, containerID, containerName)
}

// waitForHealthy 等待容器健康
func (ch *ContainerHealer) waitForHealthy(ctx context.Context, containerID, containerName string) {
	timeout := time.After(2 * time.Minute)
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			inspect, err := ch.cli.ContainerInspect(ctx, containerID)
			if err != nil {
				ch.logger.Printf("[%s] 等待健康时获取详情失败: %v", containerName, err)
				return
			}
			if inspect.State.Running {
				if inspect.State.Health == nil || inspect.State.Health.Status == "healthy" {
					ch.logger.Printf("[%s] 容器已恢复健康", containerName)
					return
				}
			}
		case <-timeout:
			ch.logger.Printf("[%s] [告警] 容器重启后 2 分钟内未恢复健康", containerName)
			return
		case <-ctx.Done():
			return
		}
	}
}

// collectContainerLogs 收集容器最近的日志用于分析
func (ch *ContainerHealer) collectContainerLogs(
	ctx context.Context,
	containerID, containerName string,
) {
	logOpts := container.LogsOptions{
		ShowStdout: true,
		ShowStderr: true,
		Tail:       "50", // 最后 50 行
		Timestamps: true,
	}

	reader, err := ch.cli.ContainerLogs(ctx, containerID, logOpts)
	if err != nil {
		ch.logger.Printf("[%s] 收集日志失败: %v", containerName, err)
		return
	}
	defer reader.Close()

	// 保存日志到文件
	logFileName := fmt.Sprintf("/tmp/container-%s-crash-%s.log",
		containerName, time.Now().Format("20060102-150405"))
	logFile, err := os.Create(logFileName)
	if err != nil {
		ch.logger.Printf("[%s] 创建日志文件失败: %v", containerName, err)
		return
	}
	defer logFile.Close()

	io.Copy(logFile, reader)
	ch.logger.Printf("[%s] 崩溃日志已保存到: %s", containerName, logFileName)
}

// watchEvents 监听容器事件
func (ch *ContainerHealer) watchEvents(ctx context.Context) {
	ch.logger.Println("开始监听 Docker 事件...")

	filterArgs := filters.NewArgs()
	filterArgs.Add("type", "container")
	filterArgs.Add("event", "die")
	filterArgs.Add("event", "oom")
	filterArgs.Add("event", "health_status")

	eventChan, errChan := ch.cli.Events(ctx, events.ListOptions{
		Filters: filterArgs,
	})

	for {
		select {
		case event := <-eventChan:
			containerName := event.Actor.Attributes["name"]
			switch event.Action {
			case "die":
				exitCode := event.Actor.Attributes["exitCode"]
				if exitCode != "0" {
					ch.logger.Printf("[事件] 容器 %s 异常退出，退出码: %s",
						containerName, exitCode)
				}
			case "oom":
				ch.logger.Printf("[事件][严重] 容器 %s OOM!", containerName)
				ch.sendAlert(containerName, "容器 OOM，需要调整内存限制")
			case "health_status: unhealthy":
				ch.logger.Printf("[事件] 容器 %s 变为不健康状态", containerName)
			}

		case err := <-errChan:
			if err != nil {
				ch.logger.Printf("事件监听错误: %v，5 秒后重试...", err)
				time.Sleep(5 * time.Second)
				// 重新开始监听
				go ch.watchEvents(ctx)
				return
			}

		case <-ctx.Done():
			ch.logger.Println("事件监听已停止")
			return
		}
	}
}

// sendAlert 发送告警通知（示例实现）
func (ch *ContainerHealer) sendAlert(containerName, message string) {
	alert := map[string]string{
		"container": containerName,
		"message":   message,
		"time":      time.Now().Format(time.RFC3339),
		"host":      getHostname(),
	}
	alertJSON, _ := json.MarshalIndent(alert, "", "  ")
	ch.logger.Printf("[告警通知]\n%s", string(alertJSON))

	// 实际生产中，这里可以集成：
	// - Slack Webhook
	// - 钉钉机器人
	// - PagerDuty
	// - 企业微信
	// - Prometheus Alertmanager
}

// GetStatus 获取所有容器的健康状态报告
func (ch *ContainerHealer) GetStatus() map[string]*ContainerHealthRecord {
	ch.mu.RLock()
	defer ch.mu.RUnlock()

	result := make(map[string]*ContainerHealthRecord, len(ch.records))
	for k, v := range ch.records {
		copied := *v
		result[k] = &copied
	}
	return result
}

// getHostname 获取主机名
func getHostname() string {
	hostname, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return hostname
}

// ============================================================
// 主函数
// ============================================================

func main() {
	// 加载配置
	config := DefaultConfig()

	// 可以从环境变量覆盖配置
	if interval := os.Getenv("CHECK_INTERVAL"); interval != "" {
		if d, err := time.ParseDuration(interval); err == nil {
			config.CheckInterval = d
		}
	}
	if label := os.Getenv("LABEL_SELECTOR"); label != "" {
		config.LabelSelector = label
	}

	// 创建 ContainerHealer
	healer, err := NewContainerHealer(config)
	if err != nil {
		log.Fatalf("初始化失败: %v", err)
	}
	defer healer.Close()

	// 设置优雅退出
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		sig := <-sigChan
		log.Printf("收到信号: %v，正在优雅退出...", sig)
		cancel()
	}()

	// 启动健康检查
	healer.Run(ctx)

	// 打印最终状态报告
	fmt.Println("\n===== 最终状态报告 =====")
	for _, record := range healer.GetStatus() {
		fmt.Printf("容器: %-20s 状态: %-15s 重启次数: %d 最后检查: %s\n",
			record.ContainerName,
			record.LastStatus,
			record.RestartCount,
			record.LastCheckTime.Format("15:04:05"),
		)
	}
}
```

---

## 常见坑点

### 1. 忘记读取 ImagePull 返回的 Reader

```go
// 错误：拉取不会完成
reader, _ := cli.ImagePull(ctx, "nginx:latest", image.PullOptions{})
// 没有读取 reader！

// 正确：必须读完 reader
reader, _ := cli.ImagePull(ctx, "nginx:latest", image.PullOptions{})
defer reader.Close()
io.Copy(io.Discard, reader) // 至少丢弃，确保拉取完成
```

### 2. 日志流的 8 字节头部

Docker 日志流每条日志前有 8 字节的头部（标识 stdout/stderr 和长度）。使用 `stdcopy.StdCopy` 可以正确分离：

```go
import "github.com/docker/docker/pkg/stdcopy"

reader, _ := cli.ContainerLogs(ctx, containerID, opts)
// 正确处理多路复用日志流
stdcopy.StdCopy(os.Stdout, os.Stderr, reader)
```

### 3. API 版本不匹配

```go
// 坑：不设置版本协商，可能报 "client version X is too new"
cli, _ := client.NewClientWithOpts(client.FromEnv)

// 正确：启用自动版本协商
cli, _ := client.NewClientWithOpts(
    client.FromEnv,
    client.WithAPIVersionNegotiation(),
)
```

### 4. Context 超时太短

```go
// 坑：镜像拉取可能很慢，5 秒超时会失败
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
cli.ImagePull(ctx, "large-image:latest", image.PullOptions{})

// 正确：为耗时操作设置合理的超时
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
```

### 5. 容器名冲突

```go
// 坑：容器名已存在会报错
cli.ContainerCreate(ctx, config, hostConfig, nil, nil, "my-app")
// Error: Conflict. The container name "/my-app" is already in use

// 正确：先检查并删除，或使用随机名
containers, _ := cli.ContainerList(ctx, container.ListOptions{All: true})
for _, c := range containers {
    for _, name := range c.Names {
        if name == "/my-app" {
            cli.ContainerRemove(ctx, c.ID, container.RemoveOptions{Force: true})
        }
    }
}
```

---

## 速查表

| 操作 | 方法 | 说明 |
|------|------|------|
| 创建客户端 | `client.NewClientWithOpts(...)` | 推荐 `FromEnv` + `WithAPIVersionNegotiation` |
| Ping | `cli.Ping(ctx)` | 验证连接 |
| 系统信息 | `cli.Info(ctx)` | Docker 系统信息 |
| 创建容器 | `cli.ContainerCreate(ctx, config, hostConfig, netConfig, platform, name)` | 返回容器 ID |
| 启动容器 | `cli.ContainerStart(ctx, id, opts)` | - |
| 停止容器 | `cli.ContainerStop(ctx, id, opts)` | 可设超时 |
| 重启容器 | `cli.ContainerRestart(ctx, id, opts)` | - |
| 删除容器 | `cli.ContainerRemove(ctx, id, opts)` | `Force: true` 可删运行中容器 |
| 容器列表 | `cli.ContainerList(ctx, opts)` | `All: true` 包含停止的 |
| 容器详情 | `cli.ContainerInspect(ctx, id)` | 包含健康检查、网络等 |
| 容器日志 | `cli.ContainerLogs(ctx, id, opts)` | `Follow: true` 实时跟踪 |
| 拉取镜像 | `cli.ImagePull(ctx, ref, opts)` | 必须读完返回的 Reader |
| 构建镜像 | `cli.ImageBuild(ctx, tarCtx, opts)` | 需要 tar 格式上下文 |
| 推送镜像 | `cli.ImagePush(ctx, ref, opts)` | 需要认证 |
| 镜像列表 | `cli.ImageList(ctx, opts)` | - |
| 删除镜像 | `cli.ImageRemove(ctx, id, opts)` | - |
| 创建网络 | `cli.NetworkCreate(ctx, name, opts)` | - |
| 网络列表 | `cli.NetworkList(ctx, opts)` | - |
| 创建卷 | `cli.VolumeCreate(ctx, opts)` | - |
| 卷列表 | `cli.VolumeList(ctx, opts)` | - |
| 事件监听 | `cli.Events(ctx, opts)` | 返回 event channel + error channel |

---

## 小结

本文系统讲解了 Go Docker SDK 的核心用法：

1. **客户端初始化**：推荐使用 `FromEnv` + `WithAPIVersionNegotiation`，封装连接管理器便于复用
2. **容器全生命周期**：创建、启动、停止、删除、列表、详情、日志一应俱全
3. **镜像管理**：拉取、构建、推送、列表、删除及清理
4. **网络与卷**：自定义网络创建、容器网络绑定、卷的创建与管理
5. **事件监听**：实时监控容器状态变化，SRE 告警的基础
6. **实战工具**：完整的容器自动健康检查与重启工具，包含日志收集、告警通知、优雅退出等生产特性

掌握 Docker SDK，你就能将日常的容器管理操作自动化，构建更强大的 SRE 工具链。
