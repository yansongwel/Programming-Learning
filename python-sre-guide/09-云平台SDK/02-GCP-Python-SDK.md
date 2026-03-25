# Google Cloud Python SDK — 用 Python 自动化管理 GCP 云资源

## 1. 是什么 & 为什么用它

Google Cloud Python SDK 是 GCP 官方提供的 Python 客户端库集合，覆盖 Compute Engine、Cloud Storage、
BigQuery、Cloud Logging 等核心服务。每个服务对应一个独立的 Python 包。

**对 SRE 的价值：**
- 自动化管理 GCP 基础设施（VM、网络、存储）
- 集成 Cloud Logging 和 Monitoring 实现可观测性
- BigQuery 日志分析和指标聚合
- 构建跨项目、跨区域的资源管理工具

## 2. 安装与环境

```bash
# 安装核心服务的 SDK
pip install google-cloud-compute        # Compute Engine
pip install google-cloud-storage        # Cloud Storage
pip install google-cloud-logging        # Cloud Logging
pip install google-cloud-bigquery       # BigQuery
pip install google-cloud-monitoring     # Cloud Monitoring
pip install google-cloud-resource-manager  # 项目管理

# 安装认证库
pip install google-auth google-auth-oauthlib

# 验证安装
python -c "import google.cloud.compute_v1; print('Compute SDK OK')"
python -c "import google.cloud.storage; print('Storage SDK OK')"
```

**认证配置：**

```bash
# 方式一：Service Account JSON Key（推荐自动化脚本使用）
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"

# 方式二：gcloud CLI 登录（开发环境）
gcloud auth application-default login

# 方式三：在 GCE/GKE/Cloud Run 中自动使用元数据服务
# 无需配置，自动获取实例 Service Account 凭证
```

## 3. 核心概念与原理

### 3.1 GCP SDK 架构

```
+-------------------+     HTTPS/gRPC    +-------------------+    +------------------+
|  Python 脚本      |------------------>|  GCP API 端点     |--->|  GCP 服务        |
|  (google-cloud-*) |   认证 + 请求     |  (*.googleapis.com)|   |  (VM/GCS/BQ...) |
+-------------------+                   +-------------------+    +------------------+
        |
        v
  认证链（Application Default Credentials）
  1. GOOGLE_APPLICATION_CREDENTIALS 环境变量
  2. gcloud auth application-default 缓存
  3. 元数据服务器（GCE/GKE/Cloud Run）
  4. Cloud SDK 默认凭证
```

### 3.2 核心服务对应关系

```
GCP 服务               Python 包                         主要用途
─────────────────────────────────────────────────────────────────────
Compute Engine         google-cloud-compute              VM 管理
Cloud Storage          google-cloud-storage              对象存储
Cloud Logging          google-cloud-logging              日志管理
Cloud Monitoring       google-cloud-monitoring           指标监控
BigQuery               google-cloud-bigquery             数据分析
IAM                    google-cloud-iam                  身份管理
Resource Manager       google-cloud-resource-manager     项目/组织管理
Pub/Sub                google-cloud-pubsub               消息队列
```

### 3.3 项目与区域模型

```
Organization（组织）
  └── Folder（文件夹，可嵌套）
       └── Project（项目）── 资源的基本组织单位
            ├── Region（区域，如 us-central1）
            │    └── Zone（可用区，如 us-central1-a）
            ├── Global（全局资源，如 VPC）
            └── Multi-Region（多区域，如 GCS US）
```

## 4. 基础用法

### 4.1 认证与客户端初始化

```python
"""
GCP 认证与客户端初始化
"""
from google.cloud import compute_v1, storage, logging as cloud_logging
from google.oauth2 import service_account
import google.auth

# === 方式一：使用默认凭证（ADC，推荐）===
def init_with_default_credentials():
    """使用 Application Default Credentials"""
    # 自动从环境检测凭证
    credentials, project = google.auth.default()
    print(f"项目：{project}")

    # 创建客户端（自动使用 ADC）
    compute_client = compute_v1.InstancesClient()
    storage_client = storage.Client()
    return compute_client, storage_client

# === 方式二：指定 Service Account Key ===
def init_with_service_account(key_path):
    """使用 Service Account JSON Key"""
    credentials = service_account.Credentials.from_service_account_file(
        key_path,
        scopes=['https://www.googleapis.com/auth/cloud-platform']
    )

    storage_client = storage.Client(credentials=credentials, project='my-project-id')
    return storage_client

# === 方式三：跨项目访问 ===
def init_cross_project(project_id, key_path=None):
    """访问不同项目的资源"""
    if key_path:
        credentials = service_account.Credentials.from_service_account_file(key_path)
        client = storage.Client(credentials=credentials, project=project_id)
    else:
        client = storage.Client(project=project_id)
    return client
```

### 4.2 Compute Engine 管理

```python
"""
Compute Engine VM 实例管理
"""
from google.cloud import compute_v1
import time

# 初始化客户端
instances_client = compute_v1.InstancesClient()
PROJECT = 'my-project-id'
ZONE = 'us-central1-a'

# === 列出实例 ===
def list_instances(project=PROJECT, zone=ZONE):
    """列出指定 zone 的所有 VM 实例"""
    request = compute_v1.ListInstancesRequest(project=project, zone=zone)
    instances = instances_client.list(request=request)

    for instance in instances:
        # 获取内网 IP
        internal_ip = ''
        external_ip = ''
        if instance.network_interfaces:
            ni = instance.network_interfaces[0]
            internal_ip = ni.network_i_p or ''
            if ni.access_configs:
                external_ip = ni.access_configs[0].nat_i_p or ''

        print(
            f"  {instance.name:30s} {instance.status:12s} "
            f"{instance.machine_type.split('/')[-1]:20s} "
            f"{internal_ip:15s} {external_ip:15s}"
        )

    return list(instances)

# === 列出所有 zone 的实例 ===
def list_all_instances(project=PROJECT):
    """列出项目中所有 zone 的实例"""
    agg_client = compute_v1.InstancesClient()
    request = compute_v1.AggregatedListInstancesRequest(project=project)
    agg_list = agg_client.aggregated_list(request=request)

    all_instances = []
    for zone, instances_scoped_list in agg_list:
        if instances_scoped_list.instances:
            for instance in instances_scoped_list.instances:
                all_instances.append({
                    'name': instance.name,
                    'zone': zone.split('/')[-1],
                    'status': instance.status,
                    'machine_type': instance.machine_type.split('/')[-1],
                })
                print(
                    f"  {zone.split('/')[-1]:20s} {instance.name:30s} "
                    f"{instance.status:12s} {instance.machine_type.split('/')[-1]}"
                )

    return all_instances

# === 创建实例 ===
def create_instance(name, machine_type='e2-micro', image_family='debian-11',
                    project=PROJECT, zone=ZONE):
    """创建 VM 实例"""
    # 构造磁盘配置
    disk = compute_v1.AttachedDisk()
    disk.auto_delete = True
    disk.boot = True
    disk.initialize_params = compute_v1.AttachedDiskInitializeParams()
    disk.initialize_params.disk_size_gb = 10
    disk.initialize_params.source_image = (
        f"projects/debian-cloud/global/images/family/{image_family}"
    )

    # 构造网络配置
    network_interface = compute_v1.NetworkInterface()
    network_interface.name = "global/networks/default"
    # 添加外网访问
    access_config = compute_v1.AccessConfig()
    access_config.name = "External NAT"
    access_config.type_ = "ONE_TO_ONE_NAT"
    network_interface.access_configs = [access_config]

    # 构造实例
    instance = compute_v1.Instance()
    instance.name = name
    instance.machine_type = f"zones/{zone}/machineTypes/{machine_type}"
    instance.disks = [disk]
    instance.network_interfaces = [network_interface]
    instance.labels = {
        'managed-by': 'sre-automation',
        'team': 'sre'
    }

    # 创建
    request = compute_v1.InsertInstanceRequest(
        project=project,
        zone=zone,
        instance_resource=instance
    )

    operation = instances_client.insert(request=request)
    print(f"创建实例中：{name}...")

    # 等待操作完成
    _wait_for_operation(operation, project, zone)
    print(f"实例已创建：{name}")

def _wait_for_operation(operation, project, zone):
    """等待区域操作完成"""
    zone_ops_client = compute_v1.ZoneOperationsClient()
    while operation.status != compute_v1.Operation.Status.DONE:
        operation = zone_ops_client.get(
            project=project,
            zone=zone,
            operation=operation.name
        )
        time.sleep(2)

    if operation.error:
        raise Exception(f"操作失败：{operation.error}")

# === 停止/启动实例 ===
def stop_instance(name, project=PROJECT, zone=ZONE):
    """停止实例"""
    operation = instances_client.stop(project=project, zone=zone, instance=name)
    _wait_for_operation(operation, project, zone)
    print(f"实例已停止：{name}")

def start_instance(name, project=PROJECT, zone=ZONE):
    """启动实例"""
    operation = instances_client.start(project=project, zone=zone, instance=name)
    _wait_for_operation(operation, project, zone)
    print(f"实例已启动：{name}")

# === 删除实例 ===
def delete_instance(name, project=PROJECT, zone=ZONE):
    """删除实例"""
    operation = instances_client.delete(project=project, zone=zone, instance=name)
    _wait_for_operation(operation, project, zone)
    print(f"实例已删除：{name}")

# === 调整机器类型 ===
def resize_instance(name, new_machine_type, project=PROJECT, zone=ZONE):
    """调整实例规格（需先停止）"""
    # 先停止实例
    stop_instance(name, project, zone)

    # 修改机器类型
    request = compute_v1.SetMachineTypeInstanceRequest(
        project=project,
        zone=zone,
        instance=name,
        instances_set_machine_type_request_resource=(
            compute_v1.InstancesSetMachineTypeRequest(
                machine_type=f"zones/{zone}/machineTypes/{new_machine_type}"
            )
        )
    )
    operation = instances_client.set_machine_type(request=request)
    _wait_for_operation(operation, project, zone)

    # 重新启动
    start_instance(name, project, zone)
    print(f"实例 {name} 已调整为 {new_machine_type}")
```

### 4.3 Cloud Storage

```python
"""
Cloud Storage 操作：存储桶和对象管理
"""
from google.cloud import storage
import os

client = storage.Client()

# === 列出存储桶 ===
def list_buckets():
    """列出所有存储桶"""
    buckets = client.list_buckets()
    for bucket in buckets:
        print(f"  {bucket.name:40s} 位置: {bucket.location:15s} 类型: {bucket.storage_class}")

# === 创建存储桶 ===
def create_bucket(name, location='US', storage_class='STANDARD'):
    """创建存储桶"""
    bucket = client.bucket(name)
    bucket.storage_class = storage_class
    new_bucket = client.create_bucket(bucket, location=location)
    print(f"存储桶已创建：{new_bucket.name}")
    return new_bucket

# === 上传文件 ===
def upload_file(bucket_name, source_file, destination_blob):
    """上传文件到 GCS"""
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_blob)
    blob.upload_from_filename(source_file)
    print(f"已上传：{source_file} -> gs://{bucket_name}/{destination_blob}")

# === 上传字符串内容 ===
def upload_string(bucket_name, content, destination_blob, content_type='text/plain'):
    """上传字符串内容到 GCS"""
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_blob)
    blob.upload_from_string(content, content_type=content_type)
    print(f"已上传内容到：gs://{bucket_name}/{destination_blob}")

# === 下载文件 ===
def download_file(bucket_name, source_blob, destination_file):
    """从 GCS 下载文件"""
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(source_blob)
    os.makedirs(os.path.dirname(destination_file), exist_ok=True)
    blob.download_to_filename(destination_file)
    print(f"已下载：gs://{bucket_name}/{source_blob} -> {destination_file}")

# === 列出对象 ===
def list_objects(bucket_name, prefix=None):
    """列出存储桶中的对象"""
    blobs = client.list_blobs(bucket_name, prefix=prefix)
    total_size = 0
    count = 0
    for blob in blobs:
        size_mb = blob.size / 1024 / 1024 if blob.size else 0
        total_size += blob.size or 0
        count += 1
        print(f"  {blob.name:60s} {size_mb:10.2f}MB  {blob.updated}")

    print(f"\n总计：{count} 个对象，{total_size / 1024 / 1024:.2f}MB")

# === 生成签名 URL ===
def generate_signed_url(bucket_name, blob_name, expiration_minutes=60):
    """生成签名 URL（临时下载链接）"""
    from datetime import timedelta

    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    url = blob.generate_signed_url(
        version='v4',
        expiration=timedelta(minutes=expiration_minutes),
        method='GET'
    )
    print(f"签名 URL（{expiration_minutes}分钟有效）：{url}")
    return url

# === 设置生命周期 ===
def set_lifecycle(bucket_name, age_days=90):
    """设置存储桶生命周期（自动删除旧对象）"""
    bucket = client.bucket(bucket_name)
    bucket.add_lifecycle_delete_rule(age=age_days)
    bucket.patch()
    print(f"已设置生命周期：{age_days} 天后自动删除")

# === 批量复制 ===
def copy_objects(src_bucket, dst_bucket, prefix=''):
    """批量复制对象"""
    src = client.bucket(src_bucket)
    dst = client.bucket(dst_bucket)

    blobs = client.list_blobs(src_bucket, prefix=prefix)
    count = 0
    for blob in blobs:
        src.copy_blob(blob, dst, blob.name)
        count += 1

    print(f"已复制 {count} 个对象从 {src_bucket} 到 {dst_bucket}")
```

### 4.4 Cloud Logging

```python
"""
Cloud Logging：写入和查询日志
"""
from google.cloud import logging as cloud_logging
import json

# 初始化客户端
logging_client = cloud_logging.Client()

# === 写入日志 ===
def write_log(log_name, message, severity='INFO', labels=None):
    """写入自定义日志"""
    logger = logging_client.logger(log_name)

    # 文本日志
    if isinstance(message, str):
        logger.log_text(message, severity=severity, labels=labels)
    # 结构化日志
    elif isinstance(message, dict):
        logger.log_struct(message, severity=severity, labels=labels)

    print(f"日志已写入：{log_name} [{severity}]")

# === 写入结构化日志 ===
def write_structured_log(log_name='sre-automation'):
    """写入结构化日志（推荐）"""
    logger = logging_client.logger(log_name)

    logger.log_struct(
        {
            'action': 'instance_restart',
            'target': 'web-server-01',
            'operator': 'sre-bot',
            'reason': '健康检查失败',
            'result': 'success',
            'duration_seconds': 45
        },
        severity='WARNING',
        labels={
            'team': 'sre',
            'service': 'auto-healing',
            'env': 'production'
        }
    )

# === 查询日志 ===
def query_logs(filter_str, max_results=100):
    """
    查询日志

    filter_str 示例：
        'resource.type="gce_instance" AND severity>=ERROR'
        'logName="projects/my-project/logs/sre-automation"'
        'textPayload:"error" AND timestamp>="2026-03-25T00:00:00Z"'
    """
    entries = logging_client.list_entries(
        filter_=filter_str,
        max_results=max_results,
        order_by=cloud_logging.DESCENDING
    )

    results = []
    for entry in entries:
        info = {
            'timestamp': entry.timestamp.isoformat() if entry.timestamp else '',
            'severity': entry.severity,
            'log_name': entry.log_name,
            'payload': entry.payload,
            'labels': dict(entry.labels) if entry.labels else {},
        }
        results.append(info)
        print(
            f"  [{info['severity']:8s}] {info['timestamp'][:19]} "
            f"{str(info['payload'])[:100]}"
        )

    return results

# === 查询错误日志 ===
def find_errors(project_id, hours=1):
    """查找最近 N 小时的错误日志"""
    from datetime import datetime, timedelta

    since = (datetime.utcnow() - timedelta(hours=hours)).strftime('%Y-%m-%dT%H:%M:%SZ')
    filter_str = (
        f'severity>=ERROR AND '
        f'timestamp>="{since}"'
    )
    return query_logs(filter_str)

# === 设置 Python logging 集成 ===
def setup_cloud_logging():
    """将 Python 标准 logging 模块集成到 Cloud Logging"""
    logging_client.setup_logging()

    import logging
    logger = logging.getLogger('sre-app')
    logger.setLevel(logging.INFO)

    # 现在标准 logging 会自动发送到 Cloud Logging
    logger.info("应用启动")
    logger.warning("磁盘空间不足")
    logger.error("数据库连接失败", extra={'labels': {'service': 'db-monitor'}})
```

### 4.5 BigQuery

```python
"""
BigQuery：数据分析和日志聚合
"""
from google.cloud import bigquery

bq_client = bigquery.Client()

# === 执行查询 ===
def run_query(sql):
    """执行 BigQuery SQL 查询"""
    query_job = bq_client.query(sql)
    results = query_job.result()

    rows = []
    for row in results:
        rows.append(dict(row))
        print(dict(row))

    print(f"\n查询完成：{results.total_rows} 行")
    return rows

# === 查询 GCE 审计日志 ===
def query_audit_logs(project_id, days=7):
    """从 BigQuery 中查询审计日志"""
    sql = f"""
    SELECT
        timestamp,
        protopayload_auditlog.methodName AS method,
        protopayload_auditlog.resourceName AS resource,
        protopayload_auditlog.authenticationInfo.principalEmail AS user,
        severity
    FROM `{project_id}.logs.cloudaudit_googleapis_com_activity_*`
    WHERE _TABLE_SUFFIX >= FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL {days} DAY))
    ORDER BY timestamp DESC
    LIMIT 100
    """
    return run_query(sql)

# === 导出查询结果到 GCS ===
def export_to_gcs(sql, destination_uri):
    """将查询结果导出到 GCS"""
    # 先创建临时表
    job_config = bigquery.QueryJobConfig(
        destination=f"{bq_client.project}.temp_dataset.export_table",
        write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE
    )
    query_job = bq_client.query(sql, job_config=job_config)
    query_job.result()

    # 导出到 GCS
    extract_config = bigquery.ExtractJobConfig(
        destination_format=bigquery.DestinationFormat.CSV
    )
    extract_job = bq_client.extract_table(
        f"{bq_client.project}.temp_dataset.export_table",
        destination_uri,
        job_config=extract_config
    )
    extract_job.result()
    print(f"已导出到：{destination_uri}")

# === 加载数据到 BigQuery ===
def load_from_gcs(bucket_uri, dataset_id, table_id):
    """从 GCS 加载数据到 BigQuery"""
    table_ref = bq_client.dataset(dataset_id).table(table_id)
    job_config = bigquery.LoadJobConfig(
        source_format=bigquery.SourceFormat.CSV,
        skip_leading_rows=1,
        autodetect=True
    )

    load_job = bq_client.load_table_from_uri(
        bucket_uri, table_ref, job_config=job_config
    )
    load_job.result()
    print(f"数据已加载到 {dataset_id}.{table_id}")
```

## 5. 进阶用法

### 5.1 Cloud Monitoring 自定义指标

```python
"""
Cloud Monitoring：创建和查询自定义指标
"""
from google.cloud import monitoring_v3
from google.protobuf.timestamp_pb2 import Timestamp
import time

PROJECT = 'my-project-id'
PROJECT_NAME = f'projects/{PROJECT}'

monitoring_client = monitoring_v3.MetricServiceClient()

# === 创建自定义指标描述符 ===
def create_metric_descriptor():
    """创建自定义指标描述符"""
    descriptor = monitoring_v3.MetricDescriptor()
    descriptor.type = 'custom.googleapis.com/sre/request_latency'
    descriptor.metric_kind = monitoring_v3.MetricDescriptor.MetricKind.GAUGE
    descriptor.value_type = monitoring_v3.MetricDescriptor.ValueType.DOUBLE
    descriptor.description = 'SRE 应用请求延迟'
    descriptor.unit = 's'  # 秒

    # 添加标签
    label = monitoring_v3.LabelDescriptor()
    label.key = 'endpoint'
    label.value_type = monitoring_v3.LabelDescriptor.ValueType.STRING
    label.description = 'API 端点'
    descriptor.labels.append(label)

    result = monitoring_client.create_metric_descriptor(
        name=PROJECT_NAME,
        metric_descriptor=descriptor
    )
    print(f"指标描述符已创建：{result.name}")

# === 写入自定义指标数据 ===
def write_metric(metric_type, value, labels=None):
    """写入自定义指标数据点"""
    series = monitoring_v3.TimeSeries()
    series.metric.type = metric_type

    # 设置标签
    if labels:
        for k, v in labels.items():
            series.metric.labels[k] = v

    # 设置资源（全局）
    series.resource.type = 'global'
    series.resource.labels['project_id'] = PROJECT

    # 设置数据点
    now = time.time()
    interval = monitoring_v3.TimeInterval()
    interval.end_time = Timestamp(seconds=int(now))

    point = monitoring_v3.Point()
    point.interval = interval
    point.value.double_value = value
    series.points.append(point)

    monitoring_client.create_time_series(
        name=PROJECT_NAME,
        time_series=[series]
    )

# === 查询指标数据 ===
def query_metrics(metric_type, hours=1):
    """查询指标时间序列数据"""
    now = time.time()
    interval = monitoring_v3.TimeInterval()
    interval.end_time = Timestamp(seconds=int(now))
    interval.start_time = Timestamp(seconds=int(now - hours * 3600))

    results = monitoring_client.list_time_series(
        request={
            'name': PROJECT_NAME,
            'filter': f'metric.type = "{metric_type}"',
            'interval': interval,
            'view': monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        }
    )

    for series in results:
        labels = dict(series.metric.labels)
        print(f"标签：{labels}")
        for point in series.points:
            ts = point.interval.end_time.seconds
            val = point.value.double_value
            print(f"  {time.strftime('%H:%M:%S', time.gmtime(ts))}: {val}")
```

### 5.2 跨项目资源管理

```python
"""
跨项目资源管理
"""
from google.cloud import resourcemanager_v3
from google.cloud import compute_v1

# === 列出所有项目 ===
def list_projects():
    """列出所有可访问的项目"""
    rm_client = resourcemanager_v3.ProjectsClient()
    request = resourcemanager_v3.SearchProjectsRequest()
    projects = rm_client.search_projects(request=request)

    result = []
    for project in projects:
        info = {
            'id': project.project_id,
            'name': project.display_name,
            'state': project.state.name,
        }
        result.append(info)
        print(f"  {info['id']:30s} {info['name']:30s} {info['state']}")

    return result

# === 跨项目扫描 VM ===
def scan_all_projects_vms():
    """扫描所有项目的 VM 实例"""
    projects = list_projects()
    all_vms = []

    for proj in projects:
        if proj['state'] != 'ACTIVE':
            continue

        try:
            client = compute_v1.InstancesClient()
            request = compute_v1.AggregatedListInstancesRequest(project=proj['id'])
            agg = client.aggregated_list(request=request)

            for zone, scoped in agg:
                if scoped.instances:
                    for instance in scoped.instances:
                        vm_info = {
                            'project': proj['id'],
                            'zone': zone.split('/')[-1],
                            'name': instance.name,
                            'status': instance.status,
                            'machine_type': instance.machine_type.split('/')[-1],
                        }
                        all_vms.append(vm_info)

        except Exception as e:
            print(f"  跳过项目 {proj['id']}：{e}")

    return all_vms
```

## 6. SRE 实战案例：GCP 资源自动化管理

```python
"""
SRE 实战：GCP 资源自动化管理与巡检
功能：
  1. 扫描所有区域的 VM 实例
  2. 识别闲置实例（低 CPU 利用率）
  3. 检查未使用的磁盘和 IP
  4. GCS 存储成本分析
  5. 日志错误汇总
  6. 生成巡检报告
"""
from google.cloud import compute_v1, storage, monitoring_v3
from google.cloud import logging as cloud_logging
from google.protobuf.timestamp_pb2 import Timestamp
import logging
import time
import json
from datetime import datetime, timedelta
from collections import defaultdict

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger('gcp-inspector')

class GCPResourceInspector:
    """
    GCP 资源巡检与优化工具
    """

    def __init__(self, project_id):
        """
        初始化巡检工具

        参数：
            project_id: GCP 项目 ID
        """
        self.project_id = project_id
        self.project_name = f'projects/{project_id}'
        self.findings = []
        self.cost_savings = 0.0

        logger.info(f"GCP 巡检工具已初始化（项目: {project_id}）")

    def _add_finding(self, category, severity, resource_id, resource_type,
                     description, estimated_savings=0.0):
        """记录发现"""
        finding = {
            'category': category,
            'severity': severity,
            'resource_id': resource_id,
            'resource_type': resource_type,
            'description': description,
            'estimated_monthly_savings': estimated_savings,
            'project': self.project_id,
            'timestamp': datetime.now().isoformat()
        }
        self.findings.append(finding)
        self.cost_savings += estimated_savings
        logger.info(f"[{severity:8s}] {resource_type}/{resource_id}: {description}")

    # ================================================================
    # VM 巡检
    # ================================================================

    def inspect_compute_instances(self):
        """巡检 Compute Engine 实例"""
        logger.info("开始 Compute Engine 巡检...")
        instances_client = compute_v1.InstancesClient()

        request = compute_v1.AggregatedListInstancesRequest(project=self.project_id)
        agg_list = instances_client.aggregated_list(request=request)

        instance_count = 0
        for zone, scoped_list in agg_list:
            if not scoped_list.instances:
                continue

            for instance in scoped_list.instances:
                instance_count += 1
                zone_name = zone.split('/')[-1]
                machine_type = instance.machine_type.split('/')[-1]

                # 检查停止的实例（仍产生磁盘费用）
                if instance.status == 'TERMINATED':
                    self._add_finding(
                        category='cost_optimization',
                        severity='low',
                        resource_id=f'{instance.name} ({zone_name})',
                        resource_type='VM Instance',
                        description=f'实例已停止但仍保留（磁盘仍产生费用），类型: {machine_type}'
                    )

                # 检查没有标签的实例
                if not instance.labels:
                    self._add_finding(
                        category='governance',
                        severity='low',
                        resource_id=f'{instance.name} ({zone_name})',
                        resource_type='VM Instance',
                        description='实例缺少标签，不利于成本分摊和资源管理'
                    )

                # 检查运行中实例的 CPU 利用率
                if instance.status == 'RUNNING':
                    avg_cpu = self._get_instance_cpu(instance.name, zone_name)
                    if avg_cpu is not None and avg_cpu < 10:
                        # 简化的费用估算
                        monthly_estimate = self._estimate_vm_cost(machine_type)
                        self._add_finding(
                            category='cost_optimization',
                            severity='medium',
                            resource_id=f'{instance.name} ({zone_name})',
                            resource_type='VM Instance',
                            description=(
                                f'实例 7 天平均 CPU 仅 {avg_cpu:.1f}%，'
                                f'类型: {machine_type}，建议缩小规格'
                            ),
                            estimated_savings=monthly_estimate * 0.5  # 假设可节省50%
                        )

        logger.info(f"共扫描 {instance_count} 个 VM 实例")

    def _get_instance_cpu(self, instance_name, zone):
        """获取实例最近 7 天的平均 CPU"""
        try:
            monitoring_client = monitoring_v3.MetricServiceClient()
            now = time.time()
            interval = monitoring_v3.TimeInterval()
            interval.end_time = Timestamp(seconds=int(now))
            interval.start_time = Timestamp(seconds=int(now - 7 * 86400))

            results = monitoring_client.list_time_series(
                request={
                    'name': self.project_name,
                    'filter': (
                        f'metric.type = "compute.googleapis.com/instance/cpu/utilization" '
                        f'AND resource.labels.instance_id = "{instance_name}"'
                    ),
                    'interval': interval,
                    'view': monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
                    'aggregation': monitoring_v3.Aggregation(
                        alignment_period={'seconds': 86400},
                        per_series_aligner=monitoring_v3.Aggregation.Aligner.ALIGN_MEAN,
                    )
                }
            )

            values = []
            for series in results:
                for point in series.points:
                    values.append(point.value.double_value * 100)  # 转为百分比

            return sum(values) / len(values) if values else None
        except Exception as e:
            logger.debug(f"获取 CPU 指标失败：{e}")
            return None

    def _estimate_vm_cost(self, machine_type):
        """简化的 VM 月成本估算"""
        cost_map = {
            'e2-micro': 7.0, 'e2-small': 14.0, 'e2-medium': 28.0,
            'n1-standard-1': 25.0, 'n1-standard-2': 50.0, 'n1-standard-4': 100.0,
            'n2-standard-2': 55.0, 'n2-standard-4': 110.0,
        }
        return cost_map.get(machine_type, 50.0)

    # ================================================================
    # 磁盘巡检
    # ================================================================

    def inspect_disks(self):
        """巡检未挂载的磁盘"""
        logger.info("开始磁盘巡检...")
        disks_client = compute_v1.DisksClient()

        request = compute_v1.AggregatedListDisksRequest(project=self.project_id)
        agg_list = disks_client.aggregated_list(request=request)

        for zone, scoped_list in agg_list:
            if not scoped_list.disks:
                continue

            for disk in scoped_list.disks:
                # 未挂载的磁盘（没有 users 字段）
                if not disk.users:
                    zone_name = zone.split('/')[-1]
                    size_gb = disk.size_gb
                    disk_type = disk.type_.split('/')[-1]

                    # 估算月费用
                    price_per_gb = {
                        'pd-standard': 0.04, 'pd-balanced': 0.10,
                        'pd-ssd': 0.17, 'pd-extreme': 0.125
                    }
                    monthly_cost = size_gb * price_per_gb.get(disk_type, 0.10)

                    self._add_finding(
                        category='cost_optimization',
                        severity='medium',
                        resource_id=f'{disk.name} ({zone_name})',
                        resource_type='Persistent Disk',
                        description=f'未挂载磁盘：{size_gb}GB {disk_type}',
                        estimated_savings=monthly_cost
                    )

    # ================================================================
    # GCS 巡检
    # ================================================================

    def inspect_storage(self):
        """巡检 Cloud Storage"""
        logger.info("开始 GCS 巡检...")
        storage_client = storage.Client(project=self.project_id)

        for bucket in storage_client.list_buckets():
            # 检查存储桶是否有生命周期策略
            if not bucket.lifecycle_rules:
                self._add_finding(
                    category='governance',
                    severity='low',
                    resource_id=bucket.name,
                    resource_type='GCS Bucket',
                    description='存储桶未设置生命周期策略，可能导致存储成本持续增长'
                )

            # 检查是否公开访问
            policy = bucket.get_iam_policy()
            for binding in policy.bindings:
                if 'allUsers' in binding['members'] or 'allAuthenticatedUsers' in binding['members']:
                    self._add_finding(
                        category='security',
                        severity='high',
                        resource_id=bucket.name,
                        resource_type='GCS Bucket',
                        description=f'存储桶允许公开访问（角色: {binding["role"]}）'
                    )

    # ================================================================
    # 日志巡检
    # ================================================================

    def inspect_logs(self, hours=24):
        """检查最近的错误日志"""
        logger.info(f"开始日志巡检（最近 {hours} 小时）...")
        try:
            logging_client = cloud_logging.Client(project=self.project_id)

            since = (datetime.utcnow() - timedelta(hours=hours)).strftime('%Y-%m-%dT%H:%M:%SZ')
            filter_str = f'severity>=ERROR AND timestamp>="{since}"'

            entries = logging_client.list_entries(
                filter_=filter_str,
                max_results=500,
                order_by=cloud_logging.DESCENDING
            )

            error_counts = defaultdict(int)
            for entry in entries:
                # 按日志来源分组统计
                log_name = entry.log_name.split('/')[-1] if entry.log_name else 'unknown'
                error_counts[log_name] += 1

            if error_counts:
                total_errors = sum(error_counts.values())
                top_sources = sorted(error_counts.items(), key=lambda x: -x[1])[:10]

                self._add_finding(
                    category='reliability',
                    severity='medium' if total_errors < 100 else 'high',
                    resource_id=f'{total_errors} 条错误',
                    resource_type='Cloud Logging',
                    description=(
                        f'最近 {hours} 小时发现 {total_errors} 条错误日志。'
                        f'Top 来源：{", ".join(f"{k}({v})" for k, v in top_sources[:5])}'
                    )
                )

        except Exception as e:
            logger.warning(f"日志巡检失败：{e}")

    # ================================================================
    # 生成报告
    # ================================================================

    def generate_report(self):
        """生成巡检报告"""
        by_category = defaultdict(list)
        by_severity = defaultdict(int)

        for f in self.findings:
            by_category[f['category']].append(f)
            by_severity[f['severity']] += 1

        report = {
            'timestamp': datetime.now().isoformat(),
            'project': self.project_id,
            'summary': {
                'total_findings': len(self.findings),
                'by_severity': dict(by_severity),
                'by_category': {k: len(v) for k, v in by_category.items()},
                'estimated_monthly_savings': round(self.cost_savings, 2),
            },
            'findings': self.findings,
        }

        logger.info("=" * 70)
        logger.info("GCP 资源巡检报告")
        logger.info("=" * 70)
        logger.info(f"项目：{self.project_id}")
        logger.info(f"总发现：{len(self.findings)}")
        logger.info(f"严重度分布：{dict(by_severity)}")
        logger.info(f"预估月节省：${self.cost_savings:.2f}")

        for category, findings in by_category.items():
            logger.info(f"\n[{category}] ({len(findings)} 项)")
            for f in findings:
                savings = f" (节省 ${f['estimated_monthly_savings']:.2f}/月)" \
                    if f['estimated_monthly_savings'] > 0 else ""
                logger.info(f"  [{f['severity']:8s}] {f['resource_type']}: {f['description']}{savings}")

        logger.info("=" * 70)
        return report

    def run_full_inspection(self):
        """执行完整巡检"""
        logger.info("开始 GCP 全面巡检...")
        self.inspect_compute_instances()
        self.inspect_disks()
        self.inspect_storage()
        self.inspect_logs()

        report = self.generate_report()

        report_file = f'/tmp/gcp_inspection_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
        with open(report_file, 'w') as f:
            json.dump(report, f, indent=2, default=str)
        logger.info(f"报告已保存：{report_file}")

        return report

# ============================================================
# 主入口
# ============================================================

def main():
    project_id = 'my-project-id'  # 替换为你的项目 ID
    inspector = GCPResourceInspector(project_id)
    report = inspector.run_full_inspection()
    print(f"\n巡检完成，共 {report['summary']['total_findings']} 个发现")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| DefaultCredentialsError | 未配置认证 | 设置 GOOGLE_APPLICATION_CREDENTIALS |
| 403 Forbidden | API 未启用或权限不足 | 在控制台启用 API，检查 IAM 角色 |
| 配额超限 | API 调用频率过高 | 使用指数退避重试 |
| 区域资源不可用 | Region 不支持某服务 | 检查服务可用区域 |
| gRPC 超时 | 网络不稳定 | 增大 timeout 配置 |

### 7.2 最佳实践

```
[✓] 使用 Service Account 而非用户凭证进行自动化
[✓] 遵循最小权限原则分配 IAM 角色
[✓] 使用 ADC 机制，避免在代码中硬编码凭证
[✓] 为所有资源打标签（team, env, service, cost-center）
[✓] 使用 Cloud Logging 集中管理日志
[✓] 配置 GCS 生命周期策略自动管理存储
[✓] 跨项目操作使用组织级 Service Account
[✓] 定期审计 IAM 权限和 Service Account Key
```

## 8. 速查表

```python
# === 安装 ===
# pip install google-cloud-compute google-cloud-storage
# pip install google-cloud-logging google-cloud-bigquery google-cloud-monitoring

# === 认证 ===
# export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
import google.auth
credentials, project = google.auth.default()

# === Compute Engine ===
from google.cloud import compute_v1
client = compute_v1.InstancesClient()
client.list(project=PROJECT, zone=ZONE)          # 列出 VM
client.insert(project, zone, instance_resource)   # 创建 VM
client.start(project, zone, instance)             # 启动
client.stop(project, zone, instance)              # 停止
client.delete(project, zone, instance)            # 删除

# === Cloud Storage ===
from google.cloud import storage
client = storage.Client()
client.list_buckets()                             # 列出桶
client.list_blobs('bucket', prefix='path/')       # 列出对象
blob.upload_from_filename('local.txt')            # 上传
blob.download_to_filename('local.txt')            # 下载
blob.generate_signed_url(expiration=timedelta(hours=1))  # 签名 URL

# === Cloud Logging ===
from google.cloud import logging as cloud_logging
client = cloud_logging.Client()
logger = client.logger('log-name')
logger.log_text('消息', severity='INFO')
logger.log_struct({'key': 'val'}, severity='WARNING')
client.list_entries(filter_='severity>=ERROR')

# === BigQuery ===
from google.cloud import bigquery
client = bigquery.Client()
results = client.query("SELECT ...").result()

# === Cloud Monitoring ===
from google.cloud import monitoring_v3
client = monitoring_v3.MetricServiceClient()
client.list_time_series(name=project, filter=..., interval=..., view=...)
client.create_time_series(name=project, time_series=[...])
```
