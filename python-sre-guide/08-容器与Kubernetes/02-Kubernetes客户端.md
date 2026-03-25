# Kubernetes Python 客户端 — 用代码管理集群资源与构建自动化运维工具

## 1. 是什么 & 为什么用它

`kubernetes`（Python 客户端）是 Kubernetes 官方维护的 Python SDK，提供对 K8s API Server 的完整访问能力。
它可以执行 kubectl 能做的所有操作，并且支持 Watch 实时事件流。

**对 SRE 的价值：**
- 编写自定义的集群运维自动化脚本
- 构建 Pod 异常诊断、自动扩缩容等工具
- 开发自定义 Controller / Operator
- 批量管理多集群资源

## 2. 安装与环境

```bash
# 安装 kubernetes 客户端
pip install kubernetes

# 可选：用于 YAML 处理
pip install pyyaml

# 验证安装
python -c "import kubernetes; print(kubernetes.__version__)"
```

**环境要求：**
- Python >= 3.6
- kubernetes >= 28.1.0
- 有效的 kubeconfig 或 in-cluster 配置
- 对目标集群的 API 访问权限

## 3. 核心概念与原理

### 3.1 客户端架构

```
+-------------------+     HTTPS     +-------------------+     etcd
|  Python 脚本      |-------------->|  API Server       |----------->  集群状态
|  (kubernetes SDK) |   REST API    |  (kube-apiserver) |
+-------------------+               +-------------------+
        |                                    |
        |  认证方式：                        |  资源操作：
        |  - kubeconfig 文件                 |  - CRUD
        |  - ServiceAccount Token            |  - Watch
        |  - Bearer Token                    |  - Patch
        |  - Client Certificate              |  - Exec
        v                                    v
  ApiClient                            核心资源
  ├── CoreV1Api        (Pod/Service/ConfigMap/Secret/Namespace...)
  ├── AppsV1Api        (Deployment/StatefulSet/DaemonSet...)
  ├── BatchV1Api       (Job/CronJob)
  ├── NetworkingV1Api  (Ingress/NetworkPolicy)
  └── CustomObjectsApi (CRD 自定义资源)
```

### 3.2 API 分组

```
API 分组                  Python 类             典型资源
─────────────────────────────────────────────────────────────────
core/v1                   CoreV1Api             Pod, Service, ConfigMap, Secret,
                                                Namespace, PV, PVC, Node
apps/v1                   AppsV1Api             Deployment, StatefulSet,
                                                DaemonSet, ReplicaSet
batch/v1                  BatchV1Api            Job, CronJob
networking.k8s.io/v1      NetworkingV1Api       Ingress, NetworkPolicy
rbac.authorization.k8s.io RbacAuthorizationV1Api Role, ClusterRole, Binding
autoscaling/v1            AutoscalingV1Api      HPA
custom (any)              CustomObjectsApi      任意 CRD
```

### 3.3 认证方式

```
方式                  适用场景              配置方法
─────────────────────────────────────────────────────────
kubeconfig 文件       本地开发/运维跳板机    config.load_kube_config()
In-Cluster            Pod 内运行的程序       config.load_incluster_config()
Bearer Token          CI/CD / 服务账号      Configuration + api_key
Client Certificate    高安全要求             Configuration + ssl_ca_cert
```

## 4. 基础用法

### 4.1 认证配置

```python
"""
Kubernetes 客户端认证配置
"""
from kubernetes import client, config
import os

# === 方式一：从 kubeconfig 文件加载（本地开发/运维）===
def connect_via_kubeconfig():
    """使用 kubeconfig 文件连接集群"""
    # 默认读取 ~/.kube/config
    config.load_kube_config()

    # 指定 kubeconfig 路径和上下文
    # config.load_kube_config(
    #     config_file='/path/to/kubeconfig',
    #     context='my-cluster-context'
    # )

    v1 = client.CoreV1Api()
    return v1

# === 方式二：In-Cluster 配置（Pod 内运行）===
def connect_in_cluster():
    """在 K8s Pod 内自动使用 ServiceAccount 认证"""
    config.load_incluster_config()
    v1 = client.CoreV1Api()
    return v1

# === 方式三：Bearer Token 认证 ===
def connect_via_token():
    """使用 Bearer Token 直接连接"""
    configuration = client.Configuration()
    configuration.host = "https://k8s-api-server:6443"
    configuration.api_key = {
        'authorization': 'Bearer YOUR_TOKEN_HERE'
    }
    configuration.verify_ssl = True
    configuration.ssl_ca_cert = '/path/to/ca.crt'

    api_client = client.ApiClient(configuration)
    v1 = client.CoreV1Api(api_client)
    return v1

# === 自动检测连接方式 ===
def auto_connect():
    """自动检测：优先 in-cluster，回退到 kubeconfig"""
    try:
        config.load_incluster_config()
        print("使用 In-Cluster 配置")
    except config.ConfigException:
        config.load_kube_config()
        print("使用 kubeconfig 配置")

    return client.CoreV1Api()

# === 多集群管理 ===
def connect_multiple_clusters():
    """连接多个集群"""
    clusters = {}

    # 读取 kubeconfig 中的所有上下文
    contexts, active_context = config.list_kube_config_contexts()

    for ctx in contexts:
        ctx_name = ctx['name']
        api_client = config.new_client_from_config(context=ctx_name)
        clusters[ctx_name] = client.CoreV1Api(api_client)
        print(f"已连接集群：{ctx_name}")

    return clusters
```

### 4.2 Pod 操作

```python
"""
Pod 的增删改查操作
"""
from kubernetes import client, config
import yaml

config.load_kube_config()
v1 = client.CoreV1Api()

# === 列出 Pod ===
def list_pods(namespace='default', label_selector=None):
    """列出指定命名空间的 Pod"""
    pods = v1.list_namespaced_pod(
        namespace=namespace,
        label_selector=label_selector  # 例如 'app=nginx,env=prod'
    )

    for pod in pods.items:
        # 获取 Pod 状态
        phase = pod.status.phase
        ready = '0/0'
        if pod.status.container_statuses:
            total = len(pod.status.container_statuses)
            running = sum(1 for c in pod.status.container_statuses if c.ready)
            ready = f'{running}/{total}'

        # 获取重启次数
        restarts = sum(
            c.restart_count for c in (pod.status.container_statuses or [])
        )

        print(f"  {pod.metadata.name:50s} {ready:6s} {phase:12s} 重启:{restarts}")

    return pods.items

# === 获取所有命名空间的 Pod ===
def list_all_pods():
    """获取集群中所有 Pod"""
    pods = v1.list_pod_for_all_namespaces()
    for pod in pods.items:
        ns = pod.metadata.namespace
        name = pod.metadata.name
        phase = pod.status.phase
        print(f"  {ns:20s} {name:50s} {phase}")
    return pods.items

# === 创建 Pod ===
def create_pod(namespace='default'):
    """通过 Python 对象创建 Pod"""
    pod_manifest = client.V1Pod(
        api_version='v1',
        kind='Pod',
        metadata=client.V1ObjectMeta(
            name='debug-pod',
            namespace=namespace,
            labels={'app': 'debug', 'team': 'sre'}
        ),
        spec=client.V1PodSpec(
            containers=[
                client.V1Container(
                    name='debug',
                    image='busybox:latest',
                    command=['sleep', '3600'],
                    resources=client.V1ResourceRequirements(
                        requests={'cpu': '100m', 'memory': '128Mi'},
                        limits={'cpu': '200m', 'memory': '256Mi'}
                    )
                )
            ],
            restart_policy='Never'
        )
    )

    pod = v1.create_namespaced_pod(namespace=namespace, body=pod_manifest)
    print(f"Pod 已创建：{pod.metadata.name}")
    return pod

# === 从 YAML 创建 Pod ===
def create_pod_from_yaml(yaml_path, namespace='default'):
    """从 YAML 文件创建 Pod"""
    with open(yaml_path) as f:
        pod_manifest = yaml.safe_load(f)

    pod = v1.create_namespaced_pod(namespace=namespace, body=pod_manifest)
    print(f"Pod 已创建：{pod.metadata.name}")
    return pod

# === 获取 Pod 日志 ===
def get_pod_logs(name, namespace='default', container=None, tail_lines=100):
    """获取 Pod 日志"""
    logs = v1.read_namespaced_pod_log(
        name=name,
        namespace=namespace,
        container=container,          # 多容器 Pod 需指定
        tail_lines=tail_lines,        # 最后 N 行
        # since_seconds=3600,         # 最近 1 小时
        # previous=True,              # 上一次运行的日志（容器重启后）
    )
    return logs

# === 在 Pod 中执行命令 ===
def exec_in_pod(name, command, namespace='default', container=None):
    """在 Pod 中执行命令"""
    from kubernetes.stream import stream

    resp = stream(
        v1.connect_get_namespaced_pod_exec,
        name=name,
        namespace=namespace,
        container=container,
        command=command if isinstance(command, list) else ['/bin/sh', '-c', command],
        stderr=True,
        stdin=False,
        stdout=True,
        tty=False
    )
    return resp

# === 删除 Pod ===
def delete_pod(name, namespace='default'):
    """删除 Pod"""
    v1.delete_namespaced_pod(
        name=name,
        namespace=namespace,
        body=client.V1DeleteOptions(
            grace_period_seconds=30  # 优雅关闭等待时间
        )
    )
    print(f"Pod 已删除：{name}")
```

### 4.3 Deployment 操作

```python
"""
Deployment 管理：创建、扩缩容、更新、回滚
"""
from kubernetes import client, config

config.load_kube_config()
apps_v1 = client.AppsV1Api()

# === 创建 Deployment ===
def create_deployment(name, image, replicas=2, namespace='default'):
    """创建 Deployment"""
    deployment = client.V1Deployment(
        api_version='apps/v1',
        kind='Deployment',
        metadata=client.V1ObjectMeta(
            name=name,
            labels={'app': name}
        ),
        spec=client.V1DeploymentSpec(
            replicas=replicas,
            selector=client.V1LabelSelector(
                match_labels={'app': name}
            ),
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(labels={'app': name}),
                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name=name,
                            image=image,
                            ports=[client.V1ContainerPort(container_port=80)],
                            resources=client.V1ResourceRequirements(
                                requests={'cpu': '100m', 'memory': '128Mi'},
                                limits={'cpu': '500m', 'memory': '512Mi'}
                            ),
                            liveness_probe=client.V1Probe(
                                http_get=client.V1HTTPGetAction(
                                    path='/health',
                                    port=80
                                ),
                                initial_delay_seconds=10,
                                period_seconds=15
                            )
                        )
                    ]
                )
            ),
            strategy=client.V1DeploymentStrategy(
                type='RollingUpdate',
                rolling_update=client.V1RollingUpdateDeployment(
                    max_surge='25%',
                    max_unavailable='25%'
                )
            )
        )
    )

    result = apps_v1.create_namespaced_deployment(namespace=namespace, body=deployment)
    print(f"Deployment 已创建：{result.metadata.name}")
    return result

# === 列出 Deployment ===
def list_deployments(namespace='default'):
    """列出 Deployment"""
    deployments = apps_v1.list_namespaced_deployment(namespace=namespace)
    for d in deployments.items:
        ready = d.status.ready_replicas or 0
        desired = d.spec.replicas or 0
        print(f"  {d.metadata.name:40s} {ready}/{desired} ready")
    return deployments.items

# === 扩缩容 ===
def scale_deployment(name, replicas, namespace='default'):
    """调整 Deployment 副本数"""
    body = {'spec': {'replicas': replicas}}
    result = apps_v1.patch_namespaced_deployment_scale(
        name=name,
        namespace=namespace,
        body=body
    )
    print(f"Deployment {name} 已扩缩容到 {replicas} 副本")
    return result

# === 更新镜像（滚动更新）===
def update_image(name, container_name, new_image, namespace='default'):
    """更新 Deployment 的容器镜像"""
    body = {
        'spec': {
            'template': {
                'spec': {
                    'containers': [{
                        'name': container_name,
                        'image': new_image
                    }]
                }
            }
        }
    }
    result = apps_v1.patch_namespaced_deployment(
        name=name,
        namespace=namespace,
        body=body
    )
    print(f"Deployment {name} 镜像已更新为 {new_image}")
    return result

# === 查看回滚历史 ===
def get_deployment_history(name, namespace='default'):
    """获取 Deployment 的 ReplicaSet 历史"""
    v1 = client.CoreV1Api()
    rs_list = apps_v1.list_namespaced_replica_set(
        namespace=namespace,
        label_selector=f'app={name}'
    )

    for rs in sorted(rs_list.items, key=lambda x: x.metadata.creation_timestamp):
        revision = rs.metadata.annotations.get('deployment.kubernetes.io/revision', '?')
        image = rs.spec.template.spec.containers[0].image
        replicas = rs.status.replicas or 0
        print(f"  Revision {revision}: {image} (副本: {replicas})")

# === 删除 Deployment ===
def delete_deployment(name, namespace='default'):
    """删除 Deployment"""
    apps_v1.delete_namespaced_deployment(
        name=name,
        namespace=namespace,
        body=client.V1DeleteOptions(
            propagation_policy='Foreground'  # 级联删除 Pod
        )
    )
    print(f"Deployment {name} 已删除")
```

### 4.4 Service 和 ConfigMap

```python
"""
Service 和 ConfigMap 管理
"""
from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()

# === 创建 Service ===
def create_service(name, selector, port=80, target_port=8080, namespace='default'):
    """创建 ClusterIP Service"""
    service = client.V1Service(
        api_version='v1',
        kind='Service',
        metadata=client.V1ObjectMeta(name=name, labels={'app': name}),
        spec=client.V1ServiceSpec(
            selector=selector,  # 例如 {'app': 'myapp'}
            ports=[client.V1ServicePort(
                port=port,
                target_port=target_port,
                protocol='TCP'
            )],
            type='ClusterIP'
        )
    )
    result = v1.create_namespaced_service(namespace=namespace, body=service)
    print(f"Service 已创建：{result.metadata.name}")
    return result

# === 创建 ConfigMap ===
def create_configmap(name, data, namespace='default'):
    """创建 ConfigMap"""
    configmap = client.V1ConfigMap(
        api_version='v1',
        kind='ConfigMap',
        metadata=client.V1ObjectMeta(name=name),
        data=data  # 字典类型，如 {'config.yaml': 'key: value', 'app.conf': '...'}
    )
    result = v1.create_namespaced_config_map(namespace=namespace, body=configmap)
    print(f"ConfigMap 已创建：{result.metadata.name}")
    return result

# === 更新 ConfigMap ===
def update_configmap(name, data, namespace='default'):
    """更新 ConfigMap"""
    body = {'data': data}
    result = v1.patch_namespaced_config_map(name=name, namespace=namespace, body=body)
    print(f"ConfigMap 已更新：{result.metadata.name}")
    return result
```

## 5. 进阶用法

### 5.1 Watch 事件流

```python
"""
Watch：实时监听 Kubernetes 资源变化
"""
from kubernetes import client, config, watch
import threading

config.load_kube_config()
v1 = client.CoreV1Api()

def watch_pods(namespace='default', label_selector=None, timeout=300):
    """
    实时监听 Pod 事件

    参数：
        namespace: 命名空间
        label_selector: 标签选择器
        timeout: 超时时间（秒）
    """
    w = watch.Watch()

    print(f"开始监听 Pod 事件（命名空间: {namespace}）...")

    for event in w.stream(
        v1.list_namespaced_pod,
        namespace=namespace,
        label_selector=label_selector,
        timeout_seconds=timeout
    ):
        event_type = event['type']          # ADDED / MODIFIED / DELETED
        pod = event['object']               # V1Pod 对象
        name = pod.metadata.name
        phase = pod.status.phase

        print(f"[{event_type:10s}] {name:50s} {phase}")

        # 检测异常状态
        if pod.status.container_statuses:
            for cs in pod.status.container_statuses:
                if cs.state.waiting:
                    reason = cs.state.waiting.reason
                    if reason in ('CrashLoopBackOff', 'ImagePullBackOff',
                                 'ErrImagePull', 'OOMKilled'):
                        print(f"  ⚠ 异常状态：{name} -> {reason}")

def watch_events(namespace='default', timeout=300):
    """监听 Kubernetes 事件（Event 资源）"""
    w = watch.Watch()

    for event in w.stream(
        v1.list_namespaced_event,
        namespace=namespace,
        timeout_seconds=timeout
    ):
        ev = event['object']
        event_type = ev.type                # Normal / Warning
        reason = ev.reason
        message = ev.message
        involved = ev.involved_object

        if event_type == 'Warning':
            print(f"[WARNING] {involved.kind}/{involved.name}: {reason} - {message}")

# 后台监听
# thread = threading.Thread(target=watch_pods, daemon=True)
# thread.start()
```

### 5.2 自定义 CRD 操作

```python
"""
操作自定义资源定义（CRD）
"""
from kubernetes import client, config

config.load_kube_config()
custom_api = client.CustomObjectsApi()

# CRD 参数
GROUP = 'stable.example.com'
VERSION = 'v1'
PLURAL = 'crontabs'  # CRD 的复数名称

# === 创建自定义资源 ===
def create_custom_resource(name, spec, namespace='default'):
    """创建自定义资源实例"""
    body = {
        'apiVersion': f'{GROUP}/{VERSION}',
        'kind': 'CronTab',
        'metadata': {
            'name': name,
            'namespace': namespace
        },
        'spec': spec
    }

    result = custom_api.create_namespaced_custom_object(
        group=GROUP,
        version=VERSION,
        namespace=namespace,
        plural=PLURAL,
        body=body
    )
    print(f"CRD 资源已创建：{result['metadata']['name']}")
    return result

# === 列出自定义资源 ===
def list_custom_resources(namespace='default'):
    """列出所有自定义资源"""
    result = custom_api.list_namespaced_custom_object(
        group=GROUP,
        version=VERSION,
        namespace=namespace,
        plural=PLURAL
    )
    for item in result.get('items', []):
        print(f"  {item['metadata']['name']}: {item.get('spec', {})}")
    return result

# === 更新自定义资源 ===
def patch_custom_resource(name, patch, namespace='default'):
    """部分更新自定义资源"""
    result = custom_api.patch_namespaced_custom_object(
        group=GROUP,
        version=VERSION,
        namespace=namespace,
        plural=PLURAL,
        name=name,
        body=patch
    )
    return result

# === 删除自定义资源 ===
def delete_custom_resource(name, namespace='default'):
    """删除自定义资源"""
    custom_api.delete_namespaced_custom_object(
        group=GROUP,
        version=VERSION,
        namespace=namespace,
        plural=PLURAL,
        name=name
    )
    print(f"CRD 资源已删除：{name}")
```

### 5.3 Operator 开发简介

```python
"""
简单的 Kubernetes Operator 框架
使用 Watch 监听资源变化并自动响应
"""
from kubernetes import client, config, watch
import logging
import time

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger('simple-operator')

class SimpleOperator:
    """
    简易 Operator：监听 Deployment 变化并自动管理相关资源
    实际生产中推荐使用 kopf 或 operator-sdk 框架
    """

    def __init__(self):
        try:
            config.load_incluster_config()
        except config.ConfigException:
            config.load_kube_config()

        self.v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()
        self.managed_label = 'operator.example.com/managed'

    def run(self, namespace='default'):
        """运行 Operator 主循环"""
        logger.info(f"Operator 启动，监听命名空间：{namespace}")

        w = watch.Watch()
        while True:
            try:
                for event in w.stream(
                    self.apps_v1.list_namespaced_deployment,
                    namespace=namespace,
                    label_selector=self.managed_label,
                    timeout_seconds=300
                ):
                    self._handle_event(event, namespace)
            except Exception as e:
                logger.error(f"Watch 异常：{e}")
                time.sleep(5)

    def _handle_event(self, event, namespace):
        """处理资源事件"""
        event_type = event['type']
        deployment = event['object']
        name = deployment.metadata.name

        logger.info(f"事件：{event_type} Deployment/{name}")

        if event_type == 'ADDED':
            self._ensure_service(name, deployment, namespace)
            self._ensure_configmap(name, deployment, namespace)
        elif event_type == 'DELETED':
            self._cleanup_resources(name, namespace)

    def _ensure_service(self, name, deployment, namespace):
        """确保对应的 Service 存在"""
        try:
            self.v1.read_namespaced_service(name=name, namespace=namespace)
            logger.info(f"Service/{name} 已存在")
        except client.exceptions.ApiException as e:
            if e.status == 404:
                # 创建 Service
                labels = deployment.spec.selector.match_labels
                port = 80
                containers = deployment.spec.template.spec.containers
                if containers and containers[0].ports:
                    port = containers[0].ports[0].container_port

                service = client.V1Service(
                    metadata=client.V1ObjectMeta(name=name, labels=labels),
                    spec=client.V1ServiceSpec(
                        selector=labels,
                        ports=[client.V1ServicePort(port=port, target_port=port)]
                    )
                )
                self.v1.create_namespaced_service(namespace=namespace, body=service)
                logger.info(f"Service/{name} 已创建")

    def _ensure_configmap(self, name, deployment, namespace):
        """确保对应的 ConfigMap 存在"""
        cm_name = f"{name}-config"
        try:
            self.v1.read_namespaced_config_map(name=cm_name, namespace=namespace)
        except client.exceptions.ApiException as e:
            if e.status == 404:
                configmap = client.V1ConfigMap(
                    metadata=client.V1ObjectMeta(name=cm_name),
                    data={'config.yaml': f'app_name: {name}\nlog_level: info\n'}
                )
                self.v1.create_namespaced_config_map(namespace=namespace, body=configmap)
                logger.info(f"ConfigMap/{cm_name} 已创建")

    def _cleanup_resources(self, name, namespace):
        """清理关联资源"""
        # 删除 Service
        try:
            self.v1.delete_namespaced_service(name=name, namespace=namespace)
            logger.info(f"Service/{name} 已清理")
        except client.exceptions.ApiException:
            pass

        # 删除 ConfigMap
        try:
            self.v1.delete_namespaced_config_map(name=f"{name}-config", namespace=namespace)
            logger.info(f"ConfigMap/{name}-config 已清理")
        except client.exceptions.ApiException:
            pass
```

## 6. SRE 实战案例：K8s Pod 异常自动诊断工具

```python
"""
SRE 实战：Kubernetes Pod 异常自动诊断工具
功能：
  1. 扫描集群中所有异常 Pod
  2. 自动收集诊断信息（日志、事件、资源使用）
  3. 分析异常原因并给出修复建议
  4. 生成诊断报告
"""
from kubernetes import client, config, watch
import logging
import json
import time
from datetime import datetime, timezone
from collections import defaultdict

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger('pod-diagnostics')

class PodDiagnostics:
    """
    Pod 异常诊断工具
    自动检测异常 Pod 并收集诊断信息
    """

    # 已知的异常状态及其含义
    KNOWN_ISSUES = {
        'CrashLoopBackOff': {
            'severity': 'critical',
            'description': '容器反复崩溃重启',
            'suggestions': [
                '检查应用日志排查崩溃原因',
                '检查 liveness probe 配置是否过于严格',
                '检查应用启动依赖是否就绪',
                '检查资源限制是否过低导致 OOM',
            ]
        },
        'ImagePullBackOff': {
            'severity': 'high',
            'description': '镜像拉取失败',
            'suggestions': [
                '检查镜像名称和 tag 是否正确',
                '检查镜像仓库认证 (imagePullSecrets)',
                '检查节点网络是否能访问镜像仓库',
                '检查镜像仓库是否可用',
            ]
        },
        'ErrImagePull': {
            'severity': 'high',
            'description': '镜像拉取错误',
            'suggestions': [
                '检查镜像是否存在',
                '检查 imagePullSecrets 配置',
            ]
        },
        'OOMKilled': {
            'severity': 'critical',
            'description': '内存溢出被杀死',
            'suggestions': [
                '增加容器 memory limits',
                '优化应用内存使用',
                '检查是否存在内存泄漏',
                '考虑使用 VPA 自动调整资源',
            ]
        },
        'Pending': {
            'severity': 'medium',
            'description': 'Pod 无法调度',
            'suggestions': [
                '检查节点资源是否充足',
                '检查 nodeSelector / affinity 配置',
                '检查 PVC 是否绑定',
                '检查是否有 taints 阻止调度',
            ]
        },
        'Evicted': {
            'severity': 'high',
            'description': 'Pod 被驱逐',
            'suggestions': [
                '检查节点磁盘空间',
                '检查节点内存压力',
                '设置合理的资源 requests 和 limits',
                '配置 PodDisruptionBudget',
            ]
        },
        'CreateContainerConfigError': {
            'severity': 'high',
            'description': '容器配置错误',
            'suggestions': [
                '检查引用的 ConfigMap/Secret 是否存在',
                '检查环境变量引用是否正确',
                '检查挂载卷配置',
            ]
        },
    }

    def __init__(self):
        try:
            config.load_incluster_config()
        except config.ConfigException:
            config.load_kube_config()

        self.v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()

    def diagnose_cluster(self, namespaces=None):
        """
        诊断整个集群

        参数：
            namespaces: 要检查的命名空间列表（None=所有）

        返回：诊断报告
        """
        logger.info("开始集群 Pod 异常诊断...")

        if namespaces:
            pods = []
            for ns in namespaces:
                ns_pods = self.v1.list_namespaced_pod(namespace=ns)
                pods.extend(ns_pods.items)
        else:
            all_pods = self.v1.list_pod_for_all_namespaces()
            pods = all_pods.items

        logger.info(f"共发现 {len(pods)} 个 Pod")

        # 筛选异常 Pod
        abnormal_pods = self._find_abnormal_pods(pods)
        logger.info(f"发现 {len(abnormal_pods)} 个异常 Pod")

        # 逐个诊断
        diagnoses = []
        for pod in abnormal_pods:
            diagnosis = self._diagnose_pod(pod)
            diagnoses.append(diagnosis)

        # 生成报告
        report = self._generate_report(pods, diagnoses)
        return report

    def _find_abnormal_pods(self, pods):
        """筛选异常 Pod"""
        abnormal = []

        for pod in pods:
            is_abnormal = False
            ns = pod.metadata.namespace
            name = pod.metadata.name

            # 跳过系统命名空间
            if ns in ('kube-system', 'kube-public', 'kube-node-lease'):
                continue

            # 检查 Phase
            phase = pod.status.phase
            if phase in ('Failed', 'Unknown'):
                is_abnormal = True

            # 检查 Pending 超过 5 分钟
            if phase == 'Pending':
                created = pod.metadata.creation_timestamp
                if created:
                    age_minutes = (datetime.now(timezone.utc) - created).total_seconds() / 60
                    if age_minutes > 5:
                        is_abnormal = True

            # 检查容器状态
            if pod.status.container_statuses:
                for cs in pod.status.container_statuses:
                    # 等待状态异常
                    if cs.state.waiting:
                        reason = cs.state.waiting.reason
                        if reason in self.KNOWN_ISSUES:
                            is_abnormal = True

                    # 频繁重启
                    if cs.restart_count > 5:
                        is_abnormal = True

                    # 上次终止原因
                    if cs.last_state and cs.last_state.terminated:
                        if cs.last_state.terminated.reason == 'OOMKilled':
                            is_abnormal = True

            if is_abnormal:
                abnormal.append(pod)

        return abnormal

    def _diagnose_pod(self, pod):
        """
        诊断单个 Pod

        返回：诊断结果字典
        """
        ns = pod.metadata.namespace
        name = pod.metadata.name

        logger.info(f"诊断 Pod: {ns}/{name}")

        diagnosis = {
            'namespace': ns,
            'name': name,
            'phase': pod.status.phase,
            'node': pod.spec.node_name or 'unscheduled',
            'image': '',
            'issues': [],
            'events': [],
            'logs': '',
            'suggestions': [],
            'resource_info': {},
            'timestamp': datetime.now().isoformat()
        }

        # 获取镜像信息
        if pod.spec.containers:
            diagnosis['image'] = pod.spec.containers[0].image

        # 分析容器状态
        self._analyze_container_status(pod, diagnosis)

        # 收集事件
        self._collect_events(pod, diagnosis)

        # 收集日志（仅运行中或刚崩溃的容器）
        self._collect_logs(pod, diagnosis)

        # 收集资源信息
        self._collect_resource_info(pod, diagnosis)

        # 生成修复建议
        self._generate_suggestions(diagnosis)

        return diagnosis

    def _analyze_container_status(self, pod, diagnosis):
        """分析容器状态"""
        if not pod.status.container_statuses:
            diagnosis['issues'].append({
                'type': 'no_container_status',
                'severity': 'medium',
                'message': '无容器状态信息（可能处于初始化阶段）'
            })
            return

        for cs in pod.status.container_statuses:
            container_name = cs.name

            # 等待状态
            if cs.state.waiting:
                reason = cs.state.waiting.reason or 'Unknown'
                message = cs.state.waiting.message or ''
                known = self.KNOWN_ISSUES.get(reason, {})

                diagnosis['issues'].append({
                    'type': reason,
                    'severity': known.get('severity', 'medium'),
                    'container': container_name,
                    'message': message or known.get('description', reason),
                })

            # 终止状态
            if cs.state.terminated:
                exit_code = cs.state.terminated.exit_code
                reason = cs.state.terminated.reason or 'Unknown'

                diagnosis['issues'].append({
                    'type': f'terminated_{reason}',
                    'severity': 'high' if exit_code != 0 else 'info',
                    'container': container_name,
                    'message': f'容器已终止：{reason}（退出码: {exit_code}）',
                    'exit_code': exit_code,
                })

            # 高重启次数
            if cs.restart_count > 5:
                diagnosis['issues'].append({
                    'type': 'high_restart_count',
                    'severity': 'high',
                    'container': container_name,
                    'message': f'容器重启次数过高：{cs.restart_count} 次',
                    'restart_count': cs.restart_count,
                })

            # OOMKilled 历史
            if cs.last_state and cs.last_state.terminated:
                if cs.last_state.terminated.reason == 'OOMKilled':
                    diagnosis['issues'].append({
                        'type': 'OOMKilled',
                        'severity': 'critical',
                        'container': container_name,
                        'message': '容器曾因内存溢出被杀死',
                    })

    def _collect_events(self, pod, diagnosis):
        """收集 Pod 相关事件"""
        try:
            events = self.v1.list_namespaced_event(
                namespace=pod.metadata.namespace,
                field_selector=f'involvedObject.name={pod.metadata.name}'
            )

            for ev in events.items:
                diagnosis['events'].append({
                    'type': ev.type,
                    'reason': ev.reason,
                    'message': ev.message,
                    'count': ev.count or 1,
                    'last_seen': ev.last_timestamp.isoformat() if ev.last_timestamp else '',
                })

                # Warning 事件加入 issues
                if ev.type == 'Warning':
                    known = self.KNOWN_ISSUES.get(ev.reason, {})
                    if known:
                        diagnosis['issues'].append({
                            'type': ev.reason,
                            'severity': known.get('severity', 'medium'),
                            'message': ev.message,
                        })
        except Exception as e:
            logger.warning(f"获取事件失败：{e}")

    def _collect_logs(self, pod, diagnosis):
        """收集容器日志"""
        try:
            # 尝试获取当前日志
            logs = self.v1.read_namespaced_pod_log(
                name=pod.metadata.name,
                namespace=pod.metadata.namespace,
                tail_lines=50,
            )
            diagnosis['logs'] = logs

        except client.exceptions.ApiException:
            # 尝试获取上一次运行的日志
            try:
                logs = self.v1.read_namespaced_pod_log(
                    name=pod.metadata.name,
                    namespace=pod.metadata.namespace,
                    tail_lines=50,
                    previous=True,
                )
                diagnosis['logs'] = f"[上次运行日志]\n{logs}"
            except Exception:
                diagnosis['logs'] = '无法获取日志'

    def _collect_resource_info(self, pod, diagnosis):
        """收集资源配置信息"""
        if pod.spec.containers:
            container = pod.spec.containers[0]
            resources = container.resources
            if resources:
                diagnosis['resource_info'] = {
                    'requests': {
                        'cpu': resources.requests.get('cpu', 'N/A') if resources.requests else 'N/A',
                        'memory': resources.requests.get('memory', 'N/A') if resources.requests else 'N/A',
                    },
                    'limits': {
                        'cpu': resources.limits.get('cpu', 'N/A') if resources.limits else 'N/A',
                        'memory': resources.limits.get('memory', 'N/A') if resources.limits else 'N/A',
                    }
                }

    def _generate_suggestions(self, diagnosis):
        """根据问题生成修复建议"""
        seen_suggestions = set()

        for issue in diagnosis['issues']:
            issue_type = issue['type']
            known = self.KNOWN_ISSUES.get(issue_type, {})

            for suggestion in known.get('suggestions', []):
                if suggestion not in seen_suggestions:
                    diagnosis['suggestions'].append(suggestion)
                    seen_suggestions.add(suggestion)

        # 通用建议
        if not diagnosis['suggestions']:
            diagnosis['suggestions'].append('检查 Pod 事件和日志获取更多信息')
            diagnosis['suggestions'].append(f'kubectl describe pod {diagnosis["name"]} -n {diagnosis["namespace"]}')

    def _generate_report(self, all_pods, diagnoses):
        """生成诊断报告"""
        # 按严重度统计
        severity_count = defaultdict(int)
        issue_types = defaultdict(int)

        for diag in diagnoses:
            for issue in diag['issues']:
                severity_count[issue.get('severity', 'unknown')] += 1
                issue_types[issue['type']] += 1

        report = {
            'timestamp': datetime.now().isoformat(),
            'summary': {
                'total_pods': len(all_pods),
                'abnormal_pods': len(diagnoses),
                'severity': dict(severity_count),
                'issue_types': dict(issue_types),
            },
            'diagnoses': diagnoses,
        }

        # 打印报告摘要
        logger.info("=" * 70)
        logger.info("Pod 异常诊断报告")
        logger.info("=" * 70)
        logger.info(f"总 Pod 数：{report['summary']['total_pods']}")
        logger.info(f"异常 Pod 数：{report['summary']['abnormal_pods']}")
        logger.info(f"严重度分布：{dict(severity_count)}")
        logger.info(f"问题类型：{dict(issue_types)}")
        logger.info("-" * 70)

        for diag in diagnoses:
            logger.info(
                f"\n[{diag['namespace']}/{diag['name']}] "
                f"阶段: {diag['phase']} | 节点: {diag['node']}"
            )
            for issue in diag['issues']:
                logger.info(f"  [{issue['severity']:8s}] {issue['type']}: {issue['message']}")
            if diag['suggestions']:
                logger.info("  修复建议：")
                for s in diag['suggestions']:
                    logger.info(f"    - {s}")

        logger.info("=" * 70)

        return report


# ============================================================
# 主入口
# ============================================================

def main():
    """运行 Pod 诊断工具"""
    diagnostics = PodDiagnostics()

    # 诊断所有命名空间
    report = diagnostics.diagnose_cluster()

    # 或指定命名空间
    # report = diagnostics.diagnose_cluster(namespaces=['default', 'production'])

    # 保存报告到文件
    report_file = f'/tmp/pod_diagnosis_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
    with open(report_file, 'w') as f:
        json.dump(report, f, indent=2, default=str)
    logger.info(f"报告已保存：{report_file}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 403 Forbidden | RBAC 权限不足 | 检查 ServiceAccount 的 Role/ClusterRole |
| Watch 断开 | 超时或网络问题 | 捕获异常并重新建立 Watch |
| 资源版本冲突 | 并发修改同一资源 | 使用 resourceVersion 乐观锁 |
| kubeconfig 找不到 | 默认路径不存在 | 指定 config_file 参数 |
| API 版本不匹配 | K8s 集群版本与 SDK 不兼容 | 升级 SDK 或使用动态客户端 |
| 大量 Pod 列表 OOM | 一次性获取过多资源 | 使用 `limit` 和 `continue` 分页 |

### 7.2 最佳实践

```
[✓] 使用自动连接检测（先 in-cluster 再 kubeconfig）
[✓] Watch 操作要处理断开重连
[✓] 列出资源时使用 label_selector 过滤
[✓] 大量资源使用分页（limit + continue）
[✓] 更新操作优先使用 patch 而非 replace
[✓] 为 ServiceAccount 配置最小权限 RBAC
[✓] 操作前检查资源是否存在（捕获 404）
[✓] 写操作加入 dry-run 测试模式
[✓] 使用 field_selector 精确筛选
[✓] Operator 开发使用成熟框架（kopf / operator-sdk）
```

## 8. 速查表

```python
# === 安装 ===
# pip install kubernetes pyyaml

# === 连接 ===
from kubernetes import client, config
config.load_kube_config()                          # 本地
config.load_incluster_config()                     # Pod 内

# === 核心 API ===
v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()
batch_v1 = client.BatchV1Api()
custom_api = client.CustomObjectsApi()

# === Pod ===
v1.list_namespaced_pod(namespace='default')
v1.list_pod_for_all_namespaces()
v1.create_namespaced_pod(namespace, body)
v1.read_namespaced_pod(name, namespace)
v1.delete_namespaced_pod(name, namespace)
v1.read_namespaced_pod_log(name, namespace, tail_lines=100)

# === Deployment ===
apps_v1.list_namespaced_deployment(namespace)
apps_v1.create_namespaced_deployment(namespace, body)
apps_v1.patch_namespaced_deployment(name, namespace, body)
apps_v1.patch_namespaced_deployment_scale(name, namespace, body)
apps_v1.delete_namespaced_deployment(name, namespace)

# === Service / ConfigMap ===
v1.create_namespaced_service(namespace, body)
v1.create_namespaced_config_map(namespace, body)
v1.patch_namespaced_config_map(name, namespace, body)

# === Watch ===
from kubernetes import watch
w = watch.Watch()
for event in w.stream(v1.list_namespaced_pod, namespace='default', timeout_seconds=300):
    print(event['type'], event['object'].metadata.name)

# === Exec ===
from kubernetes.stream import stream
resp = stream(v1.connect_get_namespaced_pod_exec, name, namespace,
              command=['/bin/sh', '-c', 'ls'], stdout=True, stderr=True)

# === CRD ===
custom_api.list_namespaced_custom_object(group, version, namespace, plural)
custom_api.create_namespaced_custom_object(group, version, namespace, plural, body)

# === 错误处理 ===
from kubernetes.client.exceptions import ApiException
try:
    v1.read_namespaced_pod('name', 'namespace')
except ApiException as e:
    if e.status == 404:
        print("资源不存在")
```
