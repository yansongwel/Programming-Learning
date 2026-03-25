# boto3 AWS SDK — 用 Python 管理 AWS 云资源与自动化运维

## 1. 是什么 & 为什么用它

`boto3` 是 AWS 官方提供的 Python SDK，可以通过代码操作几乎所有 AWS 服务（EC2、S3、Lambda、CloudWatch 等）。
它是 AWS 自动化运维的核心工具。

**对 SRE 的价值：**
- 自动化 AWS 资源管理（创建、巡检、清理）
- 编写成本优化脚本（发现闲置资源、预留实例推荐）
- 集成到 CI/CD 和告警响应流程
- 批量操作跨 Region、跨账号资源

## 2. 安装与环境

```bash
# 安装 boto3
pip install boto3

# 可选：AWS CLI（用于配置认证）
pip install awscli

# 验证安装
python -c "import boto3; print(boto3.__version__)"
```

**认证配置：**

```bash
# 方式一：AWS CLI 配置（推荐本地开发）
aws configure
# 输入 Access Key ID、Secret Access Key、Region、Output Format

# 方式二：环境变量
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1

# 方式三：IAM Role（EC2 / Lambda / ECS 推荐）
# 自动从实例元数据获取临时凭证，无需配置
```

## 3. 核心概念与原理

### 3.1 boto3 架构

```
+-------------------+        +-------------------+        +-------------------+
|  Python 脚本      |  HTTPS |  AWS API 端点     |        |  AWS 服务         |
|  (boto3 SDK)      |------->|  (Regional)       |------->|  (EC2/S3/...)     |
+-------------------+  签名  +-------------------+        +-------------------+
        |              请求
        |
        v
  认证链（Credential Chain）
  1. 代码中显式传入
  2. 环境变量
  3. ~/.aws/credentials
  4. IAM Role（实例/容器/Lambda）
  5. SSO / STS AssumeRole
```

### 3.2 Client vs Resource

```
类型        特点                          适用场景
─────────────────────────────────────────────────────────
Client      低级别 API，1:1 对应 AWS API   精确控制、新服务支持
Resource    高级别 OOP 封装                简单操作、代码可读性

# Client 示例
client = boto3.client('s3')
client.list_buckets()

# Resource 示例
s3 = boto3.resource('s3')
for bucket in s3.buckets.all():
    print(bucket.name)
```

### 3.3 Session 与多账号管理

```
Session
  ├── 默认 Session: boto3.client() / boto3.resource()
  ├── 自定义 Session: boto3.Session(profile_name='prod')
  └── 跨账号: STS AssumeRole -> 临时凭证 -> 新 Session
```

## 4. 基础用法

### 4.1 认证与 Session

```python
"""
boto3 认证与 Session 管理
"""
import boto3
from botocore.config import Config

# === 默认认证（自动链式查找）===
ec2 = boto3.client('ec2', region_name='us-east-1')

# === 使用 Profile ===
session = boto3.Session(profile_name='production')
ec2 = session.client('ec2')

# === 显式传入凭证 ===
ec2 = boto3.client(
    'ec2',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',
    aws_secret_access_key='wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
    region_name='us-east-1'
)

# === 跨账号访问（STS AssumeRole）===
def assume_role(role_arn, session_name='sre-session'):
    """切换到其他 AWS 账号"""
    sts = boto3.client('sts')
    response = sts.assume_role(
        RoleArn=role_arn,                            # arn:aws:iam::123456789012:role/SRERole
        RoleSessionName=session_name,
        DurationSeconds=3600                         # 临时凭证有效期
    )

    credentials = response['Credentials']
    session = boto3.Session(
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken']
    )
    return session

# === 自定义重试和超时 ===
custom_config = Config(
    region_name='us-east-1',
    retries={'max_attempts': 3, 'mode': 'adaptive'},
    connect_timeout=5,
    read_timeout=10
)
ec2 = boto3.client('ec2', config=custom_config)
```

### 4.2 EC2 实例管理

```python
"""
EC2 实例管理：列出、启动、停止、创建
"""
import boto3
from datetime import datetime

ec2_client = boto3.client('ec2', region_name='us-east-1')
ec2_resource = boto3.resource('ec2', region_name='us-east-1')

# === 列出所有实例 ===
def list_instances(filters=None):
    """列出 EC2 实例"""
    kwargs = {}
    if filters:
        kwargs['Filters'] = filters

    response = ec2_client.describe_instances(**kwargs)

    instances = []
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            # 获取实例名称
            name = ''
            for tag in instance.get('Tags', []):
                if tag['Key'] == 'Name':
                    name = tag['Value']

            info = {
                'id': instance['InstanceId'],
                'name': name,
                'type': instance['InstanceType'],
                'state': instance['State']['Name'],
                'private_ip': instance.get('PrivateIpAddress', 'N/A'),
                'public_ip': instance.get('PublicIpAddress', 'N/A'),
                'launch_time': instance['LaunchTime'].isoformat(),
                'az': instance['Placement']['AvailabilityZone'],
            }
            instances.append(info)
            print(
                f"  {info['id']:20s} {info['name']:30s} "
                f"{info['type']:15s} {info['state']:10s} "
                f"{info['private_ip']:15s}"
            )

    return instances

# 按标签过滤
# list_instances(filters=[{'Name': 'tag:Environment', 'Values': ['production']}])
# list_instances(filters=[{'Name': 'instance-state-name', 'Values': ['running']}])

# === 启动/停止实例 ===
def start_instances(instance_ids):
    """启动实例"""
    response = ec2_client.start_instances(InstanceIds=instance_ids)
    for change in response['StartingInstances']:
        print(f"  {change['InstanceId']}: {change['PreviousState']['Name']} -> {change['CurrentState']['Name']}")

def stop_instances(instance_ids):
    """停止实例"""
    response = ec2_client.stop_instances(InstanceIds=instance_ids)
    for change in response['StoppingInstances']:
        print(f"  {change['InstanceId']}: {change['PreviousState']['Name']} -> {change['CurrentState']['Name']}")

# === 创建实例 ===
def create_instance(image_id, instance_type='t3.micro', key_name=None,
                    subnet_id=None, security_groups=None, name=''):
    """创建 EC2 实例"""
    params = {
        'ImageId': image_id,
        'InstanceType': instance_type,
        'MinCount': 1,
        'MaxCount': 1,
        'TagSpecifications': [{
            'ResourceType': 'instance',
            'Tags': [
                {'Key': 'Name', 'Value': name},
                {'Key': 'ManagedBy', 'Value': 'sre-automation'},
                {'Key': 'CreatedAt', 'Value': datetime.now().isoformat()},
            ]
        }]
    }

    if key_name:
        params['KeyName'] = key_name
    if subnet_id:
        params['SubnetId'] = subnet_id
    if security_groups:
        params['SecurityGroupIds'] = security_groups

    response = ec2_client.run_instances(**params)
    instance_id = response['Instances'][0]['InstanceId']
    print(f"实例已创建：{instance_id}")

    # 等待实例运行
    waiter = ec2_client.get_waiter('instance_running')
    waiter.wait(InstanceIds=[instance_id])
    print(f"实例已运行：{instance_id}")

    return instance_id
```

### 4.3 S3 存储操作

```python
"""
S3 存储操作：桶管理、文件上传下载、批量操作
"""
import boto3
import os
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')
s3_resource = boto3.resource('s3')

# === 列出所有桶 ===
def list_buckets():
    """列出所有 S3 桶"""
    response = s3_client.list_buckets()
    for bucket in response['Buckets']:
        print(f"  {bucket['Name']:40s} 创建于: {bucket['CreationDate']}")
    return response['Buckets']

# === 上传文件 ===
def upload_file(local_path, bucket, s3_key, metadata=None):
    """上传文件到 S3"""
    extra_args = {}
    if metadata:
        extra_args['Metadata'] = metadata

    s3_client.upload_file(
        local_path, bucket, s3_key,
        ExtraArgs=extra_args
    )
    print(f"已上传：{local_path} -> s3://{bucket}/{s3_key}")

# === 下载文件 ===
def download_file(bucket, s3_key, local_path):
    """从 S3 下载文件"""
    os.makedirs(os.path.dirname(local_path), exist_ok=True)
    s3_client.download_file(bucket, s3_key, local_path)
    print(f"已下载：s3://{bucket}/{s3_key} -> {local_path}")

# === 列出对象 ===
def list_objects(bucket, prefix='', max_keys=1000):
    """列出 S3 桶中的对象"""
    paginator = s3_client.get_paginator('list_objects_v2')
    total_size = 0
    total_count = 0

    for page in paginator.paginate(Bucket=bucket, Prefix=prefix, MaxKeys=max_keys):
        for obj in page.get('Contents', []):
            size_mb = obj['Size'] / 1024 / 1024
            total_size += obj['Size']
            total_count += 1
            print(f"  {obj['Key']:60s} {size_mb:10.2f}MB  {obj['LastModified']}")

    print(f"\n总计：{total_count} 个对象，{total_size / 1024 / 1024:.2f}MB")

# === 生成预签名 URL ===
def generate_presigned_url(bucket, key, expiration=3600):
    """生成预签名下载 URL"""
    url = s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': bucket, 'Key': key},
        ExpiresIn=expiration
    )
    print(f"预签名 URL（{expiration}秒有效）：{url}")
    return url

# === 批量删除 ===
def delete_objects(bucket, prefix):
    """删除指定前缀下的所有对象"""
    bucket_resource = s3_resource.Bucket(bucket)
    objects = list(bucket_resource.objects.filter(Prefix=prefix))

    if not objects:
        print("没有找到要删除的对象")
        return

    print(f"将删除 {len(objects)} 个对象...")
    delete_list = [{'Key': obj.key} for obj in objects]

    # 分批删除（每批最多 1000 个）
    for i in range(0, len(delete_list), 1000):
        batch = delete_list[i:i+1000]
        s3_client.delete_objects(
            Bucket=bucket,
            Delete={'Objects': batch}
        )

    print(f"已删除 {len(delete_list)} 个对象")
```

### 4.4 CloudWatch 监控

```python
"""
CloudWatch：指标查询和告警管理
"""
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

# === 查询指标 ===
def get_ec2_cpu_metrics(instance_id, hours=1):
    """获取 EC2 实例 CPU 使用率"""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=hours)

    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,            # 5 分钟粒度
        Statistics=['Average', 'Maximum']
    )

    datapoints = sorted(response['Datapoints'], key=lambda x: x['Timestamp'])
    for dp in datapoints:
        print(
            f"  {dp['Timestamp'].strftime('%H:%M')} "
            f"平均: {dp['Average']:.1f}%  最大: {dp['Maximum']:.1f}%"
        )

    return datapoints

# === 创建告警 ===
def create_cpu_alarm(instance_id, threshold=80):
    """为 EC2 实例创建 CPU 告警"""
    cloudwatch.put_metric_alarm(
        AlarmName=f'ec2-cpu-high-{instance_id}',
        AlarmDescription=f'EC2 {instance_id} CPU 使用率超过 {threshold}%',
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
        Statistic='Average',
        Period=300,
        EvaluationPeriods=3,
        Threshold=threshold,
        ComparisonOperator='GreaterThanThreshold',
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:sre-alerts'
        ],
        Tags=[
            {'Key': 'Team', 'Value': 'SRE'},
            {'Key': 'ManagedBy', 'Value': 'automation'}
        ]
    )
    print(f"告警已创建：ec2-cpu-high-{instance_id}")

# === 发送自定义指标 ===
def put_custom_metric(namespace, metric_name, value, unit='None', dimensions=None):
    """发送自定义指标到 CloudWatch"""
    metric_data = {
        'MetricName': metric_name,
        'Value': value,
        'Unit': unit,
        'Timestamp': datetime.utcnow()
    }
    if dimensions:
        metric_data['Dimensions'] = [
            {'Name': k, 'Value': v} for k, v in dimensions.items()
        ]

    cloudwatch.put_metric_data(
        Namespace=namespace,
        MetricData=[metric_data]
    )
```

### 4.5 IAM 管理

```python
"""
IAM 管理：用户、角色、策略
"""
import boto3
import json

iam = boto3.client('iam')

# === 列出用户 ===
def list_users():
    """列出所有 IAM 用户"""
    paginator = iam.get_paginator('list_users')
    for page in paginator.paginate():
        for user in page['Users']:
            print(f"  {user['UserName']:30s} 创建于: {user['CreateDate']}")

# === 查找未使用的访问密钥 ===
def find_unused_keys(days_threshold=90):
    """查找长期未使用的 Access Key"""
    from datetime import datetime, timezone

    unused_keys = []
    paginator = iam.get_paginator('list_users')

    for page in paginator.paginate():
        for user in page['Users']:
            keys = iam.list_access_keys(UserName=user['UserName'])
            for key in keys['AccessKeyMetadata']:
                if key['Status'] == 'Active':
                    # 获取最后使用时间
                    try:
                        last_used = iam.get_access_key_last_used(
                            AccessKeyId=key['AccessKeyId']
                        )
                        last_used_date = last_used['AccessKeyLastUsed'].get('LastUsedDate')

                        if last_used_date:
                            days_unused = (datetime.now(timezone.utc) - last_used_date).days
                        else:
                            days_unused = (datetime.now(timezone.utc) - key['CreateDate']).days

                        if days_unused > days_threshold:
                            unused_keys.append({
                                'user': user['UserName'],
                                'key_id': key['AccessKeyId'],
                                'days_unused': days_unused,
                                'created': key['CreateDate'].isoformat()
                            })
                            print(
                                f"  [未使用] {user['UserName']:30s} "
                                f"{key['AccessKeyId']} ({days_unused} 天未使用)"
                            )
                    except Exception as e:
                        print(f"  检查失败：{user['UserName']} - {e}")

    return unused_keys

# === 审计 MFA 状态 ===
def audit_mfa():
    """审计用户 MFA 启用状态"""
    paginator = iam.get_paginator('list_users')
    no_mfa_users = []

    for page in paginator.paginate():
        for user in page['Users']:
            mfa_devices = iam.list_mfa_devices(UserName=user['UserName'])
            if not mfa_devices['MFADevices']:
                no_mfa_users.append(user['UserName'])
                print(f"  [无 MFA] {user['UserName']}")

    print(f"\n共 {len(no_mfa_users)} 个用户未启用 MFA")
    return no_mfa_users
```

## 5. 进阶用法

### 5.1 Lambda 函数管理

```python
"""
Lambda 函数管理：部署、调用、日志
"""
import boto3
import json
import zipfile
import io
import base64

lambda_client = boto3.client('lambda', region_name='us-east-1')

# === 创建 Lambda 函数 ===
def create_lambda_function(function_name, handler_code, role_arn,
                           runtime='python3.11', timeout=30, memory=128):
    """创建 Lambda 函数"""
    # 将代码打包为 ZIP
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
        zf.writestr('lambda_function.py', handler_code)
    zip_buffer.seek(0)

    response = lambda_client.create_function(
        FunctionName=function_name,
        Runtime=runtime,
        Role=role_arn,
        Handler='lambda_function.lambda_handler',
        Code={'ZipFile': zip_buffer.read()},
        Timeout=timeout,
        MemorySize=memory,
        Environment={
            'Variables': {
                'ENV': 'production',
                'LOG_LEVEL': 'INFO'
            }
        },
        Tags={
            'Team': 'SRE',
            'ManagedBy': 'automation'
        }
    )
    print(f"Lambda 函数已创建：{response['FunctionArn']}")

# === 调用 Lambda ===
def invoke_lambda(function_name, payload=None):
    """调用 Lambda 函数"""
    params = {
        'FunctionName': function_name,
        'InvocationType': 'RequestResponse',  # 同步调用
    }
    if payload:
        params['Payload'] = json.dumps(payload)

    response = lambda_client.invoke(**params)

    result = json.loads(response['Payload'].read())
    status_code = response['StatusCode']
    print(f"调用结果（状态码 {status_code}）：{result}")
    return result

# === 更新 Lambda 代码 ===
def update_lambda_code(function_name, handler_code):
    """更新 Lambda 函数代码"""
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
        zf.writestr('lambda_function.py', handler_code)
    zip_buffer.seek(0)

    response = lambda_client.update_function_code(
        FunctionName=function_name,
        ZipFile=zip_buffer.read()
    )
    print(f"Lambda 代码已更新：{response['LastModified']}")
```

### 5.2 多 Region 操作

```python
"""
跨 Region 批量操作
"""
import boto3
from concurrent.futures import ThreadPoolExecutor, as_completed

def get_all_regions():
    """获取所有启用的 Region"""
    ec2 = boto3.client('ec2', region_name='us-east-1')
    regions = ec2.describe_regions(
        Filters=[{'Name': 'opt-in-status', 'Values': ['opt-in-not-required', 'opted-in']}]
    )
    return [r['RegionName'] for r in regions['Regions']]

def scan_all_regions(service_name, scan_func, max_workers=10):
    """
    并发扫描所有 Region

    参数：
        service_name: AWS 服务名
        scan_func: 扫描函数，接收 (client, region) 参数
        max_workers: 最大并发数
    """
    regions = get_all_regions()
    results = {}

    def scan_region(region):
        client = boto3.client(service_name, region_name=region)
        return region, scan_func(client, region)

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(scan_region, r): r for r in regions}
        for future in as_completed(futures):
            region = futures[future]
            try:
                region_name, result = future.result()
                results[region_name] = result
            except Exception as e:
                results[region] = {'error': str(e)}

    return results

# 示例：扫描所有 Region 的运行实例
def count_running_instances(client, region):
    """统计指定 Region 的运行实例数"""
    response = client.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    count = sum(len(r['Instances']) for r in response['Reservations'])
    if count > 0:
        print(f"  {region}: {count} 个运行实例")
    return {'count': count}

# all_instances = scan_all_regions('ec2', count_running_instances)
```

## 6. SRE 实战案例：AWS 资源巡检与成本优化脚本

```python
"""
SRE 实战：AWS 资源巡检与成本优化
功能：
  1. 扫描闲置 EC2 实例（低 CPU 使用率）
  2. 查找未挂载的 EBS 卷
  3. 检查未使用的 Elastic IP
  4. 审计安全组规则
  5. S3 存储成本分析
  6. IAM 安全审计
  7. 生成优化报告
"""
import boto3
import json
import logging
from datetime import datetime, timedelta, timezone
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor, as_completed

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger('aws-inspector')

class AWSResourceInspector:
    """
    AWS 资源巡检与成本优化工具
    """

    def __init__(self, region='us-east-1', profile=None):
        """
        初始化巡检工具

        参数：
            region: AWS 区域
            profile: AWS Profile 名称
        """
        session_kwargs = {'region_name': region}
        if profile:
            session_kwargs['profile_name'] = profile

        self.session = boto3.Session(**session_kwargs)
        self.region = region
        self.findings = []
        self.cost_savings = 0.0

        logger.info(f"AWS 巡检工具已初始化（Region: {region}）")

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
            'region': self.region,
            'timestamp': datetime.now().isoformat()
        }
        self.findings.append(finding)
        self.cost_savings += estimated_savings

        level = {'critical': 40, 'high': 30, 'medium': 20, 'low': 10}.get(severity, 20)
        logger.log(level, f"[{severity:8s}] {resource_type}/{resource_id}: {description}")

    # ================================================================
    # EC2 巡检
    # ================================================================

    def inspect_ec2(self, cpu_threshold=10, days=7):
        """
        巡检 EC2 实例

        参数：
            cpu_threshold: CPU 使用率阈值（低于此值认为闲置）
            days: 统计周期（天）
        """
        logger.info("开始 EC2 巡检...")
        ec2 = self.session.client('ec2')
        cloudwatch = self.session.client('cloudwatch')

        # 获取所有运行中的实例
        response = ec2.describe_instances(
            Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
        )

        instances = []
        for reservation in response['Reservations']:
            instances.extend(reservation['Instances'])

        logger.info(f"发现 {len(instances)} 个运行中的实例")

        # 定义实例类型的大致小时费率（简化，实际应查询 Pricing API）
        pricing_estimate = {
            't3.micro': 0.0104, 't3.small': 0.0208, 't3.medium': 0.0416,
            't3.large': 0.0832, 'm5.large': 0.096, 'm5.xlarge': 0.192,
            'c5.large': 0.085, 'r5.large': 0.126,
        }

        for instance in instances:
            instance_id = instance['InstanceId']
            instance_type = instance['InstanceType']
            name = ''
            for tag in instance.get('Tags', []):
                if tag['Key'] == 'Name':
                    name = tag['Value']

            # 获取 CPU 指标
            end_time = datetime.utcnow()
            start_time = end_time - timedelta(days=days)

            try:
                metrics = cloudwatch.get_metric_statistics(
                    Namespace='AWS/EC2',
                    MetricName='CPUUtilization',
                    Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                    StartTime=start_time,
                    EndTime=end_time,
                    Period=3600,
                    Statistics=['Average']
                )

                if metrics['Datapoints']:
                    avg_cpu = sum(dp['Average'] for dp in metrics['Datapoints']) / len(metrics['Datapoints'])

                    if avg_cpu < cpu_threshold:
                        hourly_cost = pricing_estimate.get(instance_type, 0.05)
                        monthly_savings = hourly_cost * 24 * 30

                        self._add_finding(
                            category='cost_optimization',
                            severity='medium',
                            resource_id=f"{instance_id} ({name})",
                            resource_type='EC2',
                            description=(
                                f'实例 {days} 天平均 CPU 仅 {avg_cpu:.1f}%，'
                                f'类型 {instance_type}，建议缩小规格或停止'
                            ),
                            estimated_savings=monthly_savings
                        )
                else:
                    self._add_finding(
                        category='monitoring',
                        severity='low',
                        resource_id=instance_id,
                        resource_type='EC2',
                        description='无 CloudWatch 数据，可能 monitoring 未启用'
                    )

            except Exception as e:
                logger.warning(f"获取 {instance_id} 指标失败：{e}")

    # ================================================================
    # EBS 巡检
    # ================================================================

    def inspect_ebs(self):
        """巡检 EBS 卷：查找未挂载的卷"""
        logger.info("开始 EBS 巡检...")
        ec2 = self.session.client('ec2')

        # 查找未挂载的卷
        response = ec2.describe_volumes(
            Filters=[{'Name': 'status', 'Values': ['available']}]
        )

        for volume in response['Volumes']:
            vol_id = volume['VolumeId']
            size_gb = volume['Size']
            vol_type = volume['VolumeType']
            created = volume['CreateTime']

            # 估算月费用（简化）
            price_per_gb = {'gp3': 0.08, 'gp2': 0.10, 'io1': 0.125, 'st1': 0.045, 'sc1': 0.015}
            monthly_cost = size_gb * price_per_gb.get(vol_type, 0.10)

            age_days = (datetime.now(timezone.utc) - created).days

            self._add_finding(
                category='cost_optimization',
                severity='medium' if age_days > 30 else 'low',
                resource_id=vol_id,
                resource_type='EBS',
                description=(
                    f'未挂载的 EBS 卷：{size_gb}GB {vol_type}，'
                    f'已存在 {age_days} 天'
                ),
                estimated_savings=monthly_cost
            )

        # 查找旧快照
        snapshots = ec2.describe_snapshots(OwnerIds=['self'])
        old_snapshots = []
        for snap in snapshots['Snapshots']:
            age_days = (datetime.now(timezone.utc) - snap['StartTime']).days
            if age_days > 90:
                old_snapshots.append(snap)

        if old_snapshots:
            self._add_finding(
                category='cost_optimization',
                severity='low',
                resource_id=f'{len(old_snapshots)} 个旧快照',
                resource_type='EBS Snapshot',
                description=f'发现 {len(old_snapshots)} 个超过 90 天的 EBS 快照，建议审查清理'
            )

    # ================================================================
    # Elastic IP 巡检
    # ================================================================

    def inspect_eip(self):
        """巡检未使用的 Elastic IP"""
        logger.info("开始 Elastic IP 巡检...")
        ec2 = self.session.client('ec2')

        response = ec2.describe_addresses()

        for addr in response['Addresses']:
            if 'AssociationId' not in addr:
                self._add_finding(
                    category='cost_optimization',
                    severity='medium',
                    resource_id=addr['PublicIp'],
                    resource_type='Elastic IP',
                    description='未关联的 Elastic IP（闲置 EIP 会产生费用）',
                    estimated_savings=3.65  # $0.005/小时 * 730 小时
                )

    # ================================================================
    # 安全组巡检
    # ================================================================

    def inspect_security_groups(self):
        """巡检安全组规则"""
        logger.info("开始安全组巡检...")
        ec2 = self.session.client('ec2')

        response = ec2.describe_security_groups()

        for sg in response['SecurityGroups']:
            sg_id = sg['GroupId']
            sg_name = sg['GroupName']

            for rule in sg['IpPermissions']:
                for ip_range in rule.get('IpRanges', []):
                    cidr = ip_range.get('CidrIp', '')
                    if cidr == '0.0.0.0/0':
                        from_port = rule.get('FromPort', 'all')
                        to_port = rule.get('ToPort', 'all')

                        # 关键端口开放给全网是严重问题
                        critical_ports = {22: 'SSH', 3389: 'RDP', 3306: 'MySQL',
                                         5432: 'PostgreSQL', 6379: 'Redis', 27017: 'MongoDB'}

                        if from_port in critical_ports:
                            self._add_finding(
                                category='security',
                                severity='critical',
                                resource_id=f'{sg_id} ({sg_name})',
                                resource_type='Security Group',
                                description=(
                                    f'{critical_ports[from_port]}端口({from_port})对全网开放，'
                                    f'存在安全风险'
                                )
                            )
                        elif from_port == -1 or from_port == 0:
                            self._add_finding(
                                category='security',
                                severity='high',
                                resource_id=f'{sg_id} ({sg_name})',
                                resource_type='Security Group',
                                description='所有端口对全网 (0.0.0.0/0) 开放'
                            )

    # ================================================================
    # IAM 安全审计
    # ================================================================

    def inspect_iam(self, key_age_threshold=90):
        """IAM 安全审计"""
        logger.info("开始 IAM 安全审计...")
        iam = self.session.client('iam')

        # 检查 root 账号 MFA
        try:
            summary = iam.get_account_summary()
            if summary['SummaryMap'].get('AccountMFAEnabled', 0) == 0:
                self._add_finding(
                    category='security',
                    severity='critical',
                    resource_id='root',
                    resource_type='IAM',
                    description='Root 账号未启用 MFA'
                )
        except Exception:
            pass

        # 检查用户密钥年龄和 MFA
        paginator = iam.get_paginator('list_users')
        for page in paginator.paginate():
            for user in page['Users']:
                username = user['UserName']

                # 检查 MFA
                mfa = iam.list_mfa_devices(UserName=username)
                if not mfa['MFADevices']:
                    # 检查用户是否有控制台密码
                    try:
                        iam.get_login_profile(UserName=username)
                        self._add_finding(
                            category='security',
                            severity='high',
                            resource_id=username,
                            resource_type='IAM User',
                            description='用户有控制台访问权限但未启用 MFA'
                        )
                    except iam.exceptions.NoSuchEntityException:
                        pass

                # 检查密钥年龄
                keys = iam.list_access_keys(UserName=username)
                for key in keys['AccessKeyMetadata']:
                    if key['Status'] == 'Active':
                        age_days = (datetime.now(timezone.utc) - key['CreateDate']).days
                        if age_days > key_age_threshold:
                            self._add_finding(
                                category='security',
                                severity='high',
                                resource_id=f"{username}/{key['AccessKeyId']}",
                                resource_type='IAM Access Key',
                                description=f'访问密钥已使用 {age_days} 天，建议轮换'
                            )

    # ================================================================
    # 生成报告
    # ================================================================

    def generate_report(self):
        """生成巡检报告"""
        # 统计
        by_category = defaultdict(list)
        by_severity = defaultdict(int)

        for f in self.findings:
            by_category[f['category']].append(f)
            by_severity[f['severity']] += 1

        report = {
            'timestamp': datetime.now().isoformat(),
            'region': self.region,
            'summary': {
                'total_findings': len(self.findings),
                'by_severity': dict(by_severity),
                'by_category': {k: len(v) for k, v in by_category.items()},
                'estimated_monthly_savings': round(self.cost_savings, 2),
            },
            'findings': self.findings,
        }

        # 打印报告
        logger.info("=" * 70)
        logger.info("AWS 资源巡检报告")
        logger.info("=" * 70)
        logger.info(f"区域：{self.region}")
        logger.info(f"总发现：{len(self.findings)}")
        logger.info(f"严重度分布：{dict(by_severity)}")
        logger.info(f"预估月节省：${self.cost_savings:.2f}")
        logger.info("-" * 70)

        for category, findings in by_category.items():
            logger.info(f"\n[{category}] ({len(findings)} 项)")
            for f in findings:
                savings_str = f" (可节省 ${f['estimated_monthly_savings']:.2f}/月)" \
                    if f['estimated_monthly_savings'] > 0 else ""
                logger.info(
                    f"  [{f['severity']:8s}] {f['resource_type']}/{f['resource_id']}: "
                    f"{f['description']}{savings_str}"
                )

        logger.info("=" * 70)
        return report

    def run_full_inspection(self):
        """执行完整巡检"""
        logger.info("开始 AWS 全面巡检...")

        self.inspect_ec2()
        self.inspect_ebs()
        self.inspect_eip()
        self.inspect_security_groups()
        self.inspect_iam()

        report = self.generate_report()

        # 保存报告
        report_file = f'/tmp/aws_inspection_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
        with open(report_file, 'w') as f:
            json.dump(report, f, indent=2, default=str)
        logger.info(f"报告已保存：{report_file}")

        return report


# ============================================================
# 主入口
# ============================================================

def main():
    inspector = AWSResourceInspector(region='us-east-1')
    report = inspector.run_full_inspection()
    print(f"\n巡检完成，共 {report['summary']['total_findings']} 个发现")
    print(f"预估月节省：${report['summary']['estimated_monthly_savings']:.2f}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 7.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| NoCredentialsError | 未配置认证 | 检查 ~/.aws/credentials 或环境变量 |
| AccessDenied | IAM 权限不足 | 检查并添加所需 IAM 策略 |
| RegionDisabledException | Region 未启用 | 在 AWS 控制台启用 Region |
| 请求限流 (Throttling) | API 调用过快 | 使用 adaptive 重试模式 |
| 大量资源分页遗漏 | 未使用 Paginator | 使用 `get_paginator()` |

### 7.2 最佳实践

```
[✓] 使用 IAM Role 而非长期密钥
[✓] 最小权限原则配置 IAM 策略
[✓] 使用 Paginator 处理大量资源
[✓] 配置 adaptive 重试模式应对限流
[✓] 跨账号使用 STS AssumeRole
[✓] 敏感操作添加 dry-run 模式
[✓] 为自动创建的资源打标签
[✓] 使用 Session 管理多账号/多 Region
[✓] 定期轮换 Access Key
```

## 8. 速查表

```python
# === 安装 ===
# pip install boto3

# === 连接 ===
import boto3
client = boto3.client('ec2', region_name='us-east-1')
resource = boto3.resource('s3')
session = boto3.Session(profile_name='prod')

# === EC2 ===
ec2 = boto3.client('ec2')
ec2.describe_instances(Filters=[{'Name':'instance-state-name','Values':['running']}])
ec2.start_instances(InstanceIds=['i-xxx'])
ec2.stop_instances(InstanceIds=['i-xxx'])
ec2.run_instances(ImageId='ami-xxx', InstanceType='t3.micro', MinCount=1, MaxCount=1)
ec2.terminate_instances(InstanceIds=['i-xxx'])

# === S3 ===
s3 = boto3.client('s3')
s3.list_buckets()
s3.upload_file('local.txt', 'bucket', 'key.txt')
s3.download_file('bucket', 'key.txt', 'local.txt')
s3.generate_presigned_url('get_object', Params={'Bucket':'b','Key':'k'}, ExpiresIn=3600)
# 分页
paginator = s3.get_paginator('list_objects_v2')
for page in paginator.paginate(Bucket='bucket'): ...

# === CloudWatch ===
cw = boto3.client('cloudwatch')
cw.get_metric_statistics(Namespace='AWS/EC2', MetricName='CPUUtilization', ...)
cw.put_metric_data(Namespace='Custom', MetricData=[...])
cw.put_metric_alarm(AlarmName='name', ...)

# === Lambda ===
lam = boto3.client('lambda')
lam.invoke(FunctionName='func', Payload=json.dumps({}))
lam.update_function_code(FunctionName='func', ZipFile=bytes)

# === IAM ===
iam = boto3.client('iam')
iam.list_users()
iam.list_access_keys(UserName='user')
iam.get_access_key_last_used(AccessKeyId='AKIA...')

# === STS 跨账号 ===
sts = boto3.client('sts')
resp = sts.assume_role(RoleArn='arn:...', RoleSessionName='session')
session = boto3.Session(
    aws_access_key_id=resp['Credentials']['AccessKeyId'],
    aws_secret_access_key=resp['Credentials']['SecretAccessKey'],
    aws_session_token=resp['Credentials']['SessionToken']
)
```
