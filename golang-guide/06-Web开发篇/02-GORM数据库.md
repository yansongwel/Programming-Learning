# 02-GORM数据库

## 导读

数据库是绝大多数后端应用的核心基础设施。GORM 是 Go 语言中最流行的 ORM（对象关系映射）框架，它提供了丰富的功能，让你能够以 Go 的方式操作数据库，而无需手写大量 SQL。对于 SRE/DevOps 工程师，无论是构建 CMDB、工单系统、还是运维自动化平台，都离不开数据库操作。

本章将系统讲解：

- GORM 介绍与安装
- 数据库连接（MySQL/PostgreSQL/SQLite）
- 模型定义（gorm.Model、tag 配置）
- AutoMigrate 自动迁移
- CRUD 操作
- 关联关系（Has One/Has Many/Belongs To/Many To Many）
- 预加载（Preload）
- 事务处理
- Hook 回调
- 原生 SQL
- 日志与慢查询

> 基于 Go 1.24 + GORM v2，所有代码完整可运行。

---

## 一、GORM 介绍与安装

### 1.1 安装

```bash
# GORM 核心库
go get -u gorm.io/gorm

# 数据库驱动（按需安装）
go get -u gorm.io/driver/mysql      # MySQL
go get -u gorm.io/driver/postgres   # PostgreSQL
go get -u gorm.io/driver/sqlite     # SQLite（适合开发和测试）
```

### 1.2 数据库连接

```go
// 文件: database/connect.go
package database

import (
	"fmt"
	"log"
	"time"

	"gorm.io/driver/mysql"
	"gorm.io/driver/postgres"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

// ConnectMySQL 连接 MySQL
func ConnectMySQL() (*gorm.DB, error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"

	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info), // 日志级别
	})
	if err != nil {
		return nil, fmt.Errorf("连接 MySQL 失败: %w", err)
	}

	// 配置连接池
	sqlDB, err := db.DB()
	if err != nil {
		return nil, fmt.Errorf("获取底层 DB 失败: %w", err)
	}
	sqlDB.SetMaxIdleConns(10)           // 最大空闲连接数
	sqlDB.SetMaxOpenConns(100)          // 最大打开连接数
	sqlDB.SetConnMaxLifetime(time.Hour) // 连接最大存活时间
	sqlDB.SetConnMaxIdleTime(10 * time.Minute) // 空闲连接最大存活时间

	return db, nil
}

// ConnectPostgreSQL 连接 PostgreSQL
func ConnectPostgreSQL() (*gorm.DB, error) {
	dsn := "host=localhost user=postgres password=postgres dbname=mydb port=5432 sslmode=disable TimeZone=Asia/Shanghai"

	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		return nil, fmt.Errorf("连接 PostgreSQL 失败: %w", err)
	}

	sqlDB, err := db.DB()
	if err != nil {
		return nil, err
	}
	sqlDB.SetMaxIdleConns(10)
	sqlDB.SetMaxOpenConns(100)
	sqlDB.SetConnMaxLifetime(time.Hour)

	return db, nil
}

// ConnectSQLite 连接 SQLite（适合开发和测试）
func ConnectSQLite() (*gorm.DB, error) {
	db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		return nil, fmt.Errorf("连接 SQLite 失败: %w", err)
	}
	return db, nil
}

// ConnectSQLiteMemory 内存 SQLite（测试专用）
func ConnectSQLiteMemory() (*gorm.DB, error) {
	db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
	if err != nil {
		return nil, fmt.Errorf("连接内存 SQLite 失败: %w", err)
	}
	return db, nil
}
```

---

## 二、模型定义

### 2.1 gorm.Model 与 Tag 配置

```go
// 文件: models/models.go
package models

import (
	"time"

	"gorm.io/gorm"
)

// gorm.Model 内置模型，包含 ID、CreatedAt、UpdatedAt、DeletedAt 四个字段
// type Model struct {
//     ID        uint           `gorm:"primarykey"`
//     CreatedAt time.Time
//     UpdatedAt time.Time
//     DeletedAt gorm.DeletedAt `gorm:"index"`
// }

// User 用户模型——使用 gorm.Model
type User struct {
	gorm.Model           // 嵌入 ID, CreatedAt, UpdatedAt, DeletedAt
	Username  string     `gorm:"type:varchar(50);uniqueIndex;not null;comment:用户名"`
	Email     string     `gorm:"type:varchar(200);uniqueIndex;not null;comment:邮箱"`
	Password  string     `gorm:"type:varchar(255);not null;comment:密码哈希"`
	Nickname  string     `gorm:"type:varchar(100);default:'';comment:昵称"`
	Age       int        `gorm:"default:0;comment:年龄"`
	Role      string     `gorm:"type:varchar(20);default:'user';comment:角色"`
	IsActive  bool       `gorm:"default:true;comment:是否激活"`
	LastLogin *time.Time `gorm:"comment:最后登录时间"`

	// 关联
	Profile   *Profile   `gorm:"foreignKey:UserID"`          // Has One
	Orders    []Order    `gorm:"foreignKey:UserID"`          // Has Many
}

// TableName 自定义表名（默认是复数形式 users）
func (User) TableName() string {
	return "users"
}

// Profile 用户档案——Has One
type Profile struct {
	gorm.Model
	UserID    uint   `gorm:"uniqueIndex;not null;comment:用户ID"`
	Avatar    string `gorm:"type:varchar(500);comment:头像URL"`
	Bio       string `gorm:"type:text;comment:个人简介"`
	Phone     string `gorm:"type:varchar(20);comment:手机号"`
	Address   string `gorm:"type:varchar(500);comment:地址"`
}

// Product 产品模型
type Product struct {
	gorm.Model
	Name        string  `gorm:"type:varchar(200);not null;index;comment:产品名称"`
	Description string  `gorm:"type:text;comment:产品描述"`
	Price       float64 `gorm:"type:decimal(10,2);not null;comment:价格"`
	Stock       int     `gorm:"default:0;comment:库存"`
	CategoryID  uint    `gorm:"index;comment:分类ID"`

	Category Category `gorm:"foreignKey:CategoryID"` // Belongs To
	Tags     []Tag    `gorm:"many2many:product_tags"` // Many To Many
	Orders   []Order  `gorm:"many2many:order_products"` // Many To Many
}

// Category 分类模型
type Category struct {
	gorm.Model
	Name     string    `gorm:"type:varchar(100);uniqueIndex;not null;comment:分类名称"`
	ParentID *uint     `gorm:"index;comment:父分类ID"`
	Products []Product `gorm:"foreignKey:CategoryID"` // Has Many
}

// Tag 标签模型
type Tag struct {
	gorm.Model
	Name     string    `gorm:"type:varchar(50);uniqueIndex;not null;comment:标签名"`
	Products []Product `gorm:"many2many:product_tags"` // Many To Many
}

// Order 订单模型
type Order struct {
	gorm.Model
	OrderNo    string    `gorm:"type:varchar(50);uniqueIndex;not null;comment:订单号"`
	UserID     uint      `gorm:"index;not null;comment:用户ID"`
	TotalPrice float64   `gorm:"type:decimal(10,2);not null;comment:总价"`
	Status     string    `gorm:"type:varchar(20);default:'pending';comment:状态"`
	PaidAt     *time.Time `gorm:"comment:支付时间"`

	User     User      `gorm:"foreignKey:UserID"` // Belongs To
	Products []Product `gorm:"many2many:order_products"` // Many To Many
}

// Server 服务器模型（DevOps 场景）
type Server struct {
	ID        uint   `gorm:"primarykey"`
	Hostname  string `gorm:"type:varchar(200);uniqueIndex;not null"`
	IP        string `gorm:"type:varchar(50);not null"`
	OS        string `gorm:"type:varchar(100)"`
	CPU       int    `gorm:"comment:CPU核数"`
	Memory    int    `gorm:"comment:内存(GB)"`
	Disk      int    `gorm:"comment:磁盘(GB)"`
	Status    string `gorm:"type:varchar(20);default:'active'"`
	Env       string `gorm:"type:varchar(20);index"` // production/staging/development
	Labels    string `gorm:"type:text"`               // JSON 格式的标签
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

### 2.2 常用 GORM Tag

| Tag | 说明 | 示例 |
|-----|------|------|
| `primarykey` | 主键 | `gorm:"primarykey"` |
| `type` | 列类型 | `gorm:"type:varchar(100)"` |
| `not null` | 非空 | `gorm:"not null"` |
| `default` | 默认值 | `gorm:"default:0"` |
| `uniqueIndex` | 唯一索引 | `gorm:"uniqueIndex"` |
| `index` | 普通索引 | `gorm:"index"` |
| `comment` | 列注释 | `gorm:"comment:用户名"` |
| `column` | 指定列名 | `gorm:"column:user_name"` |
| `size` | 字段大小 | `gorm:"size:256"` |
| `autoIncrement` | 自增 | `gorm:"autoIncrement"` |
| `embedded` | 嵌入结构体 | `gorm:"embedded"` |
| `-` | 忽略字段 | `gorm:"-"` |

---

## 三、AutoMigrate 自动迁移

```go
package main

import (
	"fmt"
	"log"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// 使用上面定义的 models

func main() {
	db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{})
	if err != nil {
		log.Fatal(err)
	}

	// AutoMigrate 自动迁移——创建表、添加缺少的列、添加缺少的索引
	// 注意：不会删除列、不会删除索引、不会修改列类型
	err = db.AutoMigrate(
		&User{},
		&Profile{},
		&Product{},
		&Category{},
		&Tag{},
		&Order{},
		&Server{},
	)
	if err != nil {
		log.Fatalf("自动迁移失败: %v", err)
	}

	fmt.Println("数据库迁移完成")
}
```

---

## 四、CRUD 操作

### 4.1 Create 创建

```go
package main

import (
	"fmt"
	"log"
	"time"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

func main() {
	db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	db.AutoMigrate(&User{}, &Profile{})

	// ===== 创建单条记录 =====
	user := User{
		Username: "zhangsan",
		Email:    "zhangsan@example.com",
		Password: "hashed_password",
		Nickname: "张三",
		Age:      25,
	}
	result := db.Create(&user)
	if result.Error != nil {
		log.Fatalf("创建用户失败: %v", result.Error)
	}
	fmt.Printf("创建成功, ID: %d, 影响行数: %d\n", user.ID, result.RowsAffected)

	// ===== 批量创建 =====
	users := []User{
		{Username: "lisi", Email: "lisi@example.com", Password: "hash1", Age: 30},
		{Username: "wangwu", Email: "wangwu@example.com", Password: "hash2", Age: 28},
		{Username: "zhaoliu", Email: "zhaoliu@example.com", Password: "hash3", Age: 22},
	}
	result = db.Create(&users)
	fmt.Printf("批量创建 %d 条记录\n", result.RowsAffected)

	// ===== 分批创建（避免一次插入太多）=====
	bigBatch := make([]User, 1000)
	for i := range bigBatch {
		bigBatch[i] = User{
			Username: fmt.Sprintf("user_%d", i),
			Email:    fmt.Sprintf("user_%d@test.com", i),
			Password: "hash",
		}
	}
	db.CreateInBatches(&bigBatch, 100) // 每批 100 条

	// ===== 选择性创建（只插入指定字段）=====
	db.Select("Username", "Email", "Password").Create(&User{
		Username: "selective",
		Email:    "selective@test.com",
		Password: "hash",
		Age:      99, // 不会被插入
	})

	// ===== 忽略指定字段创建 =====
	db.Omit("Age", "Role").Create(&User{
		Username: "omitted",
		Email:    "omitted@test.com",
		Password: "hash",
		Age:      99, // 被忽略
	})

	// ===== 使用 Map 创建 =====
	db.Model(&User{}).Create(map[string]interface{}{
		"Username": "mapuser",
		"Email":    "map@test.com",
		"Password": "hash",
	})
}
```

### 4.2 Read 查询

```go
package main

import (
	"fmt"
	"log"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

func main() {
	db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	db.AutoMigrate(&User{})
	// 假设已有数据...

	// ===== First：查询第一条（按主键排序）=====
	var user User
	result := db.First(&user) // SELECT * FROM users ORDER BY id LIMIT 1
	if result.Error != nil {
		if result.Error == gorm.ErrRecordNotFound {
			fmt.Println("用户不存在")
		} else {
			log.Printf("查询错误: %v", result.Error)
		}
	}

	// 按主键查询
	db.First(&user, 1)        // SELECT * FROM users WHERE id = 1
	db.First(&user, "id = ?", 1) // 等效

	// ===== Find：查询多条 =====
	var users []User
	db.Find(&users) // SELECT * FROM users

	// ===== Where 条件查询 =====
	// 字符串条件
	db.Where("username = ?", "zhangsan").First(&user)
	db.Where("age > ?", 18).Find(&users)
	db.Where("age BETWEEN ? AND ?", 20, 30).Find(&users)
	db.Where("role IN ?", []string{"admin", "moderator"}).Find(&users)
	db.Where("email LIKE ?", "%@example.com").Find(&users)
	db.Where("username = ? AND age > ?", "zhangsan", 18).First(&user)

	// 结构体条件（零值字段会被忽略）
	db.Where(&User{Username: "zhangsan", Age: 25}).First(&user)
	// 注意：Age=0 时会被忽略！如果需要查询零值，用 Map

	// Map 条件（零值不会被忽略）
	db.Where(map[string]interface{}{
		"username": "zhangsan",
		"age":      0, // 不会被忽略
	}).Find(&users)

	// ===== Not 条件 =====
	db.Not("role = ?", "admin").Find(&users)
	db.Not(map[string]interface{}{"username": "zhangsan"}).Find(&users)

	// ===== Or 条件 =====
	db.Where("role = ?", "admin").Or("role = ?", "moderator").Find(&users)

	// ===== 选择特定字段 =====
	db.Select("username", "email").Find(&users)
	// SELECT username, email FROM users

	// ===== 排序 =====
	db.Order("created_at DESC").Find(&users)
	db.Order("age ASC, username DESC").Find(&users)

	// ===== 分页 =====
	page := 1
	pageSize := 10
	var total int64

	db.Model(&User{}).Count(&total) // 总数
	db.Offset((page - 1) * pageSize).Limit(pageSize).Find(&users)

	fmt.Printf("总数: %d, 当前页: %d\n", total, page)

	// ===== 分组与聚合 =====
	type RoleCount struct {
		Role  string
		Count int
	}
	var roleCounts []RoleCount
	db.Model(&User{}).Select("role, count(*) as count").Group("role").Scan(&roleCounts)

	// ===== Distinct =====
	var roles []string
	db.Model(&User{}).Distinct("role").Pluck("role", &roles)

	// ===== Pluck：查询单列到切片 =====
	var emails []string
	db.Model(&User{}).Pluck("email", &emails)

	// ===== Scan：扫描到自定义结构体 =====
	type UserSummary struct {
		Username string
		Email    string
	}
	var summaries []UserSummary
	db.Model(&User{}).Select("username", "email").Scan(&summaries)

	// ===== 子查询 =====
	subQuery := db.Model(&Order{}).Select("user_id").Where("total_price > ?", 100)
	db.Where("id IN (?)", subQuery).Find(&users)
	// SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total_price > 100)

	// ===== Scopes：可复用的查询条件 =====
	db.Scopes(ActiveUser, AgeRange(20, 30)).Find(&users)
}

// ActiveUser 活跃用户 Scope
func ActiveUser(db *gorm.DB) *gorm.DB {
	return db.Where("is_active = ?", true)
}

// AgeRange 年龄范围 Scope
func AgeRange(min, max int) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		return db.Where("age BETWEEN ? AND ?", min, max)
	}
}

// Paginate 分页 Scope
func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		if page <= 0 {
			page = 1
		}
		if pageSize <= 0 || pageSize > 100 {
			pageSize = 20
		}
		return db.Offset((page - 1) * pageSize).Limit(pageSize)
	}
}
```

### 4.3 Update 更新

```go
package main

import (
	"time"

	"gorm.io/gorm"
)

func updateExamples(db *gorm.DB) {
	// ===== Save：保存所有字段（包括零值）=====
	var user User
	db.First(&user, 1)
	user.Nickname = "新昵称"
	user.Age = 26
	db.Save(&user) // UPDATE users SET ... WHERE id = 1

	// ===== Update：更新单个字段 =====
	db.Model(&User{}).Where("id = ?", 1).Update("nickname", "另一个昵称")
	// UPDATE users SET nickname = '另一个昵称', updated_at = '...' WHERE id = 1

	// ===== Updates：更新多个字段 =====
	// 使用结构体（零值字段会被忽略）
	db.Model(&User{}).Where("id = ?", 1).Updates(User{
		Nickname: "结构体更新",
		Age:      30,
	})

	// 使用 Map（零值不会被忽略）
	db.Model(&User{}).Where("id = ?", 1).Updates(map[string]interface{}{
		"nickname": "Map更新",
		"age":      0, // 会被更新为 0
	})

	// ===== 选择/忽略字段更新 =====
	db.Model(&User{}).Where("id = ?", 1).
		Select("nickname", "age").
		Updates(User{Nickname: "选择更新", Age: 28})

	db.Model(&User{}).Where("id = ?", 1).
		Omit("password").
		Updates(map[string]interface{}{
			"nickname": "忽略更新",
			"password": "不会被更新",
		})

	// ===== 批量更新 =====
	db.Model(&User{}).Where("role = ?", "user").Update("is_active", true)
	// UPDATE users SET is_active = true WHERE role = 'user'

	// ===== 表达式更新 =====
	db.Model(&User{}).Where("id = ?", 1).
		Update("age", gorm.Expr("age + ?", 1))
	// UPDATE users SET age = age + 1 WHERE id = 1

	// ===== UpdateColumn：不触发 Hook、不更新 UpdatedAt =====
	db.Model(&User{}).Where("id = ?", 1).
		UpdateColumn("nickname", "无Hook更新")

	// ===== Upsert：不存在则创建，存在则更新（MySQL）=====
	// db.Clauses(clause.OnConflict{
	// 	Columns:   []clause.Column{{Name: "username"}},
	// 	DoUpdates: clause.AssignmentColumns([]string{"email", "updated_at"}),
	// }).Create(&user)
}
```

### 4.4 Delete 删除

```go
package main

import "gorm.io/gorm"

func deleteExamples(db *gorm.DB) {
	// ===== 软删除（使用 gorm.Model 的模型默认启用）=====
	// 不会真正删除记录，只是设置 deleted_at 字段
	db.Delete(&User{}, 1)
	// UPDATE users SET deleted_at = '...' WHERE id = 1

	// 条件删除
	db.Where("age < ?", 18).Delete(&User{})

	// 批量删除
	db.Delete(&User{}, []int{1, 2, 3})
	// UPDATE users SET deleted_at = '...' WHERE id IN (1, 2, 3)

	// ===== 查询已软删除的记录 =====
	var user User
	db.Unscoped().Where("id = ?", 1).First(&user)

	var allUsers []User
	db.Unscoped().Find(&allUsers) // 包含已删除的

	// ===== 永久删除（物理删除）=====
	db.Unscoped().Delete(&User{}, 1)
	// DELETE FROM users WHERE id = 1
}
```

---

## 五、关联关系

### 5.1 Has One / Belongs To

```go
package main

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

type Author struct {
	gorm.Model
	Name    string
	Profile AuthorProfile
}

type AuthorProfile struct {
	gorm.Model
	AuthorID uint   `gorm:"uniqueIndex"`
	Bio      string
	Website  string
}

func main() {
	db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	db.AutoMigrate(&Author{}, &AuthorProfile{})

	// 创建带关联的记录
	author := Author{
		Name: "Go 大师",
		Profile: AuthorProfile{
			Bio:     "Go 语言专家",
			Website: "https://example.com",
		},
	}
	db.Create(&author) // 自动创建 Profile

	// 查询关联
	var a Author
	db.Preload("Profile").First(&a, author.ID)
	fmt.Printf("作者: %s, 简介: %s\n", a.Name, a.Profile.Bio)

	// 添加关联
	db.Model(&a).Association("Profile").Replace(&AuthorProfile{
		Bio:     "更新后的简介",
		Website: "https://new-site.com",
	})
}
```

### 5.2 Has Many

```go
package main

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

type Department struct {
	gorm.Model
	Name      string
	Employees []Employee // Has Many
}

type Employee struct {
	gorm.Model
	Name         string
	DepartmentID uint // 外键
}

func main() {
	db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	db.AutoMigrate(&Department{}, &Employee{})

	// 创建部门和员工
	dept := Department{
		Name: "工程部",
		Employees: []Employee{
			{Name: "张三"},
			{Name: "李四"},
			{Name: "王五"},
		},
	}
	db.Create(&dept)

	// 查询部门及其员工
	var d Department
	db.Preload("Employees").First(&d, dept.ID)
	fmt.Printf("部门: %s, 员工数: %d\n", d.Name, len(d.Employees))
	for _, emp := range d.Employees {
		fmt.Printf("  - %s\n", emp.Name)
	}

	// 追加员工
	db.Model(&d).Association("Employees").Append(&Employee{Name: "赵六"})

	// 查询关联数量
	count := db.Model(&d).Association("Employees").Count()
	fmt.Printf("更新后员工数: %d\n", count)
}
```

### 5.3 Many To Many

```go
package main

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

type Student struct {
	gorm.Model
	Name    string
	Courses []Course `gorm:"many2many:student_courses;"`
}

type Course struct {
	gorm.Model
	Name     string
	Students []Student `gorm:"many2many:student_courses;"`
}

func main() {
	db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	db.AutoMigrate(&Student{}, &Course{})

	// 创建课程
	golang := Course{Name: "Go 语言编程"}
	k8s := Course{Name: "Kubernetes 实战"}
	docker := Course{Name: "Docker 容器化"}
	db.Create(&golang)
	db.Create(&k8s)
	db.Create(&docker)

	// 创建学生并关联课程
	student := Student{
		Name:    "张三",
		Courses: []Course{golang, k8s},
	}
	db.Create(&student)

	// 查询学生的课程
	var s Student
	db.Preload("Courses").First(&s, student.ID)
	fmt.Printf("学生: %s\n", s.Name)
	for _, c := range s.Courses {
		fmt.Printf("  课程: %s\n", c.Name)
	}

	// 追加课程
	db.Model(&s).Association("Courses").Append(&docker)

	// 移除课程
	db.Model(&s).Association("Courses").Delete(&k8s)

	// 替换所有课程
	db.Model(&s).Association("Courses").Replace(&golang, &docker)

	// 清除所有关联
	// db.Model(&s).Association("Courses").Clear()

	// 查询课程的学生
	var c Course
	db.Preload("Students").First(&c, golang.ID)
	fmt.Printf("课程 %s 的学生数: %d\n", c.Name, len(c.Students))
}
```

---

## 六、预加载（Preload）

```go
package main

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// 使用之前定义的 User, Profile, Order, Product 模型

func preloadExamples(db *gorm.DB) {
	// ===== 基本预加载 =====
	var users []User
	db.Preload("Profile").Find(&users)
	// SELECT * FROM users;
	// SELECT * FROM profiles WHERE user_id IN (1,2,3,...);

	// ===== 多个预加载 =====
	db.Preload("Profile").Preload("Orders").Find(&users)

	// ===== 嵌套预加载 =====
	db.Preload("Orders.Products").Find(&users)
	// 加载用户 -> 加载用户的订单 -> 加载订单的产品

	// ===== 条件预加载 =====
	db.Preload("Orders", "status = ?", "paid").Find(&users)
	// 只预加载状态为 "paid" 的订单

	// ===== 自定义预加载 SQL =====
	db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
		return db.Where("total_price > ?", 100).Order("created_at DESC").Limit(5)
	}).Find(&users)

	// ===== Joins 预加载（使用 JOIN 而不是单独查询）=====
	var user User
	db.Joins("Profile").First(&user, 1)
	// SELECT users.*, profiles.* FROM users LEFT JOIN profiles ON ...

	// ===== Preload All：预加载所有关联 =====
	db.Preload(clause.Associations).Find(&users)
}
```

---

## 七、事务处理

### 7.1 自动事务

```go
package main

import (
	"fmt"

	"gorm.io/gorm"
)

// TransferMoney 转账——使用自动事务
func TransferMoney(db *gorm.DB, fromID, toID uint, amount float64) error {
	// Transaction 自动管理事务：成功则提交，panic 或返回错误则回滚
	return db.Transaction(func(tx *gorm.DB) error {
		// 扣款
		result := tx.Model(&User{}).Where("id = ? AND balance >= ?", fromID, amount).
			Update("balance", gorm.Expr("balance - ?", amount))
		if result.Error != nil {
			return result.Error // 返回错误会自动回滚
		}
		if result.RowsAffected == 0 {
			return fmt.Errorf("余额不足或用户不存在")
		}

		// 加款
		result = tx.Model(&User{}).Where("id = ?", toID).
			Update("balance", gorm.Expr("balance + ?", amount))
		if result.Error != nil {
			return result.Error
		}
		if result.RowsAffected == 0 {
			return fmt.Errorf("收款用户不存在")
		}

		// 记录转账日志
		transferLog := map[string]interface{}{
			"from_user_id": fromID,
			"to_user_id":   toID,
			"amount":       amount,
		}
		if err := tx.Table("transfer_logs").Create(transferLog).Error; err != nil {
			return err
		}

		return nil // 返回 nil 自动提交
	})
}
```

### 7.2 手动事务

```go
// CreateOrderWithTransaction 创建订单——手动事务控制
func CreateOrderWithTransaction(db *gorm.DB, userID uint, productIDs []uint) (*Order, error) {
	// 开始事务
	tx := db.Begin()
	if tx.Error != nil {
		return nil, tx.Error
	}

	// 使用 defer 确保在 panic 时回滚
	defer func() {
		if r := recover(); r != nil {
			tx.Rollback()
		}
	}()

	// 查询产品并计算总价
	var products []Product
	if err := tx.Where("id IN ?", productIDs).Find(&products).Error; err != nil {
		tx.Rollback()
		return nil, fmt.Errorf("查询产品失败: %w", err)
	}

	var totalPrice float64
	for _, p := range products {
		totalPrice += p.Price
	}

	// 创建订单
	order := Order{
		OrderNo:    fmt.Sprintf("ORD-%d", time.Now().UnixNano()),
		UserID:     userID,
		TotalPrice: totalPrice,
		Status:     "pending",
		Products:   products,
	}

	if err := tx.Create(&order).Error; err != nil {
		tx.Rollback()
		return nil, fmt.Errorf("创建订单失败: %w", err)
	}

	// 扣减库存
	for _, p := range products {
		result := tx.Model(&Product{}).
			Where("id = ? AND stock > 0", p.ID).
			Update("stock", gorm.Expr("stock - 1"))
		if result.RowsAffected == 0 {
			tx.Rollback()
			return nil, fmt.Errorf("产品 %s 库存不足", p.Name)
		}
	}

	// 提交事务
	if err := tx.Commit().Error; err != nil {
		return nil, fmt.Errorf("提交事务失败: %w", err)
	}

	return &order, nil
}
```

### 7.3 嵌套事务（SavePoint）

```go
func NestedTransaction(db *gorm.DB) error {
	return db.Transaction(func(tx *gorm.DB) error {
		tx.Create(&User{Username: "user1", Email: "u1@test.com", Password: "hash"})

		// 嵌套事务（SavePoint）
		err := tx.Transaction(func(tx2 *gorm.DB) error {
			tx2.Create(&User{Username: "user2", Email: "u2@test.com", Password: "hash"})
			// 如果这里返回错误，只回滚 user2，不影响 user1
			return nil
		})
		if err != nil {
			// 可以选择继续或回滚外部事务
			return err
		}

		tx.Create(&User{Username: "user3", Email: "u3@test.com", Password: "hash"})

		return nil
	})
}
```

---

## 八、Hook 回调

GORM 支持在 CRUD 操作前后自动执行回调函数。

```go
package models

import (
	"crypto/sha256"
	"fmt"
	"log"
	"strings"
	"time"

	"gorm.io/gorm"
)

// User 用户模型（带 Hook）
type UserWithHooks struct {
	gorm.Model
	Username  string
	Email     string
	Password  string
	Status    string
	LoginCount int
}

// BeforeCreate 创建前的 Hook
func (u *UserWithHooks) BeforeCreate(tx *gorm.DB) error {
	// 标准化邮箱
	u.Email = strings.ToLower(strings.TrimSpace(u.Email))

	// 加密密码（示例用简单哈希，实际应用应使用 bcrypt）
	if u.Password != "" {
		hash := sha256.Sum256([]byte(u.Password))
		u.Password = fmt.Sprintf("%x", hash)
	}

	// 设置默认状态
	if u.Status == "" {
		u.Status = "active"
	}

	log.Printf("即将创建用户: %s", u.Username)
	return nil // 返回错误会取消创建
}

// AfterCreate 创建后的 Hook
func (u *UserWithHooks) AfterCreate(tx *gorm.DB) error {
	log.Printf("用户创建成功: ID=%d, Username=%s", u.ID, u.Username)
	// 可以在这里发送欢迎邮件、创建关联记录等
	return nil
}

// BeforeUpdate 更新前的 Hook
func (u *UserWithHooks) BeforeUpdate(tx *gorm.DB) error {
	// 如果密码被修改，重新加密
	if tx.Statement.Changed("Password") {
		hash := sha256.Sum256([]byte(u.Password))
		u.Password = fmt.Sprintf("%x", hash)
	}

	log.Printf("即将更新用户: ID=%d", u.ID)
	return nil
}

// AfterUpdate 更新后的 Hook
func (u *UserWithHooks) AfterUpdate(tx *gorm.DB) error {
	log.Printf("用户更新成功: ID=%d", u.ID)
	return nil
}

// BeforeDelete 删除前的 Hook
func (u *UserWithHooks) BeforeDelete(tx *gorm.DB) error {
	log.Printf("即将删除用户: ID=%d, Username=%s", u.ID, u.Username)
	// 可以做级联删除检查
	return nil
}

// AfterFind 查询后的 Hook
func (u *UserWithHooks) AfterFind(tx *gorm.DB) error {
	// 可以在查询后做数据转换
	// 注意：不要在这里做耗时操作，会影响每次查询
	return nil
}
```

可用的 Hook：

| Hook | 触发时机 |
|------|---------|
| `BeforeCreate` | 创建前 |
| `AfterCreate` | 创建后 |
| `BeforeUpdate` | 更新前 |
| `AfterUpdate` | 更新后 |
| `BeforeSave` | 创建/更新前 |
| `AfterSave` | 创建/更新后 |
| `BeforeDelete` | 删除前 |
| `AfterDelete` | 删除后 |
| `AfterFind` | 查询后 |

---

## 九、原生 SQL

```go
package main

import (
	"fmt"

	"gorm.io/gorm"
)

func rawSQLExamples(db *gorm.DB) {
	// ===== Raw：原生查询 =====
	var users []User
	db.Raw("SELECT * FROM users WHERE age > ? AND role = ?", 18, "admin").Scan(&users)

	// 查询单个值
	var count int64
	db.Raw("SELECT COUNT(*) FROM users WHERE is_active = ?", true).Scan(&count)

	// 查询到自定义结构体
	type UserStats struct {
		Role       string
		Count      int
		AvgAge     float64
		TotalUsers int
	}
	var stats []UserStats
	db.Raw(`
		SELECT role,
		       COUNT(*) as count,
		       AVG(age) as avg_age,
		       SUM(CASE WHEN is_active THEN 1 ELSE 0 END) as total_users
		FROM users
		GROUP BY role
	`).Scan(&stats)

	// ===== Exec：原生执行（INSERT/UPDATE/DELETE）=====
	db.Exec("UPDATE users SET is_active = ? WHERE last_login < ?", false, "2024-01-01")
	db.Exec("DELETE FROM sessions WHERE expired_at < NOW()")

	// 带命名参数
	db.Raw("SELECT * FROM users WHERE username = @name", map[string]interface{}{
		"name": "zhangsan",
	}).Scan(&users)

	// ===== DryRun：打印 SQL 不执行 =====
	stmt := db.Session(&gorm.Session{DryRun: true}).
		Where("age > ?", 18).
		Find(&User{}).
		Statement
	fmt.Printf("SQL: %s\n", stmt.SQL.String())
	fmt.Printf("参数: %v\n", stmt.Vars)
}
```

---

## 十、日志与慢查询

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

func main() {
	// ===== 配置日志 =====
	newLogger := logger.New(
		log.New(os.Stdout, "\r\n", log.LstdFlags), // 输出到 stdout
		logger.Config{
			SlowThreshold:             200 * time.Millisecond, // 慢查询阈值
			LogLevel:                  logger.Info,            // 日志级别
			IgnoreRecordNotFoundError: true,                   // 忽略 RecordNotFound 错误
			Colorful:                  true,                   // 彩色输出
			ParameterizedQueries:      false,                  // 显示参数值（生产环境设为 true）
		},
	)

	db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{
		Logger: newLogger,
	})
	if err != nil {
		log.Fatal(err)
	}

	// ===== 运行时更改日志级别 =====
	db.Logger = newLogger.LogMode(logger.Silent) // 静默模式
	db.Logger = newLogger.LogMode(logger.Error)  // 只记录错误
	db.Logger = newLogger.LogMode(logger.Warn)   // 记录慢查询和错误
	db.Logger = newLogger.LogMode(logger.Info)   // 记录所有 SQL

	// ===== 会话级日志控制 =====
	// 单次查询使用 Debug 模式
	db.Debug().Where("age > ?", 18).Find(&User{})

	// 单次查询静默模式
	db.Session(&gorm.Session{Logger: newLogger.LogMode(logger.Silent)}).
		Find(&User{})

	fmt.Println("日志配置完成")
}

// CustomLogger 自定义日志实现（例如输出到 ELK）
type CustomLogger struct {
	logger.Interface
}

func (l *CustomLogger) Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error) {
	elapsed := time.Since(begin)
	sql, rows := fc()

	if err != nil {
		log.Printf("[GORM ERROR] %v | %s | rows=%d | %s", err, elapsed, rows, sql)
	} else if elapsed > 200*time.Millisecond {
		log.Printf("[GORM SLOW] %s | rows=%d | %s", elapsed, rows, sql)
	} else {
		log.Printf("[GORM] %s | rows=%d | %s", elapsed, rows, sql)
	}
}
```

---

## 十一、常见坑点

### 坑点 1：结构体零值更新问题

```go
// 错误：使用结构体更新，零值字段会被忽略
db.Model(&user).Updates(User{Age: 0, IsActive: false})
// Age 和 IsActive 不会被更新！因为 0 和 false 是零值

// 正确：使用 Map 更新
db.Model(&user).Updates(map[string]interface{}{
	"age":       0,
	"is_active": false,
})

// 或使用 Select 指定字段
db.Model(&user).Select("Age", "IsActive").Updates(User{Age: 0, IsActive: false})
```

### 坑点 2：N+1 查询问题

```go
// 错误：循环中查询关联
var users []User
db.Find(&users)
for _, u := range users {
	var profile Profile
	db.Where("user_id = ?", u.ID).First(&profile) // N 次额外查询！
}

// 正确：使用 Preload
db.Preload("Profile").Find(&users)
// 只有 2 次查询：1 次查 users + 1 次查 profiles
```

### 坑点 3：事务中忘记使用 tx

```go
// 错误：在事务回调中使用 db 而不是 tx
db.Transaction(func(tx *gorm.DB) error {
	db.Create(&user1)  // 错！不在事务中
	tx.Create(&user2)  // 对！在事务中
	return nil
})
```

### 坑点 4：软删除的坑

```go
// 坑：软删除后唯一索引冲突
// 用户 zhangsan 被软删除后，无法创建新的 zhangsan
// 因为唯一索引仍然包含已删除的记录

// 解决方案1：唯一索引包含 deleted_at
// CREATE UNIQUE INDEX idx_username ON users(username, deleted_at)

// 解决方案2：使用复合唯一索引
type User struct {
	gorm.Model
	Username string `gorm:"uniqueIndex:idx_username_deleted"`
	DeletedAt gorm.DeletedAt `gorm:"uniqueIndex:idx_username_deleted"`
}
```

---

## 十二、速查表

### GORM CRUD 速查

```go
// Create
db.Create(&user)                    // 创建
db.CreateInBatches(&users, 100)     // 批量创建

// Read
db.First(&user, id)                 // 按主键查询
db.Find(&users)                     // 查询所有
db.Where("age > ?", 18).Find(&users) // 条件查询
db.Preload("Profile").Find(&users)  // 预加载

// Update
db.Save(&user)                      // 保存所有字段
db.Model(&user).Update("name", "new") // 更新单字段
db.Model(&user).Updates(User{...})  // 更新多字段
db.Model(&user).Updates(map[string]interface{}{...}) // Map 更新

// Delete
db.Delete(&user, id)                // 软删除
db.Unscoped().Delete(&user, id)     // 物理删除

// Transaction
db.Transaction(func(tx *gorm.DB) error { ... })

// Raw SQL
db.Raw("SELECT ...").Scan(&result)
db.Exec("UPDATE ...")
```
