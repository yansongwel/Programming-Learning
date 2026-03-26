# Kubernetes Operator 开发完全指南

## 导读

Operator 模式是 Kubernetes 生态最强大的扩展机制之一。它将领域特定的运维知识编码为软件，让 Kubernetes 能够自动管理复杂的有状态应用。

简单来说：**CRD（自定义资源定义）+ Controller（控制器）= Operator**。

作为 SRE/DevOps 工程师，掌握 Operator 开发意味着你可以：
- 将复杂的运维流程自动化（数据库主从切换、证书轮转等）
- 用声明式 API 管理自定义应用
- 实现"自愈"能力，减少人工干预

本文基于 **Go 1.24**，使用 **kubebuilder** 框架，系统讲解 Operator 开发的完整流程，最后实现一个 **自定义 CronJob Operator**。

### 你将学到

- Operator 模式核心概念
- kubebuilder 安装与项目初始化
- CRD 定义（types.go、验证标记）
- Controller 实现（Reconcile 循环、SetupWithManager）
- controller-runtime 核心概念
- 状态管理（Status subresource）
- Finalizer 资源清理
- RBAC 权限标记
- Webhook 验证（Validating/Mutating）
- SRE 实战：自定义 CronJob Operator

---

## 一、Operator 模式概念

### 1.1 什么是 Operator

```
┌──────────────────────────────────────────────────────────┐
│                    Operator 模式                          │
│                                                          │
│  用户定义期望状态（CRD）                                   │
│       │                                                  │
│       ▼                                                  │
│  ┌──────────┐     ┌───────────┐     ┌──────────────┐    │
│  │   CRD    │────>│ Controller│────>│ 实际资源操作   │    │
│  │ (声明式)  │     │ (协调循环) │     │ (Pod/Service  │    │
│  │          │     │           │     │  /ConfigMap等) │    │
│  └──────────┘     └─────┬─────┘     └──────────────┘    │
│                         │                                │
│                    观察当前状态                             │
│                    计算差异                               │
│                    执行操作                               │
│                    更新状态                               │
│                                                          │
│  核心思想：声明式 + 协调循环（Reconcile Loop）              │
└──────────────────────────────────────────────────────────┘
```

### 1.2 Reconcile 循环

Operator 的核心是 **Reconcile 循环**，也叫 "Level-Triggered"（水平触发），而不是 "Edge-Triggered"（边沿触发）：

```
                    ┌──────────────┐
                    │  事件触发     │
                    │  (Create/    │
                    │  Update/     │
                    │  Delete)     │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Reconcile   │<─────────────┐
                    │  函数        │               │
                    └──────┬───────┘               │
                           │                       │
                    ┌──────▼───────┐               │
                    │ 获取 CR 当前  │               │
                    │ 状态          │               │
                    └──────┬───────┘               │
                           │                       │
                    ┌──────▼───────┐               │
                    │ 比较期望状态  │               │
                    │ vs 实际状态   │               │
                    └──────┬───────┘               │
                           │                       │
                    ┌──────▼───────┐     是        │
                    │ 有差异？     ├───────────────┘
                    └──────┬───────┘  (执行操作后重新协调)
                           │ 否
                    ┌──────▼───────┐
                    │ 更新 Status  │
                    │ 返回成功     │
                    └──────────────┘
```

---

## 二、kubebuilder 安装与初始化

### 2.1 安装 kubebuilder

```bash
# 下载并安装 kubebuilder
# 使用 Go install（推荐）
go install sigs.k8s.io/kubebuilder/cmd@latest

# 或者下载二进制文件
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# 验证安装
kubebuilder version
```

### 2.2 初始化项目

```bash
# 创建项目目录
mkdir cronjob-operator && cd cronjob-operator

# 初始化项目（指定 domain 和 repo）
kubebuilder init --domain sre.example.com --repo github.com/sre-team/cronjob-operator

# 项目结构：
# .
# ├── Dockerfile              -- 构建 Operator 镜像
# ├── Makefile                -- 构建/测试/部署命令
# ├── PROJECT                 -- kubebuilder 项目元数据
# ├── cmd/
# │   └── main.go             -- 入口文件
# ├── config/
# │   ├── default/             -- Kustomize 默认配置
# │   ├── manager/             -- Manager 部署配置
# │   ├── rbac/                -- RBAC 权限配置
# │   └── crd/                 -- CRD YAML 定义
# ├── internal/
# │   └── controller/          -- 控制器代码
# ├── api/
# │   └── v1/                  -- API 类型定义
# ├── go.mod
# └── go.sum
```

### 2.3 创建 API（CRD + Controller）

```bash
# 创建 API 资源
# --group: API 组名
# --version: 版本号
# --kind: 资源类型名
kubebuilder create api --group batch --version v1 --kind CronTask

# 选择：
# Create Resource [y/n]: y    -- 创建 CRD 类型定义
# Create Controller [y/n]: y  -- 创建 Controller

# 生成的文件：
# api/v1/crontask_types.go        -- CRD 类型定义（重点修改）
# internal/controller/crontask_controller.go  -- 控制器（重点修改）
# api/v1/zz_generated.deepcopy.go -- 自动生成的 DeepCopy 方法
```

---

## 三、CRD 定义（types.go）

### 3.1 完整的 CRD 类型定义

```go
// api/v1/crontask_types.go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// ============================================================
// CronTask Spec（期望状态）
// ============================================================

// CronTaskSpec 定义 CronTask 的期望状态
type CronTaskSpec struct {
	// Schedule 是 Cron 表达式，定义执行计划
	// 例如："*/5 * * * *" 表示每 5 分钟执行一次
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:Pattern=`^(\*|([0-9]|[1-5][0-9])(-([0-9]|[1-5][0-9]))?)(\/[0-9]+)?\s+(\*|([0-9]|1[0-9]|2[0-3])(-([0-9]|1[0-9]|2[0-3]))?)(\/[0-9]+)?\s+(\*|([1-9]|[12][0-9]|3[01])(-([1-9]|[12][0-9]|3[01]))?)(\/[0-9]+)?\s+(\*|([1-9]|1[0-2])(-([1-9]|1[0-2]))?)(\/[0-9]+)?\s+(\*|[0-6](-[0-6])?)(\/[0-9]+)?$`
	Schedule string `json:"schedule"`

	// Image 是要运行的容器镜像
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	Image string `json:"image"`

	// Command 是容器启动命令
	// +optional
	Command []string `json:"command,omitempty"`

	// Args 是命令参数
	// +optional
	Args []string `json:"args,omitempty"`

	// Suspend 暂停后续执行
	// +kubebuilder:default=false
	// +optional
	Suspend *bool `json:"suspend,omitempty"`

	// SuccessfulJobsHistoryLimit 保留的成功 Job 历史数量
	// +kubebuilder:validation:Minimum=0
	// +kubebuilder:default=3
	// +optional
	SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

	// FailedJobsHistoryLimit 保留的失败 Job 历史数量
	// +kubebuilder:validation:Minimum=0
	// +kubebuilder:default=1
	// +optional
	FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`

	// ConcurrencyPolicy 并发策略
	// +kubebuilder:validation:Enum=Allow;Forbid;Replace
	// +kubebuilder:default=Allow
	// +optional
	ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

	// RestartPolicy Pod 重启策略
	// +kubebuilder:validation:Enum=OnFailure;Never
	// +kubebuilder:default=OnFailure
	// +optional
	RestartPolicy string `json:"restartPolicy,omitempty"`

	// ActiveDeadlineSeconds 任务超时时间（秒）
	// +kubebuilder:validation:Minimum=1
	// +optional
	ActiveDeadlineSeconds *int64 `json:"activeDeadlineSeconds,omitempty"`

	// BackoffLimit 失败重试次数
	// +kubebuilder:validation:Minimum=0
	// +kubebuilder:default=3
	// +optional
	BackoffLimit *int32 `json:"backoffLimit,omitempty"`

	// Env 环境变量
	// +optional
	Env map[string]string `json:"env,omitempty"`

	// Resources 资源限制
	// +optional
	Resources *ResourceRequirements `json:"resources,omitempty"`
}

// ConcurrencyPolicy 并发策略类型
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
	// AllowConcurrent 允许并发执行
	AllowConcurrent ConcurrencyPolicy = "Allow"
	// ForbidConcurrent 如果有正在运行的任务则跳过
	ForbidConcurrent ConcurrencyPolicy = "Forbid"
	// ReplaceConcurrent 取消正在运行的任务，启动新任务
	ReplaceConcurrent ConcurrencyPolicy = "Replace"
)

// ResourceRequirements 资源需求
type ResourceRequirements struct {
	// CPU 请求，如 "100m"
	// +optional
	CPURequest string `json:"cpuRequest,omitempty"`
	// CPU 限制，如 "500m"
	// +optional
	CPULimit string `json:"cpuLimit,omitempty"`
	// 内存请求，如 "128Mi"
	// +optional
	MemoryRequest string `json:"memoryRequest,omitempty"`
	// 内存限制，如 "512Mi"
	// +optional
	MemoryLimit string `json:"memoryLimit,omitempty"`
}

// ============================================================
// CronTask Status（实际状态）
// ============================================================

// CronTaskStatus 定义 CronTask 的实际状态
type CronTaskStatus struct {
	// Active 当前正在运行的 Job 列表
	// +optional
	Active []ActiveJob `json:"active,omitempty"`

	// LastScheduleTime 最后一次调度时间
	// +optional
	LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`

	// LastSuccessfulTime 最后一次成功时间
	// +optional
	LastSuccessfulTime *metav1.Time `json:"lastSuccessfulTime,omitempty"`

	// TotalSuccessCount 累计成功次数
	// +optional
	TotalSuccessCount int32 `json:"totalSuccessCount,omitempty"`

	// TotalFailureCount 累计失败次数
	// +optional
	TotalFailureCount int32 `json:"totalFailureCount,omitempty"`

	// Phase 当前阶段
	// +kubebuilder:validation:Enum=Active;Suspended;Failed
	// +optional
	Phase CronTaskPhase `json:"phase,omitempty"`

	// Conditions 状态条件列表
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// CronTaskPhase 阶段类型
type CronTaskPhase string

const (
	CronTaskPhaseActive    CronTaskPhase = "Active"
	CronTaskPhaseSuspended CronTaskPhase = "Suspended"
	CronTaskPhaseFailed    CronTaskPhase = "Failed"
)

// ActiveJob 活跃的 Job 信息
type ActiveJob struct {
	// Name Job 名称
	Name string `json:"name"`
	// StartTime 启动时间
	StartTime metav1.Time `json:"startTime"`
}

// ============================================================
// CronTask 资源定义
// ============================================================

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=ct
// +kubebuilder:printcolumn:name="Schedule",type=string,JSONPath=`.spec.schedule`
// +kubebuilder:printcolumn:name="Suspend",type=boolean,JSONPath=`.spec.suspend`
// +kubebuilder:printcolumn:name="Active",type=integer,JSONPath=`.status.active`
// +kubebuilder:printcolumn:name="Last Schedule",type=date,JSONPath=`.status.lastScheduleTime`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

// CronTask 是自定义定时任务资源
type CronTask struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CronTaskSpec   `json:"spec,omitempty"`
	Status CronTaskStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronTaskList 包含 CronTask 列表
type CronTaskList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CronTask `json:"items"`
}

func init() {
	SchemeBuilder.Register(&CronTask{}, &CronTaskList{})
}
```

### 3.2 kubebuilder 验证标记速查

```go
// 常用验证标记一览

// 必填字段
// +kubebuilder:validation:Required

// 可选字段
// +optional

// 默认值
// +kubebuilder:default=3
// +kubebuilder:default="Allow"
// +kubebuilder:default=false

// 数值范围
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Maximum=100
// +kubebuilder:validation:ExclusiveMinimum=true

// 字符串验证
// +kubebuilder:validation:MinLength=1
// +kubebuilder:validation:MaxLength=256
// +kubebuilder:validation:Pattern=`^[a-zA-Z0-9]+$`
// +kubebuilder:validation:Format=date-time

// 枚举
// +kubebuilder:validation:Enum=Allow;Forbid;Replace

// 数组验证
// +kubebuilder:validation:MinItems=1
// +kubebuilder:validation:MaxItems=10
// +kubebuilder:validation:UniqueItems=true

// 资源定义标记
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas
// +kubebuilder:resource:shortName=ct,scope=Namespaced
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

// 字段废弃
// +kubebuilder:deprecatedversion:warning="v1beta1 is deprecated"
```

---

## 四、Controller 实现

### 4.1 完整的 Controller 代码

```go
// internal/controller/crontask_controller.go
package controller

import (
	"context"
	"fmt"
	"sort"
	"time"

	batchv1 "k8s.io/api/batch/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	cronv1 "github.com/sre-team/cronjob-operator/api/v1"
	"github.com/robfig/cron/v3"
)

const (
	// cronTaskFinalizer 用于资源清理
	cronTaskFinalizer = "batch.sre.example.com/finalizer"
	// jobOwnerKey 索引键
	jobOwnerKey = ".metadata.controller"
)

// CronTaskReconciler 协调 CronTask 资源
type CronTaskReconciler struct {
	client.Client
	Scheme *runtime.Scheme
	Clock  // 可注入的时钟（便于测试）
}

// Clock 时钟接口（便于单元测试注入 mock 时钟）
type Clock interface {
	Now() time.Time
}

// RealClock 真实时钟
type RealClock struct{}

func (RealClock) Now() time.Time { return time.Now() }

// +kubebuilder:rbac:groups=batch.sre.example.com,resources=crontasks,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.sre.example.com,resources=crontasks/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=batch.sre.example.com,resources=crontasks/finalizers,verbs=update
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch

// Reconcile 是核心协调函数
// 每当 CronTask 或其关联的 Job 发生变化时，都会触发此函数
func (r *CronTaskReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)
	logger.Info("开始协调", "crontask", req.NamespacedName)

	// ========== 步骤1: 获取 CronTask 资源 ==========
	var cronTask cronv1.CronTask
	if err := r.Get(ctx, req.NamespacedName, &cronTask); err != nil {
		if errors.IsNotFound(err) {
			// 资源已被删除，无需处理
			logger.Info("CronTask 已被删除")
			return ctrl.Result{}, nil
		}
		logger.Error(err, "获取 CronTask 失败")
		return ctrl.Result{}, err
	}

	// ========== 步骤2: 处理 Finalizer ==========
	if cronTask.ObjectMeta.DeletionTimestamp.IsZero() {
		// 资源未被删除，添加 Finalizer
		if !controllerutil.ContainsFinalizer(&cronTask, cronTaskFinalizer) {
			controllerutil.AddFinalizer(&cronTask, cronTaskFinalizer)
			if err := r.Update(ctx, &cronTask); err != nil {
				return ctrl.Result{}, err
			}
		}
	} else {
		// 资源正在被删除，执行清理逻辑
		if controllerutil.ContainsFinalizer(&cronTask, cronTaskFinalizer) {
			if err := r.cleanupResources(ctx, &cronTask); err != nil {
				return ctrl.Result{}, err
			}
			// 移除 Finalizer，允许 Kubernetes 完成删除
			controllerutil.RemoveFinalizer(&cronTask, cronTaskFinalizer)
			if err := r.Update(ctx, &cronTask); err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	}

	// ========== 步骤3: 获取所有关联的 Job ==========
	var childJobs batchv1.JobList
	if err := r.List(ctx, &childJobs,
		client.InNamespace(req.Namespace),
		client.MatchingFields{jobOwnerKey: req.Name},
	); err != nil {
		logger.Error(err, "获取关联 Job 列表失败")
		return ctrl.Result{}, err
	}

	// 分类 Job
	var activeJobs []*batchv1.Job
	var successfulJobs []*batchv1.Job
	var failedJobs []*batchv1.Job

	for i, job := range childJobs.Items {
		_, finishedType := isJobFinished(&job)
		switch finishedType {
		case "":
			activeJobs = append(activeJobs, &childJobs.Items[i])
		case "Complete":
			successfulJobs = append(successfulJobs, &childJobs.Items[i])
		case "Failed":
			failedJobs = append(failedJobs, &childJobs.Items[i])
		}
	}

	logger.Info("Job 状态统计",
		"active", len(activeJobs),
		"successful", len(successfulJobs),
		"failed", len(failedJobs))

	// ========== 步骤4: 更新 Status ==========
	cronTask.Status.Active = make([]cronv1.ActiveJob, 0, len(activeJobs))
	for _, job := range activeJobs {
		cronTask.Status.Active = append(cronTask.Status.Active, cronv1.ActiveJob{
			Name:      job.Name,
			StartTime: *job.Status.StartTime,
		})
	}
	cronTask.Status.TotalSuccessCount = int32(len(successfulJobs))
	cronTask.Status.TotalFailureCount = int32(len(failedJobs))

	// 设置 Phase
	if cronTask.Spec.Suspend != nil && *cronTask.Spec.Suspend {
		cronTask.Status.Phase = cronv1.CronTaskPhaseSuspended
	} else {
		cronTask.Status.Phase = cronv1.CronTaskPhaseActive
	}

	if err := r.Status().Update(ctx, &cronTask); err != nil {
		logger.Error(err, "更新 Status 失败")
		return ctrl.Result{}, err
	}

	// ========== 步骤5: 清理历史 Job ==========
	if err := r.cleanupOldJobs(ctx, &cronTask, successfulJobs, failedJobs); err != nil {
		logger.Error(err, "清理历史 Job 失败")
		return ctrl.Result{}, err
	}

	// ========== 步骤6: 检查是否暂停 ==========
	if cronTask.Spec.Suspend != nil && *cronTask.Spec.Suspend {
		logger.Info("CronTask 已暂停，跳过调度")
		return ctrl.Result{}, nil
	}

	// ========== 步骤7: 计算下次调度时间 ==========
	missedRun, nextRun, err := getNextSchedule(&cronTask, r.Now())
	if err != nil {
		logger.Error(err, "解析 Cron 表达式失败")
		return ctrl.Result{}, nil // 不重试，Cron 表达式错误需要用户修复
	}

	// 设置下次调度的重新排队时间
	scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())}
	logger.Info("调度信息", "now", r.Now(), "nextRun", nextRun, "missedRun", missedRun)

	// ========== 步骤8: 检查是否需要执行 ==========
	if missedRun.IsZero() {
		logger.Info("还未到调度时间")
		return scheduledResult, nil
	}

	// 处理并发策略
	switch cronTask.Spec.ConcurrencyPolicy {
	case cronv1.ForbidConcurrent:
		if len(activeJobs) > 0 {
			logger.Info("有正在运行的 Job，并发策略为 Forbid，跳过本次")
			return scheduledResult, nil
		}
	case cronv1.ReplaceConcurrent:
		for _, activeJob := range activeJobs {
			logger.Info("并发策略为 Replace，终止旧 Job", "job", activeJob.Name)
			if err := r.Delete(ctx, activeJob,
				client.PropagationPolicy(metav1.DeletePropagationBackground),
			); err != nil {
				logger.Error(err, "删除旧 Job 失败", "job", activeJob.Name)
				return ctrl.Result{}, err
			}
		}
	}

	// ========== 步骤9: 创建新 Job ==========
	job, err := r.constructJob(&cronTask, missedRun)
	if err != nil {
		logger.Error(err, "构造 Job 失败")
		return ctrl.Result{}, err
	}

	if err := r.Create(ctx, job); err != nil {
		logger.Error(err, "创建 Job 失败")
		return ctrl.Result{}, err
	}
	logger.Info("Job 创建成功", "job", job.Name)

	// 更新最后调度时间
	cronTask.Status.LastScheduleTime = &metav1.Time{Time: missedRun}
	if err := r.Status().Update(ctx, &cronTask); err != nil {
		logger.Error(err, "更新 LastScheduleTime 失败")
		return ctrl.Result{}, err
	}

	return scheduledResult, nil
}

// constructJob 根据 CronTask Spec 构造 Job
func (r *CronTaskReconciler) constructJob(
	cronTask *cronv1.CronTask,
	scheduledTime time.Time,
) (*batchv1.Job, error) {

	// 生成唯一的 Job 名称
	name := fmt.Sprintf("%s-%d", cronTask.Name, scheduledTime.Unix())

	// 构建环境变量
	var envVars []corev1.EnvVar
	for k, v := range cronTask.Spec.Env {
		envVars = append(envVars, corev1.EnvVar{Name: k, Value: v})
	}

	// 构建资源限制
	resourceReqs := corev1.ResourceRequirements{}
	if cronTask.Spec.Resources != nil {
		res := cronTask.Spec.Resources
		if res.CPURequest != "" || res.MemoryRequest != "" {
			resourceReqs.Requests = corev1.ResourceList{}
			if res.CPURequest != "" {
				resourceReqs.Requests[corev1.ResourceCPU] = resource.MustParse(res.CPURequest)
			}
			if res.MemoryRequest != "" {
				resourceReqs.Requests[corev1.ResourceMemory] = resource.MustParse(res.MemoryRequest)
			}
		}
		if res.CPULimit != "" || res.MemoryLimit != "" {
			resourceReqs.Limits = corev1.ResourceList{}
			if res.CPULimit != "" {
				resourceReqs.Limits[corev1.ResourceCPU] = resource.MustParse(res.CPULimit)
			}
			if res.MemoryLimit != "" {
				resourceReqs.Limits[corev1.ResourceMemory] = resource.MustParse(res.MemoryLimit)
			}
		}
	}

	// 重启策略
	restartPolicy := corev1.RestartPolicyOnFailure
	if cronTask.Spec.RestartPolicy == "Never" {
		restartPolicy = corev1.RestartPolicyNever
	}

	job := &batchv1.Job{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: cronTask.Namespace,
			Labels: map[string]string{
				"crontask.sre.example.com/name":     cronTask.Name,
				"crontask.sre.example.com/schedule": scheduledTime.Format("20060102-150405"),
			},
			Annotations: map[string]string{
				"crontask.sre.example.com/scheduled-at": scheduledTime.Format(time.RFC3339),
			},
		},
		Spec: batchv1.JobSpec{
			ActiveDeadlineSeconds: cronTask.Spec.ActiveDeadlineSeconds,
			BackoffLimit:          cronTask.Spec.BackoffLimit,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"crontask.sre.example.com/name": cronTask.Name,
					},
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:      "task",
							Image:     cronTask.Spec.Image,
							Command:   cronTask.Spec.Command,
							Args:      cronTask.Spec.Args,
							Env:       envVars,
							Resources: resourceReqs,
						},
					},
					RestartPolicy: restartPolicy,
				},
			},
		},
	}

	// 设置 Owner Reference（级联删除）
	if err := ctrl.SetControllerReference(cronTask, job, r.Scheme); err != nil {
		return nil, err
	}

	return job, nil
}

// cleanupResources Finalizer 清理逻辑
func (r *CronTaskReconciler) cleanupResources(
	ctx context.Context,
	cronTask *cronv1.CronTask,
) error {
	logger := log.FromContext(ctx)
	logger.Info("执行 Finalizer 清理", "crontask", cronTask.Name)

	// 删除所有关联的 Job
	var childJobs batchv1.JobList
	if err := r.List(ctx, &childJobs,
		client.InNamespace(cronTask.Namespace),
		client.MatchingFields{jobOwnerKey: cronTask.Name},
	); err != nil {
		return err
	}

	for _, job := range childJobs.Items {
		logger.Info("清理 Job", "job", job.Name)
		if err := r.Delete(ctx, &job,
			client.PropagationPolicy(metav1.DeletePropagationBackground),
		); err != nil && !errors.IsNotFound(err) {
			return err
		}
	}

	logger.Info("Finalizer 清理完成")
	return nil
}

// cleanupOldJobs 清理历史 Job
func (r *CronTaskReconciler) cleanupOldJobs(
	ctx context.Context,
	cronTask *cronv1.CronTask,
	successfulJobs, failedJobs []*batchv1.Job,
) error {
	logger := log.FromContext(ctx)

	// 按创建时间排序
	sort.Slice(successfulJobs, func(i, j int) bool {
		return successfulJobs[i].CreationTimestamp.Before(&successfulJobs[j].CreationTimestamp)
	})
	sort.Slice(failedJobs, func(i, j int) bool {
		return failedJobs[i].CreationTimestamp.Before(&failedJobs[j].CreationTimestamp)
	})

	// 清理多余的成功 Job
	successLimit := int32(3) // 默认值
	if cronTask.Spec.SuccessfulJobsHistoryLimit != nil {
		successLimit = *cronTask.Spec.SuccessfulJobsHistoryLimit
	}
	for i := 0; i < len(successfulJobs)-int(successLimit); i++ {
		logger.Info("清理旧成功 Job", "job", successfulJobs[i].Name)
		if err := r.Delete(ctx, successfulJobs[i],
			client.PropagationPolicy(metav1.DeletePropagationBackground),
		); err != nil && !errors.IsNotFound(err) {
			return err
		}
	}

	// 清理多余的失败 Job
	failLimit := int32(1) // 默认值
	if cronTask.Spec.FailedJobsHistoryLimit != nil {
		failLimit = *cronTask.Spec.FailedJobsHistoryLimit
	}
	for i := 0; i < len(failedJobs)-int(failLimit); i++ {
		logger.Info("清理旧失败 Job", "job", failedJobs[i].Name)
		if err := r.Delete(ctx, failedJobs[i],
			client.PropagationPolicy(metav1.DeletePropagationBackground),
		); err != nil && !errors.IsNotFound(err) {
			return err
		}
	}

	return nil
}

// SetupWithManager 注册 Controller 到 Manager
func (r *CronTaskReconciler) SetupWithManager(mgr ctrl.Manager) error {
	// 设置 Clock 默认值
	if r.Clock == nil {
		r.Clock = RealClock{}
	}

	// 创建索引：通过 Owner 查找 Job
	if err := mgr.GetFieldIndexer().IndexField(
		context.Background(),
		&batchv1.Job{},
		jobOwnerKey,
		func(obj client.Object) []string {
			job := obj.(*batchv1.Job)
			owner := metav1.GetControllerOf(job)
			if owner == nil {
				return nil
			}
			if owner.APIVersion != cronv1.GroupVersion.String() || owner.Kind != "CronTask" {
				return nil
			}
			return []string{owner.Name}
		},
	); err != nil {
		return err
	}

	return ctrl.NewControllerManagedBy(mgr).
		For(&cronv1.CronTask{}).    // 监听 CronTask 变化
		Owns(&batchv1.Job{}).        // 监听关联的 Job 变化
		Complete(r)
}

// ============================================================
// 辅助函数
// ============================================================

// isJobFinished 判断 Job 是否完成
func isJobFinished(job *batchv1.Job) (bool, string) {
	for _, c := range job.Status.Conditions {
		if (c.Type == batchv1.JobComplete || c.Type == batchv1.JobFailed) &&
			c.Status == corev1.ConditionTrue {
			return true, string(c.Type)
		}
	}
	return false, ""
}

// getNextSchedule 计算下次调度时间
func getNextSchedule(
	cronTask *cronv1.CronTask,
	now time.Time,
) (lastMissed time.Time, next time.Time, err error) {

	sched, err := cron.ParseStandard(cronTask.Spec.Schedule)
	if err != nil {
		return time.Time{}, time.Time{}, fmt.Errorf("解析 Cron 表达式失败: %w", err)
	}

	// 确定起始时间
	var earliestTime time.Time
	if cronTask.Status.LastScheduleTime != nil {
		earliestTime = cronTask.Status.LastScheduleTime.Time
	} else {
		earliestTime = cronTask.ObjectMeta.CreationTimestamp.Time
	}

	// 计算错过的调度和下次调度
	// 防止时间漏洞导致大量补跑
	const maxMissed = 100
	missedCount := 0

	for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
		lastMissed = t
		missedCount++
		if missedCount > maxMissed {
			// 错过太多次，可能是 Cron 表达式太频繁或者 Operator 长时间未运行
			break
		}
	}

	return lastMissed, sched.Next(now), nil
}
```

---

## 五、controller-runtime 核心概念

```go
// controller-runtime 核心组件说明

package main

import (
	"context"
	"os"

	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	batchv1 "github.com/sre-team/cronjob-operator/api/v1"
	"github.com/sre-team/cronjob-operator/internal/controller"
)

var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	// 注册 Kubernetes 内置类型
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))
	// 注册自定义 CRD 类型
	utilruntime.Must(batchv1.AddToScheme(scheme))
}

func main() {
	// 配置日志
	opts := zap.Options{Development: true}
	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

	// ============================
	// Manager（管理器）
	// ============================
	// Manager 是整个 Operator 的核心，负责：
	// 1. 管理所有 Controller 的生命周期
	// 2. 提供共享的 Client（带缓存）
	// 3. 提供共享的 Cache（Informer 缓存）
	// 4. 管理 Webhook 服务器
	// 5. Leader Election（高可用）
	// 6. 健康检查端点
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme: scheme,

		// Leader Election 配置（生产环境必须开启）
		LeaderElection:   true,
		LeaderElectionID: "crontask-operator-lock",

		// 健康检查端口
		HealthProbeBindAddress: ":8081",

		// Metrics 端口
		MetricsBindAddress: ":8080",

		// Webhook 端口
		// Port: 9443,

		// 缓存命名空间过滤（空表示所有命名空间）
		// Namespace: "production",
	})
	if err != nil {
		setupLog.Error(err, "创建 Manager 失败")
		os.Exit(1)
	}

	// ============================
	// 注册 Controller
	// ============================
	if err = (&controller.CronTaskReconciler{
		Client: mgr.GetClient(),   // 带缓存的客户端
		Scheme: mgr.GetScheme(),   // Scheme（类型注册表）
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "注册 CronTask Controller 失败")
		os.Exit(1)
	}

	// ============================
	// 健康检查
	// ============================
	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "设置健康检查失败")
		os.Exit(1)
	}
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "设置就绪检查失败")
		os.Exit(1)
	}

	// ============================
	// 启动 Manager
	// ============================
	setupLog.Info("启动 Manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "Manager 运行失败")
		os.Exit(1)
	}
}
```

### controller-runtime 核心概念表

| 概念 | 说明 |
|------|------|
| **Manager** | 管理所有 Controller，提供共享客户端、缓存、Leader Election |
| **Controller** | 监听特定资源变化，调用 Reconciler 处理 |
| **Reconciler** | 核心业务逻辑，实现 `Reconcile(ctx, req) (Result, error)` |
| **Client** | 带缓存的 Kubernetes 客户端（读操作走缓存，写操作直连 API Server） |
| **Cache** | 基于 Informer 的本地缓存，自动同步 |
| **Scheme** | 类型注册表，将 GVK 映射到 Go 类型 |
| **Result** | 协调结果：`{}`=成功，`{Requeue:true}`=重新排队，`{RequeueAfter:d}`=延时排队 |
| **Predicate** | 事件过滤器，控制哪些事件触发 Reconcile |

---

## 六、Webhook 验证

### 6.1 Validating Webhook

```go
// api/v1/crontask_webhook.go
package v1

import (
	"fmt"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"

	"github.com/robfig/cron/v3"
)

var crontasklog = logf.Log.WithName("crontask-resource")

// SetupWebhookWithManager 注册 Webhook
func (r *CronTask) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

// +kubebuilder:webhook:path=/mutate-batch-sre-example-com-v1-crontask,mutating=true,failurePolicy=fail,sideEffects=None,groups=batch.sre.example.com,resources=crontasks,verbs=create;update,versions=v1,name=mcrontask.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &CronTask{}

// Default 实现 Mutating Webhook（设置默认值）
func (r *CronTask) Default() {
	crontasklog.Info("设置默认值", "name", r.Name)

	// 设置默认并发策略
	if r.Spec.ConcurrencyPolicy == "" {
		r.Spec.ConcurrencyPolicy = AllowConcurrent
	}

	// 设置默认历史保留数
	if r.Spec.SuccessfulJobsHistoryLimit == nil {
		limit := int32(3)
		r.Spec.SuccessfulJobsHistoryLimit = &limit
	}
	if r.Spec.FailedJobsHistoryLimit == nil {
		limit := int32(1)
		r.Spec.FailedJobsHistoryLimit = &limit
	}

	// 设置默认重试次数
	if r.Spec.BackoffLimit == nil {
		limit := int32(3)
		r.Spec.BackoffLimit = &limit
	}

	// 设置默认暂停状态
	if r.Spec.Suspend == nil {
		suspend := false
		r.Spec.Suspend = &suspend
	}

	// 设置默认重启策略
	if r.Spec.RestartPolicy == "" {
		r.Spec.RestartPolicy = "OnFailure"
	}

	// 添加默认标签
	if r.Labels == nil {
		r.Labels = make(map[string]string)
	}
	r.Labels["app.kubernetes.io/managed-by"] = "cronjob-operator"
}

// +kubebuilder:webhook:path=/validate-batch-sre-example-com-v1-crontask,mutating=false,failurePolicy=fail,sideEffects=None,groups=batch.sre.example.com,resources=crontasks,verbs=create;update,versions=v1,name=vcrontask.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &CronTask{}

// ValidateCreate 创建时的验证
func (r *CronTask) ValidateCreate() (admission.Warnings, error) {
	crontasklog.Info("验证创建", "name", r.Name)
	return r.validateCronTask()
}

// ValidateUpdate 更新时的验证
func (r *CronTask) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
	crontasklog.Info("验证更新", "name", r.Name)
	return r.validateCronTask()
}

// ValidateDelete 删除时的验证
func (r *CronTask) ValidateDelete() (admission.Warnings, error) {
	crontasklog.Info("验证删除", "name", r.Name)
	return nil, nil // 允许删除
}

// validateCronTask 通用验证逻辑
func (r *CronTask) validateCronTask() (admission.Warnings, error) {
	var warnings admission.Warnings

	// 验证 Cron 表达式
	_, err := cron.ParseStandard(r.Spec.Schedule)
	if err != nil {
		return nil, fmt.Errorf("无效的 Cron 表达式 '%s': %v", r.Spec.Schedule, err)
	}

	// 验证镜像不为空
	if r.Spec.Image == "" {
		return nil, fmt.Errorf("镜像不能为空")
	}

	// 警告：使用 latest 标签
	if r.Spec.Image == "latest" || len(r.Spec.Image) > 0 && r.Spec.Image[len(r.Spec.Image)-7:] == ":latest" {
		warnings = append(warnings,
			"使用 :latest 标签可能导致不可预期的行为，建议使用固定版本标签")
	}

	// 验证超时时间
	if r.Spec.ActiveDeadlineSeconds != nil && *r.Spec.ActiveDeadlineSeconds < 1 {
		return nil, fmt.Errorf("activeDeadlineSeconds 必须大于 0")
	}

	return warnings, nil
}
```

---

## 七、CRD 示例 YAML

```yaml
# config/samples/batch_v1_crontask.yaml
apiVersion: batch.sre.example.com/v1
kind: CronTask
metadata:
  name: cleanup-logs
  namespace: default
  labels:
    team: sre
    env: production
spec:
  # 每天凌晨 2 点执行
  schedule: "0 2 * * *"
  image: busybox:1.36
  command: ["sh", "-c"]
  args:
    - |
      echo "开始清理日志..."
      find /var/log -name "*.log" -mtime +7 -delete
      echo "清理完成"
  # 不允许并发
  concurrencyPolicy: Forbid
  # 保留最近 5 次成功记录
  successfulJobsHistoryLimit: 5
  # 保留最近 3 次失败记录
  failedJobsHistoryLimit: 3
  # 超时 1 小时
  activeDeadlineSeconds: 3600
  # 失败重试 2 次
  backoffLimit: 2
  restartPolicy: OnFailure
  # 环境变量
  env:
    LOG_PATH: "/var/log"
    RETENTION_DAYS: "7"
  # 资源限制
  resources:
    cpuRequest: "100m"
    cpuLimit: "500m"
    memoryRequest: "128Mi"
    memoryLimit: "256Mi"
```

---

## 八、部署 Operator

```bash
# 1. 生成 CRD YAML
make manifests

# 2. 安装 CRD 到集群
make install

# 3. 本地运行（开发调试）
make run

# 4. 构建 Docker 镜像
make docker-build IMG=registry.example.com/cronjob-operator:v0.1.0

# 5. 推送镜像
make docker-push IMG=registry.example.com/cronjob-operator:v0.1.0

# 6. 部署到集群
make deploy IMG=registry.example.com/cronjob-operator:v0.1.0

# 7. 创建 CR 实例
kubectl apply -f config/samples/batch_v1_crontask.yaml

# 8. 查看资源
kubectl get crontasks   # 或简写 kubectl get ct
kubectl describe crontask cleanup-logs

# 9. 查看关联 Job
kubectl get jobs -l crontask.sre.example.com/name=cleanup-logs

# 10. 卸载
make undeploy
make uninstall
```

---

## 常见坑点

### 1. Reconcile 必须是幂等的

```go
// 坑：每次 Reconcile 都创建新资源
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 错误：不检查是否已存在就创建
    r.Create(ctx, newPod)
    // 会导致不断创建重复资源！
}

// 正确：先检查是否存在
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var existing corev1.Pod
    err := r.Get(ctx, types.NamespacedName{Name: "my-pod", Namespace: req.Namespace}, &existing)
    if errors.IsNotFound(err) {
        // 不存在才创建
        return ctrl.Result{}, r.Create(ctx, newPod)
    }
    // 已存在，检查是否需要更新
}
```

### 2. Status 更新冲突

```go
// 坑：在更新 Spec 后立即更新 Status 会冲突
cronTask.Spec.Suspend = &suspend
r.Update(ctx, &cronTask)
cronTask.Status.Phase = "Suspended"
r.Status().Update(ctx, &cronTask)  // 可能报版本冲突！

// 正确：重新获取最新版本后再更新
r.Update(ctx, &cronTask)
// 重新获取
r.Get(ctx, req.NamespacedName, &cronTask)
cronTask.Status.Phase = "Suspended"
r.Status().Update(ctx, &cronTask)
```

### 3. 忘记设置 Owner Reference

```go
// 坑：不设置 OwnerReference，删除 CronTask 时不会自动删除关联 Job
job := &batchv1.Job{...}
r.Create(ctx, job)

// 正确：设置 OwnerReference 实现级联删除
ctrl.SetControllerReference(cronTask, job, r.Scheme)
r.Create(ctx, job)
```

### 4. Reconcile 返回值误用

```go
// Result 含义：
ctrl.Result{}                        // 成功，不需要重新排队
ctrl.Result{Requeue: true}           // 立即重新排队
ctrl.Result{RequeueAfter: 30*time.Second}  // 30秒后重新排队

// 返回 error 会自动重新排队（指数退避）
return ctrl.Result{}, fmt.Errorf("...")

// 坑：同时设置 Requeue 和 error
return ctrl.Result{Requeue: true}, err  // error 优先，Requeue 被忽略
```

### 5. Finalizer 死锁

```go
// 坑：Finalizer 清理失败后不断重试，资源永远无法删除
func (r *MyReconciler) cleanup(ctx context.Context, obj *MyResource) error {
    // 如果外部资源已不存在，这里会报错
    return externalClient.Delete(obj.Status.ExternalID) // 404 错误
}

// 正确：忽略 NotFound 错误
func (r *MyReconciler) cleanup(ctx context.Context, obj *MyResource) error {
    err := externalClient.Delete(obj.Status.ExternalID)
    if isNotFound(err) {
        return nil // 已经不存在了，清理完成
    }
    return err
}
```

---

## 速查表

| 操作 | 命令/代码 |
|------|-----------|
| 初始化项目 | `kubebuilder init --domain example.com --repo ...` |
| 创建 API | `kubebuilder create api --group batch --version v1 --kind CronTask` |
| 创建 Webhook | `kubebuilder create webhook --group batch --version v1 --kind CronTask --defaulting --programmatic-validation` |
| 生成 CRD | `make manifests` |
| 生成 DeepCopy | `make generate` |
| 安装 CRD | `make install` |
| 本地运行 | `make run` |
| 构建镜像 | `make docker-build IMG=...` |
| 部署 | `make deploy IMG=...` |
| 获取 CR | `r.Get(ctx, req.NamespacedName, &obj)` |
| 创建资源 | `r.Create(ctx, &obj)` |
| 更新资源 | `r.Update(ctx, &obj)` |
| 更新状态 | `r.Status().Update(ctx, &obj)` |
| 删除资源 | `r.Delete(ctx, &obj)` |
| 设置 Owner | `ctrl.SetControllerReference(owner, owned, scheme)` |
| 成功返回 | `return ctrl.Result{}, nil` |
| 延时重试 | `return ctrl.Result{RequeueAfter: 30*time.Second}, nil` |
| 立即重试 | `return ctrl.Result{Requeue: true}, nil` |

---

## 小结

本文系统讲解了 Kubernetes Operator 开发的完整流程：

1. **Operator 模式**：CRD + Controller = Operator，核心是声明式 API + Reconcile 循环
2. **kubebuilder**：快速初始化项目、创建 API、生成 CRD 和代码
3. **CRD 定义**：使用 kubebuilder 标记实现字段验证、默认值、枚举等
4. **Controller**：Reconcile 循环的完整实现，包含创建/更新/删除/状态管理
5. **controller-runtime**：Manager/Controller/Client/Cache 协作机制
6. **Finalizer**：资源删除前的清理机制
7. **Webhook**：Mutating（默认值注入）和 Validating（验证）
8. **实战**：完整的 CronTask Operator，支持 Cron 调度、并发策略、历史清理等

Operator 是 SRE 构建平台化工具的利器，将复杂运维逻辑编码为 Kubernetes 原生控制器。
