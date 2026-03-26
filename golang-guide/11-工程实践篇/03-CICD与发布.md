# Go CI/CD 与发布

## 导读

从代码提交到生产部署，一个完善的 CI/CD 流水线是 SRE 的核心工作之一。Go 语言天然适合 CI/CD：静态编译、交叉编译简单、二进制部署无依赖。本篇将系统讲解 Go 项目的交叉编译、GoReleaser 自动发布、Docker 多阶段构建、GitHub Actions 流水线以及版本信息注入。

**本篇核心知识点：**
- Go 交叉编译（GOOS/GOARCH）
- GoReleaser 自动发布
- Docker 多阶段构建
- GitHub Actions CI/CD 流水线
- 版本信息注入（ldflags）
- 私有模块 CI 配置

---

## 一、Go 交叉编译

### 1.1 基础知识

```go
// cross_compile.go
// Go 交叉编译演示
package main

import (
	"fmt"
	"runtime"
)

var (
	version   = "dev"
	gitCommit = "unknown"
	buildDate = "unknown"
)

func main() {
	fmt.Println("=== Go 交叉编译指南 ===\n")

	fmt.Printf("当前运行平台: %s/%s\n", runtime.GOOS, runtime.GOARCH)
	fmt.Printf("Go 版本: %s\n", runtime.Version())
	fmt.Printf("应用版本: %s (commit: %s, built: %s)\n\n", version, gitCommit, buildDate)

	fmt.Println("--- 常用平台编译命令 ---")
	fmt.Println()
	fmt.Println("# Linux AMD64（最常用的服务器平台）")
	fmt.Println("GOOS=linux GOARCH=amd64 go build -o myapp-linux-amd64")
	fmt.Println()
	fmt.Println("# Linux ARM64（AWS Graviton / Apple Silicon 的 Linux VM）")
	fmt.Println("GOOS=linux GOARCH=arm64 go build -o myapp-linux-arm64")
	fmt.Println()
	fmt.Println("# macOS ARM64（Apple Silicon）")
	fmt.Println("GOOS=darwin GOARCH=arm64 go build -o myapp-darwin-arm64")
	fmt.Println()
	fmt.Println("# macOS AMD64（Intel Mac）")
	fmt.Println("GOOS=darwin GOARCH=amd64 go build -o myapp-darwin-amd64")
	fmt.Println()
	fmt.Println("# Windows AMD64")
	fmt.Println("GOOS=windows GOARCH=amd64 go build -o myapp-windows-amd64.exe")
	fmt.Println()
	fmt.Println("# Linux MIPS（路由器/嵌入式）")
	fmt.Println("GOOS=linux GOARCH=mipsle GOMIPS=softfloat go build -o myapp-linux-mipsle")
	fmt.Println()

	fmt.Println("--- 关键环境变量 ---")
	fmt.Println("  GOOS       - 目标操作系统: linux, darwin, windows, freebsd")
	fmt.Println("  GOARCH     - 目标架构: amd64, arm64, arm, 386, mips")
	fmt.Println("  CGO_ENABLED - CGO 开关: 0=禁用（推荐交叉编译时关闭）")
	fmt.Println("  GOARM      - ARM 版本: 5, 6, 7")
	fmt.Println("  GOMIPS     - MIPS 浮点: hardfloat, softfloat")
	fmt.Println()

	fmt.Println("--- 查看支持的平台 ---")
	fmt.Println("  go tool dist list  # 列出所有支持的 GOOS/GOARCH 组合")
}
```

### 1.2 多平台编译脚本

```bash
#!/bin/bash
# scripts/build-all.sh
# 多平台交叉编译脚本

set -euo pipefail

# 项目信息
APP_NAME="${APP_NAME:-myapp}"
VERSION="${VERSION:-$(git describe --tags --always --dirty 2>/dev/null || echo dev)}"
GIT_COMMIT="${GIT_COMMIT:-$(git rev-parse --short HEAD 2>/dev/null || echo unknown)}"
BUILD_DATE="${BUILD_DATE:-$(date -u '+%Y-%m-%dT%H:%M:%SZ')}"

# 编译选项
LDFLAGS="-s -w \
  -X main.version=${VERSION} \
  -X main.gitCommit=${GIT_COMMIT} \
  -X main.buildDate=${BUILD_DATE}"

BUILD_DIR="./build"
CMD_PATH="./cmd/server"

# 目标平台列表
PLATFORMS=(
    "linux/amd64"
    "linux/arm64"
    "darwin/amd64"
    "darwin/arm64"
    "windows/amd64"
)

# 清理旧构建
rm -rf "${BUILD_DIR}"
mkdir -p "${BUILD_DIR}"

echo "=== 开始多平台编译 ==="
echo "  应用: ${APP_NAME}"
echo "  版本: ${VERSION}"
echo "  提交: ${GIT_COMMIT}"
echo "  日期: ${BUILD_DATE}"
echo ""

for PLATFORM in "${PLATFORMS[@]}"; do
    GOOS="${PLATFORM%/*}"
    GOARCH="${PLATFORM#*/}"

    OUTPUT="${BUILD_DIR}/${APP_NAME}-${GOOS}-${GOARCH}"
    if [ "${GOOS}" = "windows" ]; then
        OUTPUT="${OUTPUT}.exe"
    fi

    echo "编译 ${GOOS}/${GOARCH}..."
    CGO_ENABLED=0 GOOS="${GOOS}" GOARCH="${GOARCH}" \
        go build -trimpath -ldflags="${LDFLAGS}" \
        -o "${OUTPUT}" "${CMD_PATH}"

    # 显示文件信息
    ls -lh "${OUTPUT}"
done

echo ""
echo "=== 编译完成 ==="
ls -lh "${BUILD_DIR}/"

# 生成校验和
echo ""
echo "生成 SHA256 校验和..."
cd "${BUILD_DIR}"
sha256sum ${APP_NAME}-* > checksums.txt
cat checksums.txt
```

---

## 二、GoReleaser 自动发布

### 2.1 GoReleaser 配置

```yaml
# .goreleaser.yaml
# GoReleaser 配置文件
# 安装: go install github.com/goreleaser/goreleaser@latest
# 测试: goreleaser check
# 本地构建: goreleaser build --snapshot --clean
# 发布: goreleaser release (需要 GITHUB_TOKEN)

# 项目名称
project_name: myapp

# 构建之前执行的钩子
before:
  hooks:
    - go mod tidy
    - go generate ./...

# 构建配置
builds:
  - id: server
    main: ./cmd/server
    binary: myapp
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    # ARM 版本（仅 linux/arm 使用）
    goarm:
      - "7"
    # 忽略的组合
    ignore:
      - goos: windows
        goarch: arm64
    # 编译选项
    flags:
      - -trimpath
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.gitCommit={{.ShortCommit}}
      - -X main.buildDate={{.Date}}
    # mod_timestamp 确保可重现构建
    mod_timestamp: "{{ .CommitTimestamp }}"

  # 第二个可执行文件（如果有）
  - id: cli
    main: ./cmd/cli
    binary: myapp-cli
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64

# 归档配置
archives:
  - id: default
    builds:
      - server
      - cli
    format: tar.gz
    # Windows 使用 zip 格式
    format_overrides:
      - goos: windows
        format: zip
    # 归档文件中包含的额外文件
    files:
      - README.md
      - LICENSE
      - configs/*.yaml
    # 命名模板
    name_template: >-
      {{ .ProjectName }}_
      {{- .Version }}_
      {{- .Os }}_
      {{- .Arch }}
      {{- if .Arm }}v{{ .Arm }}{{ end }}

# 校验和
checksum:
  name_template: "checksums.txt"
  algorithm: sha256

# 更新日志
changelog:
  sort: asc
  use: github
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^ci:"
      - "^chore:"
      - "Merge pull request"
  groups:
    - title: "新功能"
      regexp: "^feat"
      order: 0
    - title: "Bug 修复"
      regexp: "^fix"
      order: 1
    - title: "性能优化"
      regexp: "^perf"
      order: 2
    - title: "其他"
      order: 999

# GitHub Release 配置
release:
  github:
    owner: myorg
    name: myapp
  # 是否为预发布
  prerelease: auto
  # Release 说明模板
  header: |
    ## {{ .ProjectName }} {{ .Tag }}

    发布日期: {{ .Date }}
  footer: |
    ---
    完整更新日志: https://github.com/myorg/myapp/compare/{{ .PreviousTag }}...{{ .Tag }}

# Docker 镜像
dockers:
  - image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}"
      - "ghcr.io/myorg/myapp:latest"
    dockerfile: Dockerfile
    build_flag_templates:
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.title={{.ProjectName}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--platform=linux/amd64"
    extra_files:
      - configs/

  # ARM64 镜像
  - image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}-arm64"
    dockerfile: Dockerfile
    goarch: arm64
    build_flag_templates:
      - "--platform=linux/arm64"

# Docker manifest（多架构镜像）
docker_manifests:
  - name_template: "ghcr.io/myorg/myapp:{{ .Tag }}"
    image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}"
      - "ghcr.io/myorg/myapp:{{ .Tag }}-arm64"

# Homebrew Tap
brews:
  - name: myapp
    repository:
      owner: myorg
      name: homebrew-tap
    homepage: "https://github.com/myorg/myapp"
    description: "我的 Go 应用描述"
    license: "MIT"
    install: |
      bin.install "myapp"
      bin.install "myapp-cli"
    test: |
      system "#{bin}/myapp", "--version"

# Snapcraft（Linux 打包）
# snapcrafts:
#   - name: myapp
#     publish: true
#     summary: "应用描述"

# 签名
# signs:
#   - artifacts: checksum
```

### 2.2 GoReleaser 使用命令

```bash
# 安装
go install github.com/goreleaser/goreleaser/v2@latest

# 初始化配置文件
goreleaser init

# 验证配置
goreleaser check

# 本地测试构建（不发布）
goreleaser build --snapshot --clean

# 本地测试完整流程（不发布）
goreleaser release --snapshot --clean

# 正式发布（需要 GITHUB_TOKEN）
# 先打 tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# 执行发布
export GITHUB_TOKEN="ghp_xxx"
goreleaser release --clean

# 指定配置文件
goreleaser release --config .goreleaser.yaml --clean
```

---

## 三、Docker 镜像构建

### 3.1 多阶段构建 Dockerfile

```dockerfile
# Dockerfile
# Go 多阶段构建 — 生产级模板

# === 第一阶段: 构建 ===
FROM golang:1.24-alpine AS builder

# 安装构建依赖
RUN apk add --no-cache git ca-certificates tzdata

# 设置工作目录
WORKDIR /app

# 先复制依赖文件（利用 Docker 层缓存）
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# 复制源代码
COPY . .

# 版本信息（通过 --build-arg 传入）
ARG VERSION=dev
ARG GIT_COMMIT=unknown
ARG BUILD_DATE=unknown

# 编译
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w \
        -X main.version=${VERSION} \
        -X main.gitCommit=${GIT_COMMIT} \
        -X main.buildDate=${BUILD_DATE}" \
    -o /app/myapp \
    ./cmd/server/

# === 第二阶段: 运行 ===
# 选项 1: scratch（最小，约 0 字节基础层）
# FROM scratch

# 选项 2: distroless（推荐，包含基本的运行时文件）
# FROM gcr.io/distroless/static-debian12:nonroot

# 选项 3: alpine（包含 shell，便于调试）
FROM alpine:3.20

# 安装运行时依赖（alpine 专用）
RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S appgroup && \
    adduser -S appuser -G appgroup

# 从构建阶段复制二进制文件
COPY --from=builder /app/myapp /usr/local/bin/myapp

# 复制配置文件
COPY --from=builder /app/configs/ /etc/myapp/

# 设置时区
ENV TZ=Asia/Shanghai

# 使用非 root 用户运行
USER appuser

# 暴露端口
EXPOSE 8080 6060

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# 启动命令
ENTRYPOINT ["myapp"]
CMD ["--config", "/etc/myapp/config.yaml"]
```

### 3.2 scratch vs distroless vs alpine

```go
// docker_base_images.go
// Docker 基础镜像选择指南
package main

import "fmt"

func main() {
	fmt.Println("=== Docker 基础镜像选择 ===\n")

	images := []struct {
		Name    string
		Size    string
		Shell   bool
		Debug   bool
		UseCase string
		Note    string
	}{
		{
			Name:    "scratch",
			Size:    "~0 MB（仅包含应用二进制）",
			Shell:   false,
			Debug:   false,
			UseCase: "对安全和大小有极致要求",
			Note:    "需要静态编译（CGO_ENABLED=0），不含 CA 证书和时区数据，需手动 COPY",
		},
		{
			Name:    "distroless/static",
			Size:    "~2 MB",
			Shell:   false,
			Debug:   false,
			UseCase: "推荐的生产环境基础镜像",
			Note:    "包含 CA 证书、时区数据、passwd 文件。无 shell = 更安全",
		},
		{
			Name:    "distroless/static:debug",
			Size:    "~5 MB",
			Shell:   true,
			Debug:   true,
			UseCase: "需要 debug 的 distroless",
			Note:    "包含 busybox shell，仅用于调试",
		},
		{
			Name:    "alpine:3.20",
			Size:    "~7 MB",
			Shell:   true,
			Debug:   true,
			UseCase: "需要 shell 和包管理器",
			Note:    "使用 musl libc，CGO 编译的程序可能有兼容问题",
		},
		{
			Name:    "debian:bookworm-slim",
			Size:    "~75 MB",
			Shell:   true,
			Debug:   true,
			UseCase: "需要 glibc 和完整包管理",
			Note:    "CGO 程序最安全的选择，但镜像较大",
		},
	}

	for _, img := range images {
		fmt.Printf("%-28s 大小: %s\n", img.Name, img.Size)
		fmt.Printf("  Shell: %v, 调试: %v\n", img.Shell, img.Debug)
		fmt.Printf("  适用: %s\n", img.UseCase)
		fmt.Printf("  说明: %s\n\n", img.Note)
	}

	fmt.Println("=== 推荐方案 ===")
	fmt.Println("  生产环境:  distroless/static:nonroot")
	fmt.Println("  开发调试:  alpine:3.20")
	fmt.Println("  CGO 应用:  debian:bookworm-slim")
	fmt.Println("  极致安全:  scratch (+ 手动 COPY CA 证书)")
}
```

### 3.3 Docker Compose 开发环境

```yaml
# deployments/docker-compose.yaml
# 开发环境 docker-compose

services:
  # 应用服务
  app:
    build:
      context: ..
      dockerfile: deployments/docker/Dockerfile
      args:
        VERSION: dev
    ports:
      - "8080:8080"   # 业务端口
      - "6060:6060"   # pprof 端口
    environment:
      - SERVER_ADDR=:8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=myapp
      - DB_PASSWORD=myapp123
      - REDIS_ADDR=redis:6379
      - LOG_LEVEL=debug
      - GOMEMLIMIT=256MiB
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ../migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Prometheus（可选：监控）
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ../configs/prometheus.yml:/etc/prometheus/prometheus.yml
    profiles:
      - monitoring

  # Grafana（可选：可视化）
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    profiles:
      - monitoring

volumes:
  postgres_data:
  redis_data:
```

---

## 四、GitHub Actions CI/CD 流水线

### 4.1 完整 CI/CD 配置

```yaml
# .github/workflows/ci.yaml
# Go 项目完整 CI/CD 流水线

name: CI/CD

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

# 权限
permissions:
  contents: write
  packages: write

# 取消之前的运行（PR 场景）
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  GO_VERSION: '1.24'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # === 代码检查 ===
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m

  # === 单元测试 ===
  test:
    name: Test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U testuser"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: 下载依赖
        run: go mod download

      - name: 运行测试
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: testuser
          DB_PASSWORD: testpass
          REDIS_ADDR: localhost:6379
        run: |
          go test -v -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: 上传覆盖率
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.out
          token: ${{ secrets.CODECOV_TOKEN }}

  # === 安全检查 ===
  security:
    name: Security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: 运行 govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: 运行 gosec
        uses: securego/gosec@master
        with:
          args: ./...

  # === 构建 ===
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
          - goos: linux
            goarch: arm64
          - goos: darwin
            goarch: amd64
          - goos: darwin
            goarch: arm64

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: 编译
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
          CGO_ENABLED: 0
        run: |
          VERSION=${GITHUB_REF_NAME:-dev}
          GIT_COMMIT=${GITHUB_SHA::7}
          BUILD_DATE=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

          go build -trimpath \
            -ldflags="-s -w \
              -X main.version=${VERSION} \
              -X main.gitCommit=${GIT_COMMIT} \
              -X main.buildDate=${BUILD_DATE}" \
            -o build/myapp-${GOOS}-${GOARCH} \
            ./cmd/server/

      - name: 上传构建产物
        uses: actions/upload-artifact@v4
        with:
          name: myapp-${{ matrix.goos }}-${{ matrix.goarch }}
          path: build/

  # === Docker 镜像构建 ===
  docker:
    name: Docker
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4

      - name: 设置 Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: 登录 GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: 提取元数据
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=

      - name: 构建并推送
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ github.ref_name }}
            GIT_COMMIT=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # === 发布 ===
  release:
    name: Release
    runs-on: ubuntu-latest
    needs: [lint, test, security]
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 4.2 依赖检查工作流

```yaml
# .github/workflows/deps.yaml
# 依赖安全检查和自动更新

name: Dependencies

on:
  schedule:
    - cron: '0 6 * * 1'  # 每周一 6:00 UTC
  workflow_dispatch:

jobs:
  vuln-check:
    name: Vulnerability Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'

      - name: 检查已知漏洞
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: 检查依赖更新
        run: |
          go list -m -u all 2>/dev/null | grep '\[' || echo "所有依赖已是最新"
```

---

## 五、版本信息注入

### 5.1 完整的版本信息管理

```go
// cmd/server/main.go
// 版本信息注入的完整实现
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"runtime"
	"runtime/debug"
)

// 通过 -ldflags 注入的版本信息
var (
	version   = "dev"
	gitCommit = "unknown"
	buildDate = "unknown"
)

// BuildInfo 构建信息
type BuildInfo struct {
	Version   string `json:"version"`
	GitCommit string `json:"git_commit"`
	BuildDate string `json:"build_date"`
	GoVersion string `json:"go_version"`
	OS        string `json:"os"`
	Arch      string `json:"arch"`
	Compiler  string `json:"compiler"`
}

// GetBuildInfo 获取构建信息
func GetBuildInfo() BuildInfo {
	info := BuildInfo{
		Version:   version,
		GitCommit: gitCommit,
		BuildDate: buildDate,
		GoVersion: runtime.Version(),
		OS:        runtime.GOOS,
		Arch:      runtime.GOARCH,
		Compiler:  runtime.Compiler,
	}

	// 如果 ldflags 没有注入，尝试从 debug.BuildInfo 获取
	if info.Version == "dev" {
		if bi, ok := debug.ReadBuildInfo(); ok {
			if bi.Main.Version != "" && bi.Main.Version != "(devel)" {
				info.Version = bi.Main.Version
			}
			for _, setting := range bi.Settings {
				switch setting.Key {
				case "vcs.revision":
					if info.GitCommit == "unknown" {
						info.GitCommit = setting.Value
						if len(info.GitCommit) > 7 {
							info.GitCommit = info.GitCommit[:7]
						}
					}
				case "vcs.time":
					if info.BuildDate == "unknown" {
						info.BuildDate = setting.Value
					}
				}
			}
		}
	}

	return info
}

func main() {
	// 处理 --version 命令行参数
	if len(os.Args) > 1 && (os.Args[1] == "--version" || os.Args[1] == "-v") {
		info := GetBuildInfo()
		fmt.Printf("%s version %s\n", os.Args[0], info.Version)
		fmt.Printf("  commit:    %s\n", info.GitCommit)
		fmt.Printf("  built:     %s\n", info.BuildDate)
		fmt.Printf("  go:        %s\n", info.GoVersion)
		fmt.Printf("  platform:  %s/%s\n", info.OS, info.Arch)
		os.Exit(0)
	}

	// 注册版本端点
	http.HandleFunc("/version", func(w http.ResponseWriter, r *http.Request) {
		info := GetBuildInfo()
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(info)
	})

	fmt.Println("服务启动 :8080")
	http.ListenAndServe(":8080", nil)
}
```

### 5.2 编译命令

```bash
# 标准版本信息注入命令
VERSION=$(git describe --tags --always --dirty)
GIT_COMMIT=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

go build -ldflags="-s -w \
  -X main.version=${VERSION} \
  -X main.gitCommit=${GIT_COMMIT} \
  -X 'main.buildDate=${BUILD_DATE}'" \
  -o myapp ./cmd/server/

# 验证
./myapp --version
```

---

## 六、私有模块 CI 配置

```go
// private_modules.go
// 私有 Go 模块的 CI/CD 配置指南
package main

import "fmt"

func main() {
	fmt.Println("=== 私有 Go 模块 CI 配置 ===\n")

	fmt.Println("--- 1. GOPRIVATE 环境变量 ---")
	fmt.Println("设置私有模块路径（不走 proxy）:")
	fmt.Println("  export GOPRIVATE=github.com/myorg/*")
	fmt.Println("  export GONOSUMDB=github.com/myorg/*")
	fmt.Println("  export GONOSUMCHECK=github.com/myorg/*")
	fmt.Println()

	fmt.Println("--- 2. Git 认证 ---")
	fmt.Println("# 方法 1：HTTPS + Token")
	fmt.Println(`  git config --global url."https://${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"`)
	fmt.Println()
	fmt.Println("# 方法 2：SSH Key")
	fmt.Println(`  git config --global url."git@github.com:".insteadOf "https://github.com/"`)
	fmt.Println()

	fmt.Println("--- 3. GitHub Actions 配置 ---")
	fmt.Println(`
  env:
    GOPRIVATE: github.com/myorg/*
  steps:
    - uses: actions/checkout@v4
    - name: 配置 Git 认证
      run: |
        git config --global url."https://${{ secrets.GH_PAT }}@github.com/".insteadOf "https://github.com/"
    - uses: actions/setup-go@v5
      with:
        go-version: '1.24'
    - run: go mod download
`)

	fmt.Println("--- 4. Dockerfile 中使用私有模块 ---")
	fmt.Println(`
  FROM golang:1.24-alpine AS builder
  ARG GITHUB_TOKEN
  RUN apk add --no-cache git
  RUN git config --global url."https://${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
  ENV GOPRIVATE=github.com/myorg/*
  WORKDIR /app
  COPY go.mod go.sum ./
  RUN go mod download
  COPY . .
  RUN CGO_ENABLED=0 go build -o /app/myapp ./cmd/server/

  # 构建命令：
  docker build --build-arg GITHUB_TOKEN=ghp_xxx -t myapp .
`)

	fmt.Println("--- 5. 安全注意事项 ---")
	fmt.Println("  - 使用 --secret 替代 --build-arg（Docker BuildKit）")
	fmt.Println("  - Token 权限最小化（只读 repo 权限）")
	fmt.Println("  - 使用 GitHub App 替代个人 Token")
	fmt.Println("  - 不要在 Dockerfile 中硬编码 Token")
}
```

---

## 七、常见坑点

```
坑点 1：CGO_ENABLED 不设为 0
  默认 CGO_ENABLED=1，会链接 C 库。
  交叉编译和 scratch/distroless 部署必须设为 0。

坑点 2：Docker 构建缓存失效
  COPY . . 放在 go mod download 之前会导致依赖每次都重新下载。
  正确顺序：先 COPY go.mod go.sum → go mod download → COPY . .

坑点 3：时区问题
  scratch 镜像不含时区数据，time.LoadLocation 会失败。
  解决：COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

坑点 4：CA 证书缺失
  scratch 不含 CA 证书，HTTPS 请求会失败。
  解决：COPY --from=builder /etc/ssl/certs /etc/ssl/certs

坑点 5：GoReleaser 的 tag 格式
  GoReleaser 要求 tag 以 v 开头：v1.0.0。
  不带 v 前缀的 tag 会报错。

坑点 6：GitHub Actions 缓存
  actions/setup-go@v5 自带 Go 模块缓存。
  不需要额外配置 actions/cache。

坑点 7：Docker 多架构构建慢
  QEMU 模拟 ARM64 构建很慢。
  解决：使用原生 ARM64 runner 或分开构建后 manifest create。

坑点 8：版本信息在 go install 时丢失
  go install 时 -ldflags 不生效。
  解决：使用 debug.ReadBuildInfo() 作为回退方案。
```

---

## 八、速查表

### 编译命令速查

```bash
# 基本编译
go build -o myapp ./cmd/server/

# 生产编译（缩小二进制 + 版本信息）
CGO_ENABLED=0 go build -trimpath \
  -ldflags="-s -w -X main.version=1.0.0" \
  -o myapp ./cmd/server/

# 交叉编译
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o myapp-linux-amd64

# PGO 编译
go build -pgo=default.pgo -o myapp

# 查看支持平台
go tool dist list
```

### Docker 命令速查

```bash
# 多阶段构建
docker build -t myapp:latest .

# 带版本参数构建
docker build --build-arg VERSION=1.0.0 -t myapp:1.0.0 .

# 多架构构建（需要 buildx）
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .

# 查看镜像大小
docker images myapp
```

### CI/CD 流程速查

```
代码提交 → lint → test → security → build → docker → release

PR:      lint → test → security → build（不发布）
Push:    lint → test → security → build → docker
Tag:     lint → test → security → goreleaser → docker → release
```
