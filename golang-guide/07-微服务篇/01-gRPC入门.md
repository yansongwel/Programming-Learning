# 01-gRPC 入门

## 导读

gRPC 是 Google 开源的高性能 RPC（Remote Procedure Call）框架，基于 HTTP/2 协议传输，使用 Protocol Buffers 作为接口描述语言（IDL）和序列化格式。相较于传统的 RESTful API，gRPC 具有更高的性能、更强的类型安全以及原生的流式通信支持。

在微服务架构中，服务间通信是核心问题。gRPC 凭借其高效的二进制序列化、多语言代码生成、四种通信模式以及丰富的中间件生态，已经成为云原生微服务间通信的事实标准。Kubernetes、Istio、etcd 等知名项目都大量使用 gRPC。

**本章你将学到：**
- Protocol Buffers 语法与 protoc 工具链
- gRPC 四种 RPC 模式的完整实现
- 拦截器/中间件机制
- 错误处理与元数据传递
- 健康检查机制

**前置要求：**
- 熟悉 Go 基础语法与并发编程
- 了解 HTTP 协议基础

---

## 一、Protocol Buffers 基础

### 1.1 什么是 Protocol Buffers

Protocol Buffers（简称 protobuf）是 Google 开发的一种语言无关、平台无关的可扩展序列化结构化数据的方式。它比 JSON 更小、更快、更简洁。

### 1.2 安装 protoc 编译器与 Go 插件

```bash
# 安装 protoc 编译器（Ubuntu/Debian）
sudo apt-get install -y protobuf-compiler

# 也可以从 GitHub Releases 下载指定版本
# https://github.com/protocolbuffers/protobuf/releases
# 例如：
# wget https://github.com/protocolbuffers/protobuf/releases/download/v25.1/protoc-25.1-linux-x86_64.zip
# unzip protoc-25.1-linux-x86_64.zip -d /usr/local

# 验证安装
protoc --version

# 安装 Go 的 protobuf 插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 确保 $GOPATH/bin 在 PATH 中
export PATH="$PATH:$(go env GOPATH)/bin"
```

### 1.3 protobuf 语法详解

```protobuf
// user.proto
// 文件头部声明

syntax = "proto3";  // 使用 proto3 语法

// Go 包路径，生成的 Go 代码将使用此包名
option go_package = "github.com/example/user/pb";

// 引入其他 proto 文件
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// 定义消息类型
message User {
  // 字段编号是 protobuf 的核心，序列化时使用编号而非字段名
  // 编号 1-15 使用 1 字节编码，16-2047 使用 2 字节，尽量把常用字段放在 1-15
  int64  id       = 1;   // 用户 ID
  string name     = 2;   // 用户名
  string email    = 3;   // 邮箱地址
  int32  age      = 4;   // 年龄
  Role   role     = 5;   // 用户角色

  // repeated 表示可重复字段（对应 Go 的切片）
  repeated string tags = 6;  // 标签列表

  // map 类型
  map<string, string> metadata = 7;  // 元数据键值对

  // 时间戳类型（需要 import）
  google.protobuf.Timestamp created_at = 8;  // 创建时间

  // optional 关键字（proto3 中可选，表示该字段可能未设置）
  optional string phone = 9;  // 可选电话号码

  // oneof 表示互斥字段，同一时间只能设置其中一个
  oneof contact {
    string wechat   = 10;  // 微信
    string dingtalk  = 11;  // 钉钉
  }
}

// 枚举类型
enum Role {
  ROLE_UNSPECIFIED = 0;  // 枚举的第一个值必须为 0
  ROLE_ADMIN       = 1;  // 管理员
  ROLE_USER        = 2;  // 普通用户
  ROLE_GUEST       = 3;  // 访客
}

// 嵌套消息
message Address {
  string province = 1;  // 省份
  string city     = 2;  // 城市
  string detail   = 3;  // 详细地址
}

message UserProfile {
  User    user    = 1;  // 用户基本信息
  Address address = 2;  // 地址信息
}
```

### 1.4 定义 gRPC 服务

```protobuf
// greeter.proto
syntax = "proto3";

option go_package = "github.com/example/greeter/pb";

package greeter;

// 定义服务
service GreeterService {
  // 一元 RPC（最常用）
  rpc SayHello (HelloRequest) returns (HelloReply);

  // 服务端流式 RPC
  rpc SayHelloServerStream (HelloRequest) returns (stream HelloReply);

  // 客户端流式 RPC
  rpc SayHelloClientStream (stream HelloRequest) returns (HelloReply);

  // 双向流式 RPC
  rpc SayHelloBidiStream (stream HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name = 1;  // 请求者姓名
}

message HelloReply {
  string message = 1;  // 回复消息
}
```

### 1.5 编译生成 Go 代码

```bash
# 创建项目目录结构
mkdir -p proto pb

# 将 .proto 文件放在 proto/ 目录

# 编译生成 Go 代码
protoc \
  --go_out=./pb \
  --go_opt=paths=source_relative \
  --go-grpc_out=./pb \
  --go-grpc_opt=paths=source_relative \
  proto/greeter.proto

# 参数说明：
# --go_out         : 生成 protobuf 消息类型的 Go 代码
# --go-grpc_out    : 生成 gRPC 服务接口的 Go 代码
# paths=source_relative : 输出路径相对于 proto 文件所在目录

# 生成的文件：
# pb/greeter.pb.go       - 消息类型定义
# pb/greeter_grpc.pb.go  - gRPC 服务接口
```

---

## 二、四种 RPC 模式完整实现

### 2.1 项目结构

```
grpc-demo/
├── go.mod
├── proto/
│   └── greeter.proto
├── pb/
│   ├── greeter.pb.go
│   └── greeter_grpc.pb.go
├── server/
│   └── main.go
└── client/
    └── main.go
```

### 2.2 初始化项目

```bash
mkdir grpc-demo && cd grpc-demo
go mod init grpc-demo

# 安装依赖
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

### 2.3 完整的 proto 定义

```protobuf
// proto/greeter.proto
syntax = "proto3";

option go_package = "grpc-demo/pb";

package greeter;

import "google/protobuf/timestamp.proto";

// 问候服务 - 包含四种 RPC 模式
service GreeterService {
  // 一元调用：发一个请求，收一个响应
  rpc SayHello (HelloRequest) returns (HelloReply);

  // 服务端流：发一个请求，收到一组流式响应
  rpc ListGreetings (HelloRequest) returns (stream HelloReply);

  // 客户端流：发送一组流式请求，收到一个汇总响应
  rpc RecordGreetings (stream HelloRequest) returns (GreetingSummary);

  // 双向流：客户端和服务端同时发送流式消息
  rpc ChatGreetings (stream HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name    = 1;  // 请求者姓名
  string message = 2;  // 自定义消息
}

message HelloReply {
  string message                    = 1;  // 回复消息
  google.protobuf.Timestamp time    = 2;  // 响应时间
}

message GreetingSummary {
  int32  count   = 1;  // 收到的问候总数
  string summary = 2;  // 汇总信息
}
```

### 2.4 服务端完整实现

```go
// server/main.go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"strings"
	"sync"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/timestamppb"

	pb "grpc-demo/pb"
)

// greeterServer 实现 GreeterService 的所有方法
type greeterServer struct {
	// 必须嵌入 UnimplementedGreeterServiceServer 以保证向前兼容
	// 当 proto 文件新增方法时，未实现的方法会返回 Unimplemented 错误
	pb.UnimplementedGreeterServiceServer

	mu       sync.Mutex       // 保护共享状态
	greetLog []string         // 存储问候日志
}

// newGreeterServer 创建服务实例
func newGreeterServer() *greeterServer {
	return &greeterServer{
		greetLog: make([]string, 0),
	}
}

// SayHello 实现一元 RPC
// 最基本的请求-响应模式，类似于 HTTP 的 GET/POST
func (s *greeterServer) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
	// 参数校验
	if req.GetName() == "" {
		// 使用 gRPC status 包返回错误，包含错误码和消息
		return nil, status.Errorf(codes.InvalidArgument, "姓名不能为空")
	}

	// 读取客户端发来的元数据
	md, ok := metadata.FromIncomingContext(ctx)
	if ok {
		if values := md.Get("client-id"); len(values) > 0 {
			log.Printf("收到来自客户端 %s 的请求", values[0])
		}
	}

	// 设置响应头元数据
	header := metadata.Pairs("server-version", "v1.0.0")
	if err := grpc.SendHeader(ctx, header); err != nil {
		log.Printf("发送元数据头失败: %v", err)
	}

	// 记录日志
	s.mu.Lock()
	s.greetLog = append(s.greetLog, req.GetName())
	s.mu.Unlock()

	// 构造响应
	reply := &pb.HelloReply{
		Message: fmt.Sprintf("你好 %s！欢迎使用 gRPC 服务。%s", req.GetName(), req.GetMessage()),
		Time:    timestamppb.Now(),
	}

	log.Printf("一元调用: 向 %s 发送问候", req.GetName())
	return reply, nil
}

// ListGreetings 实现服务端流式 RPC
// 客户端发一个请求，服务端返回多条消息（流式）
// 典型场景：查询列表、实时推送、日志流
func (s *greeterServer) ListGreetings(req *pb.HelloRequest, stream pb.GreeterService_ListGreetingsServer) error {
	if req.GetName() == "" {
		return status.Errorf(codes.InvalidArgument, "姓名不能为空")
	}

	// 模拟返回多条问候语
	greetings := []string{
		"你好",
		"早上好",
		"下午好",
		"晚上好",
		"很高兴认识你",
	}

	for i, greeting := range greetings {
		// 检查客户端是否已取消请求
		select {
		case <-stream.Context().Done():
			log.Printf("客户端取消了流式请求")
			return status.Errorf(codes.Canceled, "客户端取消请求")
		default:
		}

		reply := &pb.HelloReply{
			Message: fmt.Sprintf("[%d/%d] %s, %s!", i+1, len(greetings), greeting, req.GetName()),
			Time:    timestamppb.Now(),
		}

		// 通过流发送响应
		if err := stream.Send(reply); err != nil {
			return status.Errorf(codes.Internal, "发送流消息失败: %v", err)
		}

		// 模拟处理延迟
		time.Sleep(500 * time.Millisecond)
	}

	log.Printf("服务端流: 向 %s 发送了 %d 条问候", req.GetName(), len(greetings))
	return nil
}

// RecordGreetings 实现客户端流式 RPC
// 客户端发送多条消息，服务端在接收完毕后返回汇总结果
// 典型场景：文件上传、批量数据提交、日志收集
func (s *greeterServer) RecordGreetings(stream pb.GreeterService_RecordGreetingsServer) error {
	var names []string

	for {
		// 持续接收客户端发来的消息
		req, err := stream.Recv()
		if err == io.EOF {
			// 客户端发送完毕（调用了 CloseAndRecv）
			summary := &pb.GreetingSummary{
				Count:   int32(len(names)),
				Summary: fmt.Sprintf("共收到 %d 个问候，来自: %s", len(names), strings.Join(names, ", ")),
			}
			log.Printf("客户端流: 收到 %d 个问候", len(names))
			// SendAndClose 发送响应并关闭流
			return stream.SendAndClose(summary)
		}
		if err != nil {
			return status.Errorf(codes.Internal, "接收流消息失败: %v", err)
		}

		names = append(names, req.GetName())
		log.Printf("客户端流: 收到来自 %s 的问候", req.GetName())
	}
}

// ChatGreetings 实现双向流式 RPC
// 客户端和服务端同时发送和接收消息
// 典型场景：聊天、实时数据同步、双向通信
func (s *greeterServer) ChatGreetings(stream pb.GreeterService_ChatGreetingsServer) error {
	for {
		// 接收客户端消息
		req, err := stream.Recv()
		if err == io.EOF {
			// 客户端关闭了发送流
			log.Printf("双向流: 客户端关闭了连接")
			return nil
		}
		if err != nil {
			return status.Errorf(codes.Internal, "接收消息失败: %v", err)
		}

		log.Printf("双向流: 收到 %s 的消息: %s", req.GetName(), req.GetMessage())

		// 对每条消息立即回复（也可以积攒后批量回复）
		reply := &pb.HelloReply{
			Message: fmt.Sprintf("收到你的消息了 %s: '%s'", req.GetName(), req.GetMessage()),
			Time:    timestamppb.Now(),
		}

		if err := stream.Send(reply); err != nil {
			return status.Errorf(codes.Internal, "发送消息失败: %v", err)
		}
	}
}

func main() {
	// 监听 TCP 端口
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("无法监听端口: %v", err)
	}

	// 创建 gRPC 服务器（后面会加入拦截器）
	grpcServer := grpc.NewServer()

	// 注册服务
	pb.RegisterGreeterServiceServer(grpcServer, newGreeterServer())

	log.Printf("gRPC 服务器启动在 :50051")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("服务器启动失败: %v", err)
	}
}
```

### 2.5 客户端完整实现

```go
// client/main.go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"

	pb "grpc-demo/pb"
)

func main() {
	// 建立连接（此处使用不安全连接，生产环境应使用 TLS）
	conn, err := grpc.NewClient(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalf("连接失败: %v", err)
	}
	defer conn.Close()

	// 创建客户端 stub
	client := pb.NewGreeterServiceClient(conn)

	// 设置超时上下文
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// 依次演示四种 RPC 模式
	fmt.Println("========== 1. 一元调用 ==========")
	callUnary(ctx, client)

	fmt.Println("\n========== 2. 服务端流 ==========")
	callServerStream(ctx, client)

	fmt.Println("\n========== 3. 客户端流 ==========")
	callClientStream(ctx, client)

	fmt.Println("\n========== 4. 双向流 ==========")
	callBidiStream(ctx, client)
}

// callUnary 一元调用示例
func callUnary(ctx context.Context, client pb.GreeterServiceClient) {
	// 在上下文中附加元数据
	md := metadata.Pairs("client-id", "go-demo-client")
	ctx = metadata.NewOutgoingContext(ctx, md)

	// 用于接收响应头和尾元数据
	var header, trailer metadata.MD

	resp, err := client.SayHello(
		ctx,
		&pb.HelloRequest{
			Name:    "张三",
			Message: "这是来自 Go 客户端的消息",
		},
		grpc.Header(&header),   // 接收响应头
		grpc.Trailer(&trailer), // 接收响应尾
	)
	if err != nil {
		// 解析 gRPC 错误
		st, ok := status.FromError(err)
		if ok {
			log.Printf("gRPC 错误码: %v, 消息: %s", st.Code(), st.Message())
		} else {
			log.Printf("未知错误: %v", err)
		}
		return
	}

	fmt.Printf("响应: %s\n", resp.GetMessage())
	fmt.Printf("响应时间: %v\n", resp.GetTime().AsTime())

	// 读取响应头中的元数据
	if values := header.Get("server-version"); len(values) > 0 {
		fmt.Printf("服务器版本: %s\n", values[0])
	}
}

// callServerStream 服务端流式调用
func callServerStream(ctx context.Context, client pb.GreeterServiceClient) {
	// 发起服务端流请求
	stream, err := client.ListGreetings(ctx, &pb.HelloRequest{
		Name: "李四",
	})
	if err != nil {
		log.Printf("请求失败: %v", err)
		return
	}

	// 持续接收服务端发来的流式消息
	for {
		reply, err := stream.Recv()
		if err == io.EOF {
			// 服务端发送完毕
			fmt.Println("-- 服务端流结束 --")
			break
		}
		if err != nil {
			log.Printf("接收失败: %v", err)
			return
		}
		fmt.Printf("收到: %s (时间: %v)\n", reply.GetMessage(), reply.GetTime().AsTime().Format(time.RFC3339))
	}
}

// callClientStream 客户端流式调用
func callClientStream(ctx context.Context, client pb.GreeterServiceClient) {
	// 发起客户端流请求
	stream, err := client.RecordGreetings(ctx)
	if err != nil {
		log.Printf("请求失败: %v", err)
		return
	}

	// 发送多条消息
	names := []string{"王五", "赵六", "孙七", "周八"}
	for _, name := range names {
		req := &pb.HelloRequest{
			Name:    name,
			Message: fmt.Sprintf("来自 %s 的问候", name),
		}
		if err := stream.Send(req); err != nil {
			log.Printf("发送失败: %v", err)
			return
		}
		fmt.Printf("已发送: %s\n", name)
		time.Sleep(200 * time.Millisecond)
	}

	// 关闭发送流并接收汇总响应
	summary, err := stream.CloseAndRecv()
	if err != nil {
		log.Printf("接收汇总失败: %v", err)
		return
	}
	fmt.Printf("汇总: %s (共 %d 条)\n", summary.GetSummary(), summary.GetCount())
}

// callBidiStream 双向流式调用
func callBidiStream(ctx context.Context, client pb.GreeterServiceClient) {
	// 发起双向流请求
	stream, err := client.ChatGreetings(ctx)
	if err != nil {
		log.Printf("请求失败: %v", err)
		return
	}

	// 用 channel 等待接收完成
	done := make(chan struct{})

	// 启动 goroutine 接收服务端消息
	go func() {
		defer close(done)
		for {
			reply, err := stream.Recv()
			if err == io.EOF {
				fmt.Println("-- 双向流结束 --")
				return
			}
			if err != nil {
				log.Printf("接收失败: %v", err)
				return
			}
			fmt.Printf("  <- 收到回复: %s\n", reply.GetMessage())
		}
	}()

	// 发送消息
	messages := []struct {
		name string
		msg  string
	}{
		{"张三", "你好啊"},
		{"张三", "今天天气不错"},
		{"张三", "再见"},
	}

	for _, m := range messages {
		if err := stream.Send(&pb.HelloRequest{
			Name:    m.name,
			Message: m.msg,
		}); err != nil {
			log.Printf("发送失败: %v", err)
			return
		}
		fmt.Printf("  -> 已发送: %s\n", m.msg)
		time.Sleep(300 * time.Millisecond)
	}

	// 关闭发送流（告知服务端发送完毕）
	if err := stream.CloseSend(); err != nil {
		log.Printf("关闭发送流失败: %v", err)
	}

	// 等待接收完成
	<-done
}
```

---

## 三、拦截器/中间件

gRPC 拦截器类似于 HTTP 中间件，可以在请求前后执行通用逻辑，如日志记录、认证、限流、链路追踪等。

### 3.1 一元拦截器

```go
// interceptors/unary.go
package interceptors

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

// LoggingUnaryServerInterceptor 日志记录拦截器
// 记录每个请求的方法名、耗时、错误信息
func LoggingUnaryServerInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()

	// 调用实际处理函数
	resp, err := handler(ctx, req)

	duration := time.Since(start)

	// 记录日志
	if err != nil {
		st, _ := status.FromError(err)
		log.Printf("[gRPC] 方法=%s 耗时=%v 错误码=%v 错误=%v",
			info.FullMethod, duration, st.Code(), st.Message())
	} else {
		log.Printf("[gRPC] 方法=%s 耗时=%v 状态=OK", info.FullMethod, duration)
	}

	return resp, err
}

// AuthUnaryServerInterceptor 认证拦截器
// 从元数据中提取 token 并验证
func AuthUnaryServerInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	// 从元数据中提取 authorization token
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Errorf(codes.Unauthenticated, "缺少元数据")
	}

	tokens := md.Get("authorization")
	if len(tokens) == 0 {
		return nil, status.Errorf(codes.Unauthenticated, "缺少认证令牌")
	}

	token := tokens[0]

	// 验证 token（这里是简化示例，实际应调用认证服务）
	if token != "Bearer valid-token-123" {
		return nil, status.Errorf(codes.Unauthenticated, "无效的认证令牌")
	}

	// 将用户信息注入上下文
	newCtx := context.WithValue(ctx, "user_id", "user-001")

	return handler(newCtx, req)
}

// RecoveryUnaryServerInterceptor panic 恢复拦截器
// 防止单个请求的 panic 导致整个服务崩溃
func RecoveryUnaryServerInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (resp interface{}, err error) {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("[gRPC-Recovery] 方法=%s 捕获到panic: %v", info.FullMethod, r)
			err = status.Errorf(codes.Internal, "服务内部错误")
		}
	}()

	return handler(ctx, req)
}

// RateLimitUnaryServerInterceptor 简单限流拦截器
func RateLimitUnaryServerInterceptor(maxConcurrent int) grpc.UnaryServerInterceptor {
	sem := make(chan struct{}, maxConcurrent)

	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		// 尝试获取令牌
		select {
		case sem <- struct{}{}:
			defer func() { <-sem }()
			return handler(ctx, req)
		default:
			return nil, status.Errorf(codes.ResourceExhausted, "服务器繁忙，请稍后重试")
		}
	}
}
```

### 3.2 流式拦截器

```go
// interceptors/stream.go
package interceptors

import (
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/status"
)

// LoggingStreamServerInterceptor 流式日志拦截器
func LoggingStreamServerInterceptor(
	srv interface{},
	ss grpc.ServerStream,
	info *grpc.StreamServerInfo,
	handler grpc.StreamHandler,
) error {
	start := time.Now()

	// 用包装器包装 ServerStream 以拦截 Send/Recv
	wrappedStream := &loggingServerStream{
		ServerStream: ss,
		method:       info.FullMethod,
	}

	// 调用实际处理函数
	err := handler(srv, wrappedStream)

	duration := time.Since(start)
	if err != nil {
		st, _ := status.FromError(err)
		log.Printf("[gRPC-Stream] 方法=%s 耗时=%v 发送=%d 接收=%d 错误码=%v",
			info.FullMethod, duration, wrappedStream.sendCount, wrappedStream.recvCount, st.Code())
	} else {
		log.Printf("[gRPC-Stream] 方法=%s 耗时=%v 发送=%d 接收=%d 状态=OK",
			info.FullMethod, duration, wrappedStream.sendCount, wrappedStream.recvCount)
	}

	return err
}

// loggingServerStream 包装 grpc.ServerStream 用于统计收发消息数
type loggingServerStream struct {
	grpc.ServerStream
	method    string
	sendCount int
	recvCount int
}

func (s *loggingServerStream) SendMsg(m interface{}) error {
	err := s.ServerStream.SendMsg(m)
	if err == nil {
		s.sendCount++
	}
	return err
}

func (s *loggingServerStream) RecvMsg(m interface{}) error {
	err := s.ServerStream.RecvMsg(m)
	if err == nil {
		s.recvCount++
	}
	return err
}
```

### 3.3 客户端拦截器

```go
// interceptors/client.go
package interceptors

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

// LoggingUnaryClientInterceptor 客户端日志拦截器
func LoggingUnaryClientInterceptor(
	ctx context.Context,
	method string,
	req, reply interface{},
	cc *grpc.ClientConn,
	invoker grpc.UnaryInvoker,
	opts ...grpc.CallOption,
) error {
	start := time.Now()

	// 自动注入客户端标识到元数据
	md, ok := metadata.FromOutgoingContext(ctx)
	if !ok {
		md = metadata.New(nil)
	}
	md.Set("x-request-time", start.Format(time.RFC3339))
	ctx = metadata.NewOutgoingContext(ctx, md)

	// 调用实际的 RPC
	err := invoker(ctx, method, req, reply, cc, opts...)

	duration := time.Since(start)
	if err != nil {
		st, _ := status.FromError(err)
		log.Printf("[gRPC-Client] 方法=%s 耗时=%v 错误码=%v", method, duration, st.Code())
	} else {
		log.Printf("[gRPC-Client] 方法=%s 耗时=%v 状态=OK", method, duration)
	}

	return err
}
```

### 3.4 注册拦截器

```go
// 服务端：注册多个拦截器（使用 ChainUnaryInterceptor 链式组合）
grpcServer := grpc.NewServer(
    // 一元拦截器链（按顺序执行）
    grpc.ChainUnaryInterceptor(
        interceptors.RecoveryUnaryServerInterceptor,      // 第一个：panic 恢复
        interceptors.LoggingUnaryServerInterceptor,       // 第二个：日志记录
        interceptors.RateLimitUnaryServerInterceptor(100), // 第三个：限流
    ),
    // 流式拦截器
    grpc.ChainStreamInterceptor(
        interceptors.LoggingStreamServerInterceptor,
    ),
)

// 客户端：注册拦截器
conn, err := grpc.NewClient(
    "localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithChainUnaryInterceptor(
        interceptors.LoggingUnaryClientInterceptor,
    ),
)
```

---

## 四、gRPC 错误处理

### 4.1 错误码一览

```go
// error_handling/errors.go
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// gRPC 标准错误码说明
func showErrorCodes() {
	errorCodes := map[codes.Code]string{
		codes.OK:                 "成功",
		codes.Canceled:           "操作被取消（通常是客户端取消）",
		codes.Unknown:            "未知错误",
		codes.InvalidArgument:    "无效参数（客户端错误）",
		codes.DeadlineExceeded:   "超时（客户端设置的截止时间已过）",
		codes.NotFound:           "资源不存在",
		codes.AlreadyExists:      "资源已存在（创建冲突）",
		codes.PermissionDenied:   "权限不足",
		codes.ResourceExhausted:  "资源耗尽（如限流）",
		codes.FailedPrecondition: "前置条件不满足",
		codes.Aborted:            "操作中止（如事务冲突）",
		codes.OutOfRange:         "超出有效范围",
		codes.Unimplemented:      "方法未实现",
		codes.Internal:           "服务内部错误",
		codes.Unavailable:        "服务暂时不可用（可重试）",
		codes.DataLoss:           "数据丢失",
		codes.Unauthenticated:    "未认证（缺少有效凭证）",
	}

	for code, desc := range errorCodes {
		fmt.Printf("  %v: %s\n", code, desc)
	}
}

// 创建带详细信息的错误
func createDetailedError() error {
	// 创建基础错误
	st := status.New(codes.InvalidArgument, "请求参数无效")

	// 附加错误详情（使用 Google 的标准错误详情类型）
	detailed, err := st.WithDetails(
		// 描述哪些字段有问题
		&errdetails.BadRequest{
			FieldViolations: []*errdetails.BadRequest_FieldViolation{
				{
					Field:       "name",
					Description: "姓名不能为空",
				},
				{
					Field:       "age",
					Description: "年龄必须大于 0",
				},
			},
		},
	)
	if err != nil {
		// 如果附加详情失败，返回原始错误
		return st.Err()
	}

	return detailed.Err()
}

// 解析带详细信息的错误
func parseDetailedError(err error) {
	st, ok := status.FromError(err)
	if !ok {
		log.Printf("非 gRPC 错误: %v", err)
		return
	}

	fmt.Printf("错误码: %v\n", st.Code())
	fmt.Printf("错误消息: %s\n", st.Message())

	// 解析错误详情
	for _, detail := range st.Details() {
		switch d := detail.(type) {
		case *errdetails.BadRequest:
			for _, violation := range d.GetFieldViolations() {
				fmt.Printf("  字段 '%s': %s\n", violation.GetField(), violation.GetDescription())
			}
		}
	}
}

// 服务端错误处理最佳实践
func handleServiceError(ctx context.Context) error {
	// 检查上下文是否已取消
	if ctx.Err() == context.Canceled {
		return status.Errorf(codes.Canceled, "请求已取消")
	}
	if ctx.Err() == context.DeadlineExceeded {
		return status.Errorf(codes.DeadlineExceeded, "请求超时")
	}

	// 业务逻辑错误映射
	// 不要直接返回内部错误信息给客户端，避免泄露内部实现
	// 错误的做法：return status.Errorf(codes.Internal, "数据库错误: %v", dbErr)
	// 正确的做法：记录内部日志，返回通用错误给客户端
	log.Printf("内部错误详情: ...")
	return status.Errorf(codes.Internal, "服务内部错误，请稍后重试")
}
```

---

## 五、gRPC 元数据（Metadata）

元数据类似于 HTTP Header，用于在客户端和服务端之间传递附加信息。

```go
// metadata_demo/main.go
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
)

// ===== 客户端发送元数据 =====

func clientSendMetadata() {
	// 方式一：使用 metadata.Pairs 创建
	md := metadata.Pairs(
		"authorization", "Bearer my-token",
		"x-request-id",  "req-001",
		"x-client-version", "v1.0",
	)
	ctx := metadata.NewOutgoingContext(context.Background(), md)

	// 方式二：追加元数据到已有上下文
	ctx = metadata.AppendToOutgoingContext(ctx,
		"x-trace-id", "trace-abc123",
	)

	// 使用 ctx 发起 RPC 调用...
	_ = ctx
}

// ===== 服务端读取元数据 =====

func serverReadMetadata(ctx context.Context) {
	// 从上下文提取传入的元数据
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		log.Println("没有收到元数据")
		return
	}

	// 读取特定的元数据字段（键名自动转为小写）
	if values := md.Get("authorization"); len(values) > 0 {
		fmt.Printf("认证令牌: %s\n", values[0])
	}

	if values := md.Get("x-request-id"); len(values) > 0 {
		fmt.Printf("请求 ID: %s\n", values[0])
	}

	// 遍历所有元数据
	for key, values := range md {
		fmt.Printf("  %s = %v\n", key, values)
	}
}

// ===== 服务端发送元数据 =====

func serverSendMetadata(ctx context.Context) error {
	// 发送响应头（在第一个响应消息之前发送）
	header := metadata.Pairs(
		"x-server-id",      "server-01",
		"x-response-time",  "2024-01-01T00:00:00Z",
	)
	if err := grpc.SendHeader(ctx, header); err != nil {
		return err
	}

	// 设置响应尾（在 RPC 结束时发送）
	trailer := metadata.Pairs(
		"x-processing-time", "150ms",
	)
	if err := grpc.SetTrailer(ctx, trailer); err != nil {
		return err
	}

	return nil
}

// ===== 客户端接收元数据 =====

func clientReceiveMetadata() {
	// 一元 RPC 接收元数据
	var header, trailer metadata.MD

	// 调用时通过 CallOption 指定接收容器
	// resp, err := client.SayHello(ctx, req,
	//     grpc.Header(&header),
	//     grpc.Trailer(&trailer),
	// )

	// 流式 RPC 接收元数据
	// stream, err := client.ListGreetings(ctx, req)
	// header, err := stream.Header()  // 获取响应头
	// trailer := stream.Trailer()     // 获取响应尾

	_ = header
	_ = trailer
}
```

---

## 六、健康检查

gRPC 标准的健康检查协议，被 Kubernetes、负载均衡器等广泛支持。

```go
// health/main.go
package main

import (
	"context"
	"log"
	"net"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

func main() {
	// ===== 服务端设置健康检查 =====
	go startServerWithHealth()

	time.Sleep(time.Second) // 等待服务器启动

	// ===== 客户端检查服务健康状态 =====
	checkHealth()
}

func startServerWithHealth() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("监听失败: %v", err)
	}

	grpcServer := grpc.NewServer()

	// 创建健康检查服务
	healthServer := health.NewServer()

	// 注册健康检查服务到 gRPC 服务器
	healthpb.RegisterHealthServer(grpcServer, healthServer)

	// 设置各服务的健康状态
	// 空字符串表示整个服务器的健康状态
	healthServer.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)

	// 也可以设置特定服务的健康状态
	healthServer.SetServingStatus("greeter.GreeterService",
		healthpb.HealthCheckResponse_SERVING)

	// 模拟：5 秒后将某个服务标记为不健康
	go func() {
		time.Sleep(5 * time.Second)
		log.Println("模拟故障：将 GreeterService 标记为 NOT_SERVING")
		healthServer.SetServingStatus("greeter.GreeterService",
			healthpb.HealthCheckResponse_NOT_SERVING)
	}()

	log.Println("gRPC 服务器启动（含健康检查）在 :50051")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("服务器启动失败: %v", err)
	}
}

func checkHealth() {
	conn, err := grpc.NewClient(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalf("连接失败: %v", err)
	}
	defer conn.Close()

	healthClient := healthpb.NewHealthClient(conn)

	// 方式一：单次检查（Check）
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	resp, err := healthClient.Check(ctx, &healthpb.HealthCheckRequest{
		Service: "greeter.GreeterService", // 空字符串表示检查整个服务器
	})
	if err != nil {
		log.Printf("健康检查失败: %v", err)
		return
	}
	log.Printf("健康状态: %v", resp.GetStatus())

	// 方式二：持续监听（Watch）
	watchCtx, watchCancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer watchCancel()

	stream, err := healthClient.Watch(watchCtx, &healthpb.HealthCheckRequest{
		Service: "greeter.GreeterService",
	})
	if err != nil {
		log.Printf("健康监听失败: %v", err)
		return
	}

	// 持续接收健康状态变化
	for {
		watchResp, err := stream.Recv()
		if err != nil {
			log.Printf("监听结束: %v", err)
			return
		}
		log.Printf("健康状态变化: %v", watchResp.GetStatus())
	}
}
```

Kubernetes 中配置 gRPC 健康检查：

```yaml
# kubernetes-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: grpc-service
  template:
    metadata:
      labels:
        app: grpc-service
    spec:
      containers:
      - name: grpc-service
        image: my-grpc-service:latest
        ports:
        - containerPort: 50051
        # Kubernetes 1.24+ 原生支持 gRPC 健康检查
        livenessProbe:
          grpc:
            port: 50051
            service: ""  # 空字符串检查整个服务器
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          grpc:
            port: 50051
            service: "greeter.GreeterService"
          initialDelaySeconds: 3
          periodSeconds: 5
```

---

## 七、实战场景：SRE 监控告警 gRPC 服务

```go
// sre_alert/alert.proto 的对应实现
// 一个完整的告警通知 gRPC 服务示例

package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"sync"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/timestamppb"
)

// 以下类型模拟 protobuf 生成的代码结构

// AlertLevel 告警级别
type AlertLevel int32

const (
	AlertLevel_INFO     AlertLevel = 0
	AlertLevel_WARNING  AlertLevel = 1
	AlertLevel_CRITICAL AlertLevel = 2
)

// Alert 告警消息
type Alert struct {
	Id        string
	Service   string
	Level     AlertLevel
	Message   string
	Labels    map[string]string
	Timestamp *timestamppb.Timestamp
}

// AlertResponse 告警响应
type AlertResponse struct {
	Accepted bool
	AlertId  string
	Message  string
}

// AlertService 告警服务实现
type AlertService struct {
	mu     sync.RWMutex
	alerts []*Alert // 告警存储（生产环境应使用数据库）

	// 订阅者管理：支持多个客户端订阅告警流
	subscribers   map[string]chan *Alert
	subscribersMu sync.RWMutex
}

func NewAlertService() *AlertService {
	return &AlertService{
		alerts:      make([]*Alert, 0),
		subscribers: make(map[string]chan *Alert),
	}
}

// SendAlert 接收告警（一元 RPC）
func (s *AlertService) SendAlert(ctx context.Context, alert *Alert) (*AlertResponse, error) {
	if alert.Service == "" {
		return nil, status.Errorf(codes.InvalidArgument, "服务名不能为空")
	}
	if alert.Message == "" {
		return nil, status.Errorf(codes.InvalidArgument, "告警消息不能为空")
	}

	// 生成告警 ID
	alert.Id = fmt.Sprintf("alert-%d", time.Now().UnixNano())
	alert.Timestamp = timestamppb.Now()

	// 存储告警
	s.mu.Lock()
	s.alerts = append(s.alerts, alert)
	s.mu.Unlock()

	// 通知所有订阅者
	s.subscribersMu.RLock()
	for _, ch := range s.subscribers {
		select {
		case ch <- alert:
		default:
			// 订阅者处理不过来，丢弃（生产环境应有更好的策略）
		}
	}
	s.subscribersMu.RUnlock()

	log.Printf("收到告警: [%v] %s - %s", alert.Level, alert.Service, alert.Message)

	return &AlertResponse{
		Accepted: true,
		AlertId:  alert.Id,
		Message:  "告警已接收",
	}, nil
}

// Subscribe 订阅告警（服务端流式 RPC）
func (s *AlertService) Subscribe(subscriberID string, sendFunc func(*Alert) error) error {
	ch := make(chan *Alert, 100)

	// 注册订阅者
	s.subscribersMu.Lock()
	s.subscribers[subscriberID] = ch
	s.subscribersMu.Unlock()

	// 清理：函数退出时移除订阅
	defer func() {
		s.subscribersMu.Lock()
		delete(s.subscribers, subscriberID)
		s.subscribersMu.Unlock()
		close(ch)
		log.Printf("订阅者 %s 已断开", subscriberID)
	}()

	log.Printf("订阅者 %s 已连接", subscriberID)

	// 持续将告警推送给订阅者
	for alert := range ch {
		if err := sendFunc(alert); err != nil {
			return err
		}
	}

	return nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("监听失败: %v", err)
	}

	grpcServer := grpc.NewServer(
		grpc.ChainUnaryInterceptor(
			// 恢复拦截器
			func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
				defer func() {
					if r := recover(); r != nil {
						log.Printf("panic 恢复: %v", r)
					}
				}()
				return handler(ctx, req)
			},
		),
	)

	_ = NewAlertService()
	// 在实际项目中这里会注册 protobuf 生成的服务接口
	// pb.RegisterAlertServiceServer(grpcServer, alertService)

	log.Println("告警 gRPC 服务启动在 :50051")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("启动失败: %v", err)
	}
}
```

---

## 八、常见坑点

### 坑点 1：忘记嵌入 UnimplementedXxxServer

```go
// 错误：没有嵌入 Unimplemented 结构体
type myServer struct{}

// 正确：必须嵌入以保证向前兼容
type myServer struct {
    pb.UnimplementedGreeterServiceServer
}
// 这样当 proto 文件新增方法时，未实现的方法会自动返回 codes.Unimplemented
// 而不是编译错误，保证了向前兼容
```

### 坑点 2：连接复用问题

```go
// 错误：每次调用都创建新连接（昂贵且浪费）
func callService() {
    conn, _ := grpc.NewClient("localhost:50051", ...)
    defer conn.Close()
    client := pb.NewServiceClient(conn)
    client.DoSomething(ctx, req)
}

// 正确：复用连接，gRPC 连接内部使用 HTTP/2 多路复用
// 一个连接可以承载大量并发请求
var conn *grpc.ClientConn // 全局或者通过依赖注入共享

func init() {
    var err error
    conn, err = grpc.NewClient("localhost:50051", ...)
    if err != nil {
        log.Fatal(err)
    }
}
```

### 坑点 3：忽略 Context 超时

```go
// 错误：没有设置超时，可能永远阻塞
resp, err := client.SayHello(context.Background(), req)

// 正确：始终设置合理的超时时间
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.SayHello(ctx, req)
```

### 坑点 4：流式 RPC 忘记处理 io.EOF

```go
// 错误：把 io.EOF 当成异常错误
reply, err := stream.Recv()
if err != nil {
    log.Fatalf("出错了: %v", err) // io.EOF 不是错误！
}

// 正确：io.EOF 表示流正常结束
reply, err := stream.Recv()
if err == io.EOF {
    // 流正常结束
    break
}
if err != nil {
    // 这才是真正的错误
    log.Printf("接收错误: %v", err)
    return err
}
```

### 坑点 5：proto 文件字段编号修改

```protobuf
// 错误：修改已发布的字段编号（会导致反序列化失败）
message User {
  string name = 2;  // 原来是 1，改成了 2 —— 灾难！
}

// 正确：字段编号一旦发布就不能修改
// 如果要废弃字段，使用 reserved 标记
message User {
  reserved 1;          // 保留编号 1，不再使用
  reserved "old_name"; // 保留字段名
  string name = 2;     // 使用新的编号
}
```

### 坑点 6：大消息超出默认限制

```go
// gRPC 默认消息大小限制为 4MB
// 超过时会报 "rpc error: code = ResourceExhausted desc = grpc: received message larger than max"

// 服务端调整
grpcServer := grpc.NewServer(
    grpc.MaxRecvMsgSize(50 * 1024 * 1024), // 50MB
    grpc.MaxSendMsgSize(50 * 1024 * 1024), // 50MB
)

// 客户端调整
conn, err := grpc.NewClient(
    "localhost:50051",
    grpc.WithDefaultCallOptions(
        grpc.MaxCallRecvMsgSize(50 * 1024 * 1024),
        grpc.MaxCallSendMsgSize(50 * 1024 * 1024),
    ),
)

// 更好的做法：对于大数据传输，使用流式 RPC 分片传输
```

---

## 九、速查表

### gRPC 核心概念速查

| 概念 | 说明 |
|------|------|
| Protocol Buffers | 接口定义语言 + 高效二进制序列化格式 |
| protoc | Protocol Buffers 编译器 |
| protoc-gen-go | 生成 Go 消息类型代码的插件 |
| protoc-gen-go-grpc | 生成 Go gRPC 服务接口代码的插件 |
| Unary RPC | 一元调用（一请求一响应） |
| Server Streaming | 服务端流（一请求多响应） |
| Client Streaming | 客户端流（多请求一响应） |
| Bidi Streaming | 双向流（多请求多响应） |
| Interceptor | 拦截器/中间件 |
| Metadata | 元数据（类似 HTTP Header） |
| status/codes | gRPC 错误处理包 |

### 常用 gRPC 错误码速查

| 错误码 | 场景 | 是否可重试 |
|--------|------|-----------|
| `InvalidArgument` | 参数校验失败 | 否（修正参数后重试） |
| `NotFound` | 资源不存在 | 否 |
| `AlreadyExists` | 资源已存在 | 否 |
| `PermissionDenied` | 权限不足 | 否 |
| `Unauthenticated` | 未认证 | 否（提供凭证后重试） |
| `ResourceExhausted` | 限流/超出配额 | 是（退避后重试） |
| `Unavailable` | 服务不可用 | 是（退避后重试） |
| `DeadlineExceeded` | 超时 | 是（可尝试重试） |
| `Internal` | 服务内部错误 | 视情况而定 |
| `Unimplemented` | 方法未实现 | 否 |

### protoc 编译命令速查

```bash
# 基本编译
protoc --go_out=. --go-grpc_out=. proto/*.proto

# 指定路径模式
protoc --go_out=./pb --go_opt=paths=source_relative \
       --go-grpc_out=./pb --go-grpc_opt=paths=source_relative \
       proto/*.proto

# 引入外部 proto（如 google/protobuf/timestamp.proto）
protoc -I=proto -I=/usr/local/include \
       --go_out=./pb --go_opt=paths=source_relative \
       --go-grpc_out=./pb --go-grpc_opt=paths=source_relative \
       proto/*.proto

# 使用 buf 工具（推荐替代 protoc 的现代工具）
# go install github.com/bufbuild/buf/cmd/buf@latest
# buf generate
```

### 依赖包速查

```bash
# 核心包
go get google.golang.org/grpc                  # gRPC 框架
go get google.golang.org/protobuf              # protobuf 运行时
go get google.golang.org/genproto              # Google API proto 定义

# 代码生成工具
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 常用中间件（grpc-ecosystem）
go get github.com/grpc-ecosystem/go-grpc-middleware/v2  # 拦截器工具集
go get github.com/grpc-ecosystem/grpc-gateway/v2        # REST 网关
```

---

## 小结

本章完整介绍了 gRPC 在 Go 中的使用方法：

1. **Protocol Buffers** 是 gRPC 的基石，掌握 proto 语法和 protoc 工具链是第一步
2. **四种 RPC 模式** 覆盖了所有通信场景：一元调用用于常规请求，服务端流用于数据推送，客户端流用于数据收集，双向流用于实时通信
3. **拦截器** 是 gRPC 的中间件机制，用于实现日志、认证、限流等横切关注点
4. **错误处理** 使用 status/codes 包，提供语义化的错误码和可扩展的错误详情
5. **元数据** 提供了类似 HTTP Header 的附加信息传递能力
6. **健康检查** 是微服务部署的必备功能，与 Kubernetes 无缝集成

下一章我们将学习服务注册与发现，让 gRPC 服务能够在分布式环境中自动寻找彼此。
