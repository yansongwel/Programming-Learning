# Azure SDK for Python — SRE 深度实战指南

> **适用读者**：有运维经验的 SRE 工程师，希望通过 Python 自动化管理 Azure 云资源。
> **Python 版本**：3.9+
> **Azure SDK 版本**：azure-identity 1.x / azure-mgmt-* 系列最新稳定版

---

## 一、是什么 & 为什么用它

### 1.1 Azure SDK for Python 是什么

Azure SDK for Python 是微软官方提供的 Python 客户端库集合，覆盖 Azure 几乎所有服务。SDK 分为两大类：

- **管理面（Management Plane）**：`azure-mgmt-*` 系列，用于创建/删除/配置资源（对应 Azure Resource Manager API）。
- **数据面（Data Plane）**：如 `azure-storage-blob`、`azure-monitor-query`，用于操作资源内部数据。

### 1.2 SRE 为什么要用它

| 场景 | 手动操作 (Portal/CLI) | Python SDK |
|------|----------------------|------------|
| 批量管理 500+ VM | 逐个点击，耗时数小时 | 脚本 5 分钟完成 |
| 定期巡检未使用资源 | 容易遗漏，无法标准化 | 自动化 + 报告输出 |
| 成本分析与告警 | 依赖 Portal 图表 | 自定义维度聚合，集成告警 |
| 安全合规检查 | 人工逐条核对 | 代码化策略，CI/CD 集成 |
| 故障自愈 | 7x24 值班手动恢复 | 事件触发自动处置 |

---

## 二、安装与环境配置

### 2.1 安装依赖

```bash
# 创建虚拟环境
python3 -m venv azure-sre-env
source azure-sre-env/bin/activate

# 核心认证库
pip install azure-identity

# 资源管理
pip install azure-mgmt-resource
pip install azure-mgmt-compute
pip install azure-mgmt-network
pip install azure-mgmt-storage
pip install azure-mgmt-monitor

# 数据面操作
pip install azure-storage-blob
pip install azure-monitor-query

# 可选：成本管理
pip install azure-mgmt-costmanagement

# 工具库
pip install tabulate  # 表格输出
```

### 2.2 环境变量配置

```bash
# 服务主体认证所需（推荐写入 ~/.bashrc 或使用密钥管理工具）
export AZURE_TENANT_ID="你的租户ID"
export AZURE_CLIENT_ID="你的应用ID"
export AZURE_CLIENT_SECRET="你的客户端密钥"
export AZURE_SUBSCRIPTION_ID="你的订阅ID"
```

### 2.3 在 Azure 门户创建服务主体

```bash
# 使用 Azure CLI 创建（也可在门户操作）
az ad sp create-for-rbac \
  --name "sre-automation-sp" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>
```

---

## 三、核心概念与原理

### 3.1 Azure SDK 认证链架构

```
┌─────────────────────────────────────────────────────────────┐
│                  DefaultAzureCredential                     │
│  (按顺序尝试以下认证方式，第一个成功即返回)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │ Environment  │──▶│ Managed      │──▶│ Azure CLI      │  │
│  │ Credential   │   │ Identity     │   │ Credential     │  │
│  │              │   │              │   │                │  │
│  │ 环境变量认证  │   │ VM/AKS托管   │   │ az login 认证  │  │
│  └──────────────┘   │ 身份认证     │   └────────────────┘  │
│         │           └──────────────┘          │            │
│         ▼                  │                  ▼            │
│  ┌──────────────┐          ▼           ┌────────────────┐  │
│  │ VS Code      │   ┌──────────────┐   │ PowerShell     │  │
│  │ Credential   │   │ Azure        │   │ Credential     │  │
│  │              │   │ Developer    │   │                │  │
│  │ VS Code插件  │   │ CLI Cred     │   │ PS 模块认证    │  │
│  └──────────────┘   └──────────────┘   └────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 SDK 请求处理流程

```
                    Python 应用代码
                         │
                         ▼
              ┌─────────────────────┐
              │   Azure SDK Client  │
              │  (如 ComputeClient) │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   认证中间件        │
              │ (Token Acquisition) │
              └──────────┬──────────┘
                         │  Bearer Token
                         ▼
              ┌─────────────────────┐
              │   HTTP Pipeline     │
              │  ┌───────────────┐  │
              │  │ 重试策略      │  │
              │  │ (指数退避)    │  │
              │  ├───────────────┤  │
              │  │ 日志记录      │  │
              │  ├───────────────┤  │
              │  │ 请求/响应Hook │  │
              │  └───────────────┘  │
              └──────────┬──────────┘
                         │  HTTPS
                         ▼
              ┌─────────────────────┐
              │  Azure REST API     │
              │  management.azure   │
              │  .com               │
              └─────────────────────┘
```

### 3.3 资源层级模型

```
Azure 账户 (Account)
  └── 租户 (Tenant) ─── Azure AD
        └── 订阅 (Subscription) ─── 计费边界
              └── 资源组 (Resource Group) ─── 生命周期管理单元
                    ├── 虚拟机 (VM)
                    ├── 存储账户 (Storage Account)
                    │     └── Blob 容器 (Container)
                    │           └── Blob 对象
                    ├── 虚拟网络 (VNet)
                    │     └── 子网 (Subnet)
                    ├── 网络安全组 (NSG)
                    │     └── 安全规则 (Security Rules)
                    └── ... 其他资源
```

---

## 四、基础用法（逐行注释）

### 4.1 认证方式

```python
#!/usr/bin/env python3
"""
Azure SDK 认证方式演示
包含 DefaultAzureCredential 和 ClientSecretCredential 两种方式
"""

import os
from azure.identity import (
    DefaultAzureCredential,      # 默认凭据链，自动尝试多种认证
    ClientSecretCredential,      # 服务主体密钥认证
    ManagedIdentityCredential,   # 托管身份认证（VM/AKS内部使用）
)

# ============================================================
# 方式一：DefaultAzureCredential（推荐）
# 自动按优先级尝试多种认证方式，开发和生产环境通用
# ============================================================
def get_default_credential():
    """获取默认凭据，自动适配运行环境"""
    credential = DefaultAzureCredential(
        # 可选：排除不需要的认证方式以加快失败速度
        exclude_visual_studio_code_credential=True,
        exclude_powershell_credential=True,
    )
    return credential


# ============================================================
# 方式二：ClientSecretCredential（适合自动化流水线）
# 使用服务主体的 client_id + client_secret 认证
# ============================================================
def get_sp_credential():
    """通过服务主体密钥认证"""
    # 从环境变量读取敏感信息，切勿硬编码
    tenant_id = os.environ["AZURE_TENANT_ID"]
    client_id = os.environ["AZURE_CLIENT_ID"]
    client_secret = os.environ["AZURE_CLIENT_SECRET"]

    credential = ClientSecretCredential(
        tenant_id=tenant_id,
        client_id=client_id,
        client_secret=client_secret,
    )
    return credential


# ============================================================
# 方式三：ManagedIdentityCredential（Azure 内部资源推荐）
# 在 Azure VM / AKS / App Service 等环境中使用
# 无需管理密钥，由 Azure 自动轮转
# ============================================================
def get_managed_identity_credential():
    """获取托管身份凭据（仅在 Azure 环境内有效）"""
    credential = ManagedIdentityCredential(
        # 如果使用用户分配的托管身份，需指定 client_id
        # client_id="用户分配托管身份的client_id",
    )
    return credential


# ============================================================
# 验证凭据是否可用
# ============================================================
def verify_credential(credential):
    """验证凭据有效性，获取 token 测试"""
    try:
        # 请求 Azure 管理面的 token
        token = credential.get_token("https://management.azure.com/.default")
        print(f"[OK] 认证成功，Token 前20字符: {token.token[:20]}...")
        print(f"[OK] Token 过期时间戳: {token.expires_on}")
        return True
    except Exception as e:
        print(f"[FAIL] 认证失败: {e}")
        return False


if __name__ == "__main__":
    # 默认使用 DefaultAzureCredential
    cred = get_default_credential()
    verify_credential(cred)
```

### 4.2 VM 管理基础操作

```python
#!/usr/bin/env python3
"""
Azure VM 管理基础操作
包含：列表、启动、停止、获取详情
"""

import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
from azure.mgmt.compute.models import (
    VirtualMachine,
    HardwareProfile,
    StorageProfile,
    ImageReference,
    OSDisk,
    ManagedDiskParameters,
    OSProfile,
    NetworkProfile,
    NetworkInterfaceReference,
)

# 初始化客户端
SUBSCRIPTION_ID = os.environ["AZURE_SUBSCRIPTION_ID"]
credential = DefaultAzureCredential()
compute_client = ComputeManagementClient(credential, SUBSCRIPTION_ID)


def list_all_vms():
    """列出订阅下所有虚拟机及其状态"""
    print("=" * 70)
    print(f"{'资源组':<20} {'VM名称':<20} {'位置':<15} {'VM大小':<15}")
    print("=" * 70)

    # list_all() 返回订阅下所有 VM，跨资源组
    for vm in compute_client.virtual_machines.list_all():
        # 从 VM ID 中提取资源组名称
        rg_name = vm.id.split("/")[4]
        print(
            f"{rg_name:<20} "
            f"{vm.name:<20} "
            f"{vm.location:<15} "
            f"{vm.hardware_profile.vm_size:<15}"
        )


def get_vm_status(resource_group, vm_name):
    """获取 VM 的详细运行状态"""
    # instance_view 包含 VM 的运行时状态信息
    vm_instance = compute_client.virtual_machines.instance_view(
        resource_group_name=resource_group,
        vm_name=vm_name,
    )

    # 电源状态在 statuses 列表中，code 格式为 "PowerState/状态"
    for status in vm_instance.statuses:
        if status.code.startswith("PowerState"):
            power_state = status.code.split("/")[1]
            print(f"VM [{vm_name}] 电源状态: {power_state}")
            return power_state

    return "unknown"


def start_vm(resource_group, vm_name):
    """启动虚拟机（异步操作，返回 poller）"""
    print(f"正在启动 VM [{vm_name}]...")
    # begin_start 返回 LROPoller（长时间运行操作轮询器）
    poller = compute_client.virtual_machines.begin_start(
        resource_group_name=resource_group,
        vm_name=vm_name,
    )
    # result() 阻塞等待操作完成
    poller.result()
    print(f"VM [{vm_name}] 已成功启动")


def stop_vm(resource_group, vm_name, deallocate=True):
    """
    停止虚拟机
    deallocate=True: 解除分配（停止计费）
    deallocate=False: 仅停止操作系统（仍计费）
    """
    if deallocate:
        print(f"正在解除分配 VM [{vm_name}]（将停止计费）...")
        poller = compute_client.virtual_machines.begin_deallocate(
            resource_group_name=resource_group,
            vm_name=vm_name,
        )
    else:
        print(f"正在停止 VM [{vm_name}]（仍会产生费用）...")
        poller = compute_client.virtual_machines.begin_power_off(
            resource_group_name=resource_group,
            vm_name=vm_name,
        )

    poller.result()
    print(f"VM [{vm_name}] 操作完成")


def delete_vm(resource_group, vm_name):
    """删除虚拟机（注意：不会自动删除关联的磁盘、网卡等）"""
    print(f"[警告] 即将删除 VM [{vm_name}]，此操作不可逆！")
    poller = compute_client.virtual_machines.begin_delete(
        resource_group_name=resource_group,
        vm_name=vm_name,
    )
    poller.result()
    print(f"VM [{vm_name}] 已删除")


if __name__ == "__main__":
    list_all_vms()
```

### 4.3 创建虚拟机（完整流程）

```python
#!/usr/bin/env python3
"""
创建 Azure 虚拟机完整流程
依赖：VNet、Subnet、NIC、Public IP 均需提前创建或在此脚本中创建
"""

import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
from azure.mgmt.network import NetworkManagementClient
from azure.mgmt.resource import ResourceManagementClient

SUBSCRIPTION_ID = os.environ["AZURE_SUBSCRIPTION_ID"]
credential = DefaultAzureCredential()

# 初始化各服务客户端
resource_client = ResourceManagementClient(credential, SUBSCRIPTION_ID)
network_client = NetworkManagementClient(credential, SUBSCRIPTION_ID)
compute_client = ComputeManagementClient(credential, SUBSCRIPTION_ID)

# 配置参数
RESOURCE_GROUP = "sre-demo-rg"
LOCATION = "eastasia"           # 东亚区域
VNET_NAME = "sre-demo-vnet"
SUBNET_NAME = "sre-demo-subnet"
NIC_NAME = "sre-demo-nic"
IP_NAME = "sre-demo-ip"
VM_NAME = "sre-demo-vm"


def create_resource_group():
    """第一步：创建资源组"""
    print(f"创建资源组 [{RESOURCE_GROUP}]...")
    rg = resource_client.resource_groups.create_or_update(
        RESOURCE_GROUP,
        {"location": LOCATION, "tags": {"env": "demo", "team": "sre"}},
    )
    print(f"资源组已创建: {rg.name} ({rg.location})")
    return rg


def create_vnet_and_subnet():
    """第二步：创建虚拟网络和子网"""
    print(f"创建 VNet [{VNET_NAME}]...")
    poller = network_client.virtual_networks.begin_create_or_update(
        RESOURCE_GROUP,
        VNET_NAME,
        {
            "location": LOCATION,
            "address_space": {"address_prefixes": ["10.0.0.0/16"]},
            "subnets": [
                {"name": SUBNET_NAME, "address_prefix": "10.0.1.0/24"}
            ],
        },
    )
    vnet = poller.result()
    print(f"VNet 已创建: {vnet.name}，地址空间: 10.0.0.0/16")

    # 获取子网信息（后续创建 NIC 需要子网 ID）
    subnet = network_client.subnets.get(
        RESOURCE_GROUP, VNET_NAME, SUBNET_NAME
    )
    return subnet


def create_public_ip():
    """第三步：创建公网 IP"""
    print(f"创建公网 IP [{IP_NAME}]...")
    poller = network_client.public_ip_addresses.begin_create_or_update(
        RESOURCE_GROUP,
        IP_NAME,
        {
            "location": LOCATION,
            "sku": {"name": "Standard"},           # Standard SKU
            "public_ip_allocation_method": "Static", # 静态 IP
        },
    )
    ip = poller.result()
    print(f"公网 IP 已创建: {ip.ip_address}")
    return ip


def create_nic(subnet, public_ip):
    """第四步：创建网络接口卡"""
    print(f"创建 NIC [{NIC_NAME}]...")
    poller = network_client.network_interfaces.begin_create_or_update(
        RESOURCE_GROUP,
        NIC_NAME,
        {
            "location": LOCATION,
            "ip_configurations": [
                {
                    "name": "default",
                    "subnet": {"id": subnet.id},
                    "public_ip_address": {"id": public_ip.id},
                }
            ],
        },
    )
    nic = poller.result()
    print(f"NIC 已创建: {nic.id}")
    return nic


def create_vm(nic):
    """第五步：创建虚拟机"""
    print(f"创建 VM [{VM_NAME}]（这可能需要几分钟）...")
    poller = compute_client.virtual_machines.begin_create_or_update(
        RESOURCE_GROUP,
        VM_NAME,
        {
            "location": LOCATION,
            "tags": {
                "env": "demo",
                "team": "sre",
                "auto_shutdown": "true",   # 自定义标签：标记自动关机
            },
            # 硬件配置
            "hardware_profile": {"vm_size": "Standard_B2s"},
            # 操作系统配置
            "os_profile": {
                "computer_name": VM_NAME,
                "admin_username": "azuresre",
                "linux_configuration": {
                    "disable_password_authentication": True,
                    "ssh": {
                        "public_keys": [
                            {
                                "path": "/home/azuresre/.ssh/authorized_keys",
                                "key_data": "ssh-rsa AAAA...你的公钥...",
                            }
                        ]
                    },
                },
            },
            # 存储配置
            "storage_profile": {
                "image_reference": {
                    "publisher": "Canonical",
                    "offer": "0001-com-ubuntu-server-jammy",
                    "sku": "22_04-lts-gen2",
                    "version": "latest",
                },
                "os_disk": {
                    "name": f"{VM_NAME}-osdisk",
                    "caching": "ReadWrite",
                    "create_option": "FromImage",
                    "managed_disk": {"storage_account_type": "Premium_LRS"},
                    "disk_size_gb": 64,
                },
            },
            # 网络配置
            "network_profile": {
                "network_interfaces": [{"id": nic.id, "primary": True}]
            },
        },
    )

    vm = poller.result()
    print(f"VM 已创建: {vm.name} ({vm.hardware_profile.vm_size})")
    return vm


if __name__ == "__main__":
    create_resource_group()
    subnet = create_vnet_and_subnet()
    public_ip = create_public_ip()
    nic = create_nic(subnet, public_ip)
    create_vm(nic)
    print("\n全部资源创建完成！")
```

---

## 五、Blob Storage 操作

### 5.1 容器与 Blob 管理

```python
#!/usr/bin/env python3
"""
Azure Blob Storage 操作
包含：容器管理、文件上传/下载/列表、SAS Token 生成
"""

import os
import io
import json
from datetime import datetime, timedelta, timezone
from azure.identity import DefaultAzureCredential
from azure.storage.blob import (
    BlobServiceClient,           # Blob 服务顶层客户端
    ContainerClient,             # 容器级别客户端
    BlobClient,                  # 单个 Blob 操作客户端
    generate_blob_sas,           # 生成 Blob 级别 SAS
    BlobSasPermissions,          # SAS 权限枚举
    ContentSettings,             # 内容类型设置
)

# 存储账户 URL 格式：https://<account_name>.blob.core.windows.net
STORAGE_ACCOUNT_URL = "https://your_account.blob.core.windows.net"
CONTAINER_NAME = "sre-backup"

# 使用 DefaultAzureCredential 认证（需要存储账户的 RBAC 角色）
credential = DefaultAzureCredential()
blob_service_client = BlobServiceClient(
    account_url=STORAGE_ACCOUNT_URL,
    credential=credential,
)


def create_container():
    """创建 Blob 容器"""
    try:
        container_client = blob_service_client.create_container(
            CONTAINER_NAME,
            # 容器级别的元数据
            metadata={"team": "sre", "purpose": "backup"},
            # 公共访问级别：None(私有)/blob(Blob级别)/container(容器级别)
            public_access=None,
        )
        print(f"容器 [{CONTAINER_NAME}] 创建成功")
        return container_client
    except Exception as e:
        if "ContainerAlreadyExists" in str(e):
            print(f"容器 [{CONTAINER_NAME}] 已存在，跳过创建")
            return blob_service_client.get_container_client(CONTAINER_NAME)
        raise


def upload_blob(container_name, blob_name, data, content_type="application/octet-stream"):
    """
    上传数据到 Blob
    data: 可以是 bytes、str、文件流
    """
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=blob_name,
    )

    # 如果是字符串，转为 bytes
    if isinstance(data, str):
        data = data.encode("utf-8")

    blob_client.upload_blob(
        data,
        overwrite=True,  # 覆盖已有 Blob
        content_settings=ContentSettings(content_type=content_type),
        metadata={"uploaded_by": "sre-script", "timestamp": datetime.now().isoformat()},
    )
    print(f"已上传: {blob_name} ({len(data)} bytes)")
    return blob_client


def upload_file(container_name, blob_name, file_path):
    """从本地文件上传到 Blob"""
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=blob_name,
    )

    with open(file_path, "rb") as f:
        blob_client.upload_blob(f, overwrite=True)

    file_size = os.path.getsize(file_path)
    print(f"已上传文件: {file_path} -> {blob_name} ({file_size} bytes)")


def download_blob(container_name, blob_name):
    """下载 Blob 到内存"""
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=blob_name,
    )

    # download_blob() 返回 StorageStreamDownloader
    downloader = blob_client.download_blob()
    content = downloader.readall()
    print(f"已下载: {blob_name} ({len(content)} bytes)")
    return content


def download_blob_to_file(container_name, blob_name, file_path):
    """下载 Blob 到本地文件"""
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=blob_name,
    )

    with open(file_path, "wb") as f:
        downloader = blob_client.download_blob()
        # readinto() 流式写入文件，适合大文件
        downloader.readinto(f)

    print(f"已下载到文件: {blob_name} -> {file_path}")


def list_blobs(container_name, prefix=None):
    """
    列出容器中的所有 Blob
    prefix: 按前缀过滤（模拟目录结构）
    """
    container_client = blob_service_client.get_container_client(container_name)

    print(f"\n容器 [{container_name}] 中的 Blob 列表:")
    print(f"{'名称':<40} {'大小(bytes)':<15} {'最后修改':<25} {'层级':<10}")
    print("-" * 90)

    for blob in container_client.list_blobs(name_starts_with=prefix):
        print(
            f"{blob.name:<40} "
            f"{blob.size:<15} "
            f"{str(blob.last_modified):<25} "
            f"{blob.blob_tier or 'N/A':<10}"
        )


def delete_old_blobs(container_name, days_old=30):
    """删除指定天数前的旧 Blob（用于自动清理）"""
    container_client = blob_service_client.get_container_client(container_name)
    cutoff = datetime.now(timezone.utc) - timedelta(days=days_old)
    deleted_count = 0

    for blob in container_client.list_blobs():
        if blob.last_modified < cutoff:
            container_client.delete_blob(blob.name)
            print(f"已删除过期 Blob: {blob.name} (最后修改: {blob.last_modified})")
            deleted_count += 1

    print(f"\n共删除 {deleted_count} 个过期 Blob（>{days_old} 天）")


if __name__ == "__main__":
    # 创建容器
    create_container()

    # 上传示例数据
    report_data = json.dumps({"status": "ok", "check_time": datetime.now().isoformat()})
    upload_blob(CONTAINER_NAME, "reports/daily/health-check.json", report_data, "application/json")

    # 列出所有 Blob
    list_blobs(CONTAINER_NAME)
```

---

## 六、Azure Monitor 指标与日志查询

```python
#!/usr/bin/env python3
"""
Azure Monitor 指标查询与日志分析
用于 SRE 监控告警自动化
"""

import os
from datetime import datetime, timedelta, timezone
from azure.identity import DefaultAzureCredential
from azure.monitor.query import (
    MetricsQueryClient,    # 指标查询客户端
    LogsQueryClient,       # 日志（Log Analytics）查询客户端
    MetricsQueryGranularity,
    LogsQueryStatus,
)

SUBSCRIPTION_ID = os.environ["AZURE_SUBSCRIPTION_ID"]
credential = DefaultAzureCredential()

# 初始化查询客户端
metrics_client = MetricsQueryClient(credential)
logs_client = LogsQueryClient(credential)


def query_vm_cpu_metrics(resource_id, hours=4):
    """
    查询 VM 的 CPU 使用率指标
    resource_id: VM 的完整 Azure 资源 ID
    """
    end_time = datetime.now(timezone.utc)
    start_time = end_time - timedelta(hours=hours)

    # 查询 Percentage CPU 指标
    response = metrics_client.query_resource(
        resource_uri=resource_id,
        metric_names=["Percentage CPU"],
        timespan=(start_time, end_time),
        granularity=MetricsQueryGranularity.PT1H,  # 1小时粒度
        aggregations=["Average", "Maximum"],
    )

    print(f"\nVM CPU 使用率（过去 {hours} 小时）:")
    print(f"{'时间':<25} {'平均值(%)':<15} {'最大值(%)':<15}")
    print("-" * 55)

    for metric in response.metrics:
        for ts in metric.timeseries:
            for data_point in ts.data:
                if data_point.average is not None:
                    print(
                        f"{str(data_point.timestamp):<25} "
                        f"{data_point.average:<15.2f} "
                        f"{data_point.maximum or 0:<15.2f}"
                    )

    return response


def query_vm_disk_metrics(resource_id, hours=4):
    """查询 VM 磁盘 IOPS 和吞吐量"""
    end_time = datetime.now(timezone.utc)
    start_time = end_time - timedelta(hours=hours)

    response = metrics_client.query_resource(
        resource_uri=resource_id,
        metric_names=[
            "Disk Read Operations/Sec",
            "Disk Write Operations/Sec",
            "Disk Read Bytes",
            "Disk Write Bytes",
        ],
        timespan=(start_time, end_time),
        granularity=MetricsQueryGranularity.PT1H,
        aggregations=["Average"],
    )

    print(f"\nVM 磁盘指标（过去 {hours} 小时）:")
    for metric in response.metrics:
        print(f"\n  指标: {metric.name}")
        for ts in metric.timeseries:
            for dp in ts.data:
                if dp.average is not None:
                    print(f"    {dp.timestamp}: {dp.average:.2f}")

    return response


def query_logs(workspace_id, query, hours=24):
    """
    使用 KQL 查询 Log Analytics 工作区日志
    workspace_id: Log Analytics 工作区 ID
    query: KQL 查询语句
    """
    end_time = datetime.now(timezone.utc)
    start_time = end_time - timedelta(hours=hours)

    response = logs_client.query_workspace(
        workspace_id=workspace_id,
        query=query,
        timespan=(start_time, end_time),
    )

    if response.status == LogsQueryStatus.SUCCESS:
        print(f"\n日志查询结果（过去 {hours} 小时）:")
        for table in response.tables:
            # 打印列头
            headers = [col.name for col in table.columns]
            print(" | ".join(f"{h:<20}" for h in headers))
            print("-" * (22 * len(headers)))
            # 打印数据行
            for row in table.rows:
                print(" | ".join(f"{str(v):<20}" for v in row))
    elif response.status == LogsQueryStatus.PARTIAL:
        print(f"[警告] 查询返回部分结果: {response.partial_error}")
    else:
        print(f"[错误] 查询失败")

    return response


# 常用 KQL 查询模板
KQL_TEMPLATES = {
    # 查询最近的错误事件
    "recent_errors": """
        Syslog
        | where SeverityLevel == "err" or SeverityLevel == "crit"
        | summarize count() by Computer, SeverityLevel
        | order by count_ desc
        | take 20
    """,

    # VM 心跳丢失检测
    "heartbeat_missing": """
        Heartbeat
        | summarize LastHeartbeat = max(TimeGenerated) by Computer
        | where LastHeartbeat < ago(5m)
        | project Computer, LastHeartbeat, MinutesSinceLastBeat = datetime_diff("minute", now(), LastHeartbeat)
    """,

    # 磁盘使用率超过 85% 的主机
    "disk_usage_high": """
        Perf
        | where ObjectName == "Logical Disk" and CounterName == "% Used Space"
        | where InstanceName != "_Total"
        | summarize AvgUsage = avg(CounterValue) by Computer, InstanceName
        | where AvgUsage > 85
        | order by AvgUsage desc
    """,

    # 网络连接异常统计
    "network_anomalies": """
        VMConnection
        | where TimeGenerated > ago(1h)
        | where Direction == "inbound"
        | summarize
            TotalConnections = count(),
            FailedConnections = countif(LinksFailed > 0)
            by DestinationIp, DestinationPort
        | where FailedConnections > 10
        | order by FailedConnections desc
    """,
}


if __name__ == "__main__":
    # 示例：查询 VM CPU 指标
    # 资源 ID 格式示例
    vm_resource_id = (
        f"/subscriptions/{SUBSCRIPTION_ID}"
        "/resourceGroups/sre-demo-rg"
        "/providers/Microsoft.Compute/virtualMachines/sre-demo-vm"
    )
    query_vm_cpu_metrics(vm_resource_id)

    # 示例：日志查询（需要替换为实际的 workspace_id）
    # workspace_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    # query_logs(workspace_id, KQL_TEMPLATES["heartbeat_missing"])
```

---

## 七、Network 管理（VNet / NSG）

```python
#!/usr/bin/env python3
"""
Azure 网络资源管理
包含：VNet、Subnet、NSG 规则管理
"""

import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.network import NetworkManagementClient

SUBSCRIPTION_ID = os.environ["AZURE_SUBSCRIPTION_ID"]
credential = DefaultAzureCredential()
network_client = NetworkManagementClient(credential, SUBSCRIPTION_ID)


def list_vnets(resource_group=None):
    """列出虚拟网络"""
    print("\n虚拟网络列表:")
    print(f"{'资源组':<20} {'VNet名称':<25} {'地址空间':<20} {'子网数':<10}")
    print("=" * 75)

    if resource_group:
        vnets = network_client.virtual_networks.list(resource_group)
    else:
        vnets = network_client.virtual_networks.list_all()

    for vnet in vnets:
        rg = vnet.id.split("/")[4]
        addr_space = ", ".join(vnet.address_space.address_prefixes)
        subnet_count = len(vnet.subnets) if vnet.subnets else 0
        print(f"{rg:<20} {vnet.name:<25} {addr_space:<20} {subnet_count:<10}")


def create_nsg(resource_group, nsg_name, location):
    """创建网络安全组"""
    print(f"创建 NSG [{nsg_name}]...")
    poller = network_client.network_security_groups.begin_create_or_update(
        resource_group,
        nsg_name,
        {
            "location": location,
            "tags": {"team": "sre", "managed_by": "python-sdk"},
        },
    )
    nsg = poller.result()
    print(f"NSG 已创建: {nsg.name}")
    return nsg


def add_nsg_rule(resource_group, nsg_name, rule_name, rule_config):
    """
    添加 NSG 安全规则
    rule_config 示例: {
        "priority": 100,
        "direction": "Inbound",
        "access": "Allow",
        "protocol": "Tcp",
        "source_address_prefix": "10.0.0.0/8",
        "source_port_range": "*",
        "destination_address_prefix": "*",
        "destination_port_range": "22",
    }
    """
    print(f"添加安全规则 [{rule_name}] 到 NSG [{nsg_name}]...")
    poller = network_client.security_rules.begin_create_or_update(
        resource_group,
        nsg_name,
        rule_name,
        rule_config,
    )
    rule = poller.result()
    print(f"规则已添加: {rule.name} (优先级: {rule.priority}, {rule.access})")
    return rule


def audit_nsg_rules(resource_group=None):
    """
    审计 NSG 规则 — 检查危险配置
    检测项：
    1. 入站规则允许 0.0.0.0/0 访问敏感端口
    2. 优先级冲突
    3. 过于宽泛的规则
    """
    # 敏感端口列表
    sensitive_ports = {"22", "3389", "3306", "5432", "27017", "6379", "9200"}

    if resource_group:
        nsgs = network_client.network_security_groups.list(resource_group)
    else:
        nsgs = network_client.network_security_groups.list_all()

    findings = []

    for nsg in nsgs:
        rg = nsg.id.split("/")[4]
        all_rules = list(nsg.security_rules or [])

        for rule in all_rules:
            # 仅检查入站允许规则
            if rule.direction != "Inbound" or rule.access != "Allow":
                continue

            source = rule.source_address_prefix or ""
            dest_port = rule.destination_port_range or ""

            # 检查1：源地址为任意(0.0.0.0/0 或 * 或 Internet)
            is_open_source = source in ("*", "0.0.0.0/0", "Internet")

            # 检查2：目标端口是否为敏感端口或全部端口
            is_sensitive_port = dest_port in sensitive_ports or dest_port == "*"

            if is_open_source and is_sensitive_port:
                finding = {
                    "severity": "HIGH",
                    "nsg": nsg.name,
                    "resource_group": rg,
                    "rule": rule.name,
                    "source": source,
                    "port": dest_port,
                    "message": f"NSG [{nsg.name}] 规则 [{rule.name}] "
                               f"允许来自 [{source}] 访问端口 [{dest_port}]",
                }
                findings.append(finding)
                print(f"  [HIGH] {finding['message']}")

    print(f"\n审计完成，共发现 {len(findings)} 个高风险规则")
    return findings


# SRE 常用 NSG 规则模板
NSG_RULE_TEMPLATES = {
    # 仅允许内网 SSH
    "allow_internal_ssh": {
        "priority": 100,
        "direction": "Inbound",
        "access": "Allow",
        "protocol": "Tcp",
        "source_address_prefix": "10.0.0.0/8",
        "source_port_range": "*",
        "destination_address_prefix": "*",
        "destination_port_range": "22",
    },
    # 拒绝公网 SSH
    "deny_public_ssh": {
        "priority": 200,
        "direction": "Inbound",
        "access": "Deny",
        "protocol": "Tcp",
        "source_address_prefix": "Internet",
        "source_port_range": "*",
        "destination_address_prefix": "*",
        "destination_port_range": "22",
    },
    # 允许 HTTPS
    "allow_https": {
        "priority": 110,
        "direction": "Inbound",
        "access": "Allow",
        "protocol": "Tcp",
        "source_address_prefix": "*",
        "source_port_range": "*",
        "destination_address_prefix": "*",
        "destination_port_range": "443",
    },
}


if __name__ == "__main__":
    list_vnets()
    print("\n--- NSG 安全审计 ---")
    audit_nsg_rules()
```

---

## 八、Resource Management（资源组 / 订阅 / 标签）

```python
#!/usr/bin/env python3
"""
Azure 资源管理操作
包含：资源组管理、资源标签管理、订阅信息查询
"""

import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient, SubscriptionClient

SUBSCRIPTION_ID = os.environ["AZURE_SUBSCRIPTION_ID"]
credential = DefaultAzureCredential()

resource_client = ResourceManagementClient(credential, SUBSCRIPTION_ID)
subscription_client = SubscriptionClient(credential)


def list_subscriptions():
    """列出当前凭据可访问的所有订阅"""
    print("\n可访问的 Azure 订阅:")
    print(f"{'订阅ID':<40} {'名称':<30} {'状态':<12}")
    print("=" * 82)
    for sub in subscription_client.subscriptions.list():
        print(f"{sub.subscription_id:<40} {sub.display_name:<30} {sub.state:<12}")


def list_resource_groups():
    """列出当前订阅下的所有资源组"""
    print(f"\n订阅 [{SUBSCRIPTION_ID}] 下的资源组:")
    print(f"{'名称':<30} {'位置':<15} {'标签':<40}")
    print("=" * 85)
    for rg in resource_client.resource_groups.list():
        tags_str = str(rg.tags) if rg.tags else "无标签"
        # 截断过长的标签字符串
        if len(tags_str) > 38:
            tags_str = tags_str[:35] + "..."
        print(f"{rg.name:<30} {rg.location:<15} {tags_str:<40}")


def list_resources_in_group(resource_group):
    """列出资源组中的所有资源"""
    print(f"\n资源组 [{resource_group}] 中的资源:")
    print(f"{'资源名称':<30} {'类型':<45} {'位置':<15}")
    print("=" * 90)
    for resource in resource_client.resources.list_by_resource_group(resource_group):
        # 简化资源类型显示
        res_type = resource.type.replace("Microsoft.", "")
        print(f"{resource.name:<30} {res_type:<45} {resource.location:<15}")


def find_untagged_resources():
    """查找缺少必要标签的资源（合规检查）"""
    required_tags = {"env", "team", "owner"}
    non_compliant = []

    print("\n检查资源标签合规性...")
    print(f"必要标签: {required_tags}")
    print("-" * 80)

    for resource in resource_client.resources.list():
        resource_tags = set(resource.tags.keys()) if resource.tags else set()
        missing = required_tags - resource_tags

        if missing:
            non_compliant.append({
                "name": resource.name,
                "type": resource.type,
                "resource_group": resource.id.split("/")[4],
                "missing_tags": missing,
            })

    # 输出不合规资源
    if non_compliant:
        print(f"\n发现 {len(non_compliant)} 个不合规资源:")
        for item in non_compliant:
            print(
                f"  [{item['resource_group']}] {item['name']} "
                f"({item['type']}) - 缺少标签: {item['missing_tags']}"
            )
    else:
        print("所有资源标签合规！")

    return non_compliant


def batch_tag_resources(resource_group, tags_to_add):
    """批量为资源组中的所有资源添加标签"""
    print(f"\n为资源组 [{resource_group}] 中的资源批量添加标签: {tags_to_add}")
    updated_count = 0

    for resource in resource_client.resources.list_by_resource_group(resource_group):
        # 合并现有标签和新标签
        current_tags = resource.tags or {}
        current_tags.update(tags_to_add)

        try:
            # 使用 begin_update_by_id 更新资源标签
            resource_client.tags.begin_create_or_update_at_scope(
                scope=resource.id,
                parameters={"properties": {"tags": current_tags}, "operation": "Merge"},
            ).result()
            updated_count += 1
            print(f"  已更新: {resource.name}")
        except Exception as e:
            print(f"  更新失败 [{resource.name}]: {e}")

    print(f"\n共更新 {updated_count} 个资源的标签")


if __name__ == "__main__":
    list_subscriptions()
    list_resource_groups()
    find_untagged_resources()
```

---

## 九、SRE 实战：Azure 资源批量管理与巡检脚本

### 9.1 完整巡检脚本

```python
#!/usr/bin/env python3
"""
============================================================
Azure SRE 全面巡检脚本
============================================================
功能：
  1. 未使用资源检测（孤立磁盘、空闲公网IP、未关联NSG）
  2. VM 运行状态与性能巡检
  3. 存储账户安全检查
  4. NSG 安全规则审计
  5. 资源标签合规检查
  6. 成本异常初步分析
  7. 汇总报告输出

用法：
  export AZURE_SUBSCRIPTION_ID="your-sub-id"
  python3 azure_sre_patrol.py
============================================================
"""

import os
import sys
import json
from datetime import datetime, timedelta, timezone
from collections import defaultdict

# Azure SDK 导入
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
from azure.mgmt.network import NetworkManagementClient
from azure.mgmt.storage import StorageManagementClient
from azure.mgmt.resource import ResourceManagementClient
from azure.mgmt.monitor import MonitorManagementClient
from azure.core.exceptions import HttpResponseError

# ============================================================
# 全局配置
# ============================================================
SUBSCRIPTION_ID = os.environ.get("AZURE_SUBSCRIPTION_ID", "")
if not SUBSCRIPTION_ID:
    print("[错误] 请设置环境变量 AZURE_SUBSCRIPTION_ID")
    sys.exit(1)

# 初始化凭据和所有客户端
credential = DefaultAzureCredential()
compute_client = ComputeManagementClient(credential, SUBSCRIPTION_ID)
network_client = NetworkManagementClient(credential, SUBSCRIPTION_ID)
storage_client = StorageManagementClient(credential, SUBSCRIPTION_ID)
resource_client = ResourceManagementClient(credential, SUBSCRIPTION_ID)
monitor_client = MonitorManagementClient(credential, SUBSCRIPTION_ID)

# 巡检结果收集器
class PatrolReport:
    """巡检报告收集器"""
    def __init__(self):
        self.findings = []          # 所有发现的问题
        self.stats = defaultdict(int)  # 统计计数器
        self.start_time = datetime.now()

    def add_finding(self, severity, category, resource, message):
        """
        添加一条巡检发现
        severity: CRITICAL / HIGH / MEDIUM / LOW / INFO
        category: 分类（如 security, cost, compliance）
        """
        self.findings.append({
            "severity": severity,
            "category": category,
            "resource": resource,
            "message": message,
            "timestamp": datetime.now().isoformat(),
        })
        self.stats[severity] += 1
        # 实时输出
        icon = {"CRITICAL": "!!!!", "HIGH": " !! ", "MEDIUM": " !  ", "LOW": " .  ", "INFO": " i  "}
        print(f"  [{icon.get(severity, '    ')}] [{severity}] {message}")

    def summary(self):
        """输出巡检摘要"""
        elapsed = (datetime.now() - self.start_time).total_seconds()
        print("\n" + "=" * 70)
        print("                    巡 检 报 告 摘 要")
        print("=" * 70)
        print(f"  巡检时间: {self.start_time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"  耗时: {elapsed:.1f} 秒")
        print(f"  订阅: {SUBSCRIPTION_ID}")
        print("-" * 70)
        print(f"  CRITICAL : {self.stats.get('CRITICAL', 0)}")
        print(f"  HIGH     : {self.stats.get('HIGH', 0)}")
        print(f"  MEDIUM   : {self.stats.get('MEDIUM', 0)}")
        print(f"  LOW      : {self.stats.get('LOW', 0)}")
        print(f"  INFO     : {self.stats.get('INFO', 0)}")
        print(f"  总计     : {len(self.findings)}")
        print("=" * 70)
        return self.findings


# 创建全局报告实例
report = PatrolReport()


# ============================================================
# 检查模块 1：未使用资源检测
# ============================================================
def check_orphan_disks():
    """检查孤立磁盘（未挂载到任何 VM 的托管磁盘）"""
    print("\n[1/7] 检查孤立磁盘...")
    orphan_count = 0
    total_waste_gb = 0

    for disk in compute_client.disks.list():
        # managed_by 为 None 表示磁盘未挂载到任何 VM
        if disk.managed_by is None:
            orphan_count += 1
            size_gb = disk.disk_size_gb or 0
            total_waste_gb += size_gb
            rg = disk.id.split("/")[4]

            report.add_finding(
                severity="MEDIUM",
                category="cost",
                resource=f"{rg}/{disk.name}",
                message=f"孤立磁盘: {disk.name} ({size_gb}GB, {disk.sku.name}) "
                        f"在资源组 [{rg}]，建议清理以节省成本",
            )

    if orphan_count == 0:
        print("  未发现孤立磁盘")
    else:
        report.add_finding(
            severity="INFO",
            category="cost",
            resource="subscription",
            message=f"共发现 {orphan_count} 个孤立磁盘，"
                    f"总计浪费 {total_waste_gb}GB 存储空间",
        )


def check_unused_public_ips():
    """检查未关联的公网 IP（空闲公网 IP 仍会产生费用）"""
    print("\n[2/7] 检查空闲公网 IP...")
    idle_count = 0

    for ip in network_client.public_ip_addresses.list_all():
        # ip_configuration 为 None 表示公网 IP 未关联任何资源
        if ip.ip_configuration is None:
            idle_count += 1
            rg = ip.id.split("/")[4]

            report.add_finding(
                severity="LOW",
                category="cost",
                resource=f"{rg}/{ip.name}",
                message=f"空闲公网 IP: {ip.name} ({ip.ip_address or '未分配'}) "
                        f"在资源组 [{rg}]，Standard SKU 空闲仍计费",
            )

    if idle_count == 0:
        print("  未发现空闲公网 IP")


def check_unassociated_nsgs():
    """检查未关联子网或网卡的 NSG"""
    print("\n[3/7] 检查未关联的 NSG...")

    for nsg in network_client.network_security_groups.list_all():
        has_subnet = bool(nsg.subnets)
        has_nic = bool(nsg.network_interfaces)

        if not has_subnet and not has_nic:
            rg = nsg.id.split("/")[4]
            report.add_finding(
                severity="LOW",
                category="compliance",
                resource=f"{rg}/{nsg.name}",
                message=f"NSG [{nsg.name}] 未关联任何子网或网卡，可能是残留资源",
            )


# ============================================================
# 检查模块 2：VM 状态巡检
# ============================================================
def check_vm_status():
    """检查所有 VM 的运行状态和配置"""
    print("\n[4/7] VM 状态巡检...")
    vm_count = 0
    running_count = 0
    stopped_billing_count = 0  # 已停止但仍计费的 VM

    for vm in compute_client.virtual_machines.list_all():
        vm_count += 1
        rg = vm.id.split("/")[4]

        try:
            # 获取 VM 实例视图（包含运行时状态）
            instance_view = compute_client.virtual_machines.instance_view(
                resource_group_name=rg,
                vm_name=vm.name,
            )

            power_state = "unknown"
            for status in instance_view.statuses:
                if status.code.startswith("PowerState"):
                    power_state = status.code.split("/")[1]

            if power_state == "running":
                running_count += 1

            # 检查：VM 处于 stopped 状态（非 deallocated，仍计费）
            if power_state == "stopped":
                stopped_billing_count += 1
                report.add_finding(
                    severity="HIGH",
                    category="cost",
                    resource=f"{rg}/{vm.name}",
                    message=f"VM [{vm.name}] 处于 stopped 状态（仍在计费！），"
                            f"建议 deallocate 或删除。大小: {vm.hardware_profile.vm_size}",
                )

            # 检查：VM 缺少关键标签
            tags = vm.tags or {}
            if "owner" not in tags:
                report.add_finding(
                    severity="MEDIUM",
                    category="compliance",
                    resource=f"{rg}/{vm.name}",
                    message=f"VM [{vm.name}] 缺少 'owner' 标签，无法追溯责任人",
                )

        except HttpResponseError as e:
            report.add_finding(
                severity="MEDIUM",
                category="availability",
                resource=f"{rg}/{vm.name}",
                message=f"无法获取 VM [{vm.name}] 的状态: {e.message}",
            )

    report.add_finding(
        severity="INFO",
        category="inventory",
        resource="subscription",
        message=f"VM 统计: 共 {vm_count} 台，运行中 {running_count} 台，"
                f"stopped(仍计费) {stopped_billing_count} 台",
    )


# ============================================================
# 检查模块 3：存储账户安全检查
# ============================================================
def check_storage_security():
    """检查存储账户安全配置"""
    print("\n[5/7] 存储账户安全检查...")

    for account in storage_client.storage_accounts.list():
        rg = account.id.split("/")[4]
        issues = []

        # 检查1：HTTPS 是否强制开启
        if not account.enable_https_traffic_only:
            issues.append("未强制 HTTPS")
            report.add_finding(
                severity="HIGH",
                category="security",
                resource=f"{rg}/{account.name}",
                message=f"存储账户 [{account.name}] 未强制 HTTPS 传输，存在数据泄露风险",
            )

        # 检查2：Blob 公共访问是否关闭
        if account.allow_blob_public_access:
            report.add_finding(
                severity="HIGH",
                category="security",
                resource=f"{rg}/{account.name}",
                message=f"存储账户 [{account.name}] 允许 Blob 公共访问，"
                        f"可能导致数据泄露",
            )

        # 检查3：最低 TLS 版本
        min_tls = account.minimum_tls_version
        if min_tls and min_tls != "TLS1_2":
            report.add_finding(
                severity="MEDIUM",
                category="security",
                resource=f"{rg}/{account.name}",
                message=f"存储账户 [{account.name}] 最低 TLS 版本为 {min_tls}，"
                        f"建议设置为 TLS1_2",
            )

        # 检查4：网络规则（是否允许所有网络访问）
        if account.network_rule_set:
            default_action = account.network_rule_set.default_action
            if default_action == "Allow":
                report.add_finding(
                    severity="MEDIUM",
                    category="security",
                    resource=f"{rg}/{account.name}",
                    message=f"存储账户 [{account.name}] 网络规则默认动作为 Allow，"
                            f"建议设置为 Deny 并配置白名单",
                )

        if not issues:
            print(f"  [{account.name}] 安全配置正常")


# ============================================================
# 检查模块 4：NSG 安全规则审计
# ============================================================
def check_nsg_security():
    """审计 NSG 入站规则，检测高风险配置"""
    print("\n[6/7] NSG 安全规则审计...")
    # 高危端口定义
    high_risk_ports = {
        "22": "SSH",
        "3389": "RDP",
        "3306": "MySQL",
        "5432": "PostgreSQL",
        "1433": "MSSQL",
        "27017": "MongoDB",
        "6379": "Redis",
        "9200": "Elasticsearch",
        "5601": "Kibana",
    }

    for nsg in network_client.network_security_groups.list_all():
        rg = nsg.id.split("/")[4]

        for rule in (nsg.security_rules or []):
            # 只关注入站允许规则
            if rule.direction != "Inbound" or rule.access != "Allow":
                continue

            src = rule.source_address_prefix or ""
            dst_port = rule.destination_port_range or ""

            # 判断源是否为公网任意地址
            is_public = src in ("*", "0.0.0.0/0", "Internet")

            if not is_public:
                continue

            # 检查端口范围
            if dst_port == "*":
                report.add_finding(
                    severity="CRITICAL",
                    category="security",
                    resource=f"{rg}/{nsg.name}",
                    message=f"NSG [{nsg.name}] 规则 [{rule.name}] 允许公网访问所有端口！"
                            f" 优先级: {rule.priority}",
                )
            elif dst_port in high_risk_ports:
                svc = high_risk_ports[dst_port]
                report.add_finding(
                    severity="HIGH",
                    category="security",
                    resource=f"{rg}/{nsg.name}",
                    message=f"NSG [{nsg.name}] 规则 [{rule.name}] 允许公网访问 "
                            f"{svc}(端口 {dst_port})",
                )
            elif "-" in dst_port:
                # 端口范围情况，如 "1000-2000"
                try:
                    start_p, end_p = dst_port.split("-")
                    port_range_size = int(end_p) - int(start_p) + 1
                    if port_range_size > 100:
                        report.add_finding(
                            severity="MEDIUM",
                            category="security",
                            resource=f"{rg}/{nsg.name}",
                            message=f"NSG [{nsg.name}] 规则 [{rule.name}] 允许公网访问 "
                                    f"端口范围 {dst_port} ({port_range_size} 个端口)",
                        )
                except ValueError:
                    pass


# ============================================================
# 检查模块 5：资源标签合规检查
# ============================================================
def check_tag_compliance():
    """检查资源标签是否符合组织策略"""
    print("\n[7/7] 资源标签合规检查...")
    required_tags = {"env", "team"}  # 必需的标签
    non_compliant_count = 0

    for rg in resource_client.resource_groups.list():
        rg_tags = set(rg.tags.keys()) if rg.tags else set()
        missing = required_tags - rg_tags

        if missing:
            non_compliant_count += 1
            report.add_finding(
                severity="LOW",
                category="compliance",
                resource=f"{rg.name}",
                message=f"资源组 [{rg.name}] 缺少必需标签: {missing}",
            )

    if non_compliant_count == 0:
        print("  所有资源组标签合规")


# ============================================================
# 报告导出
# ============================================================
def export_report(findings, output_file="patrol_report.json"):
    """将巡检结果导出为 JSON 文件"""
    report_data = {
        "report_time": datetime.now().isoformat(),
        "subscription_id": SUBSCRIPTION_ID,
        "total_findings": len(findings),
        "severity_summary": dict(report.stats),
        "findings": findings,
    }

    with open(output_file, "w", encoding="utf-8") as f:
        json.dump(report_data, f, ensure_ascii=False, indent=2)

    print(f"\n巡检报告已导出: {output_file}")


# ============================================================
# 主函数
# ============================================================
def main():
    """执行全面巡检"""
    print("=" * 70)
    print("           Azure SRE 自动化巡检")
    print(f"           订阅: {SUBSCRIPTION_ID}")
    print(f"           时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 70)

    # 依次执行各检查模块
    try:
        check_orphan_disks()          # 孤立磁盘
        check_unused_public_ips()     # 空闲公网 IP
        check_unassociated_nsgs()     # 未关联 NSG
        check_vm_status()             # VM 状态
        check_storage_security()      # 存储安全
        check_nsg_security()          # NSG 安全规则
        check_tag_compliance()        # 标签合规
    except HttpResponseError as e:
        print(f"\n[错误] Azure API 调用失败: {e.message}")
        print(f"状态码: {e.status_code}")
    except Exception as e:
        print(f"\n[错误] 巡检过程中发生异常: {e}")

    # 输出摘要
    findings = report.summary()

    # 导出报告
    export_report(findings)

    # 返回退出码（有 CRITICAL 发现时返回非零）
    if report.stats.get("CRITICAL", 0) > 0:
        sys.exit(2)
    elif report.stats.get("HIGH", 0) > 0:
        sys.exit(1)
    else:
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## 十、常见坑点与最佳实践

### Do's (推荐做法)

```python
# -------------------------------------------------------
# DO: 使用 DefaultAzureCredential，自动适配多种环境
# -------------------------------------------------------
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()

# -------------------------------------------------------
# DO: 使用长时间运行操作的 poller 并处理超时
# -------------------------------------------------------
poller = compute_client.virtual_machines.begin_create_or_update(rg, name, params)
try:
    result = poller.result(timeout=600)  # 设置10分钟超时
except Exception as e:
    print(f"操作超时或失败: {e}")
    # 可以通过 poller.status() 查看当前状态

# -------------------------------------------------------
# DO: 对 API 调用进行异常处理和重试
# -------------------------------------------------------
from azure.core.exceptions import (
    HttpResponseError,
    ResourceNotFoundError,
    ClientAuthenticationError,
)

try:
    vm = compute_client.virtual_machines.get(rg, vm_name)
except ResourceNotFoundError:
    print(f"VM [{vm_name}] 不存在")
except ClientAuthenticationError:
    print("认证失败，请检查凭据")
except HttpResponseError as e:
    if e.status_code == 429:  # 触发限流
        print("API 限流，请稍后重试")
    else:
        raise

# -------------------------------------------------------
# DO: 分页处理大量资源
# -------------------------------------------------------
# SDK 默认使用惰性分页，直接 for 循环即可
all_vms = []
for vm in compute_client.virtual_machines.list_all():
    all_vms.append(vm)
    # SDK 会自动处理分页请求

# -------------------------------------------------------
# DO: 使用标签进行资源分组和成本追踪
# -------------------------------------------------------
tags = {
    "env": "production",
    "team": "sre",
    "owner": "zhangsan@company.com",
    "cost_center": "CC-1234",
    "auto_shutdown": "false",
}

# -------------------------------------------------------
# DO: deallocate 而不是 power_off 来停止计费
# -------------------------------------------------------
# begin_deallocate = 解除分配（不计费）
poller = compute_client.virtual_machines.begin_deallocate(rg, vm_name)
# begin_power_off = 仅关机（仍然计费！）
# poller = compute_client.virtual_machines.begin_power_off(rg, vm_name)
```

### Don'ts (避免做法)

```python
# -------------------------------------------------------
# DON'T: 硬编码凭据（严重安全隐患）
# -------------------------------------------------------
# 错误示例：
# credential = ClientSecretCredential(
#     tenant_id="abc-123",
#     client_id="def-456",
#     client_secret="supersecretpassword123",  # 切勿硬编码！
# )
# 正确做法：使用环境变量或 Azure Key Vault

# -------------------------------------------------------
# DON'T: 忽略 begin_ 前缀方法的异步特性
# -------------------------------------------------------
# 错误示例：不等待操作完成就进行后续操作
poller = compute_client.virtual_machines.begin_delete(rg, vm_name)
# 直接创建同名 VM —— 可能因删除未完成而失败！
# compute_client.virtual_machines.begin_create_or_update(rg, vm_name, ...)
#
# 正确做法：
poller = compute_client.virtual_machines.begin_delete(rg, vm_name)
poller.result()  # 等待删除完成
# 然后再进行后续操作

# -------------------------------------------------------
# DON'T: 删除 VM 后忘记清理关联资源
# -------------------------------------------------------
# 删除 VM 不会自动删除：磁盘、网卡、公网IP、NSG
# 正确做法：先记录关联资源 ID，删除 VM 后逐一清理

# -------------------------------------------------------
# DON'T: 不设置超时直接调用 result()
# -------------------------------------------------------
# 错误示例：
# poller.result()  # 可能永远阻塞
# 正确做法：
# poller.result(timeout=300)

# -------------------------------------------------------
# DON'T: 在循环中逐个创建资源（效率低下）
# -------------------------------------------------------
# 错误示例：
# for name in vm_names:
#     poller = client.begin_create(...)
#     poller.result()  # 串行等待，极慢
#
# 正确做法：并行发起，统一等待
pollers = []
for name in vm_names:
    p = compute_client.virtual_machines.begin_create_or_update(rg, name, params)
    pollers.append(p)
# 统一等待所有操作完成
for p in pollers:
    p.result(timeout=600)

# -------------------------------------------------------
# DON'T: 在生产环境使用 list_all() 且不设限制
# -------------------------------------------------------
# 大型订阅可能有数千资源，全量拉取非常慢
# 建议按资源组过滤或使用 filter 参数
```

---

## 十一、速查表

### 认证方式速查

| 认证方式 | 适用场景 | 配置要求 |
|---------|---------|---------|
| `DefaultAzureCredential` | 通用（推荐） | 自动检测环境 |
| `ClientSecretCredential` | CI/CD、自动化 | TENANT/CLIENT_ID/SECRET |
| `ManagedIdentityCredential` | Azure VM/AKS 内部 | 开启托管身份 |
| `AzureCliCredential` | 本地开发 | `az login` |
| `CertificateCredential` | 高安全要求 | 客户端证书 |

### 常用客户端初始化

```python
from azure.identity import DefaultAzureCredential
cred = DefaultAzureCredential()
sub = "你的订阅ID"

# 计算（VM/磁盘）
from azure.mgmt.compute import ComputeManagementClient
compute = ComputeManagementClient(cred, sub)

# 网络（VNet/NSG/LB）
from azure.mgmt.network import NetworkManagementClient
network = NetworkManagementClient(cred, sub)

# 存储（存储账户管理面）
from azure.mgmt.storage import StorageManagementClient
storage_mgmt = StorageManagementClient(cred, sub)

# Blob 存储（数据面）
from azure.storage.blob import BlobServiceClient
blob = BlobServiceClient(account_url="https://xxx.blob.core.windows.net", credential=cred)

# 资源管理
from azure.mgmt.resource import ResourceManagementClient
resource = ResourceManagementClient(cred, sub)

# 监控
from azure.monitor.query import MetricsQueryClient, LogsQueryClient
metrics = MetricsQueryClient(cred)
logs = LogsQueryClient(cred)
```

### VM 操作速查

| 操作 | 方法 | 是否异步 |
|------|------|---------|
| 列出所有 VM | `compute.virtual_machines.list_all()` | 否（分页迭代） |
| 列出资源组 VM | `compute.virtual_machines.list(rg)` | 否 |
| 获取 VM 详情 | `compute.virtual_machines.get(rg, name)` | 否 |
| 获取运行状态 | `compute.virtual_machines.instance_view(rg, name)` | 否 |
| 创建/更新 VM | `compute.virtual_machines.begin_create_or_update(rg, name, params)` | 是 |
| 启动 VM | `compute.virtual_machines.begin_start(rg, name)` | 是 |
| 关机(仍计费) | `compute.virtual_machines.begin_power_off(rg, name)` | 是 |
| 解除分配(停止计费) | `compute.virtual_machines.begin_deallocate(rg, name)` | 是 |
| 重启 VM | `compute.virtual_machines.begin_restart(rg, name)` | 是 |
| 删除 VM | `compute.virtual_machines.begin_delete(rg, name)` | 是 |

### Blob Storage 操作速查

| 操作 | 方法 |
|------|------|
| 创建容器 | `blob_service.create_container(name)` |
| 列出容器 | `blob_service.list_containers()` |
| 上传 Blob | `blob_client.upload_blob(data, overwrite=True)` |
| 下载 Blob | `blob_client.download_blob().readall()` |
| 删除 Blob | `blob_client.delete_blob()` |
| 列出 Blob | `container_client.list_blobs(name_starts_with=prefix)` |
| 设置 Blob 层级 | `blob_client.set_standard_blob_tier(tier)` |

### 异常处理速查

```python
from azure.core.exceptions import (
    HttpResponseError,           # HTTP 错误（4xx/5xx）
    ResourceNotFoundError,       # 404 资源不存在
    ResourceExistsError,         # 409 资源已存在
    ClientAuthenticationError,   # 认证失败
    ServiceRequestError,         # 网络连接失败
)
```

### NSG 规则优先级参考

| 优先级范围 | 建议用途 |
|-----------|---------|
| 100-199 | 核心业务允许规则 |
| 200-299 | 管理访问规则（SSH/RDP） |
| 300-399 | 监控和运维规则 |
| 400-499 | 第三方集成规则 |
| 4000-4096 | 兜底拒绝规则 |

### 常用 Azure 区域代码

| 区域 | 代码 | 区域 | 代码 |
|------|------|------|------|
| 东亚（香港） | `eastasia` | 东南亚（新加坡） | `southeastasia` |
| 中国东部 | `chinaeast` | 中国东部2 | `chinaeast2` |
| 中国北部 | `chinanorth` | 中国北部2 | `chinanorth2` |
| 美国东部 | `eastus` | 美国西部 | `westus` |
| 西欧 | `westeurope` | 日本东部 | `japaneast` |

---

> **提示**：本文档所有代码在正确配置 Azure 凭据和订阅 ID 后可直接运行。生产环境使用前请在测试订阅中充分验证，避免对线上资源造成影响。
