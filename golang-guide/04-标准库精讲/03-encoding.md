# 03 - encoding 编码与序列化

> **适用版本**: Go 1.24 | **面向读者**: 有编程经验的 SRE / DevOps 工程师

---

## 导读

SRE/DevOps 工程师每天都在和各种数据格式打交道：JSON API 的请求与响应、CSV 格式的监控数据导出、XML 配置文件、Base64 编码的密钥……Go 标准库的 `encoding` 家族提供了统一的序列化/反序列化能力。

本篇涵盖：

1. `encoding/json` —— JSON 编解码（重点）
2. `encoding/xml` —— XML 基础
3. `encoding/csv` —— CSV 读写
4. `encoding/gob` —— Go 特有二进制序列化
5. `encoding/base64` —— Base64 编解码
6. 实战：解析 Kubernetes API 返回的 JSON
7. SRE 实战场景
8. 常见坑点与速查表

---

## 1. encoding/json

### 1.1 Marshal —— Go 结构体 → JSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

// struct tag 控制 JSON 字段名
type Server struct {
    Name      string    `json:"name"`                // 字段名映射为 "name"
    IP        string    `json:"ip"`                  // 字段名映射为 "ip"
    Port      int       `json:"port"`                // 字段名映射为 "port"
    IsHealthy bool      `json:"is_healthy"`          // 下划线命名
    Tags      []string  `json:"tags,omitempty"`      // omitempty: 空切片时省略该字段
    Password  string    `json:"-"`                   // "-": 永远不输出到 JSON
    CreatedAt time.Time `json:"created_at"`
}

func main() {
    server := Server{
        Name:      "web-01",
        IP:        "192.168.1.10",
        Port:      8080,
        IsHealthy: true,
        Tags:      []string{"production", "web"},
        Password:  "secret123",
        CreatedAt: time.Now(),
    }

    // Marshal: 紧凑格式
    data, err := json.Marshal(server)
    if err != nil {
        panic(err)
    }
    fmt.Println("紧凑格式:")
    fmt.Println(string(data))

    // MarshalIndent: 格式化输出
    prettyData, err := json.MarshalIndent(server, "", "  ")
    if err != nil {
        panic(err)
    }
    fmt.Println("\n格式化输出:")
    fmt.Println(string(prettyData))

    // 注意：Password 字段被 json:"-" 隐藏了
}
```

### 1.2 Unmarshal —— JSON → Go 结构体

```go
package main

import (
    "encoding/json"
    "fmt"
)

type AlertRule struct {
    Name        string   `json:"name"`
    Severity    string   `json:"severity"`
    Threshold   float64  `json:"threshold"`
    Labels      []string `json:"labels"`
    Description string   `json:"description,omitempty"`
}

func main() {
    // 模拟从 API 收到的 JSON
    jsonData := `{
        "name": "high_cpu_usage",
        "severity": "critical",
        "threshold": 90.5,
        "labels": ["production", "infrastructure"],
        "description": "CPU 使用率超过阈值"
    }`

    var rule AlertRule
    err := json.Unmarshal([]byte(jsonData), &rule)
    if err != nil {
        panic(err)
    }

    fmt.Printf("规则名称:  %s\n", rule.Name)
    fmt.Printf("严重级别:  %s\n", rule.Severity)
    fmt.Printf("阈值:      %.1f\n", rule.Threshold)
    fmt.Printf("标签:      %v\n", rule.Labels)
    fmt.Printf("描述:      %s\n", rule.Description)
}
```

### 1.3 嵌套结构体

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Cluster struct {
    Name  string `json:"name"`
    Nodes []Node `json:"nodes"`
}

type Node struct {
    Hostname string     `json:"hostname"`
    IP       string     `json:"ip"`
    Role     string     `json:"role"`
    Status   NodeStatus `json:"status"`
}

type NodeStatus struct {
    CPUUsage    float64 `json:"cpu_usage"`
    MemoryUsage float64 `json:"memory_usage"`
    DiskUsage   float64 `json:"disk_usage"`
    IsReady     bool    `json:"is_ready"`
}

func main() {
    cluster := Cluster{
        Name: "k8s-prod",
        Nodes: []Node{
            {
                Hostname: "master-01",
                IP:       "10.0.1.1",
                Role:     "master",
                Status:   NodeStatus{CPUUsage: 45.2, MemoryUsage: 60.1, DiskUsage: 30.5, IsReady: true},
            },
            {
                Hostname: "worker-01",
                IP:       "10.0.1.10",
                Role:     "worker",
                Status:   NodeStatus{CPUUsage: 78.9, MemoryUsage: 85.3, DiskUsage: 55.2, IsReady: true},
            },
            {
                Hostname: "worker-02",
                IP:       "10.0.1.11",
                Role:     "worker",
                Status:   NodeStatus{CPUUsage: 92.1, MemoryUsage: 90.0, DiskUsage: 80.3, IsReady: false},
            },
        },
    }

    data, _ := json.MarshalIndent(cluster, "", "  ")
    fmt.Println(string(data))

    // 反序列化
    var parsed Cluster
    json.Unmarshal(data, &parsed)
    fmt.Printf("\n集群 %s 有 %d 个节点\n", parsed.Name, len(parsed.Nodes))
    for _, node := range parsed.Nodes {
        status := "就绪"
        if !node.Status.IsReady {
            status = "异常"
        }
        fmt.Printf("  %s (%s) - CPU: %.1f%%, 状态: %s\n",
            node.Hostname, node.Role, node.Status.CPUUsage, status)
    }
}
```

### 1.4 动态 JSON —— map 与 interface{}

```go
package main

import (
    "encoding/json"
    "fmt"
)

func main() {
    // 当 JSON 结构不固定时，使用 map[string]interface{}
    jsonData := `{
        "event": "deployment",
        "metadata": {
            "cluster": "prod",
            "namespace": "default",
            "replicas": 3
        },
        "tags": ["k8s", "deployment", "v2"]
    }`

    var result map[string]interface{}
    err := json.Unmarshal([]byte(jsonData), &result)
    if err != nil {
        panic(err)
    }

    fmt.Println("事件:", result["event"])

    // 嵌套 map 需要类型断言
    metadata, ok := result["metadata"].(map[string]interface{})
    if ok {
        fmt.Println("集群:", metadata["cluster"])
        fmt.Println("命名空间:", metadata["namespace"])
        // 注意：JSON 数字默认解析为 float64
        fmt.Printf("副本数: %.0f\n", metadata["replicas"].(float64))
    }

    // 数组需要类型断言为 []interface{}
    tags, ok := result["tags"].([]interface{})
    if ok {
        fmt.Print("标签: ")
        for _, tag := range tags {
            fmt.Printf("%s ", tag.(string))
        }
        fmt.Println()
    }
}
```

### 1.5 自定义 MarshalJSON / UnmarshalJSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
    "time"
)

// 自定义时间格式
type CustomTime struct {
    time.Time
}

const customTimeFormat = "2006-01-02 15:04:05"

func (ct CustomTime) MarshalJSON() ([]byte, error) {
    // 自定义序列化格式
    formatted := fmt.Sprintf(`"%s"`, ct.Format(customTimeFormat))
    return []byte(formatted), nil
}

func (ct *CustomTime) UnmarshalJSON(data []byte) error {
    // 去掉引号
    s := strings.Trim(string(data), `"`)
    if s == "null" {
        return nil
    }
    t, err := time.Parse(customTimeFormat, s)
    if err != nil {
        return err
    }
    ct.Time = t
    return nil
}

type Event struct {
    Name      string     `json:"name"`
    Timestamp CustomTime `json:"timestamp"`
    Level     string     `json:"level"`
}

func main() {
    event := Event{
        Name:      "server_restart",
        Timestamp: CustomTime{time.Now()},
        Level:     "warning",
    }

    // 序列化
    data, _ := json.MarshalIndent(event, "", "  ")
    fmt.Println("序列化结果:")
    fmt.Println(string(data))

    // 反序列化
    jsonStr := `{"name":"disk_full","timestamp":"2024-06-15 14:30:00","level":"critical"}`
    var parsed Event
    json.Unmarshal([]byte(jsonStr), &parsed)
    fmt.Printf("\n反序列化: %s 发生在 %s\n", parsed.Name, parsed.Timestamp.Format(customTimeFormat))
}
```

### 1.6 json.Decoder —— 流式解析

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

type LogEntry struct {
    Timestamp string `json:"timestamp"`
    Level     string `json:"level"`
    Message   string `json:"message"`
}

func main() {
    // 模拟 JSON Lines 格式（每行一个 JSON 对象）
    // 这种格式在日志系统中非常常见（如 Docker 日志、Elasticsearch Bulk API）
    jsonLines := `{"timestamp":"2024-01-15T10:00:00Z","level":"INFO","message":"服务启动"}
{"timestamp":"2024-01-15T10:00:01Z","level":"WARN","message":"内存使用率偏高"}
{"timestamp":"2024-01-15T10:00:02Z","level":"ERROR","message":"数据库连接失败"}
{"timestamp":"2024-01-15T10:00:03Z","level":"INFO","message":"重连成功"}
`

    decoder := json.NewDecoder(strings.NewReader(jsonLines))

    var entries []LogEntry
    for decoder.More() {
        var entry LogEntry
        if err := decoder.Decode(&entry); err != nil {
            fmt.Println("解析错误:", err)
            continue
        }
        entries = append(entries, entry)
    }

    fmt.Printf("共解析 %d 条日志\n", len(entries))
    for _, e := range entries {
        fmt.Printf("  [%s] %s: %s\n", e.Timestamp, e.Level, e.Message)
    }
}
```

### 1.7 json.Encoder —— 流式编码

```go
package main

import (
    "encoding/json"
    "os"
)

type Metric struct {
    Name  string  `json:"name"`
    Value float64 `json:"value"`
    Unit  string  `json:"unit"`
}

func main() {
    metrics := []Metric{
        {Name: "cpu_usage", Value: 65.5, Unit: "percent"},
        {Name: "memory_free", Value: 2048, Unit: "MB"},
        {Name: "disk_iops", Value: 1500, Unit: "ops/s"},
        {Name: "network_rx", Value: 125.3, Unit: "MB/s"},
    }

    // 流式编码到标准输出
    encoder := json.NewEncoder(os.Stdout)
    encoder.SetIndent("", "  ") // 设置缩进

    for _, m := range metrics {
        if err := encoder.Encode(m); err != nil {
            panic(err)
        }
    }
}
```

### 1.8 struct tag 详解

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Config struct {
    // 基本映射
    Host string `json:"host"` // JSON 字段名为 "host"

    // omitempty: 零值时省略
    Port int `json:"port,omitempty"` // 0 时不输出

    // string: 将数值/布尔序列化为 JSON 字符串
    Timeout int  `json:"timeout,string"` // 输出为 "timeout": "30"
    Debug   bool `json:"debug,string"`   // 输出为 "debug": "true"

    // -: 完全忽略
    InternalKey string `json:"-"` // 永不序列化

    // 匿名字段（嵌入）会被展平
    Metadata
}

type Metadata struct {
    Version string `json:"version"`
    Region  string `json:"region"`
}

func main() {
    cfg := Config{
        Host:        "10.0.1.1",
        Port:        0,        // omitempty，不会出现在 JSON 中
        Timeout:     30,
        Debug:       true,
        InternalKey: "secret",
        Metadata: Metadata{
            Version: "1.0",
            Region:  "us-east-1",
        },
    }

    data, _ := json.MarshalIndent(cfg, "", "  ")
    fmt.Println(string(data))
    // 注意：
    // - port 因为 omitempty 且值为 0 而被省略
    // - InternalKey 因为 json:"-" 而被隐藏
    // - timeout 和 debug 被序列化为字符串
    // - Metadata 的字段被展平到顶层
}
```

---

## 2. encoding/xml

```go
package main

import (
    "encoding/xml"
    "fmt"
)

// XML 结构定义
type PomXML struct {
    XMLName      xml.Name     `xml:"project"`
    ModelVersion string       `xml:"modelVersion"`
    GroupID      string       `xml:"groupId"`
    ArtifactID   string       `xml:"artifactId"`
    Version      string       `xml:"version"`
    Dependencies []Dependency `xml:"dependencies>dependency"`
}

type Dependency struct {
    GroupID    string `xml:"groupId"`
    ArtifactID string `xml:"artifactId"`
    Version    string `xml:"version"`
    Scope      string `xml:"scope,omitempty"`
}

func main() {
    // 解析 XML（类似 Maven pom.xml）
    xmlData := `<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-service</artifactId>
    <version>1.0.0</version>
    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>`

    var pom PomXML
    err := xml.Unmarshal([]byte(xmlData), &pom)
    if err != nil {
        panic(err)
    }

    fmt.Printf("项目: %s:%s:%s\n", pom.GroupID, pom.ArtifactID, pom.Version)
    fmt.Println("依赖:")
    for _, dep := range pom.Dependencies {
        scope := dep.Scope
        if scope == "" {
            scope = "compile"
        }
        fmt.Printf("  %s:%s:%s (%s)\n", dep.GroupID, dep.ArtifactID, dep.Version, scope)
    }

    // 序列化
    output, _ := xml.MarshalIndent(pom, "", "  ")
    fmt.Println("\n序列化结果:")
    fmt.Println(xml.Header + string(output))
}
```

---

## 3. encoding/csv

### 3.1 读取 CSV

```go
package main

import (
    "encoding/csv"
    "fmt"
    "os"
    "strings"
)

func main() {
    // 模拟 CSV 数据（监控指标导出）
    csvData := `hostname,cpu_usage,memory_usage,disk_usage,status
web-01,45.2,60.1,30.5,healthy
web-02,78.9,85.3,55.2,healthy
web-03,92.1,90.0,80.3,critical
db-01,55.0,70.2,45.0,healthy
db-02,30.5,40.1,25.3,healthy`

    reader := csv.NewReader(strings.NewReader(csvData))

    // 读取所有行
    records, err := reader.ReadAll()
    if err != nil {
        panic(err)
    }

    // 第一行是表头
    header := records[0]
    fmt.Printf("%-12s %-10s %-12s %-10s %-10s\n",
        header[0], header[1], header[2], header[3], header[4])
    fmt.Println(strings.Repeat("-", 60))

    // 遍历数据行
    for _, row := range records[1:] {
        fmt.Printf("%-12s %-10s %-12s %-10s %-10s\n",
            row[0], row[1], row[2], row[3], row[4])
    }

    // 逐行读取（适合大文件）
    fmt.Println("\n--- 逐行读取 ---")
    reader2 := csv.NewReader(strings.NewReader(csvData))
    for {
        record, err := reader2.Read()
        if err != nil {
            break // io.EOF
        }
        fmt.Println(record)
    }
}
```

### 3.2 写入 CSV

```go
package main

import (
    "encoding/csv"
    "fmt"
    "os"
    "strconv"
    "time"
)

type MetricRecord struct {
    Timestamp time.Time
    Host      string
    CPU       float64
    Memory    float64
}

func main() {
    file, err := os.Create("/tmp/metrics_export.csv")
    if err != nil {
        panic(err)
    }
    defer file.Close()

    writer := csv.NewWriter(file)
    defer writer.Flush() // 重要：必须 Flush

    // 写入表头
    writer.Write([]string{"timestamp", "hostname", "cpu_usage", "memory_usage"})

    // 模拟写入数据
    records := []MetricRecord{
        {time.Now().Add(-5 * time.Minute), "web-01", 45.2, 60.1},
        {time.Now().Add(-4 * time.Minute), "web-01", 48.5, 62.3},
        {time.Now().Add(-3 * time.Minute), "web-01", 52.1, 65.0},
        {time.Now().Add(-2 * time.Minute), "web-01", 55.8, 68.7},
        {time.Now().Add(-1 * time.Minute), "web-01", 60.3, 71.2},
    }

    for _, r := range records {
        row := []string{
            r.Timestamp.Format(time.RFC3339),
            r.Host,
            strconv.FormatFloat(r.CPU, 'f', 1, 64),
            strconv.FormatFloat(r.Memory, 'f', 1, 64),
        }
        writer.Write(row)
    }

    fmt.Println("CSV 已写入 /tmp/metrics_export.csv")
}
```

---

## 4. encoding/gob

Go 特有的二进制序列化格式，比 JSON 更高效，但只能在 Go 程序之间使用。

```go
package main

import (
    "bytes"
    "encoding/gob"
    "fmt"
    "time"
)

type Session struct {
    UserID    string
    Token     string
    CreatedAt time.Time
    Data      map[string]interface{}
}

func main() {
    // 序列化
    original := Session{
        UserID:    "sre-admin",
        Token:     "abc123def456",
        CreatedAt: time.Now(),
        Data: map[string]interface{}{
            "role":       "admin",
            "last_login": "2024-01-15",
        },
    }

    var buf bytes.Buffer
    encoder := gob.NewEncoder(&buf)
    err := encoder.Encode(original)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Gob 编码大小: %d 字节\n", buf.Len())

    // 反序列化
    var decoded Session
    decoder := gob.NewDecoder(&buf)
    err = decoder.Decode(&decoded)
    if err != nil {
        panic(err)
    }

    fmt.Printf("用户ID: %s\n", decoded.UserID)
    fmt.Printf("Token: %s\n", decoded.Token)
    fmt.Printf("创建时间: %s\n", decoded.CreatedAt.Format(time.RFC3339))
    fmt.Printf("数据: %v\n", decoded.Data)

    // 对比 JSON 大小
    jsonBuf := &bytes.Buffer{}
    fmt.Fprintf(jsonBuf, `{"user_id":"%s","token":"%s"}`, original.UserID, original.Token)
    fmt.Printf("\nJSON 大小（简化版）: %d 字节\n", jsonBuf.Len())
}
```

---

## 5. encoding/base64

```go
package main

import (
    "encoding/base64"
    "fmt"
)

func main() {
    // 标准 Base64 编码
    original := "Hello, SRE World! 这是一段中文内容。"

    // 编码
    encoded := base64.StdEncoding.EncodeToString([]byte(original))
    fmt.Println("标准 Base64 编码:", encoded)

    // 解码
    decoded, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil {
        panic(err)
    }
    fmt.Println("解码结果:", string(decoded))

    // URL 安全的 Base64（用 - 和 _ 替代 + 和 /）
    urlEncoded := base64.URLEncoding.EncodeToString([]byte(original))
    fmt.Println("\nURL安全 Base64:", urlEncoded)

    // 无 padding 的 Base64（去掉末尾的 =）
    rawEncoded := base64.RawStdEncoding.EncodeToString([]byte(original))
    fmt.Println("无 padding Base64:", rawEncoded)

    // SRE 场景：解码 Kubernetes Secret
    k8sSecretValue := "cGFzc3dvcmQxMjM=" // base64("password123")
    secretDecoded, _ := base64.StdEncoding.DecodeString(k8sSecretValue)
    fmt.Printf("\nK8s Secret 解码: %s\n", string(secretDecoded))

    // 编码 Docker config.json 认证信息
    dockerAuth := "registry-user:registry-pass"
    dockerEncoded := base64.StdEncoding.EncodeToString([]byte(dockerAuth))
    fmt.Printf("Docker 认证编码: %s\n", dockerEncoded)
}
```

---

## 6. 实战：解析 Kubernetes API 返回的 JSON

### 6.1 解析 Pod 列表

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

// Kubernetes API 响应结构（简化版）
type PodList struct {
    APIVersion string `json:"apiVersion"`
    Kind       string `json:"kind"`
    Items      []Pod  `json:"items"`
}

type Pod struct {
    Metadata PodMetadata `json:"metadata"`
    Spec     PodSpec     `json:"spec"`
    Status   PodStatus   `json:"status"`
}

type PodMetadata struct {
    Name      string            `json:"name"`
    Namespace string            `json:"namespace"`
    Labels    map[string]string `json:"labels,omitempty"`
}

type PodSpec struct {
    Containers []Container `json:"containers"`
    NodeName   string      `json:"nodeName,omitempty"`
}

type Container struct {
    Name  string `json:"name"`
    Image string `json:"image"`
}

type PodStatus struct {
    Phase             string             `json:"phase"`
    ContainerStatuses []ContainerStatus  `json:"containerStatuses,omitempty"`
    Conditions        []PodCondition     `json:"conditions,omitempty"`
}

type ContainerStatus struct {
    Name         string `json:"name"`
    Ready        bool   `json:"ready"`
    RestartCount int    `json:"restartCount"`
}

type PodCondition struct {
    Type   string `json:"type"`
    Status string `json:"status"`
}

func main() {
    // 模拟 kubectl get pods -o json 的输出
    k8sJSON := `{
        "apiVersion": "v1",
        "kind": "PodList",
        "items": [
            {
                "metadata": {
                    "name": "nginx-deployment-abc123",
                    "namespace": "production",
                    "labels": {
                        "app": "nginx",
                        "version": "1.25"
                    }
                },
                "spec": {
                    "containers": [
                        {"name": "nginx", "image": "nginx:1.25"},
                        {"name": "sidecar", "image": "envoy:1.28"}
                    ],
                    "nodeName": "worker-01"
                },
                "status": {
                    "phase": "Running",
                    "containerStatuses": [
                        {"name": "nginx", "ready": true, "restartCount": 0},
                        {"name": "sidecar", "ready": true, "restartCount": 2}
                    ]
                }
            },
            {
                "metadata": {
                    "name": "redis-master-xyz789",
                    "namespace": "production",
                    "labels": {
                        "app": "redis",
                        "role": "master"
                    }
                },
                "spec": {
                    "containers": [
                        {"name": "redis", "image": "redis:7.2"}
                    ],
                    "nodeName": "worker-02"
                },
                "status": {
                    "phase": "Running",
                    "containerStatuses": [
                        {"name": "redis", "ready": true, "restartCount": 0}
                    ]
                }
            },
            {
                "metadata": {
                    "name": "app-backend-failed456",
                    "namespace": "production",
                    "labels": {
                        "app": "backend"
                    }
                },
                "spec": {
                    "containers": [
                        {"name": "backend", "image": "myapp:latest"}
                    ],
                    "nodeName": "worker-01"
                },
                "status": {
                    "phase": "CrashLoopBackOff",
                    "containerStatuses": [
                        {"name": "backend", "ready": false, "restartCount": 15}
                    ]
                }
            }
        ]
    }`

    var podList PodList
    err := json.Unmarshal([]byte(k8sJSON), &podList)
    if err != nil {
        panic(err)
    }

    fmt.Printf("API 版本: %s, 类型: %s\n", podList.APIVersion, podList.Kind)
    fmt.Printf("共 %d 个 Pod\n\n", len(podList.Items))

    // 表格输出
    fmt.Printf("%-35s %-12s %-10s %-25s %s\n",
        "NAME", "NAMESPACE", "STATUS", "IMAGES", "RESTARTS")
    fmt.Println(strings.Repeat("-", 100))

    for _, pod := range podList.Items {
        var images []string
        for _, c := range pod.Spec.Containers {
            images = append(images, c.Image)
        }

        totalRestarts := 0
        for _, cs := range pod.Status.ContainerStatuses {
            totalRestarts += cs.RestartCount
        }

        fmt.Printf("%-35s %-12s %-10s %-25s %d\n",
            pod.Metadata.Name,
            pod.Metadata.Namespace,
            pod.Status.Phase,
            strings.Join(images, ","),
            totalRestarts,
        )
    }

    // 找出异常 Pod
    fmt.Println("\n--- 异常 Pod ---")
    for _, pod := range podList.Items {
        if pod.Status.Phase != "Running" && pod.Status.Phase != "Succeeded" {
            fmt.Printf("告警: %s 状态异常 (%s)\n", pod.Metadata.Name, pod.Status.Phase)
        }
        for _, cs := range pod.Status.ContainerStatuses {
            if cs.RestartCount > 5 {
                fmt.Printf("告警: %s/%s 重启 %d 次\n", pod.Metadata.Name, cs.Name, cs.RestartCount)
            }
        }
    }
}
```

### 6.2 解析 Kubernetes Service

```go
package main

import (
    "encoding/json"
    "fmt"
)

type K8sService struct {
    Metadata struct {
        Name      string `json:"name"`
        Namespace string `json:"namespace"`
    } `json:"metadata"`
    Spec struct {
        Type      string            `json:"type"`
        ClusterIP string            `json:"clusterIP"`
        Ports     []ServicePort     `json:"ports"`
        Selector  map[string]string `json:"selector"`
    } `json:"spec"`
}

type ServicePort struct {
    Name       string `json:"name"`
    Protocol   string `json:"protocol"`
    Port       int    `json:"port"`
    TargetPort int    `json:"targetPort"`
    NodePort   int    `json:"nodePort,omitempty"`
}

func main() {
    svcJSON := `{
        "metadata": {"name": "web-service", "namespace": "production"},
        "spec": {
            "type": "NodePort",
            "clusterIP": "10.96.0.10",
            "ports": [
                {"name": "http", "protocol": "TCP", "port": 80, "targetPort": 8080, "nodePort": 30080},
                {"name": "https", "protocol": "TCP", "port": 443, "targetPort": 8443, "nodePort": 30443}
            ],
            "selector": {"app": "web", "version": "v2"}
        }
    }`

    var svc K8sService
    json.Unmarshal([]byte(svcJSON), &svc)

    fmt.Printf("Service: %s/%s\n", svc.Metadata.Namespace, svc.Metadata.Name)
    fmt.Printf("Type: %s, ClusterIP: %s\n", svc.Spec.Type, svc.Spec.ClusterIP)
    fmt.Printf("Selector: %v\n", svc.Spec.Selector)
    fmt.Println("Ports:")
    for _, p := range svc.Spec.Ports {
        fmt.Printf("  %s: %d → %d (NodePort: %d)\n", p.Name, p.Port, p.TargetPort, p.NodePort)
    }
}
```

---

## 7. SRE 实战场景

### 7.1 JSON 配置文件加载器

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type AppConfig struct {
    Server struct {
        Host         string `json:"host"`
        Port         int    `json:"port"`
        ReadTimeout  string `json:"read_timeout"`
        WriteTimeout string `json:"write_timeout"`
    } `json:"server"`
    Database struct {
        Host     string `json:"host"`
        Port     int    `json:"port"`
        Name     string `json:"name"`
        User     string `json:"user"`
        Password string `json:"password"`
        MaxConns int    `json:"max_conns"`
    } `json:"database"`
    Logging struct {
        Level  string `json:"level"`
        Format string `json:"format"`
        Output string `json:"output"`
    } `json:"logging"`
}

func LoadConfig(filename string) (*AppConfig, error) {
    file, err := os.Open(filename)
    if err != nil {
        return nil, fmt.Errorf("打开配置文件失败: %w", err)
    }
    defer file.Close()

    var cfg AppConfig
    decoder := json.NewDecoder(file)
    decoder.DisallowUnknownFields() // 拒绝未知字段（防止拼写错误的配置项）
    if err := decoder.Decode(&cfg); err != nil {
        return nil, fmt.Errorf("解析配置文件失败: %w", err)
    }

    return &cfg, nil
}

func main() {
    // 创建测试配置文件
    configJSON := `{
        "server": {
            "host": "0.0.0.0",
            "port": 8080,
            "read_timeout": "15s",
            "write_timeout": "15s"
        },
        "database": {
            "host": "localhost",
            "port": 5432,
            "name": "myapp",
            "user": "admin",
            "password": "secret",
            "max_conns": 20
        },
        "logging": {
            "level": "info",
            "format": "json",
            "output": "/var/log/app.log"
        }
    }`
    os.WriteFile("/tmp/app_config.json", []byte(configJSON), 0644)

    cfg, err := LoadConfig("/tmp/app_config.json")
    if err != nil {
        fmt.Println("错误:", err)
        return
    }

    fmt.Printf("服务器: %s:%d\n", cfg.Server.Host, cfg.Server.Port)
    fmt.Printf("数据库: %s@%s:%d/%s\n",
        cfg.Database.User, cfg.Database.Host, cfg.Database.Port, cfg.Database.Name)
    fmt.Printf("日志: 级别=%s, 格式=%s\n", cfg.Logging.Level, cfg.Logging.Format)
}
```

### 7.2 CSV 格式的监控报告生成器

```go
package main

import (
    "encoding/csv"
    "fmt"
    "math/rand"
    "os"
    "strconv"
    "time"
)

func main() {
    file, err := os.Create("/tmp/monitoring_report.csv")
    if err != nil {
        panic(err)
    }
    defer file.Close()

    writer := csv.NewWriter(file)
    defer writer.Flush()

    // 写入表头
    writer.Write([]string{
        "timestamp", "hostname", "cpu_percent", "memory_percent",
        "disk_percent", "network_rx_mbps", "network_tx_mbps", "status",
    })

    hosts := []string{"web-01", "web-02", "web-03", "db-01", "cache-01"}
    baseTime := time.Now().Add(-1 * time.Hour)

    // 生成模拟数据
    for i := 0; i < 60; i++ {
        timestamp := baseTime.Add(time.Duration(i) * time.Minute)
        for _, host := range hosts {
            cpu := 30.0 + rand.Float64()*60.0
            mem := 40.0 + rand.Float64()*50.0
            disk := 20.0 + rand.Float64()*30.0
            rxMbps := rand.Float64() * 100
            txMbps := rand.Float64() * 50

            status := "healthy"
            if cpu > 85 || mem > 90 {
                status = "warning"
            }
            if cpu > 95 || mem > 95 {
                status = "critical"
            }

            writer.Write([]string{
                timestamp.Format(time.RFC3339),
                host,
                strconv.FormatFloat(cpu, 'f', 1, 64),
                strconv.FormatFloat(mem, 'f', 1, 64),
                strconv.FormatFloat(disk, 'f', 1, 64),
                strconv.FormatFloat(rxMbps, 'f', 2, 64),
                strconv.FormatFloat(txMbps, 'f', 2, 64),
                status,
            })
        }
    }

    fmt.Println("监控报告已生成: /tmp/monitoring_report.csv")
    fmt.Printf("共 %d 条记录 (%d 台主机 x %d 分钟)\n", 60*len(hosts), len(hosts), 60)
}
```

---

## 常见坑点

### 坑 1：JSON 数字默认解析为 float64

```go
// 当使用 map[string]interface{} 解析时，所有数字都是 float64
var data map[string]interface{}
json.Unmarshal([]byte(`{"count": 42}`), &data)
count := data["count"].(float64) // 是 float64，不是 int！

// 如果需要精确控制，使用 json.Decoder + UseNumber
decoder := json.NewDecoder(strings.NewReader(`{"count": 42}`))
decoder.UseNumber()
decoder.Decode(&data)
num := data["count"].(json.Number)
intVal, _ := num.Int64() // 精确获取 int64
```

### 坑 2：未导出字段不会被序列化

```go
type Config struct {
    Host     string `json:"host"`
    port     int    `json:"port"` // 小写开头 → 未导出 → JSON 忽略！
    Password string `json:"-"`    // 显式忽略
}
// port 字段永远不会出现在 JSON 中（也不会被反序列化填充）
```

### 坑 3：nil slice 和 空 slice 序列化不同

```go
var nilSlice []string        // nil
emptySlice := []string{}     // 空但非 nil

json.Marshal(nilSlice)       // "null"
json.Marshal(emptySlice)     // "[]"

// 如果 API 契约要求数组不能为 null，初始化为空切片
```

### 坑 4：time.Time 的默认 JSON 格式

```go
// time.Time 默认序列化为 RFC3339 格式
// "2024-01-15T10:00:00Z"
// 如果需要自定义格式，使用自定义 MarshalJSON（见 1.5 节）
```

### 坑 5：json.Decoder 读取后 Body 被消费

```go
// HTTP handler 中，Body 只能读一次
func handler(w http.ResponseWriter, r *http.Request) {
    var data map[string]interface{}
    json.NewDecoder(r.Body).Decode(&data) // Body 被消费
    // r.Body 已经空了，不能再次读取！

    // 如果需要多次读取，先保存：
    // bodyBytes, _ := io.ReadAll(r.Body)
    // json.Unmarshal(bodyBytes, &data1)
    // json.Unmarshal(bodyBytes, &data2)
}
```

---

## 速查表

```text
┌──────────────────────────────────────────────────────────────────────┐
│                    encoding 速查表                                     │
├──────────────────────────────────────────────────────────────────────┤
│ JSON                                                                  │
│   结构体→JSON     │ json.Marshal(v) / json.MarshalIndent(v,"","  ")  │
│   JSON→结构体     │ json.Unmarshal(data, &v)                         │
│   流式编码        │ json.NewEncoder(w).Encode(v)                     │
│   流式解码        │ json.NewDecoder(r).Decode(&v)                    │
│   字段映射        │ `json:"name"`                                    │
│   省略零值        │ `json:"name,omitempty"`                          │
│   忽略字段        │ `json:"-"`                                       │
│   数值转字符串    │ `json:"name,string"`                             │
├──────────────────────────────────────────────────────────────────────┤
│ XML                                                                   │
│   编码            │ xml.Marshal(v) / xml.MarshalIndent(v,"","  ")    │
│   解码            │ xml.Unmarshal(data, &v)                          │
│   根元素          │ `xml:"tagname"` 在 XMLName 字段                  │
│   子元素          │ `xml:"parent>child"`                             │
├──────────────────────────────────────────────────────────────────────┤
│ CSV                                                                   │
│   读取全部        │ csv.NewReader(r).ReadAll()                       │
│   逐行读取        │ reader.Read()                                    │
│   写入            │ csv.NewWriter(w).Write(record)                   │
│   刷新            │ writer.Flush()                                   │
├──────────────────────────────────────────────────────────────────────┤
│ Base64                                                                │
│   标准编码        │ base64.StdEncoding.EncodeToString(data)          │
│   标准解码        │ base64.StdEncoding.DecodeString(str)             │
│   URL安全         │ base64.URLEncoding                               │
│   无padding       │ base64.RawStdEncoding                            │
├──────────────────────────────────────────────────────────────────────┤
│ Gob (Go专用)                                                         │
│   编码            │ gob.NewEncoder(w).Encode(v)                      │
│   解码            │ gob.NewDecoder(r).Decode(&v)                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 小结

- `encoding/json` 是最常用的编码包，掌握 struct tag 规则和流式编解码是关键。
- 动态 JSON 用 `map[string]interface{}`，但要注意数字类型为 `float64`。
- 自定义序列化通过实现 `MarshalJSON` / `UnmarshalJSON` 接口。
- CSV 适合导出监控数据和报表，注意 Writer 必须 `Flush`。
- Base64 在 K8s Secret、Docker 认证等场景中高频出现。
- XML 用于解析老系统的配置文件（如 Maven pom.xml、Nginx 的部分配置）。
- Gob 是 Go 专有格式，适合 Go 微服务之间的高效通信。
