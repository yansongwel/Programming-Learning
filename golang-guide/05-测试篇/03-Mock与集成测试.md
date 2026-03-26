# 03-Mock与集成测试

## 导读

单元测试验证单个函数的正确性，但真实系统中函数往往依赖外部服务——数据库、HTTP API、消息队列、缓存等。作为 SRE/DevOps 工程师，你的代码几乎不可避免地会与这些外部系统交互。如何在不依赖真实外部服务的情况下测试这些代码？答案是 **Mock（模拟）**。而当你需要验证与真实外部服务的交互时，就需要 **集成测试**。

本章将系统讲解：

- 接口 Mock：利用 Go 接口实现 mock、手写 mock vs gomock/mockgen
- testify 断言库：assert/require/suite/mock
- httptest 包：测试 HTTP Handler
- 数据库测试：sqlmock、testcontainers-go
- 集成测试组织与 build tag

> 基于 Go 1.24，所有代码完整可运行。

---

## 一、接口 Mock 基础

### 1.1 为什么需要 Mock

```go
// 文件: orderservice/order.go
package orderservice

import (
	"context"
	"fmt"
	"time"
)

// Order 订单
type Order struct {
	ID        string
	UserID    string
	Amount    float64
	Status    string
	CreatedAt time.Time
}

// 问题：OrderService 直接依赖数据库和外部支付 API
// 测试时需要真实的数据库和支付服务才能运行——这不是好的设计

// type BadOrderService struct {
// 	db     *sql.DB           // 直接依赖数据库连接
// 	payAPI *PaymentAPIClient // 直接依赖外部 API 客户端
// }
```

### 1.2 利用接口实现可测试的设计

```go
// 文件: orderservice/interfaces.go
package orderservice

import "context"

// OrderRepository 订单存储接口——抽象数据库操作
type OrderRepository interface {
	Create(ctx context.Context, order *Order) error
	GetByID(ctx context.Context, id string) (*Order, error)
	UpdateStatus(ctx context.Context, id string, status string) error
	ListByUser(ctx context.Context, userID string) ([]*Order, error)
}

// PaymentGateway 支付网关接口——抽象外部支付 API
type PaymentGateway interface {
	Charge(ctx context.Context, userID string, amount float64) (transactionID string, err error)
	Refund(ctx context.Context, transactionID string) error
}

// NotificationService 通知服务接口——抽象消息发送
type NotificationService interface {
	SendEmail(ctx context.Context, to, subject, body string) error
	SendSMS(ctx context.Context, to, message string) error
}
```

```go
// 文件: orderservice/service.go
package orderservice

import (
	"context"
	"fmt"
	"time"
)

// OrderService 订单服务——依赖接口而非具体实现
type OrderService struct {
	repo     OrderRepository
	payment  PaymentGateway
	notifier NotificationService
}

// NewOrderService 创建订单服务
func NewOrderService(repo OrderRepository, payment PaymentGateway, notifier NotificationService) *OrderService {
	return &OrderService{
		repo:     repo,
		payment:  payment,
		notifier: notifier,
	}
}

// CreateOrder 创建订单
func (s *OrderService) CreateOrder(ctx context.Context, userID string, amount float64) (*Order, error) {
	if amount <= 0 {
		return nil, fmt.Errorf("订单金额必须大于零")
	}

	order := &Order{
		ID:        fmt.Sprintf("ORD-%d", time.Now().UnixNano()),
		UserID:    userID,
		Amount:    amount,
		Status:    "pending",
		CreatedAt: time.Now(),
	}

	// 保存到数据库
	if err := s.repo.Create(ctx, order); err != nil {
		return nil, fmt.Errorf("保存订单失败: %w", err)
	}

	// 发起支付
	txID, err := s.payment.Charge(ctx, userID, amount)
	if err != nil {
		// 支付失败，更新订单状态
		s.repo.UpdateStatus(ctx, order.ID, "payment_failed")
		return nil, fmt.Errorf("支付失败: %w", err)
	}

	// 支付成功，更新状态
	order.Status = "paid"
	if err := s.repo.UpdateStatus(ctx, order.ID, "paid"); err != nil {
		return nil, fmt.Errorf("更新订单状态失败: %w", err)
	}

	// 发送通知（异步，不影响主流程）
	go func() {
		subject := fmt.Sprintf("订单 %s 支付成功", order.ID)
		body := fmt.Sprintf("您的订单已支付成功，交易号: %s，金额: %.2f", txID, amount)
		s.notifier.SendEmail(context.Background(), userID, subject, body)
	}()

	return order, nil
}

// GetOrder 获取订单
func (s *OrderService) GetOrder(ctx context.Context, id string) (*Order, error) {
	order, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("获取订单失败: %w", err)
	}
	if order == nil {
		return nil, fmt.Errorf("订单不存在: %s", id)
	}
	return order, nil
}

// RefundOrder 退款
func (s *OrderService) RefundOrder(ctx context.Context, orderID, transactionID string) error {
	order, err := s.repo.GetByID(ctx, orderID)
	if err != nil {
		return fmt.Errorf("获取订单失败: %w", err)
	}
	if order == nil {
		return fmt.Errorf("订单不存在: %s", orderID)
	}
	if order.Status != "paid" {
		return fmt.Errorf("订单状态不允许退款: %s", order.Status)
	}

	if err := s.payment.Refund(ctx, transactionID); err != nil {
		return fmt.Errorf("退款失败: %w", err)
	}

	if err := s.repo.UpdateStatus(ctx, orderID, "refunded"); err != nil {
		return fmt.Errorf("更新退款状态失败: %w", err)
	}

	return nil
}
```

### 1.3 手写 Mock

```go
// 文件: orderservice/service_test.go
package orderservice

import (
	"context"
	"fmt"
	"sync"
	"testing"
)

// ===== 手写 Mock 实现 =====

// MockOrderRepository 模拟订单存储
type MockOrderRepository struct {
	mu     sync.Mutex
	orders map[string]*Order

	// 控制行为的字段
	CreateFunc       func(ctx context.Context, order *Order) error
	GetByIDFunc      func(ctx context.Context, id string) (*Order, error)
	UpdateStatusFunc func(ctx context.Context, id string, status string) error
	ListByUserFunc   func(ctx context.Context, userID string) ([]*Order, error)

	// 记录调用的字段
	CreateCalled       int
	GetByIDCalled      int
	UpdateStatusCalled int
}

func NewMockOrderRepository() *MockOrderRepository {
	return &MockOrderRepository{
		orders: make(map[string]*Order),
	}
}

func (m *MockOrderRepository) Create(ctx context.Context, order *Order) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.CreateCalled++

	if m.CreateFunc != nil {
		return m.CreateFunc(ctx, order)
	}
	// 默认行为：保存到内存
	m.orders[order.ID] = order
	return nil
}

func (m *MockOrderRepository) GetByID(ctx context.Context, id string) (*Order, error) {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.GetByIDCalled++

	if m.GetByIDFunc != nil {
		return m.GetByIDFunc(ctx, id)
	}
	order, ok := m.orders[id]
	if !ok {
		return nil, nil
	}
	return order, nil
}

func (m *MockOrderRepository) UpdateStatus(ctx context.Context, id string, status string) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.UpdateStatusCalled++

	if m.UpdateStatusFunc != nil {
		return m.UpdateStatusFunc(ctx, id, status)
	}
	if order, ok := m.orders[id]; ok {
		order.Status = status
	}
	return nil
}

func (m *MockOrderRepository) ListByUser(ctx context.Context, userID string) ([]*Order, error) {
	m.mu.Lock()
	defer m.mu.Unlock()

	if m.ListByUserFunc != nil {
		return m.ListByUserFunc(ctx, userID)
	}
	var result []*Order
	for _, order := range m.orders {
		if order.UserID == userID {
			result = append(result, order)
		}
	}
	return result, nil
}

// MockPaymentGateway 模拟支付网关
type MockPaymentGateway struct {
	ChargeFunc func(ctx context.Context, userID string, amount float64) (string, error)
	RefundFunc func(ctx context.Context, transactionID string) error

	ChargeCalled int
	RefundCalled int
}

func (m *MockPaymentGateway) Charge(ctx context.Context, userID string, amount float64) (string, error) {
	m.ChargeCalled++
	if m.ChargeFunc != nil {
		return m.ChargeFunc(ctx, userID, amount)
	}
	return "TX-MOCK-123", nil
}

func (m *MockPaymentGateway) Refund(ctx context.Context, transactionID string) error {
	m.RefundCalled++
	if m.RefundFunc != nil {
		return m.RefundFunc(ctx, transactionID)
	}
	return nil
}

// MockNotificationService 模拟通知服务
type MockNotificationService struct {
	SendEmailFunc func(ctx context.Context, to, subject, body string) error
	SendSMSFunc   func(ctx context.Context, to, message string) error

	EmailsSent []struct{ To, Subject, Body string }
}

func (m *MockNotificationService) SendEmail(ctx context.Context, to, subject, body string) error {
	m.EmailsSent = append(m.EmailsSent, struct{ To, Subject, Body string }{to, subject, body})
	if m.SendEmailFunc != nil {
		return m.SendEmailFunc(ctx, to, subject, body)
	}
	return nil
}

func (m *MockNotificationService) SendSMS(ctx context.Context, to, message string) error {
	if m.SendSMSFunc != nil {
		return m.SendSMSFunc(ctx, to, message)
	}
	return nil
}

// ===== 测试 =====

func TestOrderService_CreateOrder(t *testing.T) {
	t.Run("成功创建订单", func(t *testing.T) {
		repo := NewMockOrderRepository()
		payment := &MockPaymentGateway{}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		order, err := svc.CreateOrder(ctx, "user-1", 99.99)
		if err != nil {
			t.Fatalf("创建订单失败: %v", err)
		}

		// 验证订单属性
		if order.UserID != "user-1" {
			t.Errorf("UserID = %q; 期望 %q", order.UserID, "user-1")
		}
		if order.Amount != 99.99 {
			t.Errorf("Amount = %f; 期望 %f", order.Amount, 99.99)
		}
		if order.Status != "paid" {
			t.Errorf("Status = %q; 期望 %q", order.Status, "paid")
		}

		// 验证 Mock 被调用
		if repo.CreateCalled != 1 {
			t.Errorf("repo.Create 被调用 %d 次; 期望 1 次", repo.CreateCalled)
		}
		if payment.ChargeCalled != 1 {
			t.Errorf("payment.Charge 被调用 %d 次; 期望 1 次", payment.ChargeCalled)
		}
	})

	t.Run("金额为零返回错误", func(t *testing.T) {
		repo := NewMockOrderRepository()
		payment := &MockPaymentGateway{}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		_, err := svc.CreateOrder(ctx, "user-1", 0)
		if err == nil {
			t.Fatal("金额为零应该返回错误")
		}

		// 验证数据库和支付都没有被调用
		if repo.CreateCalled != 0 {
			t.Errorf("repo.Create 不应被调用")
		}
		if payment.ChargeCalled != 0 {
			t.Errorf("payment.Charge 不应被调用")
		}
	})

	t.Run("支付失败", func(t *testing.T) {
		repo := NewMockOrderRepository()
		payment := &MockPaymentGateway{
			ChargeFunc: func(ctx context.Context, userID string, amount float64) (string, error) {
				return "", fmt.Errorf("余额不足")
			},
		}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		_, err := svc.CreateOrder(ctx, "user-1", 99.99)
		if err == nil {
			t.Fatal("支付失败应该返回错误")
		}

		// 验证订单状态被更新为支付失败
		if repo.UpdateStatusCalled != 1 {
			t.Errorf("repo.UpdateStatus 应该被调用 1 次; 实际 %d 次", repo.UpdateStatusCalled)
		}
	})

	t.Run("数据库保存失败", func(t *testing.T) {
		repo := NewMockOrderRepository()
		repo.CreateFunc = func(ctx context.Context, order *Order) error {
			return fmt.Errorf("数据库连接失败")
		}
		payment := &MockPaymentGateway{}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		_, err := svc.CreateOrder(ctx, "user-1", 99.99)
		if err == nil {
			t.Fatal("数据库失败应该返回错误")
		}

		// 验证支付没有被调用（数据库失败应该提前返回）
		if payment.ChargeCalled != 0 {
			t.Error("数据库失败后不应该发起支付")
		}
	})
}

func TestOrderService_RefundOrder(t *testing.T) {
	t.Run("成功退款", func(t *testing.T) {
		repo := NewMockOrderRepository()
		// 预置一个已支付的订单
		repo.orders["ORD-001"] = &Order{
			ID:     "ORD-001",
			UserID: "user-1",
			Amount: 50.00,
			Status: "paid",
		}
		payment := &MockPaymentGateway{}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		err := svc.RefundOrder(ctx, "ORD-001", "TX-123")
		if err != nil {
			t.Fatalf("退款失败: %v", err)
		}

		if payment.RefundCalled != 1 {
			t.Error("payment.Refund 应该被调用")
		}
	})

	t.Run("未支付订单不能退款", func(t *testing.T) {
		repo := NewMockOrderRepository()
		repo.orders["ORD-002"] = &Order{
			ID:     "ORD-002",
			Status: "pending",
		}
		payment := &MockPaymentGateway{}
		notifier := &MockNotificationService{}

		svc := NewOrderService(repo, payment, notifier)
		ctx := context.Background()

		err := svc.RefundOrder(ctx, "ORD-002", "TX-123")
		if err == nil {
			t.Fatal("未支付订单退款应该返回错误")
		}

		if payment.RefundCalled != 0 {
			t.Error("payment.Refund 不应该被调用")
		}
	})
}
```

### 1.4 gomock/mockgen 自动生成 Mock

```bash
# 安装 mockgen
go install go.uber.org/mock/mockgen@latest

# 从接口生成 Mock
# 方式1：源码模式（推荐）
mockgen -source=interfaces.go -destination=mock_interfaces_test.go -package=orderservice

# 方式2：反射模式
mockgen -destination=mock_repo_test.go -package=orderservice \
  myproject/orderservice OrderRepository,PaymentGateway

# Go 1.24 中也可以使用 go generate
```

```go
// 文件: orderservice/interfaces.go 中添加 go:generate 注释
//go:generate mockgen -source=interfaces.go -destination=mock_interfaces_test.go -package=orderservice
```

```go
// 使用 gomock 生成的 Mock 编写测试
package orderservice

import (
	"context"
	"testing"

	"go.uber.org/mock/gomock"
)

func TestOrderService_CreateOrder_WithGomock(t *testing.T) {
	ctrl := gomock.NewController(t)
	// Go 1.24 中 ctrl.Finish() 不再需要手动调用，
	// gomock.NewController 会自动注册 Cleanup

	// 创建 Mock 对象
	mockRepo := NewMockOrderRepository(ctrl)
	mockPayment := NewMockPaymentGateway(ctrl)
	mockNotifier := NewMockNotificationService(ctrl)

	// 设置期望
	mockRepo.EXPECT().
		Create(gomock.Any(), gomock.Any()).
		Return(nil).
		Times(1)

	mockPayment.EXPECT().
		Charge(gomock.Any(), "user-1", 99.99).
		Return("TX-001", nil).
		Times(1)

	mockRepo.EXPECT().
		UpdateStatus(gomock.Any(), gomock.Any(), "paid").
		Return(nil).
		Times(1)

	// 通知是异步的，可能调也可能没调
	mockNotifier.EXPECT().
		SendEmail(gomock.Any(), gomock.Any(), gomock.Any(), gomock.Any()).
		Return(nil).
		AnyTimes()

	// 执行测试
	svc := NewOrderService(mockRepo, mockPayment, mockNotifier)
	ctx := context.Background()

	order, err := svc.CreateOrder(ctx, "user-1", 99.99)
	if err != nil {
		t.Fatalf("创建订单失败: %v", err)
	}
	if order.Status != "paid" {
		t.Errorf("Status = %q; 期望 %q", order.Status, "paid")
	}
}

func TestOrderService_CreateOrder_PaymentFail_WithGomock(t *testing.T) {
	ctrl := gomock.NewController(t)

	mockRepo := NewMockOrderRepository(ctrl)
	mockPayment := NewMockPaymentGateway(ctrl)
	mockNotifier := NewMockNotificationService(ctrl)

	// 使用 gomock.InOrder 确保调用顺序
	gomock.InOrder(
		mockRepo.EXPECT().Create(gomock.Any(), gomock.Any()).Return(nil),
		mockPayment.EXPECT().Charge(gomock.Any(), "user-1", 50.0).Return("", fmt.Errorf("支付超时")),
		mockRepo.EXPECT().UpdateStatus(gomock.Any(), gomock.Any(), "payment_failed").Return(nil),
	)

	svc := NewOrderService(mockRepo, mockPayment, mockNotifier)
	_, err := svc.CreateOrder(context.Background(), "user-1", 50.0)

	if err == nil {
		t.Fatal("支付失败应该返回错误")
	}
}
```

---

## 二、testify 断言库

`testify` 是 Go 生态中最流行的测试辅助库，提供 assert、require、suite、mock 等功能。

### 2.1 安装

```bash
go get github.com/stretchr/testify
```

### 2.2 assert 与 require

```go
package testifyexample

import (
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

// User 用户结构体
type User struct {
	ID    int
	Name  string
	Email string
	Age   int
	Tags  []string
}

func TestAssertExamples(t *testing.T) {
	user := &User{
		ID:    1,
		Name:  "张三",
		Email: "zhangsan@example.com",
		Age:   25,
		Tags:  []string{"admin", "active"},
	}

	// ===== assert: 失败后继续执行 =====

	// 相等性断言
	assert.Equal(t, 1, user.ID, "ID 应该等于 1")
	assert.Equal(t, "张三", user.Name)
	assert.NotEqual(t, "李四", user.Name)

	// nil 检查
	assert.NotNil(t, user)
	var nilUser *User
	assert.Nil(t, nilUser)

	// 布尔断言
	assert.True(t, user.Age >= 18, "用户应该是成年人")
	assert.False(t, user.Age < 0)

	// 字符串断言
	assert.Contains(t, user.Email, "@")
	assert.HasPrefix(t, user.Email, "zhangsan")
	assert.HasSuffix(t, user.Email, ".com")
	assert.Regexp(t, `^[a-z]+@`, user.Email)

	// 集合断言
	assert.Len(t, user.Tags, 2)
	assert.Contains(t, user.Tags, "admin")
	assert.NotContains(t, user.Tags, "disabled")
	assert.ElementsMatch(t, []string{"active", "admin"}, user.Tags) // 忽略顺序

	// 数值断言
	assert.Greater(t, user.Age, 0)
	assert.GreaterOrEqual(t, user.Age, 18)
	assert.Less(t, user.Age, 200)
	assert.InDelta(t, 25.0, float64(user.Age), 0.1) // 浮点数近似比较

	// 类型断言
	assert.IsType(t, &User{}, user)

	// 错误断言
	err := errors.New("连接超时")
	assert.Error(t, err)
	assert.EqualError(t, err, "连接超时")
	assert.ErrorContains(t, err, "超时")

	assert.NoError(t, nil)
}

func TestRequireExamples(t *testing.T) {
	// ===== require: 失败后立即终止 =====
	// require 的 API 与 assert 完全一致，区别在于失败时调用 t.FailNow()

	user := &User{ID: 1, Name: "张三"}

	// 当后续断言依赖当前断言时，使用 require
	require.NotNil(t, user, "user 不应为 nil")
	// 如果 user 为 nil，下面的代码不会执行（避免空指针 panic）
	require.Equal(t, "张三", user.Name)
	require.Equal(t, 1, user.ID)
}

func TestAssertWithTableDriven(t *testing.T) {
	tests := []struct {
		name     string
		a, b     int
		expected int
	}{
		{"正数相加", 2, 3, 5},
		{"负数相加", -1, -2, -3},
		{"零值", 0, 0, 0},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := tt.a + tt.b
			assert.Equal(t, tt.expected, result, "计算结果不正确")
		})
	}
}
```

### 2.3 testify/suite 测试套件

```go
package testifyexample

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/stretchr/testify/suite"
)

// UserStore 用户存储接口
type UserStore interface {
	Save(ctx context.Context, user *User) error
	FindByID(ctx context.Context, id int) (*User, error)
	Delete(ctx context.Context, id int) error
}

// InMemoryUserStore 内存实现
type InMemoryUserStore struct {
	users map[int]*User
}

func NewInMemoryUserStore() *InMemoryUserStore {
	return &InMemoryUserStore{users: make(map[int]*User)}
}

func (s *InMemoryUserStore) Save(ctx context.Context, user *User) error {
	s.users[user.ID] = user
	return nil
}

func (s *InMemoryUserStore) FindByID(ctx context.Context, id int) (*User, error) {
	user, ok := s.users[id]
	if !ok {
		return nil, nil
	}
	return user, nil
}

func (s *InMemoryUserStore) Delete(ctx context.Context, id int) error {
	delete(s.users, id)
	return nil
}

// UserStoreSuite 用户存储测试套件
type UserStoreSuite struct {
	suite.Suite
	store UserStore
	ctx   context.Context
}

// SetupSuite 在所有测试之前运行一次
func (s *UserStoreSuite) SetupSuite() {
	s.T().Log("=== 测试套件初始化 ===")
	s.ctx = context.Background()
}

// TearDownSuite 在所有测试之后运行一次
func (s *UserStoreSuite) TearDownSuite() {
	s.T().Log("=== 测试套件清理 ===")
}

// SetupTest 在每个测试之前运行
func (s *UserStoreSuite) SetupTest() {
	s.T().Log("--- 测试用例初始化 ---")
	// 每个测试用例使用全新的存储实例
	s.store = NewInMemoryUserStore()
}

// TearDownTest 在每个测试之后运行
func (s *UserStoreSuite) TearDownTest() {
	s.T().Log("--- 测试用例清理 ---")
}

// TestSaveAndFind 测试保存和查询
func (s *UserStoreSuite) TestSaveAndFind() {
	user := &User{ID: 1, Name: "张三", Email: "zhangsan@test.com"}

	err := s.store.Save(s.ctx, user)
	require.NoError(s.T(), err)

	found, err := s.store.FindByID(s.ctx, 1)
	require.NoError(s.T(), err)
	require.NotNil(s.T(), found)

	assert.Equal(s.T(), "张三", found.Name)
	assert.Equal(s.T(), "zhangsan@test.com", found.Email)
}

// TestFindNotExist 测试查询不存在的用户
func (s *UserStoreSuite) TestFindNotExist() {
	found, err := s.store.FindByID(s.ctx, 999)
	require.NoError(s.T(), err)
	assert.Nil(s.T(), found)
}

// TestDelete 测试删除
func (s *UserStoreSuite) TestDelete() {
	user := &User{ID: 2, Name: "李四"}
	err := s.store.Save(s.ctx, user)
	require.NoError(s.T(), err)

	err = s.store.Delete(s.ctx, 2)
	require.NoError(s.T(), err)

	found, err := s.store.FindByID(s.ctx, 2)
	require.NoError(s.T(), err)
	assert.Nil(s.T(), found, "删除后不应能查到用户")
}

// TestSaveOverwrite 测试覆盖保存
func (s *UserStoreSuite) TestSaveOverwrite() {
	user1 := &User{ID: 3, Name: "王五"}
	s.store.Save(s.ctx, user1)

	user2 := &User{ID: 3, Name: "王五（更新后）"}
	s.store.Save(s.ctx, user2)

	found, _ := s.store.FindByID(s.ctx, 3)
	require.NotNil(s.T(), found)
	assert.Equal(s.T(), "王五（更新后）", found.Name)
}

// 运行测试套件
func TestUserStoreSuite(t *testing.T) {
	suite.Run(t, new(UserStoreSuite))
}
```

### 2.4 testify/mock

```go
package testifyexample

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

// MockUserStore 使用 testify/mock 实现的 Mock
type MockUserStore struct {
	mock.Mock
}

func (m *MockUserStore) Save(ctx context.Context, user *User) error {
	args := m.Called(ctx, user)
	return args.Error(0)
}

func (m *MockUserStore) FindByID(ctx context.Context, id int) (*User, error) {
	args := m.Called(ctx, id)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserStore) Delete(ctx context.Context, id int) error {
	args := m.Called(ctx, id)
	return args.Error(0)
}

// UserManager 用户管理器
type UserManager struct {
	store UserStore
}

func (m *UserManager) GetUser(ctx context.Context, id int) (*User, error) {
	return m.store.FindByID(ctx, id)
}

func (m *UserManager) CreateUser(ctx context.Context, user *User) error {
	return m.store.Save(ctx, user)
}

func TestUserManager_GetUser_WithTestifyMock(t *testing.T) {
	// 创建 Mock
	mockStore := new(MockUserStore)

	// 设置期望：当调用 FindByID(ctx, 1) 时返回指定用户
	expectedUser := &User{ID: 1, Name: "张三"}
	mockStore.On("FindByID", mock.Anything, 1).Return(expectedUser, nil)

	// 设置期望：当调用 FindByID(ctx, 999) 时返回 nil
	mockStore.On("FindByID", mock.Anything, 999).Return(nil, nil)

	manager := &UserManager{store: mockStore}
	ctx := context.Background()

	// 测试找到用户
	user, err := manager.GetUser(ctx, 1)
	assert.NoError(t, err)
	assert.NotNil(t, user)
	assert.Equal(t, "张三", user.Name)

	// 测试找不到用户
	user, err = manager.GetUser(ctx, 999)
	assert.NoError(t, err)
	assert.Nil(t, user)

	// 验证期望被满足
	mockStore.AssertExpectations(t)

	// 验证调用次数
	mockStore.AssertNumberOfCalls(t, "FindByID", 2)
}

func TestUserManager_CreateUser_WithTestifyMock(t *testing.T) {
	mockStore := new(MockUserStore)

	// 使用 MatchedBy 自定义匹配逻辑
	mockStore.On("Save", mock.Anything, mock.MatchedBy(func(u *User) bool {
		return u.Name != "" && u.Email != ""
	})).Return(nil)

	manager := &UserManager{store: mockStore}
	ctx := context.Background()

	err := manager.CreateUser(ctx, &User{ID: 1, Name: "张三", Email: "a@b.com"})
	assert.NoError(t, err)

	mockStore.AssertExpectations(t)
}
```

---

## 三、httptest 包

`net/http/httptest` 是 Go 标准库中用于测试 HTTP 处理器的工具包。

### 3.1 httptest.NewRecorder 测试 Handler

```go
// 文件: api/handlers.go
package api

import (
	"encoding/json"
	"net/http"
	"strconv"
	"sync"
)

// Response 统一响应结构
type Response struct {
	Code    int         `json:"code"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
}

// UserHandler 用户处理器
type UserHandler struct {
	mu    sync.RWMutex
	users map[int]*User
}

type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func NewUserHandler() *UserHandler {
	return &UserHandler{
		users: map[int]*User{
			1: {ID: 1, Name: "张三", Email: "zhangsan@test.com"},
			2: {ID: 2, Name: "李四", Email: "lisi@test.com"},
		},
	}
}

// GetUser 获取用户
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	idStr := r.URL.Query().Get("id")
	if idStr == "" {
		writeJSON(w, http.StatusBadRequest, Response{
			Code: 400, Message: "缺少 id 参数",
		})
		return
	}

	id, err := strconv.Atoi(idStr)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, Response{
			Code: 400, Message: "id 必须是数字",
		})
		return
	}

	h.mu.RLock()
	user, ok := h.users[id]
	h.mu.RUnlock()

	if !ok {
		writeJSON(w, http.StatusNotFound, Response{
			Code: 404, Message: "用户不存在",
		})
		return
	}

	writeJSON(w, http.StatusOK, Response{
		Code: 200, Message: "success", Data: user,
	})
}

// CreateUser 创建用户
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeJSON(w, http.StatusMethodNotAllowed, Response{
			Code: 405, Message: "只支持 POST 方法",
		})
		return
	}

	var user User
	if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
		writeJSON(w, http.StatusBadRequest, Response{
			Code: 400, Message: "请求体格式错误",
		})
		return
	}

	if user.Name == "" || user.Email == "" {
		writeJSON(w, http.StatusBadRequest, Response{
			Code: 400, Message: "name 和 email 不能为空",
		})
		return
	}

	h.mu.Lock()
	user.ID = len(h.users) + 1
	h.users[user.ID] = &user
	h.mu.Unlock()

	writeJSON(w, http.StatusCreated, Response{
		Code: 201, Message: "创建成功", Data: user,
	})
}

// HealthCheck 健康检查
func HealthCheck(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, Response{
		Code: 200, Message: "ok",
	})
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}
```

```go
// 文件: api/handlers_test.go
package api

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestHealthCheck(t *testing.T) {
	// 创建请求
	req := httptest.NewRequest("GET", "/health", nil)
	// 创建响应记录器
	w := httptest.NewRecorder()

	// 调用 handler
	HealthCheck(w, req)

	// 检查响应
	resp := w.Result()
	assert.Equal(t, http.StatusOK, resp.StatusCode)
	assert.Equal(t, "application/json", resp.Header.Get("Content-Type"))

	var body Response
	err := json.NewDecoder(resp.Body).Decode(&body)
	require.NoError(t, err)
	assert.Equal(t, 200, body.Code)
	assert.Equal(t, "ok", body.Message)
}

func TestUserHandler_GetUser(t *testing.T) {
	handler := NewUserHandler()

	tests := []struct {
		name         string
		queryParams  string
		wantStatus   int
		wantCode     int
		wantUserName string // 期望的用户名（如果有）
	}{
		{
			name:         "获取存在的用户",
			queryParams:  "?id=1",
			wantStatus:   http.StatusOK,
			wantCode:     200,
			wantUserName: "张三",
		},
		{
			name:       "获取不存在的用户",
			queryParams: "?id=999",
			wantStatus: http.StatusNotFound,
			wantCode:   404,
		},
		{
			name:       "缺少id参数",
			queryParams: "",
			wantStatus: http.StatusBadRequest,
			wantCode:   400,
		},
		{
			name:       "id不是数字",
			queryParams: "?id=abc",
			wantStatus: http.StatusBadRequest,
			wantCode:   400,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			req := httptest.NewRequest("GET", "/user"+tt.queryParams, nil)
			w := httptest.NewRecorder()

			handler.GetUser(w, req)

			resp := w.Result()
			assert.Equal(t, tt.wantStatus, resp.StatusCode)

			var body Response
			err := json.NewDecoder(resp.Body).Decode(&body)
			require.NoError(t, err)
			assert.Equal(t, tt.wantCode, body.Code)

			if tt.wantUserName != "" {
				// 解析 Data 字段
				userData, ok := body.Data.(map[string]interface{})
				require.True(t, ok, "Data 应该是 map 类型")
				assert.Equal(t, tt.wantUserName, userData["name"])
			}
		})
	}
}

func TestUserHandler_CreateUser(t *testing.T) {
	handler := NewUserHandler()

	t.Run("成功创建用户", func(t *testing.T) {
		body := `{"name": "王五", "email": "wangwu@test.com"}`
		req := httptest.NewRequest("POST", "/user", bytes.NewBufferString(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		handler.CreateUser(w, req)

		resp := w.Result()
		assert.Equal(t, http.StatusCreated, resp.StatusCode)

		var result Response
		json.NewDecoder(resp.Body).Decode(&result)
		assert.Equal(t, 201, result.Code)
	})

	t.Run("缺少必填字段", func(t *testing.T) {
		body := `{"name": ""}`
		req := httptest.NewRequest("POST", "/user", bytes.NewBufferString(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		handler.CreateUser(w, req)

		assert.Equal(t, http.StatusBadRequest, w.Code)
	})

	t.Run("请求体格式错误", func(t *testing.T) {
		body := `invalid json`
		req := httptest.NewRequest("POST", "/user", bytes.NewBufferString(body))
		w := httptest.NewRecorder()

		handler.CreateUser(w, req)

		assert.Equal(t, http.StatusBadRequest, w.Code)
	})

	t.Run("错误的HTTP方法", func(t *testing.T) {
		req := httptest.NewRequest("GET", "/user", nil)
		w := httptest.NewRecorder()

		handler.CreateUser(w, req)

		assert.Equal(t, http.StatusMethodNotAllowed, w.Code)
	})
}
```

### 3.2 httptest.NewServer 测试完整 HTTP 服务

```go
// 文件: api/server_test.go
package api

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

// setupTestServer 创建测试服务器
func setupTestServer() *httptest.Server {
	handler := NewUserHandler()

	mux := http.NewServeMux()
	mux.HandleFunc("/health", HealthCheck)
	mux.HandleFunc("/user", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case "GET":
			handler.GetUser(w, r)
		case "POST":
			handler.CreateUser(w, r)
		default:
			w.WriteHeader(http.StatusMethodNotAllowed)
		}
	})

	return httptest.NewServer(mux)
}

func TestIntegration_FullAPI(t *testing.T) {
	server := setupTestServer()
	defer server.Close()

	client := server.Client()
	// 可以配置客户端超时
	client.Timeout = 5 * time.Second

	baseURL := server.URL

	t.Run("健康检查", func(t *testing.T) {
		resp, err := client.Get(baseURL + "/health")
		require.NoError(t, err)
		defer resp.Body.Close()

		assert.Equal(t, http.StatusOK, resp.StatusCode)
	})

	t.Run("创建然后查询用户", func(t *testing.T) {
		// 创建用户
		body := `{"name": "测试用户", "email": "test@example.com"}`
		resp, err := client.Post(baseURL+"/user", "application/json", strings.NewReader(body))
		require.NoError(t, err)
		defer resp.Body.Close()

		assert.Equal(t, http.StatusCreated, resp.StatusCode)

		var createResult Response
		json.NewDecoder(resp.Body).Decode(&createResult)
		userData := createResult.Data.(map[string]interface{})
		userID := int(userData["id"].(float64))

		// 查询刚创建的用户
		resp2, err := client.Get(fmt.Sprintf("%s/user?id=%d", baseURL, userID))
		require.NoError(t, err)
		defer resp2.Body.Close()

		assert.Equal(t, http.StatusOK, resp2.StatusCode)

		var getResult Response
		json.NewDecoder(resp2.Body).Decode(&getResult)
		getUserData := getResult.Data.(map[string]interface{})
		assert.Equal(t, "测试用户", getUserData["name"])
	})
}

// TestHTTPClient_WithTestServer 测试 HTTP 客户端逻辑
func TestHTTPClient_WithTestServer(t *testing.T) {
	// 创建一个模拟外部 API 的测试服务器
	externalAPI := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/api/data":
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusOK)
			fmt.Fprint(w, `{"result": "success", "items": [1,2,3]}`)
		case "/api/error":
			w.WriteHeader(http.StatusInternalServerError)
			fmt.Fprint(w, `{"error": "internal error"}`)
		case "/api/slow":
			time.Sleep(2 * time.Second)
			w.WriteHeader(http.StatusOK)
		default:
			w.WriteHeader(http.StatusNotFound)
		}
	}))
	defer externalAPI.Close()

	t.Run("正常请求", func(t *testing.T) {
		resp, err := http.Get(externalAPI.URL + "/api/data")
		require.NoError(t, err)
		defer resp.Body.Close()

		body, _ := io.ReadAll(resp.Body)
		assert.Contains(t, string(body), "success")
	})

	t.Run("服务器错误", func(t *testing.T) {
		resp, err := http.Get(externalAPI.URL + "/api/error")
		require.NoError(t, err)
		defer resp.Body.Close()

		assert.Equal(t, http.StatusInternalServerError, resp.StatusCode)
	})

	t.Run("客户端超时", func(t *testing.T) {
		client := &http.Client{Timeout: 100 * time.Millisecond}
		_, err := client.Get(externalAPI.URL + "/api/slow")
		assert.Error(t, err, "应该因超时而报错")
	})
}

// TestHTTPSServer TLS 测试服务器
func TestHTTPSServer(t *testing.T) {
	server := httptest.NewTLSServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("安全连接"))
	}))
	defer server.Close()

	// 使用服务器提供的 TLS 客户端
	client := server.Client()
	resp, err := client.Get(server.URL)
	require.NoError(t, err)
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	assert.Equal(t, "安全连接", string(body))
}
```

---

## 四、数据库测试

### 4.1 sqlmock 测试数据库交互

```bash
go get github.com/DATA-DOG/go-sqlmock
```

```go
// 文件: repository/user_repo.go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

// User 用户模型
type User struct {
	ID        int
	Name      string
	Email     string
	CreatedAt time.Time
}

// UserRepository 用户仓库
type UserRepository struct {
	db *sql.DB
}

// NewUserRepository 创建用户仓库
func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

// Create 创建用户
func (r *UserRepository) Create(ctx context.Context, user *User) error {
	query := `INSERT INTO users (name, email, created_at) VALUES (?, ?, ?)`
	result, err := r.db.ExecContext(ctx, query, user.Name, user.Email, time.Now())
	if err != nil {
		return fmt.Errorf("插入用户失败: %w", err)
	}
	id, err := result.LastInsertId()
	if err != nil {
		return fmt.Errorf("获取插入ID失败: %w", err)
	}
	user.ID = int(id)
	return nil
}

// GetByID 根据 ID 查询用户
func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
	query := `SELECT id, name, email, created_at FROM users WHERE id = ?`
	row := r.db.QueryRowContext(ctx, query, id)

	user := &User{}
	err := row.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("查询用户失败: %w", err)
	}
	return user, nil
}

// ListAll 列出所有用户
func (r *UserRepository) ListAll(ctx context.Context) ([]*User, error) {
	query := `SELECT id, name, email, created_at FROM users ORDER BY id`
	rows, err := r.db.QueryContext(ctx, query)
	if err != nil {
		return nil, fmt.Errorf("查询用户列表失败: %w", err)
	}
	defer rows.Close()

	var users []*User
	for rows.Next() {
		user := &User{}
		if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt); err != nil {
			return nil, fmt.Errorf("扫描用户行失败: %w", err)
		}
		users = append(users, user)
	}
	return users, rows.Err()
}

// Update 更新用户
func (r *UserRepository) Update(ctx context.Context, user *User) error {
	query := `UPDATE users SET name = ?, email = ? WHERE id = ?`
	result, err := r.db.ExecContext(ctx, query, user.Name, user.Email, user.ID)
	if err != nil {
		return fmt.Errorf("更新用户失败: %w", err)
	}
	rowsAffected, _ := result.RowsAffected()
	if rowsAffected == 0 {
		return fmt.Errorf("用户不存在: %d", user.ID)
	}
	return nil
}

// Delete 删除用户
func (r *UserRepository) Delete(ctx context.Context, id int) error {
	query := `DELETE FROM users WHERE id = ?`
	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return fmt.Errorf("删除用户失败: %w", err)
	}
	rowsAffected, _ := result.RowsAffected()
	if rowsAffected == 0 {
		return fmt.Errorf("用户不存在: %d", id)
	}
	return nil
}
```

```go
// 文件: repository/user_repo_test.go
package repository

import (
	"context"
	"testing"
	"time"

	"github.com/DATA-DOG/go-sqlmock"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestUserRepository_Create(t *testing.T) {
	// 创建 sqlmock
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()

	repo := NewUserRepository(db)
	ctx := context.Background()

	t.Run("成功创建", func(t *testing.T) {
		// 设置期望
		mock.ExpectExec("INSERT INTO users").
			WithArgs("张三", "zhangsan@test.com", sqlmock.AnyArg()). // AnyArg 匹配时间参数
			WillReturnResult(sqlmock.NewResult(1, 1))                 // LastInsertId=1, RowsAffected=1

		user := &User{Name: "张三", Email: "zhangsan@test.com"}
		err := repo.Create(ctx, user)

		assert.NoError(t, err)
		assert.Equal(t, 1, user.ID) // ID 应该被设置

		// 验证所有期望都被满足
		assert.NoError(t, mock.ExpectationsWereMet())
	})

	t.Run("数据库错误", func(t *testing.T) {
		mock.ExpectExec("INSERT INTO users").
			WillReturnError(fmt.Errorf("duplicate key"))

		user := &User{Name: "重复", Email: "dup@test.com"}
		err := repo.Create(ctx, user)

		assert.Error(t, err)
		assert.Contains(t, err.Error(), "插入用户失败")
		assert.NoError(t, mock.ExpectationsWereMet())
	})
}

func TestUserRepository_GetByID(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()

	repo := NewUserRepository(db)
	ctx := context.Background()
	now := time.Now()

	t.Run("找到用户", func(t *testing.T) {
		rows := sqlmock.NewRows([]string{"id", "name", "email", "created_at"}).
			AddRow(1, "张三", "zhangsan@test.com", now)

		mock.ExpectQuery("SELECT .+ FROM users WHERE id = ?").
			WithArgs(1).
			WillReturnRows(rows)

		user, err := repo.GetByID(ctx, 1)

		require.NoError(t, err)
		require.NotNil(t, user)
		assert.Equal(t, "张三", user.Name)
		assert.Equal(t, "zhangsan@test.com", user.Email)
		assert.NoError(t, mock.ExpectationsWereMet())
	})

	t.Run("用户不存在", func(t *testing.T) {
		rows := sqlmock.NewRows([]string{"id", "name", "email", "created_at"})
		// 空结果集

		mock.ExpectQuery("SELECT .+ FROM users WHERE id = ?").
			WithArgs(999).
			WillReturnRows(rows)

		user, err := repo.GetByID(ctx, 999)

		assert.NoError(t, err)
		assert.Nil(t, user)
		assert.NoError(t, mock.ExpectationsWereMet())
	})
}

func TestUserRepository_ListAll(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()

	repo := NewUserRepository(db)
	ctx := context.Background()
	now := time.Now()

	rows := sqlmock.NewRows([]string{"id", "name", "email", "created_at"}).
		AddRow(1, "张三", "a@test.com", now).
		AddRow(2, "李四", "b@test.com", now).
		AddRow(3, "王五", "c@test.com", now)

	mock.ExpectQuery("SELECT .+ FROM users ORDER BY id").
		WillReturnRows(rows)

	users, err := repo.ListAll(ctx)

	require.NoError(t, err)
	assert.Len(t, users, 3)
	assert.Equal(t, "张三", users[0].Name)
	assert.Equal(t, "李四", users[1].Name)
	assert.Equal(t, "王五", users[2].Name)
	assert.NoError(t, mock.ExpectationsWereMet())
}

func TestUserRepository_Update(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()

	repo := NewUserRepository(db)
	ctx := context.Background()

	t.Run("成功更新", func(t *testing.T) {
		mock.ExpectExec("UPDATE users SET").
			WithArgs("张三（新）", "new@test.com", 1).
			WillReturnResult(sqlmock.NewResult(0, 1))

		err := repo.Update(ctx, &User{ID: 1, Name: "张三（新）", Email: "new@test.com"})
		assert.NoError(t, err)
		assert.NoError(t, mock.ExpectationsWereMet())
	})

	t.Run("更新不存在的用户", func(t *testing.T) {
		mock.ExpectExec("UPDATE users SET").
			WithArgs("不存在", "no@test.com", 999).
			WillReturnResult(sqlmock.NewResult(0, 0)) // RowsAffected=0

		err := repo.Update(ctx, &User{ID: 999, Name: "不存在", Email: "no@test.com"})
		assert.Error(t, err)
		assert.Contains(t, err.Error(), "用户不存在")
		assert.NoError(t, mock.ExpectationsWereMet())
	})
}

func TestUserRepository_Delete(t *testing.T) {
	db, mock, err := sqlmock.New()
	require.NoError(t, err)
	defer db.Close()

	repo := NewUserRepository(db)
	ctx := context.Background()

	mock.ExpectExec("DELETE FROM users WHERE id = ?").
		WithArgs(1).
		WillReturnResult(sqlmock.NewResult(0, 1))

	err = repo.Delete(ctx, 1)
	assert.NoError(t, err)
	assert.NoError(t, mock.ExpectationsWereMet())
}
```

### 4.2 testcontainers-go 真实数据库容器测试

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

```go
// 文件: repository/user_repo_integration_test.go
//go:build integration

package repository

import (
	"context"
	"database/sql"
	"fmt"
	"testing"
	"time"

	_ "github.com/lib/pq"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
	"github.com/testcontainers/testcontainers-go/wait"
)

// setupPostgresContainer 启动 PostgreSQL 容器
func setupPostgresContainer(t *testing.T) (*sql.DB, func()) {
	ctx := context.Background()

	// 启动 PostgreSQL 容器
	pgContainer, err := postgres.Run(ctx,
		"postgres:16-alpine",
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).
				WithStartupTimeout(30*time.Second),
		),
	)
	require.NoError(t, err)

	// 获取连接字符串
	connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
	require.NoError(t, err)

	// 连接数据库
	db, err := sql.Open("postgres", connStr)
	require.NoError(t, err)

	// 创建表
	_, err = db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			name VARCHAR(100) NOT NULL,
			email VARCHAR(200) NOT NULL,
			created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
		)
	`)
	require.NoError(t, err)

	// 返回清理函数
	cleanup := func() {
		db.Close()
		pgContainer.Terminate(ctx)
	}

	return db, cleanup
}

func TestUserRepository_Integration(t *testing.T) {
	db, cleanup := setupPostgresContainer(t)
	defer cleanup()

	repo := NewUserRepository(db)
	ctx := context.Background()

	t.Run("完整CRUD流程", func(t *testing.T) {
		// Create
		user := &User{Name: "集成测试用户", Email: "integration@test.com"}
		err := repo.Create(ctx, user)
		require.NoError(t, err)
		assert.Greater(t, user.ID, 0)

		// Read
		found, err := repo.GetByID(ctx, user.ID)
		require.NoError(t, err)
		require.NotNil(t, found)
		assert.Equal(t, "集成测试用户", found.Name)
		assert.Equal(t, "integration@test.com", found.Email)

		// Update
		found.Name = "更新后的名称"
		err = repo.Update(ctx, found)
		require.NoError(t, err)

		updated, err := repo.GetByID(ctx, user.ID)
		require.NoError(t, err)
		assert.Equal(t, "更新后的名称", updated.Name)

		// Delete
		err = repo.Delete(ctx, user.ID)
		require.NoError(t, err)

		deleted, err := repo.GetByID(ctx, user.ID)
		require.NoError(t, err)
		assert.Nil(t, deleted)
	})

	t.Run("列出所有用户", func(t *testing.T) {
		// 插入多个用户
		for i := 0; i < 3; i++ {
			user := &User{
				Name:  fmt.Sprintf("用户%d", i),
				Email: fmt.Sprintf("user%d@test.com", i),
			}
			err := repo.Create(ctx, user)
			require.NoError(t, err)
		}

		users, err := repo.ListAll(ctx)
		require.NoError(t, err)
		assert.GreaterOrEqual(t, len(users), 3)
	})
}
```

```bash
# 运行集成测试（需要 Docker）
go test -tags=integration -v -timeout=120s ./repository/
```

---

## 五、集成测试组织与 Build Tag

### 5.1 使用 Build Tag 隔离集成测试

```go
// 文件: integration_test.go
//go:build integration

// 这个文件只在指定 -tags=integration 时才会编译和运行
package myapp

import "testing"

func TestExternalServiceIntegration(t *testing.T) {
	// 需要真实外部服务的测试
}
```

```go
// 文件: e2e_test.go
//go:build e2e

// 端到端测试
package myapp

import "testing"

func TestEndToEnd(t *testing.T) {
	// 需要完整环境的测试
}
```

```bash
# 只运行单元测试（默认不包含任何 tag）
go test ./...

# 运行集成测试
go test -tags=integration ./...

# 运行端到端测试
go test -tags=e2e ./...

# 运行所有测试
go test -tags="integration e2e" ./...
```

### 5.2 使用环境变量控制

```go
package myapp

import (
	"os"
	"testing"
)

func TestDatabaseIntegration(t *testing.T) {
	dbURL := os.Getenv("TEST_DATABASE_URL")
	if dbURL == "" {
		t.Skip("跳过数据库集成测试: 未设置 TEST_DATABASE_URL")
	}
	// 使用真实数据库进行测试
}

func TestRedisIntegration(t *testing.T) {
	redisURL := os.Getenv("TEST_REDIS_URL")
	if redisURL == "" {
		t.Skip("跳过 Redis 集成测试: 未设置 TEST_REDIS_URL")
	}
	// 使用真实 Redis 进行测试
}
```

### 5.3 使用 testing.Short() 控制

```go
package myapp

import "testing"

func TestQuickUnit(t *testing.T) {
	// 快速单元测试，总是运行
	result := 1 + 1
	if result != 2 {
		t.Error("数学坏了")
	}
}

func TestSlowIntegration(t *testing.T) {
	if testing.Short() {
		t.Skip("短模式跳过慢速测试")
	}
	// 耗时的集成测试
}
```

```bash
# 快速模式（跳过慢速测试）
go test -short ./...

# 正常模式（运行所有测试）
go test ./...
```

---

## 六、常见坑点

### 坑点 1：Mock 对象未验证期望

```go
func TestMockNotVerified(t *testing.T) {
	mockStore := new(MockUserStore)
	mockStore.On("FindByID", mock.Anything, 1).Return(&User{Name: "test"}, nil)

	// 错误：忘记调用 mockStore.AssertExpectations(t)
	// 即使 FindByID 没有被调用，测试也会通过

	// 正确：始终验证期望
	// mockStore.AssertExpectations(t)

	// 或者使用 t.Cleanup 自动验证
	t.Cleanup(func() {
		mockStore.AssertExpectations(t)
	})
}
```

### 坑点 2：sqlmock 的查询匹配

```go
func TestSqlmockQueryMatch(t *testing.T) {
	db, mock, _ := sqlmock.New()
	defer db.Close()

	// 坑：默认使用正则匹配，特殊字符需要转义
	// mock.ExpectQuery("SELECT * FROM users")  // * 在正则中是特殊字符！

	// 方案1：转义特殊字符
	mock.ExpectQuery(`SELECT \* FROM users`)

	// 方案2：使用 sqlmock.QueryMatcherEqual 精确匹配
	// db, mock, _ := sqlmock.New(sqlmock.QueryMatcherOption(sqlmock.QueryMatcherEqual))
}
```

### 坑点 3：httptest 服务器忘记关闭

```go
func TestHTTPServerLeak(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))
	// 错误：忘记 defer server.Close()
	// 会导致 goroutine 泄漏

	// 正确：
	defer server.Close()
	// 或使用 t.Cleanup
	t.Cleanup(server.Close)
}
```

### 坑点 4：testcontainers 超时配置

```go
func TestContainerTimeout(t *testing.T) {
	// 坑：默认超时可能太短，特别是第一次拉取镜像时
	// 解决：设置足够的超时时间

	// 在 go test 命令中：
	// go test -timeout=300s -tags=integration ./...

	// 在测试内部也要设置合理的上下文超时
	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()
	_ = ctx
}
```

### 坑点 5：接口设计太宽泛

```go
// 错误：接口太大，Mock 困难
type BadRepository interface {
	Create(ctx context.Context, entity interface{}) error
	Read(ctx context.Context, id interface{}) (interface{}, error)
	Update(ctx context.Context, entity interface{}) error
	Delete(ctx context.Context, id interface{}) error
	Query(ctx context.Context, filter interface{}) ([]interface{}, error)
	Count(ctx context.Context, filter interface{}) (int64, error)
	Transaction(ctx context.Context, fn func(tx interface{}) error) error
	// ... 更多方法
}

// 正确：接口应该小而精确（接口隔离原则）
type UserReader interface {
	GetByID(ctx context.Context, id int) (*User, error)
	ListAll(ctx context.Context) ([]*User, error)
}

type UserWriter interface {
	Create(ctx context.Context, user *User) error
	Update(ctx context.Context, user *User) error
	Delete(ctx context.Context, id int) error
}

// 需要同时读写时组合接口
type UserRepository interface {
	UserReader
	UserWriter
}
```

---

## 七、速查表

### Mock 方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 手写 Mock | 简单直接，无依赖 | 代码量大，接口变更需同步 | 小项目、简单接口 |
| gomock/mockgen | 自动生成，类型安全 | 需要额外工具 | 中大项目、接口多 |
| testify/mock | API 友好，社区广泛 | 运行时类型检查 | 中等项目 |
| 接口 + 内存实现 | 可复用，逻辑完整 | 实现成本较高 | 需要完整行为的场景 |

### testify 常用断言

```go
// 相等
assert.Equal(t, expected, actual)
assert.NotEqual(t, expected, actual)
assert.EqualValues(t, expected, actual)  // 类型转换后比较

// nil
assert.Nil(t, object)
assert.NotNil(t, object)

// 布尔
assert.True(t, condition)
assert.False(t, condition)

// 错误
assert.NoError(t, err)
assert.Error(t, err)
assert.ErrorIs(t, err, target)
assert.ErrorAs(t, err, &target)
assert.ErrorContains(t, err, "消息")

// 字符串
assert.Contains(t, str, substr)
assert.NotContains(t, str, substr)
assert.Regexp(t, pattern, str)

// 集合
assert.Len(t, collection, length)
assert.Empty(t, collection)
assert.NotEmpty(t, collection)
assert.ElementsMatch(t, expected, actual) // 忽略顺序

// 数值
assert.Greater(t, a, b)
assert.Less(t, a, b)
assert.InDelta(t, expected, actual, delta) // 浮点数

// Panic
assert.Panics(t, func() { panic("boom") })
assert.NotPanics(t, func() { /* 正常代码 */ })

// 时间
assert.WithinDuration(t, expected, actual, delta)
```

### 测试组织最佳实践

```
项目结构:
├── handler/
│   ├── user.go
│   ├── user_test.go              # 单元测试（手写 Mock）
│   └── user_integration_test.go  # 集成测试（build tag）
├── repository/
│   ├── user.go
│   ├── user_test.go              # 使用 sqlmock 的单元测试
│   └── user_integration_test.go  # 使用 testcontainers 的集成测试
├── service/
│   ├── user.go
│   ├── user_test.go              # 使用 gomock 的单元测试
│   └── mock_test.go              # gomock 生成的文件
└── testutil/
    └── helpers.go                # 共享的测试辅助函数

运行策略:
  开发时:    go test ./...              # 只跑快速单元测试
  CI 流水线: go test -tags=integration ./... -timeout=300s
  发布前:    go test -tags="integration e2e" ./... -timeout=600s
```
