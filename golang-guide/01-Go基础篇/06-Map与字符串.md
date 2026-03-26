# 06 - Map 与字符串

> **适用读者**：有 Python / Shell 经验的 SRE / DevOps 工程师
> **Go 版本**：Go 1.24

---

## 导读

Map（映射）是 Go 中的哈希表实现，类似 Python 的 dict。字符串是 Go 中最常处理的数据类型之一——解析配置、处理日志、拼接 URL 都离不开它。本篇将深入讲解 map 的使用方式和并发安全问题，以及字符串操作的性能陷阱。

---

## 1. Map 的声明与初始化

```go
package main

import "fmt"

func main() {
    // ============ 方式一：make 创建 ============
    m1 := make(map[string]int)
    m1["web-01"] = 8080
    m1["api-01"] = 9090
    fmt.Println("make:", m1)

    // 预分配容量（推荐，减少扩容）
    m2 := make(map[string]int, 100) // 预计 100 个 key
    m2["key"] = 1
    fmt.Println("make with cap:", m2)

    // ============ 方式二：字面量创建 ============
    m3 := map[string]int{
        "web-01": 8080,
        "web-02": 8081,
        "api-01": 9090,
    }
    fmt.Println("字面量:", m3)

    // 复杂类型
    servers := map[string][]string{
        "production": {"web-01", "web-02", "web-03"},
        "staging":    {"stg-01"},
        "dev":        {"dev-01", "dev-02"},
    }
    fmt.Println("复杂类型:", servers)

    // ============ nil map vs 空 map ============
    var nilMap map[string]int  // nil map
    emptyMap := map[string]int{} // 空 map

    fmt.Printf("\nnil map: %v, nil=%v\n", nilMap, nilMap == nil)
    fmt.Printf("空 map: %v, nil=%v\n", emptyMap, emptyMap == nil)

    // nil map 可以读，但不能写！
    _ = nilMap["key"]        // 返回零值，不 panic
    // nilMap["key"] = 1     // panic: assignment to entry in nil map

    // 空 map 可读可写
    emptyMap["key"] = 1      // 正常
    fmt.Println("空 map:", emptyMap)
}
```

> **与 Python 对比**：
> - Python: `d = {}` 或 `d = dict()` — 都创建空字典
> - Go: `var m map[K]V` 是 nil map（不能写），`m := map[K]V{}` 或 `m := make(map[K]V)` 才能写

---

## 2. Map 增删改查

```go
package main

import "fmt"

func main() {
    config := map[string]string{
        "host":     "localhost",
        "port":     "8080",
        "database": "myapp",
    }

    // ============ 增/改 ============
    config["host"] = "0.0.0.0"      // 修改
    config["timeout"] = "30s"        // 新增
    fmt.Println("增改后:", config)

    // ============ 查 ============
    host := config["host"]
    fmt.Println("host:", host)

    // 访问不存在的 key，返回零值（不会 panic）
    missing := config["not_exist"]
    fmt.Printf("不存在的 key: %q (空字符串)\n", missing)

    // ============ 删 ============
    delete(config, "timeout")
    fmt.Println("删除后:", config)

    // 删除不存在的 key 不会 panic
    delete(config, "not_exist") // 静默操作

    // ============ 清空 map ============
    // 方式一：Go 1.21+ clear 内建函数
    temp := map[string]int{"a": 1, "b": 2}
    clear(temp)
    fmt.Println("clear 后:", temp, "len:", len(temp))

    // 方式二：重新赋值（Go 1.21 之前）
    // temp = make(map[string]int)

    // ============ 获取 map 大小 ============
    fmt.Println("配置项数量:", len(config))
}
```

---

## 3. 判断 key 是否存在（comma ok 模式）

```go
package main

import "fmt"

func main() {
    ports := map[string]int{
        "http":  80,
        "https": 443,
        "ssh":   22,
        "ftp":   0, // 注意：值为 0 是合法的
    }

    // ============ comma ok 模式（最重要的 map 用法） ============
    // 第二个返回值 ok 表示 key 是否存在
    if port, ok := ports["ssh"]; ok {
        fmt.Println("SSH 端口:", port)
    } else {
        fmt.Println("SSH 端口未配置")
    }

    // 区分"key 不存在"和"值为零值"
    // 如果不用 comma ok，无法区分这两种情况
    port1 := ports["ftp"]
    port2 := ports["telnet"]
    fmt.Printf("ftp=%d, telnet=%d (都是 0，但含义不同！)\n", port1, port2)

    // 正确方式
    if _, ok := ports["ftp"]; ok {
        fmt.Println("ftp 存在，端口为 0")
    }
    if _, ok := ports["telnet"]; !ok {
        fmt.Println("telnet 不存在")
    }

    // ============ 实际场景：环境变量查找 ============
    envVars := map[string]string{
        "HOME":     "/home/devops",
        "PATH":     "/usr/bin:/usr/local/bin",
        "SHELL":    "/bin/bash",
        "LANG":     "",  // 空字符串也是有效值
    }

    getEnv := func(key, defaultVal string) string {
        if val, ok := envVars[key]; ok {
            return val // 即使是空字符串也返回
        }
        return defaultVal
    }

    fmt.Println("\nHOME:", getEnv("HOME", "/root"))
    fmt.Println("LANG:", getEnv("LANG", "en_US.UTF-8")) // 返回 ""
    fmt.Println("EDITOR:", getEnv("EDITOR", "vim"))      // 返回 "vim"

    // ============ 实际场景：计数器 ============
    logs := []string{"ERROR", "INFO", "ERROR", "WARN", "INFO", "INFO", "ERROR"}
    counter := make(map[string]int)
    for _, level := range logs {
        counter[level]++ // key 不存在时自动用零值 0，然后 +1
    }
    fmt.Println("\n日志统计:", counter)
}
```

> **与 Python 对比**：
> - Python: `if key in d:` 或 `d.get(key, default)`
> - Go: `val, ok := m[key]` — comma ok 模式是 Go 的标准写法

---

## 4. Map 遍历的随机性

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    m := map[string]int{
        "cherry": 3,
        "apple":  1,
        "banana": 2,
        "date":   4,
        "elder":  5,
    }

    // ============ map 遍历顺序是随机的 ============
    fmt.Println("=== 第一次遍历 ===")
    for k, v := range m {
        fmt.Printf("  %s: %d\n", k, v)
    }

    fmt.Println("=== 第二次遍历（顺序可能不同） ===")
    for k, v := range m {
        fmt.Printf("  %s: %d\n", k, v)
    }

    // Go 故意将 map 遍历顺序随机化
    // 防止开发者依赖特定的遍历顺序

    // ============ 有序遍历：先排序 key ============
    fmt.Println("\n=== 有序遍历 ===")

    // 收集所有 key
    keys := make([]string, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    sort.Strings(keys)

    // 按排序后的 key 遍历
    for _, k := range keys {
        fmt.Printf("  %s: %d\n", k, m[k])
    }

    // ============ 按值排序 ============
    fmt.Println("\n=== 按值排序 ===")
    type kv struct {
        Key   string
        Value int
    }
    pairs := make([]kv, 0, len(m))
    for k, v := range m {
        pairs = append(pairs, kv{k, v})
    }
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].Value < pairs[j].Value
    })
    for _, p := range pairs {
        fmt.Printf("  %s: %d\n", p.Key, p.Value)
    }
}
```

---

## 5. Map 并发安全问题

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // ============ map 不是并发安全的！ ============
    // 多个 goroutine 同时读写一个 map 会导致 panic:
    // fatal error: concurrent map read and map write

    // 错误示例（取消注释会 panic）：
    /*
    m := make(map[int]int)
    go func() {
        for i := 0; i < 10000; i++ {
            m[i] = i
        }
    }()
    go func() {
        for i := 0; i < 10000; i++ {
            _ = m[i]
        }
    }()
    time.Sleep(time.Second)
    */

    // ============ 解决方案一：sync.Mutex ============
    fmt.Println("=== sync.Mutex ===")
    safeMap := &SafeMap{
        data: make(map[string]int),
    }

    var wg sync.WaitGroup
    // 并发写
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", n)
            safeMap.Set(key, n)
        }(i)
    }
    wg.Wait()
    fmt.Printf("SafeMap 大小: %d\n", safeMap.Len())

    // ============ 解决方案二：sync.RWMutex（读多写少场景推荐） ============
    fmt.Println("\n=== sync.RWMutex ===")
    rwMap := &RWMap{
        data: make(map[string]int),
    }

    // 并发读写
    for i := 0; i < 50; i++ {
        wg.Add(2)
        go func(n int) {
            defer wg.Done()
            rwMap.Set(fmt.Sprintf("key-%d", n), n)
        }(i)
        go func(n int) {
            defer wg.Done()
            rwMap.Get(fmt.Sprintf("key-%d", n))
        }(i)
    }
    wg.Wait()
    fmt.Printf("RWMap 大小: %d\n", rwMap.Len())
}

// SafeMap 使用 Mutex 的线程安全 map
type SafeMap struct {
    mu   sync.Mutex
    data map[string]int
}

func (m *SafeMap) Set(key string, value int) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *SafeMap) Get(key string) (int, bool) {
    m.mu.Lock()
    defer m.mu.Unlock()
    v, ok := m.data[key]
    return v, ok
}

func (m *SafeMap) Len() int {
    m.mu.Lock()
    defer m.mu.Unlock()
    return len(m.data)
}

// RWMap 使用 RWMutex 的线程安全 map（读多写少场景性能更好）
type RWMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func (m *RWMap) Set(key string, value int) {
    m.mu.Lock()         // 写锁
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *RWMap) Get(key string) (int, bool) {
    m.mu.RLock()        // 读锁（多个 goroutine 可以同时持有读锁）
    defer m.mu.RUnlock()
    v, ok := m.data[key]
    return v, ok
}

func (m *RWMap) Len() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.data)
}
```

---

## 6. sync.Map 简介

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // sync.Map 是标准库提供的并发安全 map
    // 适用场景：
    // 1. key 相对稳定，以读为主
    // 2. 多个 goroutine 读写不同的 key
    // 不适用场景：
    // 1. 频繁的写操作
    // 2. 需要获取 map 大小
    // 3. 需要遍历的场景

    var m sync.Map

    // Store：存储
    m.Store("web-01", 8080)
    m.Store("api-01", 9090)
    m.Store("db-01", 5432)

    // Load：读取
    if port, ok := m.Load("web-01"); ok {
        fmt.Println("web-01 端口:", port)
    }

    // LoadOrStore：如果 key 存在就返回，不存在就存储
    actual, loaded := m.LoadOrStore("web-01", 8081)
    fmt.Printf("LoadOrStore web-01: actual=%v, loaded=%v (已存在)\n", actual, loaded)

    actual, loaded = m.LoadOrStore("cache-01", 6379)
    fmt.Printf("LoadOrStore cache-01: actual=%v, loaded=%v (新存储)\n", actual, loaded)

    // LoadAndDelete：原子地读取并删除
    old, loaded := m.LoadAndDelete("db-01")
    fmt.Printf("LoadAndDelete db-01: old=%v, loaded=%v\n", old, loaded)

    // Delete：删除
    m.Delete("api-01")

    // Range：遍历
    fmt.Println("\n遍历:")
    m.Range(func(key, value interface{}) bool {
        fmt.Printf("  %v: %v\n", key, value)
        return true // 返回 false 可以提前终止
    })

    // CompareAndSwap / CompareAndDelete（Go 1.20+）
    m.Store("counter", 0)
    m.CompareAndSwap("counter", 0, 1) // 如果当前值是 0，就改为 1
    v, _ := m.Load("counter")
    fmt.Println("\nCompareAndSwap 后:", v)

    // ============ 注意事项 ============
    // 1. sync.Map 没有 Len() 方法
    // 2. 值的类型是 interface{}，需要类型断言
    // 3. 大多数场景下 Mutex + map 就够了
    // 4. 只有在性能测试证明需要时才使用 sync.Map

    // 并发安全演示
    var wg sync.WaitGroup
    var sm sync.Map

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", n%10)
            sm.Store(key, n)
            sm.Load(key)
        }(i)
    }
    wg.Wait()
    fmt.Println("并发操作完成，无 panic")
}
```

---

## 7. 字符串不可变性

```go
package main

import "fmt"

func main() {
    // Go 字符串是不可变的字节序列
    s := "Hello"

    // 不能修改单个字节
    // s[0] = 'h' // 编译错误：cannot assign to s[0]

    // 可以重新赋值（创建新字符串）
    s = "hello" // 这是一个全新的字符串，不是修改

    // 要修改字符内容，必须转换为 []byte 或 []rune
    bs := []byte(s)
    bs[0] = 'H'
    s = string(bs)
    fmt.Println(s) // "Hello"

    // ============ 字符串的底层结构 ============
    // type stringHeader struct {
    //     Data unsafe.Pointer  // 指向字节数组的指针
    //     Len  int             // 字节长度
    // }
    // 字符串头只有 16 字节，赋值和传参不会复制底层数据

    s1 := "Hello, World!"
    s2 := s1           // 只复制了字符串头（指针+长度），不复制内容
    s3 := s1[0:5]      // 子字符串也共享底层数据

    fmt.Println(s2, s3) // Hello, World! Hello

    // 注意：子字符串会阻止原字符串被 GC
    // 如果原字符串很大，子字符串很小，可能导致内存浪费
    // 解决：使用 strings.Clone（Go 1.20+）或 string([]byte(s[0:5]))
}
```

---

## 8. string / []byte / []rune 转换

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "Hello, 世界! 🌍"

    // ============ 三种视角看同一个字符串 ============
    fmt.Println("=== 原始字符串 ===")
    fmt.Println("字符串:", s)
    fmt.Println("len():", len(s))                                  // 字节数：20
    fmt.Println("utf8.RuneCountInString():", utf8.RuneCountInString(s)) // 字符数：11

    // string -> []byte (UTF-8 字节序列)
    fmt.Println("\n=== []byte 视角 ===")
    bs := []byte(s)
    fmt.Printf("字节数: %d\n", len(bs))
    fmt.Printf("字节: %v\n", bs)
    for i, b := range bs {
        fmt.Printf("  [%2d] %3d (0x%02X)\n", i, b, b)
    }

    // string -> []rune (Unicode 码点序列)
    fmt.Println("\n=== []rune 视角 ===")
    rs := []rune(s)
    fmt.Printf("字符数: %d\n", len(rs))
    for i, r := range rs {
        fmt.Printf("  [%d] %c (U+%04X, %d bytes)\n", i, r, r, utf8.RuneLen(r))
    }

    // ============ 转换开销 ============
    // string <-> []byte 会复制数据（分配新内存）
    // string <-> []rune 也会复制数据
    // 频繁转换有性能开销！

    // ============ 实际应用 ============
    // 1. 需要修改字符串：转 []byte 或 []rune
    // 2. 需要处理非 ASCII 字符：转 []rune
    // 3. 需要操作字节（加密、编码）：转 []byte

    // 修改 ASCII 字符串用 []byte
    greeting := "hello world"
    b := []byte(greeting)
    b[0] = 'H'
    b[6] = 'W'
    fmt.Println("\n修改后:", string(b))

    // 修改含中文的字符串用 []rune
    chinese := "你好世界"
    r := []rune(chinese)
    r[2] = '地'
    r[3] = '球'
    fmt.Println("修改后:", string(r)) // "你好地球"

    // 安全地截取含多字节字符的字符串
    mixed := "Go语言编程"
    // 错误：直接按字节截取可能截断多字节字符
    // fmt.Println(mixed[:4]) // 可能显示乱码

    // 正确：按 rune 截取
    runes := []rune(mixed)
    fmt.Println("前3个字符:", string(runes[:3])) // "Go语"
}
```

---

## 9. strings 包常用函数

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // ============ 查找与判断 ============
    fmt.Println("=== 查找与判断 ===")

    s := "Hello, World! Hello, Go!"

    fmt.Println("Contains:", strings.Contains(s, "World"))          // true
    fmt.Println("ContainsAny:", strings.ContainsAny(s, "xyz"))      // false
    fmt.Println("HasPrefix:", strings.HasPrefix(s, "Hello"))        // true
    fmt.Println("HasSuffix:", strings.HasSuffix(s, "Go!"))          // true
    fmt.Println("Index:", strings.Index(s, "World"))                 // 7
    fmt.Println("LastIndex:", strings.LastIndex(s, "Hello"))         // 14
    fmt.Println("Count:", strings.Count(s, "Hello"))                 // 2
    fmt.Println("EqualFold:", strings.EqualFold("Go", "go"))         // true（大小写不敏感比较）

    // ============ 转换 ============
    fmt.Println("\n=== 转换 ===")

    fmt.Println("ToUpper:", strings.ToUpper("hello"))                 // HELLO
    fmt.Println("ToLower:", strings.ToLower("HELLO"))                 // hello
    fmt.Println("ToTitle:", strings.ToTitle("hello world"))           // HELLO WORLD
    fmt.Println("Title:", strings.Title("hello world"))               // Hello World（已废弃）

    // ============ 修剪 ============
    fmt.Println("\n=== 修剪 ===")

    fmt.Println("TrimSpace:", strings.TrimSpace("  hello  "))        // "hello"
    fmt.Println("Trim:", strings.Trim("***hello***", "*"))            // "hello"
    fmt.Println("TrimLeft:", strings.TrimLeft("***hello***", "*"))    // "hello***"
    fmt.Println("TrimRight:", strings.TrimRight("***hello***", "*"))  // "***hello"
    fmt.Println("TrimPrefix:", strings.TrimPrefix("HelloWorld", "Hello")) // "World"
    fmt.Println("TrimSuffix:", strings.TrimSuffix("hello.go", ".go"))     // "hello"

    // ============ 分割与合并 ============
    fmt.Println("\n=== 分割与合并 ===")

    // Split
    csv := "web-01,web-02,api-01,db-01"
    parts := strings.Split(csv, ",")
    fmt.Println("Split:", parts) // [web-01 web-02 api-01 db-01]

    // SplitN（限制分割次数）
    kv := "host=localhost:8080"
    pair := strings.SplitN(kv, "=", 2)
    fmt.Println("SplitN:", pair) // [host localhost:8080]

    // Fields（按空白分割，自动处理多个空格）
    line := "  web-01   8080   running  "
    fields := strings.Fields(line)
    fmt.Println("Fields:", fields) // [web-01 8080 running]

    // Join
    servers := []string{"web-01", "web-02", "api-01"}
    joined := strings.Join(servers, ", ")
    fmt.Println("Join:", joined) // "web-01, web-02, api-01"

    // ============ 替换 ============
    fmt.Println("\n=== 替换 ===")

    template := "Hello, {name}! Welcome to {place}."
    result := strings.ReplaceAll(template, "{name}", "DevOps")
    result = strings.ReplaceAll(result, "{place}", "Go World")
    fmt.Println("ReplaceAll:", result)

    // Replace（限制替换次数）
    s2 := "aaa-bbb-ccc-ddd"
    fmt.Println("Replace(2):", strings.Replace(s2, "-", "_", 2)) // "aaa_bbb_ccc-ddd"

    // ============ 重复与填充 ============
    fmt.Println("\n=== 重复 ===")
    fmt.Println("Repeat:", strings.Repeat("=", 40))
    fmt.Println("Repeat:", strings.Repeat("ab", 5))

    // ============ Replacer（高效的多重替换） ============
    fmt.Println("\n=== Replacer ===")
    replacer := strings.NewReplacer(
        "{host}", "localhost",
        "{port}", "8080",
        "{db}", "myapp",
    )
    dsn := replacer.Replace("postgres://{host}:{port}/{db}")
    fmt.Println("DSN:", dsn)

    // ============ Map 函数（逐字符转换） ============
    fmt.Println("\n=== Map ===")
    // 删除所有数字
    noDigits := strings.Map(func(r rune) rune {
        if r >= '0' && r <= '9' {
            return -1 // 返回负数表示删除
        }
        return r
    }, "web-01 port: 8080")
    fmt.Println("删除数字:", noDigits)

    // ============ Cut（Go 1.18+，非常实用） ============
    fmt.Println("\n=== Cut ===")
    addr := "localhost:8080"
    host, port, found := strings.Cut(addr, ":")
    fmt.Printf("Cut(%q, \":\"): host=%q, port=%q, found=%v\n", addr, host, port, found)

    headerLine := "Content-Type: application/json"
    key, value, _ := strings.Cut(headerLine, ": ")
    fmt.Printf("Header: key=%q, value=%q\n", key, value)

    // CutPrefix / CutSuffix（Go 1.20+）
    path := "/api/v1/users"
    after, found := strings.CutPrefix(path, "/api/v1")
    fmt.Printf("CutPrefix: %q, found=%v\n", after, found) // "/users", true

    filename := "config.yaml"
    before, found := strings.CutSuffix(filename, ".yaml")
    fmt.Printf("CutSuffix: %q, found=%v\n", before, found) // "config", true
}
```

---

## 10. strconv 包类型转换

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // ============ 字符串 -> 数字 ============
    fmt.Println("=== 字符串 -> 数字 ===")

    // Atoi: 字符串 -> int
    n, err := strconv.Atoi("42")
    fmt.Printf("Atoi(\"42\") = %d, err = %v\n", n, err)

    // ParseInt: 字符串 -> int64（支持进制）
    n64, _ := strconv.ParseInt("FF", 16, 64)
    fmt.Printf("ParseInt(\"FF\", 16) = %d\n", n64)

    n64, _ = strconv.ParseInt("0755", 8, 64)
    fmt.Printf("ParseInt(\"0755\", 8) = %d\n", n64)

    // ParseFloat: 字符串 -> float64
    f, _ := strconv.ParseFloat("3.14159", 64)
    fmt.Printf("ParseFloat(\"3.14159\") = %f\n", f)

    // ParseBool: 字符串 -> bool
    b, _ := strconv.ParseBool("true")
    fmt.Printf("ParseBool(\"true\") = %v\n", b)
    // 接受: 1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False

    // ParseUint: 字符串 -> uint64
    u, _ := strconv.ParseUint("255", 10, 8)
    fmt.Printf("ParseUint(\"255\", 10, 8) = %d\n", u)

    // ============ 数字 -> 字符串 ============
    fmt.Println("\n=== 数字 -> 字符串 ===")

    // Itoa: int -> 字符串
    s := strconv.Itoa(42)
    fmt.Printf("Itoa(42) = %q\n", s)

    // FormatInt: int64 -> 字符串（支持进制）
    s = strconv.FormatInt(255, 16)
    fmt.Printf("FormatInt(255, 16) = %q\n", s) // "ff"

    s = strconv.FormatInt(255, 2)
    fmt.Printf("FormatInt(255, 2) = %q\n", s) // "11111111"

    // FormatFloat: float64 -> 字符串
    s = strconv.FormatFloat(3.14, 'f', 2, 64) // 'f'格式, 2位精度, float64
    fmt.Printf("FormatFloat(3.14, 'f', 2) = %q\n", s)

    s = strconv.FormatFloat(3.14, 'e', -1, 64)
    fmt.Printf("FormatFloat(3.14, 'e') = %q\n", s) // 科学计数法

    s = strconv.FormatFloat(3.14, 'g', -1, 64) // 自动选择
    fmt.Printf("FormatFloat(3.14, 'g') = %q\n", s)

    // FormatBool: bool -> 字符串
    s = strconv.FormatBool(true)
    fmt.Printf("FormatBool(true) = %q\n", s)

    // ============ Append 系列（高性能场景） ============
    fmt.Println("\n=== Append 系列 ===")
    buf := []byte("端口号: ")
    buf = strconv.AppendInt(buf, 8080, 10)
    fmt.Println(string(buf))

    // ============ Quote 系列 ============
    fmt.Println("\n=== Quote 系列 ===")
    fmt.Println(strconv.Quote("Hello\tWorld\n"))  // "Hello\tWorld\n"
    fmt.Println(strconv.QuoteRune('中'))           // '中'
    s2, _ := strconv.Unquote(`"Hello\tWorld"`)
    fmt.Println("Unquote:", s2)

    // ============ 错误处理 ============
    fmt.Println("\n=== 错误处理 ===")
    _, err = strconv.Atoi("abc")
    if err != nil {
        // err 的类型是 *strconv.NumError
        if numErr, ok := err.(*strconv.NumError); ok {
            fmt.Printf("转换失败: Func=%s, Num=%s, Err=%v\n",
                numErr.Func, numErr.Num, numErr.Err)
        }
    }
}
```

---

## 11. 字符串拼接性能比较

```go
package main

import (
    "bytes"
    "fmt"
    "strings"
    "time"
)

const iterations = 100000

func main() {
    // ============ 方式一：+ 运算符 ============
    start := time.Now()
    s := ""
    for i := 0; i < iterations; i++ {
        s += "a" // 每次都创建新字符串！
    }
    fmt.Printf("+ 运算符:         %v (len=%d)\n", time.Since(start), len(s))

    // ============ 方式二：fmt.Sprintf ============
    start = time.Now()
    s = ""
    for i := 0; i < iterations; i++ {
        s = fmt.Sprintf("%s%s", s, "a")
    }
    fmt.Printf("fmt.Sprintf:      %v (len=%d)\n", time.Since(start), len(s))

    // ============ 方式三：strings.Builder（推荐） ============
    start = time.Now()
    var builder strings.Builder
    builder.Grow(iterations) // 预分配容量
    for i := 0; i < iterations; i++ {
        builder.WriteString("a")
    }
    s = builder.String()
    fmt.Printf("strings.Builder:  %v (len=%d)\n", time.Since(start), len(s))

    // ============ 方式四：bytes.Buffer ============
    start = time.Now()
    var buf bytes.Buffer
    buf.Grow(iterations)
    for i := 0; i < iterations; i++ {
        buf.WriteString("a")
    }
    s = buf.String()
    fmt.Printf("bytes.Buffer:     %v (len=%d)\n", time.Since(start), len(s))

    // ============ 方式五：[]byte + append ============
    start = time.Now()
    bs := make([]byte, 0, iterations)
    for i := 0; i < iterations; i++ {
        bs = append(bs, 'a')
    }
    s = string(bs)
    fmt.Printf("[]byte append:    %v (len=%d)\n", time.Since(start), len(s))

    // ============ 方式六：strings.Join（已有切片时最优） ============
    parts := make([]string, iterations)
    for i := range parts {
        parts[i] = "a"
    }
    start = time.Now()
    s = strings.Join(parts, "")
    fmt.Printf("strings.Join:     %v (len=%d)\n", time.Since(start), len(s))

    // ============ 性能建议 ============
    fmt.Println("\n=== 性能建议 ===")
    fmt.Println("1. 少量拼接（< 5次）：直接用 + 或 fmt.Sprintf")
    fmt.Println("2. 循环拼接：用 strings.Builder（最快）")
    fmt.Println("3. 已有切片：用 strings.Join")
    fmt.Println("4. 需要 io.Writer：用 bytes.Buffer")
    fmt.Println("5. 永远不要在循环中用 += 拼接！")
}
```

---

## 12. 正则表达式基础（regexp 包）

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    // ============ 编译正则表达式 ============
    // Compile：返回 error
    re, err := regexp.Compile(`\d+`)
    if err != nil {
        fmt.Println("正则编译错误:", err)
        return
    }

    // MustCompile：编译失败直接 panic（适合包级变量）
    reIP := regexp.MustCompile(`(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})`)
    reEmail := regexp.MustCompile(`[\w.+-]+@[\w-]+\.[\w.]+`)
    reLogLine := regexp.MustCompile(`(\d{4}-\d{2}-\d{2}T[\d:]+Z)\s+(\w+)\s+(\S+)\s+(.+)`)

    // ============ 匹配 ============
    fmt.Println("=== 匹配 ===")
    fmt.Println("MatchString:", re.MatchString("hello 123 world"))  // true
    fmt.Println("MatchString:", re.MatchString("hello world"))       // false

    matched, _ := regexp.MatchString(`^[a-z]+$`, "hello")
    fmt.Println("直接匹配:", matched) // true

    // ============ 查找 ============
    fmt.Println("\n=== 查找 ===")

    // FindString：找第一个匹配
    text := "端口 8080 和 9090"
    fmt.Println("FindString:", re.FindString(text)) // "8080"

    // FindAllString：找所有匹配
    fmt.Println("FindAll:", re.FindAllString(text, -1)) // ["8080", "9090"]

    // FindAllString 限制数量
    fmt.Println("FindAll(1):", re.FindAllString(text, 1)) // ["8080"]

    // ============ 子匹配（捕获组） ============
    fmt.Println("\n=== 捕获组 ===")

    ipText := "服务器 IP: 192.168.1.100, 网关: 192.168.1.1"
    matches := reIP.FindAllStringSubmatch(ipText, -1)
    for _, match := range matches {
        fmt.Printf("完整匹配: %s, 各段: %s.%s.%s.%s\n",
            match[0], match[1], match[2], match[3], match[4])
    }

    // 命名捕获组
    reNamed := regexp.MustCompile(`(?P<host>[\w.-]+):(?P<port>\d+)`)
    addr := "localhost:8080"
    match := reNamed.FindStringSubmatch(addr)
    if match != nil {
        for i, name := range reNamed.SubexpNames() {
            if i > 0 {
                fmt.Printf("  %s = %s\n", name, match[i])
            }
        }
    }

    // ============ 替换 ============
    fmt.Println("\n=== 替换 ===")

    // 简单替换
    result := re.ReplaceAllString("端口 8080 和 9090", "****")
    fmt.Println("替换:", result) // "端口 **** 和 ****"

    // 使用匹配内容替换
    result = re.ReplaceAllStringFunc("端口 8080 和 9090", func(s string) string {
        return "[" + s + "]"
    })
    fmt.Println("函数替换:", result) // "端口 [8080] 和 [9090]"

    // 使用引用替换
    reDate := regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
    dateStr := "日期: 2025-03-15"
    result = reDate.ReplaceAllString(dateStr, "$3/$2/$1")
    fmt.Println("日期格式转换:", result) // "日期: 15/03/2025"

    // ============ 分割 ============
    fmt.Println("\n=== 分割 ===")
    reSep := regexp.MustCompile(`[,;|\s]+`)
    parts := reSep.Split("web-01,web-02;api-01|db-01 cache-01", -1)
    fmt.Println("分割:", parts)

    // ============ 实际场景：解析日志 ============
    fmt.Println("\n=== 日志解析 ===")
    logLine := "2025-03-15T10:30:00Z ERROR api-01 Connection refused to redis:6379"
    logMatch := reLogLine.FindStringSubmatch(logLine)
    if logMatch != nil {
        fmt.Printf("  时间: %s\n", logMatch[1])
        fmt.Printf("  级别: %s\n", logMatch[2])
        fmt.Printf("  服务: %s\n", logMatch[3])
        fmt.Printf("  消息: %s\n", logMatch[4])
    }

    // ============ 实际场景：邮箱提取 ============
    fmt.Println("\n=== 邮箱提取 ===")
    emailText := "联系人: admin@example.com, 技术: ops+alert@company.io"
    emails := reEmail.FindAllString(emailText, -1)
    fmt.Println("邮箱:", emails)

    // ============ 性能提示 ============
    fmt.Println("\n=== 性能提示 ===")
    fmt.Println("1. 将正则编译为包级变量，避免重复编译")
    fmt.Println("2. 简单的字符串操作用 strings 包，不要用正则")
    fmt.Println("3. Go 的正则使用 RE2 引擎，保证线性时间复杂度")
    fmt.Println("4. 不支持回溯（lookbehind、backreference）")
}
```

---

## 实战场景：配置文件解析器

```go
package main

import (
    "fmt"
    "regexp"
    "strconv"
    "strings"
)

// Config 配置项
type Config struct {
    data     map[string]string
    sections map[string]map[string]string
}

// NewConfig 创建新配置
func NewConfig() *Config {
    return &Config{
        data:     make(map[string]string),
        sections: make(map[string]map[string]string),
    }
}

var (
    reSection = regexp.MustCompile(`^\[([^\]]+)\]$`)
    reKeyVal  = regexp.MustCompile(`^([^=]+)=(.*)$`)
    reComment = regexp.MustCompile(`^\s*[#;]`)
)

// Parse 解析 INI 格式的配置
func (c *Config) Parse(content string) error {
    currentSection := ""

    for lineNum, line := range strings.Split(content, "\n") {
        line = strings.TrimSpace(line)

        // 跳过空行和注释
        if line == "" || reComment.MatchString(line) {
            continue
        }

        // 检查 section
        if match := reSection.FindStringSubmatch(line); match != nil {
            currentSection = match[1]
            if _, exists := c.sections[currentSection]; !exists {
                c.sections[currentSection] = make(map[string]string)
            }
            continue
        }

        // 解析 key=value
        if match := reKeyVal.FindStringSubmatch(line); match != nil {
            key := strings.TrimSpace(match[1])
            value := strings.TrimSpace(match[2])

            // 移除行内注释
            if idx := strings.Index(value, " #"); idx >= 0 {
                value = strings.TrimSpace(value[:idx])
            }

            // 移除引号
            value = strings.Trim(value, `"'`)

            if currentSection != "" {
                c.sections[currentSection][key] = value
            } else {
                c.data[key] = value
            }
            continue
        }

        return fmt.Errorf("第 %d 行语法错误: %q", lineNum+1, line)
    }
    return nil
}

// Get 获取全局配置
func (c *Config) Get(key, defaultVal string) string {
    if v, ok := c.data[key]; ok {
        return v
    }
    return defaultVal
}

// GetSection 获取 section 下的配置
func (c *Config) GetSection(section, key, defaultVal string) string {
    if s, ok := c.sections[section]; ok {
        if v, ok := s[key]; ok {
            return v
        }
    }
    return defaultVal
}

// GetInt 获取整数配置
func (c *Config) GetInt(section, key string, defaultVal int) int {
    s := c.GetSection(section, key, "")
    if s == "" {
        return defaultVal
    }
    n, err := strconv.Atoi(s)
    if err != nil {
        return defaultVal
    }
    return n
}

// GetBool 获取布尔配置
func (c *Config) GetBool(section, key string, defaultVal bool) bool {
    s := c.GetSection(section, key, "")
    if s == "" {
        return defaultVal
    }
    b, err := strconv.ParseBool(s)
    if err != nil {
        return defaultVal
    }
    return b
}

// Dump 输出所有配置
func (c *Config) Dump() string {
    var sb strings.Builder

    // 全局配置
    if len(c.data) > 0 {
        sb.WriteString("[global]\n")
        for k, v := range c.data {
            fmt.Fprintf(&sb, "  %s = %s\n", k, v)
        }
    }

    // 分节配置
    for section, kvs := range c.sections {
        fmt.Fprintf(&sb, "[%s]\n", section)
        for k, v := range kvs {
            fmt.Fprintf(&sb, "  %s = %s\n", k, v)
        }
    }

    return sb.String()
}

func main() {
    configContent := `
# 应用配置文件
app_name = "health-checker"
version = 1.0

[server]
host = 0.0.0.0
port = 8080
read_timeout = 30  # 秒
write_timeout = 30
debug = true

[database]
host = db-primary.internal
port = 5432
name = myapp
user = admin
password = "s3cret"
max_connections = 20

[redis]
host = cache-01.internal
port = 6379
db = 0
pool_size = 10

[logging]
level = info
format = json
output = stdout
`

    cfg := NewConfig()
    if err := cfg.Parse(configContent); err != nil {
        fmt.Println("解析错误:", err)
        return
    }

    fmt.Println("=== 解析结果 ===")
    fmt.Println(cfg.Dump())

    fmt.Println("=== 配置读取 ===")
    fmt.Println("应用名:", cfg.Get("app_name", "unknown"))
    fmt.Println("服务器:", cfg.GetSection("server", "host", "localhost"))
    fmt.Println("端口:", cfg.GetInt("server", "port", 80))
    fmt.Println("调试模式:", cfg.GetBool("server", "debug", false))
    fmt.Println("数据库:", cfg.GetSection("database", "host", "localhost"))
    fmt.Println("日志级别:", cfg.GetSection("logging", "level", "info"))

    // 构建 DSN
    var dsn strings.Builder
    fmt.Fprintf(&dsn, "postgres://%s:%s@%s:%s/%s?sslmode=disable",
        cfg.GetSection("database", "user", ""),
        cfg.GetSection("database", "password", ""),
        cfg.GetSection("database", "host", "localhost"),
        cfg.GetSection("database", "port", "5432"),
        cfg.GetSection("database", "name", ""))
    fmt.Println("\nDSN:", dsn.String())
}
```

---

## 常见坑点

### 坑点 1：nil map 写入 panic

```go
package main

func main() {
    var m map[string]int
    // m["key"] = 1 // panic: assignment to entry in nil map

    // 正确：先初始化
    m = make(map[string]int)
    m["key"] = 1
}
```

### 坑点 2：map 不能取地址

```go
package main

import "fmt"

func main() {
    m := map[string]int{"a": 1}

    // p := &m["a"] // 编译错误：cannot take address of m["a"]

    // 原因：map 的值可能因为扩容而移动，地址不稳定
    // 解决：用临时变量
    v := m["a"]
    fmt.Println(&v)
}
```

### 坑点 3：string(int) 不是数字转字符串

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    n := 65
    fmt.Println(string(n))       // "A"（Unicode 码点）
    fmt.Println(strconv.Itoa(n)) // "65"（正确的转换）
}
```

### 坑点 4：字符串循环拼接性能差

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // 错误：O(n^2) 时间复杂度
    s := ""
    for i := 0; i < 1000; i++ {
        s += "a" // 每次创建新字符串
    }

    // 正确：O(n) 时间复杂度
    var builder strings.Builder
    for i := 0; i < 1000; i++ {
        builder.WriteString("a")
    }
    s = builder.String()
    fmt.Println("长度:", len(s))
}
```

---

## 速查表

### Map 操作速查

| 操作 | 代码 | Python 等价 |
|------|------|-------------|
| 创建 | `m := make(map[K]V)` | `d = {}` |
| 字面量 | `m := map[K]V{k:v}` | `d = {k:v}` |
| 写入 | `m[k] = v` | `d[k] = v` |
| 读取 | `v := m[k]` | `v = d[k]` |
| 检查存在 | `v, ok := m[k]` | `k in d` |
| 删除 | `delete(m, k)` | `del d[k]` |
| 长度 | `len(m)` | `len(d)` |
| 遍历 | `for k, v := range m` | `for k, v in d.items()` |
| 清空 | `clear(m)` | `d.clear()` |

### strings 包速查

| 函数 | 用途 | 示例 |
|------|------|------|
| `Contains` | 包含 | `strings.Contains("hello", "ell")` |
| `HasPrefix` | 前缀 | `strings.HasPrefix("/api", "/")` |
| `HasSuffix` | 后缀 | `strings.HasSuffix("a.go", ".go")` |
| `Split` | 分割 | `strings.Split("a,b,c", ",")` |
| `Join` | 合并 | `strings.Join([]string{"a","b"}, ",")` |
| `Replace` | 替换 | `strings.ReplaceAll(s, old, new)` |
| `TrimSpace` | 去空白 | `strings.TrimSpace("  hi  ")` |
| `ToLower` | 小写 | `strings.ToLower("ABC")` |
| `ToUpper` | 大写 | `strings.ToUpper("abc")` |
| `Cut` | 切分 | `strings.Cut("k:v", ":")` |
| `Fields` | 按空白分割 | `strings.Fields("a  b  c")` |
| `Builder` | 高效拼接 | `var b strings.Builder` |

---

## 小结

1. **Map 必须初始化才能写入**：`var m map[K]V` 是 nil map，写入会 panic。
2. **comma ok 模式**是判断 key 是否存在的标准方式。
3. **Map 遍历顺序随机**：需要有序遍历时先排序 key。
4. **Map 不是并发安全的**：多 goroutine 读写需要加锁（sync.Mutex / sync.RWMutex），或使用 sync.Map。
5. **字符串不可变**：修改内容需要转 `[]byte` 或 `[]rune`。
6. **循环拼接用 strings.Builder**：性能比 `+=` 好几个数量级。
7. **strings.Cut（Go 1.18+）**是解析 key:value 格式的最佳选择。
8. **正则表达式编译为包级变量**：避免重复编译；简单操作用 strings 包更快。

下一篇我们将学习 Go 的结构体与方法——面向对象编程的 Go 风格。
