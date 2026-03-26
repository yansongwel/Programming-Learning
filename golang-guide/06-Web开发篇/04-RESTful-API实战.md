# 04-RESTful-API实战

## 导读

构建 RESTful API 是后端开发的核心技能。对于 SRE/DevOps 工程师，你可能需要构建 CMDB 资产管理系统、自动化运维平台、监控告警接口等。本章将综合前面学到的 Gin 框架、GORM 数据库、Redis 缓存，从零构建一个完整的、生产级别的用户管理 CRUD API。

本章将系统讲解：

- RESTful API 设计规范
- 项目分层结构（controller/service/repository/model/middleware）
- JWT 认证（生成与验证 token、中间件）
- Swagger 文档（swaggo/swag 自动生成）
- 请求参数校验
- 统一错误处理
- 分页查询
- 优雅关闭（signal + http.Server.Shutdown）
- 完整用户管理 CRUD API 示例

> 基于 Go 1.24 + Gin + GORM + JWT，所有代码完整可运行。

---

## 一、RESTful API 设计规范

### 1.1 URL 设计

```
# 资源命名使用名词复数
GET    /api/v1/users          # 获取用户列表
GET    /api/v1/users/:id      # 获取单个用户
POST   /api/v1/users          # 创建用户
PUT    /api/v1/users/:id      # 全量更新用户
PATCH  /api/v1/users/:id      # 部分更新用户
DELETE /api/v1/users/:id      # 删除用户

# 子资源
GET    /api/v1/users/:id/orders      # 获取用户的订单列表
POST   /api/v1/users/:id/orders      # 为用户创建订单

# 查询参数用于过滤、排序、分页
GET    /api/v1/users?page=1&page_size=20&sort=created_at&order=desc
GET    /api/v1/users?role=admin&status=active

# 非 CRUD 操作用动词
POST   /api/v1/users/:id/activate    # 激活用户
POST   /api/v1/users/:id/deactivate  # 停用用户
POST   /api/v1/auth/login             # 登录
POST   /api/v1/auth/refresh           # 刷新 token
```

### 1.2 统一响应格式

```json
// 成功响应
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "name": "张三"
  }
}

// 列表响应（带分页）
{
  "code": 200,
  "message": "success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "page_size": 20
  }
}

// 错误响应
{
  "code": 400,
  "message": "参数校验失败",
  "errors": {
    "email": "邮箱格式不正确",
    "password": "密码长度至少6位"
  }
}
```

### 1.3 HTTP 状态码使用

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| 200 | OK | GET 成功、PUT/PATCH 更新成功 |
| 201 | Created | POST 创建成功 |
| 204 | No Content | DELETE 删除成功 |
| 400 | Bad Request | 请求参数错误 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突（如重复创建） |
| 422 | Unprocessable Entity | 参数校验失败 |
| 429 | Too Many Requests | 请求限流 |
| 500 | Internal Server Error | 服务器内部错误 |

---

## 二、项目结构

```
user-api/
├── main.go                    # 入口文件
├── go.mod
├── go.sum
├── config/
│   └── config.go              # 配置管理
├── model/
│   ├── user.go                # 用户模型
│   └── response.go            # 统一响应结构
├── repository/
│   └── user_repo.go           # 数据访问层
├── service/
│   └── user_service.go        # 业务逻辑层
├── controller/
│   ├── auth_controller.go     # 认证控制器
│   └── user_controller.go     # 用户控制器
├── middleware/
│   ├── auth.go                # JWT 认证中间件
│   ├── cors.go                # CORS 中间件
│   ├── logger.go              # 日志中间件
│   └── recovery.go            # 错误恢复中间件
├── pkg/
│   ├── jwt/
│   │   └── jwt.go             # JWT 工具
│   └── errors/
│       └── errors.go          # 自定义错误
└── router/
    └── router.go              # 路由定义
```

---

## 三、代码实现

### 3.1 配置管理

```go
// 文件: config/config.go
package config

import (
	"time"
)

// Config 应用配置
type Config struct {
	Server   ServerConfig
	Database DatabaseConfig
	JWT      JWTConfig
	Redis    RedisConfig
}

// ServerConfig 服务器配置
type ServerConfig struct {
	Port         string
	Mode         string // debug, release, test
	ReadTimeout  time.Duration
	WriteTimeout time.Duration
}

// DatabaseConfig 数据库配置
type DatabaseConfig struct {
	DSN          string
	MaxIdleConns int
	MaxOpenConns int
	MaxLifetime  time.Duration
}

// JWTConfig JWT 配置
type JWTConfig struct {
	Secret     string
	Issuer     string
	ExpireTime time.Duration
}

// RedisConfig Redis 配置
type RedisConfig struct {
	Addr     string
	Password string
	DB       int
}

// DefaultConfig 默认配置
func DefaultConfig() *Config {
	return &Config{
		Server: ServerConfig{
			Port:         ":8080",
			Mode:         "debug",
			ReadTimeout:  10 * time.Second,
			WriteTimeout: 10 * time.Second,
		},
		Database: DatabaseConfig{
			DSN:          "root:password@tcp(127.0.0.1:3306)/userdb?charset=utf8mb4&parseTime=True&loc=Local",
			MaxIdleConns: 10,
			MaxOpenConns: 100,
			MaxLifetime:  time.Hour,
		},
		JWT: JWTConfig{
			Secret:     "your-secret-key-change-in-production",
			Issuer:     "user-api",
			ExpireTime: 24 * time.Hour,
		},
		Redis: RedisConfig{
			Addr:     "localhost:6379",
			Password: "",
			DB:       0,
		},
	}
}
```

### 3.2 统一响应与错误处理

```go
// 文件: model/response.go
package model

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// Response 统一响应结构
type Response struct {
	Code    int         `json:"code"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
	Errors  interface{} `json:"errors,omitempty"`
}

// PageData 分页数据
type PageData struct {
	Items    interface{} `json:"items"`
	Total    int64       `json:"total"`
	Page     int         `json:"page"`
	PageSize int         `json:"page_size"`
}

// Success 成功响应
func Success(c *gin.Context, data interface{}) {
	c.JSON(http.StatusOK, Response{
		Code:    200,
		Message: "success",
		Data:    data,
	})
}

// Created 创建成功响应
func Created(c *gin.Context, data interface{}) {
	c.JSON(http.StatusCreated, Response{
		Code:    201,
		Message: "创建成功",
		Data:    data,
	})
}

// NoContent 删除成功响应
func NoContent(c *gin.Context) {
	c.JSON(http.StatusOK, Response{
		Code:    200,
		Message: "删除成功",
	})
}

// PageSuccess 分页成功响应
func PageSuccess(c *gin.Context, items interface{}, total int64, page, pageSize int) {
	c.JSON(http.StatusOK, Response{
		Code:    200,
		Message: "success",
		Data: PageData{
			Items:    items,
			Total:    total,
			Page:     page,
			PageSize: pageSize,
		},
	})
}

// BadRequest 400 错误
func BadRequest(c *gin.Context, message string) {
	c.JSON(http.StatusBadRequest, Response{
		Code:    400,
		Message: message,
	})
}

// ValidationError 422 参数校验错误
func ValidationError(c *gin.Context, errors interface{}) {
	c.JSON(http.StatusUnprocessableEntity, Response{
		Code:    422,
		Message: "参数校验失败",
		Errors:  errors,
	})
}

// Unauthorized 401 未认证
func Unauthorized(c *gin.Context, message string) {
	if message == "" {
		message = "未授权，请登录"
	}
	c.JSON(http.StatusUnauthorized, Response{
		Code:    401,
		Message: message,
	})
}

// Forbidden 403 无权限
func Forbidden(c *gin.Context, message string) {
	if message == "" {
		message = "没有访问权限"
	}
	c.JSON(http.StatusForbidden, Response{
		Code:    403,
		Message: message,
	})
}

// NotFound 404 资源不存在
func NotFound(c *gin.Context, message string) {
	if message == "" {
		message = "资源不存在"
	}
	c.JSON(http.StatusNotFound, Response{
		Code:    404,
		Message: message,
	})
}

// Conflict 409 冲突
func Conflict(c *gin.Context, message string) {
	c.JSON(http.StatusConflict, Response{
		Code:    409,
		Message: message,
	})
}

// InternalError 500 内部错误
func InternalError(c *gin.Context, message string) {
	if message == "" {
		message = "服务器内部错误"
	}
	c.JSON(http.StatusInternalServerError, Response{
		Code:    500,
		Message: message,
	})
}
```

### 3.3 用户模型

```go
// 文件: model/user.go
package model

import (
	"time"

	"gorm.io/gorm"
)

// User 用户模型
type User struct {
	ID        uint           `json:"id" gorm:"primarykey"`
	Username  string         `json:"username" gorm:"type:varchar(50);uniqueIndex;not null"`
	Email     string         `json:"email" gorm:"type:varchar(200);uniqueIndex;not null"`
	Password  string         `json:"-" gorm:"type:varchar(255);not null"` // json:"-" 不返回密码
	Nickname  string         `json:"nickname" gorm:"type:varchar(100)"`
	Avatar    string         `json:"avatar" gorm:"type:varchar(500)"`
	Role      string         `json:"role" gorm:"type:varchar(20);default:'user'"`
	Status    int            `json:"status" gorm:"default:1;comment:1-正常 0-禁用"`
	LastLogin *time.Time     `json:"last_login"`
	CreatedAt time.Time      `json:"created_at"`
	UpdatedAt time.Time      `json:"updated_at"`
	DeletedAt gorm.DeletedAt `json:"-" gorm:"index"`
}

// TableName 指定表名
func (User) TableName() string {
	return "users"
}

// ===== 请求 DTO =====

// CreateUserRequest 创建用户请求
type CreateUserRequest struct {
	Username string `json:"username" binding:"required,min=3,max=50"`
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=6,max=64"`
	Nickname string `json:"nickname" binding:"omitempty,max=100"`
	Role     string `json:"role" binding:"omitempty,oneof=user admin moderator"`
}

// UpdateUserRequest 更新用户请求
type UpdateUserRequest struct {
	Nickname *string `json:"nickname" binding:"omitempty,max=100"`
	Avatar   *string `json:"avatar" binding:"omitempty,url"`
	Role     *string `json:"role" binding:"omitempty,oneof=user admin moderator"`
	Status   *int    `json:"status" binding:"omitempty,oneof=0 1"`
}

// LoginRequest 登录请求
type LoginRequest struct {
	Username string `json:"username" binding:"required"`
	Password string `json:"password" binding:"required"`
}

// ChangePasswordRequest 修改密码请求
type ChangePasswordRequest struct {
	OldPassword string `json:"old_password" binding:"required"`
	NewPassword string `json:"new_password" binding:"required,min=6,max=64"`
}

// ListUserRequest 列表查询请求
type ListUserRequest struct {
	Page     int    `form:"page" binding:"omitempty,gte=1"`
	PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
	Keyword  string `form:"keyword"`
	Role     string `form:"role" binding:"omitempty,oneof=user admin moderator"`
	Status   *int   `form:"status" binding:"omitempty,oneof=0 1"`
	Sort     string `form:"sort" binding:"omitempty,oneof=created_at updated_at username"`
	Order    string `form:"order" binding:"omitempty,oneof=asc desc"`
}

// ===== 响应 DTO =====

// LoginResponse 登录响应
type LoginResponse struct {
	Token     string `json:"token"`
	ExpiresAt int64  `json:"expires_at"`
	User      *User  `json:"user"`
}

// UserResponse 用户响应（可以在这里定义不同于模型的响应结构）
type UserResponse struct {
	ID        uint       `json:"id"`
	Username  string     `json:"username"`
	Email     string     `json:"email"`
	Nickname  string     `json:"nickname"`
	Avatar    string     `json:"avatar"`
	Role      string     `json:"role"`
	Status    int        `json:"status"`
	LastLogin *time.Time `json:"last_login"`
	CreatedAt time.Time  `json:"created_at"`
}
```

### 3.4 JWT 工具

```go
// 文件: pkg/jwt/jwt.go
package jwt

import (
	"errors"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// Claims 自定义 JWT Claims
type Claims struct {
	UserID   uint   `json:"user_id"`
	Username string `json:"username"`
	Role     string `json:"role"`
	jwt.RegisteredClaims
}

// JWTManager JWT 管理器
type JWTManager struct {
	secret     []byte
	issuer     string
	expireTime time.Duration
}

// NewJWTManager 创建 JWT 管理器
func NewJWTManager(secret, issuer string, expireTime time.Duration) *JWTManager {
	return &JWTManager{
		secret:     []byte(secret),
		issuer:     issuer,
		expireTime: expireTime,
	}
}

// GenerateToken 生成 JWT Token
func (j *JWTManager) GenerateToken(userID uint, username, role string) (string, int64, error) {
	expiresAt := time.Now().Add(j.expireTime)

	claims := Claims{
		UserID:   userID,
		Username: username,
		Role:     role,
		RegisteredClaims: jwt.RegisteredClaims{
			Issuer:    j.issuer,
			ExpiresAt: jwt.NewNumericDate(expiresAt),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			NotBefore: jwt.NewNumericDate(time.Now()),
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString(j.secret)
	if err != nil {
		return "", 0, err
	}

	return tokenString, expiresAt.Unix(), nil
}

// ParseToken 解析 JWT Token
func (j *JWTManager) ParseToken(tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
		// 验证签名方法
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, errors.New("无效的签名方法")
		}
		return j.secret, nil
	})

	if err != nil {
		return nil, err
	}

	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid {
		return nil, errors.New("无效的 token")
	}

	return claims, nil
}
```

### 3.5 数据访问层（Repository）

```go
// 文件: repository/user_repo.go
package repository

import (
	"context"
	"fmt"

	"user-api/model"

	"gorm.io/gorm"
)

// UserRepository 用户数据仓库
type UserRepository struct {
	db *gorm.DB
}

// NewUserRepository 创建用户仓库
func NewUserRepository(db *gorm.DB) *UserRepository {
	return &UserRepository{db: db}
}

// Create 创建用户
func (r *UserRepository) Create(ctx context.Context, user *model.User) error {
	return r.db.WithContext(ctx).Create(user).Error
}

// GetByID 根据 ID 查询
func (r *UserRepository) GetByID(ctx context.Context, id uint) (*model.User, error) {
	var user model.User
	err := r.db.WithContext(ctx).First(&user, id).Error
	if err == gorm.ErrRecordNotFound {
		return nil, nil
	}
	return &user, err
}

// GetByUsername 根据用户名查询
func (r *UserRepository) GetByUsername(ctx context.Context, username string) (*model.User, error) {
	var user model.User
	err := r.db.WithContext(ctx).Where("username = ?", username).First(&user).Error
	if err == gorm.ErrRecordNotFound {
		return nil, nil
	}
	return &user, err
}

// GetByEmail 根据邮箱查询
func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*model.User, error) {
	var user model.User
	err := r.db.WithContext(ctx).Where("email = ?", email).First(&user).Error
	if err == gorm.ErrRecordNotFound {
		return nil, nil
	}
	return &user, err
}

// List 分页查询用户列表
func (r *UserRepository) List(ctx context.Context, req *model.ListUserRequest) ([]model.User, int64, error) {
	var users []model.User
	var total int64

	query := r.db.WithContext(ctx).Model(&model.User{})

	// 关键词搜索
	if req.Keyword != "" {
		keyword := "%" + req.Keyword + "%"
		query = query.Where("username LIKE ? OR email LIKE ? OR nickname LIKE ?",
			keyword, keyword, keyword)
	}

	// 角色过滤
	if req.Role != "" {
		query = query.Where("role = ?", req.Role)
	}

	// 状态过滤
	if req.Status != nil {
		query = query.Where("status = ?", *req.Status)
	}

	// 计算总数
	if err := query.Count(&total).Error; err != nil {
		return nil, 0, err
	}

	// 排序
	sort := "created_at"
	if req.Sort != "" {
		sort = req.Sort
	}
	order := "desc"
	if req.Order != "" {
		order = req.Order
	}
	query = query.Order(fmt.Sprintf("%s %s", sort, order))

	// 分页
	page := req.Page
	if page <= 0 {
		page = 1
	}
	pageSize := req.PageSize
	if pageSize <= 0 {
		pageSize = 20
	}
	offset := (page - 1) * pageSize

	err := query.Offset(offset).Limit(pageSize).Find(&users).Error
	return users, total, err
}

// Update 更新用户
func (r *UserRepository) Update(ctx context.Context, user *model.User) error {
	return r.db.WithContext(ctx).Save(user).Error
}

// UpdateFields 更新指定字段
func (r *UserRepository) UpdateFields(ctx context.Context, id uint, fields map[string]interface{}) error {
	return r.db.WithContext(ctx).Model(&model.User{}).Where("id = ?", id).Updates(fields).Error
}

// Delete 删除用户（软删除）
func (r *UserRepository) Delete(ctx context.Context, id uint) error {
	return r.db.WithContext(ctx).Delete(&model.User{}, id).Error
}
```

### 3.6 业务逻辑层（Service）

```go
// 文件: service/user_service.go
package service

import (
	"context"
	"fmt"
	"time"

	"user-api/model"
	jwtpkg "user-api/pkg/jwt"
	"user-api/repository"

	"golang.org/x/crypto/bcrypt"
)

// UserService 用户服务
type UserService struct {
	repo       *repository.UserRepository
	jwtManager *jwtpkg.JWTManager
}

// NewUserService 创建用户服务
func NewUserService(repo *repository.UserRepository, jwtManager *jwtpkg.JWTManager) *UserService {
	return &UserService{
		repo:       repo,
		jwtManager: jwtManager,
	}
}

// Register 注册用户
func (s *UserService) Register(ctx context.Context, req *model.CreateUserRequest) (*model.User, error) {
	// 检查用户名是否已存在
	existing, err := s.repo.GetByUsername(ctx, req.Username)
	if err != nil {
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}
	if existing != nil {
		return nil, fmt.Errorf("用户名已存在")
	}

	// 检查邮箱是否已存在
	existing, err = s.repo.GetByEmail(ctx, req.Email)
	if err != nil {
		return nil, fmt.Errorf("查询邮箱失败: %w", err)
	}
	if existing != nil {
		return nil, fmt.Errorf("邮箱已被注册")
	}

	// 加密密码
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		return nil, fmt.Errorf("密码加密失败: %w", err)
	}

	// 创建用户
	user := &model.User{
		Username: req.Username,
		Email:    req.Email,
		Password: string(hashedPassword),
		Nickname: req.Nickname,
		Role:     "user",
		Status:   1,
	}
	if req.Role != "" {
		user.Role = req.Role
	}

	if err := s.repo.Create(ctx, user); err != nil {
		return nil, fmt.Errorf("创建用户失败: %w", err)
	}

	return user, nil
}

// Login 用户登录
func (s *UserService) Login(ctx context.Context, req *model.LoginRequest) (*model.LoginResponse, error) {
	// 查找用户
	user, err := s.repo.GetByUsername(ctx, req.Username)
	if err != nil {
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}
	if user == nil {
		return nil, fmt.Errorf("用户名或密码错误")
	}

	// 检查用户状态
	if user.Status != 1 {
		return nil, fmt.Errorf("账号已被禁用")
	}

	// 验证密码
	if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
		return nil, fmt.Errorf("用户名或密码错误")
	}

	// 生成 JWT
	token, expiresAt, err := s.jwtManager.GenerateToken(user.ID, user.Username, user.Role)
	if err != nil {
		return nil, fmt.Errorf("生成 token 失败: %w", err)
	}

	// 更新最后登录时间
	now := time.Now()
	s.repo.UpdateFields(ctx, user.ID, map[string]interface{}{
		"last_login": now,
	})
	user.LastLogin = &now

	return &model.LoginResponse{
		Token:     token,
		ExpiresAt: expiresAt,
		User:      user,
	}, nil
}

// GetByID 获取用户
func (s *UserService) GetByID(ctx context.Context, id uint) (*model.User, error) {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}
	if user == nil {
		return nil, fmt.Errorf("用户不存在")
	}
	return user, nil
}

// List 获取用户列表
func (s *UserService) List(ctx context.Context, req *model.ListUserRequest) ([]model.User, int64, error) {
	return s.repo.List(ctx, req)
}

// Update 更新用户
func (s *UserService) Update(ctx context.Context, id uint, req *model.UpdateUserRequest) (*model.User, error) {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}
	if user == nil {
		return nil, fmt.Errorf("用户不存在")
	}

	// 更新字段
	fields := make(map[string]interface{})
	if req.Nickname != nil {
		fields["nickname"] = *req.Nickname
	}
	if req.Avatar != nil {
		fields["avatar"] = *req.Avatar
	}
	if req.Role != nil {
		fields["role"] = *req.Role
	}
	if req.Status != nil {
		fields["status"] = *req.Status
	}

	if len(fields) > 0 {
		if err := s.repo.UpdateFields(ctx, id, fields); err != nil {
			return nil, fmt.Errorf("更新用户失败: %w", err)
		}
	}

	// 重新查询返回最新数据
	return s.repo.GetByID(ctx, id)
}

// Delete 删除用户
func (s *UserService) Delete(ctx context.Context, id uint) error {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("查询用户失败: %w", err)
	}
	if user == nil {
		return fmt.Errorf("用户不存在")
	}

	return s.repo.Delete(ctx, id)
}

// ChangePassword 修改密码
func (s *UserService) ChangePassword(ctx context.Context, userID uint, req *model.ChangePasswordRequest) error {
	user, err := s.repo.GetByID(ctx, userID)
	if err != nil {
		return fmt.Errorf("查询用户失败: %w", err)
	}
	if user == nil {
		return fmt.Errorf("用户不存在")
	}

	// 验证旧密码
	if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.OldPassword)); err != nil {
		return fmt.Errorf("旧密码不正确")
	}

	// 加密新密码
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.NewPassword), bcrypt.DefaultCost)
	if err != nil {
		return fmt.Errorf("密码加密失败: %w", err)
	}

	return s.repo.UpdateFields(ctx, userID, map[string]interface{}{
		"password": string(hashedPassword),
	})
}
```

### 3.7 控制器层（Controller）

```go
// 文件: controller/auth_controller.go
package controller

import (
	"user-api/model"
	"user-api/service"

	"github.com/gin-gonic/gin"
)

// AuthController 认证控制器
type AuthController struct {
	userService *service.UserService
}

// NewAuthController 创建认证控制器
func NewAuthController(userService *service.UserService) *AuthController {
	return &AuthController{userService: userService}
}

// Register 用户注册
// @Summary 用户注册
// @Tags 认证
// @Accept json
// @Produce json
// @Param body body model.CreateUserRequest true "注册信息"
// @Success 201 {object} model.Response{data=model.User}
// @Failure 400 {object} model.Response
// @Router /api/v1/auth/register [post]
func (ctrl *AuthController) Register(c *gin.Context) {
	var req model.CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		model.ValidationError(c, err.Error())
		return
	}

	user, err := ctrl.userService.Register(c.Request.Context(), &req)
	if err != nil {
		model.BadRequest(c, err.Error())
		return
	}

	model.Created(c, user)
}

// Login 用户登录
// @Summary 用户登录
// @Tags 认证
// @Accept json
// @Produce json
// @Param body body model.LoginRequest true "登录信息"
// @Success 200 {object} model.Response{data=model.LoginResponse}
// @Failure 400 {object} model.Response
// @Router /api/v1/auth/login [post]
func (ctrl *AuthController) Login(c *gin.Context) {
	var req model.LoginRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		model.ValidationError(c, err.Error())
		return
	}

	resp, err := ctrl.userService.Login(c.Request.Context(), &req)
	if err != nil {
		model.Unauthorized(c, err.Error())
		return
	}

	model.Success(c, resp)
}
```

```go
// 文件: controller/user_controller.go
package controller

import (
	"strconv"

	"user-api/model"
	"user-api/service"

	"github.com/gin-gonic/gin"
)

// UserController 用户控制器
type UserController struct {
	userService *service.UserService
}

// NewUserController 创建用户控制器
func NewUserController(userService *service.UserService) *UserController {
	return &UserController{userService: userService}
}

// List 获取用户列表
// @Summary 获取用户列表
// @Tags 用户管理
// @Accept json
// @Produce json
// @Param page query int false "页码" default(1)
// @Param page_size query int false "每页数量" default(20)
// @Param keyword query string false "搜索关键词"
// @Param role query string false "角色筛选"
// @Param sort query string false "排序字段" Enums(created_at, updated_at, username)
// @Param order query string false "排序方向" Enums(asc, desc)
// @Success 200 {object} model.Response{data=model.PageData}
// @Security Bearer
// @Router /api/v1/users [get]
func (ctrl *UserController) List(c *gin.Context) {
	var req model.ListUserRequest
	if err := c.ShouldBindQuery(&req); err != nil {
		model.ValidationError(c, err.Error())
		return
	}

	users, total, err := ctrl.userService.List(c.Request.Context(), &req)
	if err != nil {
		model.InternalError(c, err.Error())
		return
	}

	page := req.Page
	if page <= 0 {
		page = 1
	}
	pageSize := req.PageSize
	if pageSize <= 0 {
		pageSize = 20
	}

	model.PageSuccess(c, users, total, page, pageSize)
}

// GetByID 获取用户详情
// @Summary 获取用户详情
// @Tags 用户管理
// @Produce json
// @Param id path int true "用户ID"
// @Success 200 {object} model.Response{data=model.User}
// @Failure 404 {object} model.Response
// @Security Bearer
// @Router /api/v1/users/{id} [get]
func (ctrl *UserController) GetByID(c *gin.Context) {
	id, err := strconv.ParseUint(c.Param("id"), 10, 64)
	if err != nil {
		model.BadRequest(c, "无效的用户 ID")
		return
	}

	user, err := ctrl.userService.GetByID(c.Request.Context(), uint(id))
	if err != nil {
		model.NotFound(c, err.Error())
		return
	}

	model.Success(c, user)
}

// Update 更新用户
// @Summary 更新用户信息
// @Tags 用户管理
// @Accept json
// @Produce json
// @Param id path int true "用户ID"
// @Param body body model.UpdateUserRequest true "更新信息"
// @Success 200 {object} model.Response{data=model.User}
// @Failure 400 {object} model.Response
// @Security Bearer
// @Router /api/v1/users/{id} [put]
func (ctrl *UserController) Update(c *gin.Context) {
	id, err := strconv.ParseUint(c.Param("id"), 10, 64)
	if err != nil {
		model.BadRequest(c, "无效的用户 ID")
		return
	}

	var req model.UpdateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		model.ValidationError(c, err.Error())
		return
	}

	user, err := ctrl.userService.Update(c.Request.Context(), uint(id), &req)
	if err != nil {
		model.BadRequest(c, err.Error())
		return
	}

	model.Success(c, user)
}

// Delete 删除用户
// @Summary 删除用户
// @Tags 用户管理
// @Produce json
// @Param id path int true "用户ID"
// @Success 200 {object} model.Response
// @Failure 404 {object} model.Response
// @Security Bearer
// @Router /api/v1/users/{id} [delete]
func (ctrl *UserController) Delete(c *gin.Context) {
	id, err := strconv.ParseUint(c.Param("id"), 10, 64)
	if err != nil {
		model.BadRequest(c, "无效的用户 ID")
		return
	}

	if err := ctrl.userService.Delete(c.Request.Context(), uint(id)); err != nil {
		model.NotFound(c, err.Error())
		return
	}

	model.NoContent(c)
}

// ChangePassword 修改密码
// @Summary 修改密码
// @Tags 用户管理
// @Accept json
// @Produce json
// @Param body body model.ChangePasswordRequest true "密码信息"
// @Success 200 {object} model.Response
// @Security Bearer
// @Router /api/v1/users/password [put]
func (ctrl *UserController) ChangePassword(c *gin.Context) {
	// 从 JWT 中获取当前用户 ID
	userID, exists := c.Get("user_id")
	if !exists {
		model.Unauthorized(c, "")
		return
	}

	var req model.ChangePasswordRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		model.ValidationError(c, err.Error())
		return
	}

	if err := ctrl.userService.ChangePassword(c.Request.Context(), userID.(uint), &req); err != nil {
		model.BadRequest(c, err.Error())
		return
	}

	model.Success(c, gin.H{"message": "密码修改成功"})
}
```

### 3.8 中间件

```go
// 文件: middleware/auth.go
package middleware

import (
	"strings"

	"user-api/model"
	jwtpkg "user-api/pkg/jwt"

	"github.com/gin-gonic/gin"
)

// JWTAuth JWT 认证中间件
func JWTAuth(jwtManager *jwtpkg.JWTManager) gin.HandlerFunc {
	return func(c *gin.Context) {
		// 从 Authorization 头获取 token
		authHeader := c.GetHeader("Authorization")
		if authHeader == "" {
			model.Unauthorized(c, "缺少认证信息")
			c.Abort()
			return
		}

		// 解析 "Bearer <token>" 格式
		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || parts[0] != "Bearer" {
			model.Unauthorized(c, "认证格式错误，应为: Bearer <token>")
			c.Abort()
			return
		}

		// 解析 token
		claims, err := jwtManager.ParseToken(parts[1])
		if err != nil {
			model.Unauthorized(c, "token 无效或已过期")
			c.Abort()
			return
		}

		// 将用户信息存入 Context
		c.Set("user_id", claims.UserID)
		c.Set("username", claims.Username)
		c.Set("role", claims.Role)

		c.Next()
	}
}

// RequireRole 角色检查中间件
func RequireRole(roles ...string) gin.HandlerFunc {
	return func(c *gin.Context) {
		userRole, exists := c.Get("role")
		if !exists {
			model.Unauthorized(c, "")
			c.Abort()
			return
		}

		roleStr := userRole.(string)
		for _, role := range roles {
			if roleStr == role {
				c.Next()
				return
			}
		}

		model.Forbidden(c, "没有访问权限，需要角色: "+strings.Join(roles, " 或 "))
		c.Abort()
	}
}
```

```go
// 文件: middleware/cors.go
package middleware

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// CORS 跨域中间件
func CORS() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Header("Access-Control-Allow-Origin", "*")
		c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
		c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")
		c.Header("Access-Control-Expose-Headers", "X-Request-ID, X-Total-Count")
		c.Header("Access-Control-Max-Age", "86400")

		if c.Request.Method == "OPTIONS" {
			c.AbortWithStatus(http.StatusNoContent)
			return
		}

		c.Next()
	}
}
```

```go
// 文件: middleware/logger.go
package middleware

import (
	"fmt"
	"log"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
)

// RequestLogger 请求日志中间件
func RequestLogger() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 生成请求 ID
		requestID := c.GetHeader("X-Request-ID")
		if requestID == "" {
			requestID = uuid.New().String()
		}
		c.Set("request_id", requestID)
		c.Header("X-Request-ID", requestID)

		start := time.Now()
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery

		c.Next()

		duration := time.Since(start)
		status := c.Writer.Status()

		logEntry := fmt.Sprintf("[%s] %s %s %s | %d | %v | %s | %s",
			requestID,
			c.Request.Method,
			path,
			query,
			status,
			duration,
			c.ClientIP(),
			c.Request.UserAgent(),
		)

		if status >= 500 {
			log.Printf("ERROR %s", logEntry)
		} else if status >= 400 {
			log.Printf("WARN  %s", logEntry)
		} else {
			log.Printf("INFO  %s", logEntry)
		}
	}
}
```

### 3.9 路由定义

```go
// 文件: router/router.go
package router

import (
	"user-api/controller"
	"user-api/middleware"
	jwtpkg "user-api/pkg/jwt"

	"github.com/gin-gonic/gin"
)

// SetupRouter 配置路由
func SetupRouter(
	authCtrl *controller.AuthController,
	userCtrl *controller.UserController,
	jwtManager *jwtpkg.JWTManager,
) *gin.Engine {
	r := gin.New()

	// 全局中间件
	r.Use(gin.Recovery())
	r.Use(middleware.CORS())
	r.Use(middleware.RequestLogger())

	// 健康检查（不需要认证）
	r.GET("/health", func(c *gin.Context) {
		c.JSON(200, gin.H{"status": "ok"})
	})

	// API v1
	v1 := r.Group("/api/v1")
	{
		// 认证相关（公开路由）
		auth := v1.Group("/auth")
		{
			auth.POST("/register", authCtrl.Register)
			auth.POST("/login", authCtrl.Login)
		}

		// 用户管理（需要认证）
		users := v1.Group("/users")
		users.Use(middleware.JWTAuth(jwtManager))
		{
			users.GET("", userCtrl.List)
			users.GET("/:id", userCtrl.GetByID)
			users.PUT("/password", userCtrl.ChangePassword)

			// 管理员才能执行的操作
			admin := users.Group("")
			admin.Use(middleware.RequireRole("admin"))
			{
				admin.PUT("/:id", userCtrl.Update)
				admin.DELETE("/:id", userCtrl.Delete)
			}
		}
	}

	return r
}
```

### 3.10 Swagger 文档

```bash
# 安装 swag CLI
go install github.com/swaggo/swag/cmd/swag@latest

# 安装 gin-swagger
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```

```go
// 文件: main.go（添加 Swagger 注解）

// @title           用户管理 API
// @version         1.0
// @description     基于 Go + Gin + GORM 的用户管理 RESTful API
// @host            localhost:8080
// @BasePath        /
// @securityDefinitions.apikey Bearer
// @in header
// @name Authorization
// @description 输入 "Bearer {token}"
package main
```

```bash
# 生成 Swagger 文档
swag init

# 文件会生成在 docs/ 目录下：
# docs/docs.go
# docs/swagger.json
# docs/swagger.yaml
```

```go
// 在路由中添加 Swagger UI
import (
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"
	_ "user-api/docs" // 导入生成的文档
)

// 在 SetupRouter 中添加：
r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
// 访问 http://localhost:8080/swagger/index.html
```

### 3.11 优雅关闭

```go
// 文件: main.go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"user-api/config"
	"user-api/controller"
	jwtpkg "user-api/pkg/jwt"
	"user-api/model"
	"user-api/repository"
	"user-api/router"
	"user-api/service"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

func main() {
	// 加载配置
	cfg := config.DefaultConfig()

	// 初始化数据库（这里用 SQLite 方便演示）
	db, err := gorm.Open(sqlite.Open("users.db"), &gorm.Config{})
	if err != nil {
		log.Fatalf("数据库连接失败: %v", err)
	}

	// 自动迁移
	db.AutoMigrate(&model.User{})

	// 初始化 JWT
	jwtManager := jwtpkg.NewJWTManager(
		cfg.JWT.Secret,
		cfg.JWT.Issuer,
		cfg.JWT.ExpireTime,
	)

	// 初始化各层
	userRepo := repository.NewUserRepository(db)
	userService := service.NewUserService(userRepo, jwtManager)
	authCtrl := controller.NewAuthController(userService)
	userCtrl := controller.NewUserController(userService)

	// 设置路由
	r := router.SetupRouter(authCtrl, userCtrl, jwtManager)

	// 创建 HTTP 服务器
	srv := &http.Server{
		Addr:         cfg.Server.Port,
		Handler:      r,
		ReadTimeout:  cfg.Server.ReadTimeout,
		WriteTimeout: cfg.Server.WriteTimeout,
	}

	// 在 goroutine 中启动服务器
	go func() {
		fmt.Printf("服务启动，监听 %s\n", cfg.Server.Port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("服务启动失败: %v", err)
		}
	}()

	// ===== 优雅关闭 =====
	// 等待中断信号
	quit := make(chan os.Signal, 1)
	// SIGINT (Ctrl+C), SIGTERM (kill), SIGHUP (终端关闭)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)
	sig := <-quit

	fmt.Printf("\n收到信号 %v，开始优雅关闭...\n", sig)

	// 创建超时 context（最多等待 30 秒）
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// 关闭 HTTP 服务器
	// Shutdown 会：
	// 1. 停止接受新连接
	// 2. 等待所有活跃连接处理完成
	// 3. 超时后强制关闭
	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("服务器关闭出错: %v", err)
	}

	// 关闭数据库连接
	sqlDB, _ := db.DB()
	if sqlDB != nil {
		sqlDB.Close()
	}

	fmt.Println("服务已安全关闭")
}
```

---

## 四、测试 API

```bash
# 注册用户
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "email": "admin@example.com",
    "password": "123456",
    "nickname": "管理员",
    "role": "admin"
  }'

# 登录
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "123456"
  }'
# 返回: {"code":200,"data":{"token":"eyJhbG...","expires_at":...,"user":{...}}}

# 获取用户列表（需要 token）
TOKEN="eyJhbG..."  # 从登录响应中获取
curl http://localhost:8080/api/v1/users \
  -H "Authorization: Bearer $TOKEN"

# 获取单个用户
curl http://localhost:8080/api/v1/users/1 \
  -H "Authorization: Bearer $TOKEN"

# 更新用户（需要 admin 角色）
curl -X PUT http://localhost:8080/api/v1/users/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"nickname": "超级管理员", "status": 1}'

# 删除用户（需要 admin 角色）
curl -X DELETE http://localhost:8080/api/v1/users/2 \
  -H "Authorization: Bearer $TOKEN"

# 分页查询
curl "http://localhost:8080/api/v1/users?page=1&page_size=10&keyword=admin&sort=created_at&order=desc" \
  -H "Authorization: Bearer $TOKEN"

# 修改密码
curl -X PUT http://localhost:8080/api/v1/users/password \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"old_password": "123456", "new_password": "newpass123"}'

# 健康检查
curl http://localhost:8080/health
```

---

## 五、常见坑点

### 坑点 1：JWT Secret 硬编码

```go
// 错误：密钥硬编码在代码中
jwtManager := jwtpkg.NewJWTManager("my-secret", ...)

// 正确：从环境变量或配置文件读取
secret := os.Getenv("JWT_SECRET")
if secret == "" {
	log.Fatal("JWT_SECRET 环境变量未设置")
}
jwtManager := jwtpkg.NewJWTManager(secret, ...)
```

### 坑点 2：密码明文存储

```go
// 错误：明文存储密码
user.Password = req.Password

// 正确：使用 bcrypt 加密
hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
user.Password = string(hashedPassword)
```

### 坑点 3：不安全的 ID 类型

```go
// 错误：使用自增 ID 暴露业务量
// GET /api/v1/users/1, /api/v1/users/2, ...
// 攻击者可以遍历所有用户

// 改进方案：
// 1. 使用 UUID 作为外部标识
// 2. 对 ID 加密/混淆
// 3. 添加访问控制，只允许查看自己的信息
```

### 坑点 4：Shutdown 超时太短

```go
// 错误：超时时间太短，请求可能被强制中断
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)

// 正确：根据实际情况设置合理的超时
// 考虑最慢的请求处理时间
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
```

### 坑点 5：忘记验证路由参数

```go
// 错误：直接使用路由参数，不做转换和验证
func (ctrl *UserController) GetByID(c *gin.Context) {
	idStr := c.Param("id")
	// 如果 id 不是数字呢？如果是负数呢？
}

// 正确：转换并验证
func (ctrl *UserController) GetByID(c *gin.Context) {
	id, err := strconv.ParseUint(c.Param("id"), 10, 64)
	if err != nil || id == 0 {
		model.BadRequest(c, "无效的用户 ID")
		return
	}
}
```

---

## 六、速查表

### 项目分层职责

| 层 | 职责 | 文件 |
|----|------|------|
| Controller | 处理 HTTP 请求/响应，参数校验 | `controller/*.go` |
| Service | 业务逻辑，事务管理 | `service/*.go` |
| Repository | 数据访问，SQL 查询 | `repository/*.go` |
| Model | 数据模型，DTO 定义 | `model/*.go` |
| Middleware | 横切关注点（认证、日志等） | `middleware/*.go` |
| Router | 路由注册与分组 | `router/*.go` |

### RESTful API 设计规范速查

```
GET    /resources           # 列表查询
GET    /resources/:id       # 获取详情
POST   /resources           # 创建
PUT    /resources/:id       # 全量更新
PATCH  /resources/:id       # 部分更新
DELETE /resources/:id       # 删除

# 分页: ?page=1&page_size=20
# 搜索: ?keyword=xxx
# 排序: ?sort=field&order=asc|desc
# 过滤: ?status=active&role=admin
```

### 优雅关闭流程

```
1. 接收到终止信号 (SIGINT/SIGTERM)
2. 停止接受新的 HTTP 连接
3. 等待所有活跃请求处理完成
4. 关闭数据库连接池
5. 关闭 Redis 连接
6. 关闭其他资源（消息队列等）
7. 退出进程
```

### 安全注意事项

```
1. JWT Secret 从环境变量读取，不要硬编码
2. 密码使用 bcrypt 加密存储
3. 输入参数全部校验
4. SQL 使用参数化查询（GORM 默认支持）
5. 敏感字段（如 Password）不返回给前端 (json:"-")
6. 启用 CORS 白名单（生产环境不要用 *）
7. 启用请求限流
8. 使用 HTTPS
9. 日志不要打印敏感信息
10. 生产环境使用 gin.ReleaseMode
```
