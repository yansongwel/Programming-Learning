# Jenkins API — 用 python-jenkins 自动化流水线管理

## 1. 是什么 & 为什么用它

`python-jenkins` 是 Python 操作 Jenkins 的官方客户端库，通过 Jenkins REST API 实现自动化管理。
对 SRE 来说，Jenkins 自动化是 CI/CD 的核心：

- **批量管理 Job**：创建、修改、触发数百个 Job
- **构建监控**：自动监控构建状态，失败告警
- **流水线编排**：程序化管理 Pipeline
- **数据导出**：构建历史、统计分析

手动操作 Jenkins Web UI 的痛点：
- 重复性操作多（创建类似 Job）
- 无法批量操作
- 无法纳入版本控制
- 缺乏审计追踪

## 2. 安装与环境

```bash
# 安装 python-jenkins
pip install python-jenkins

# 验证
python3 -c "import jenkins; print('python-jenkins 可用')"

# Jenkins 端需要：
# 1. 创建 API Token（用户设置 → API Token）
# 2. 安装必要插件
```

## 3. 核心概念与原理

### 3.1 Jenkins 核心概念

```
Jenkins Master
  ├── Job（任务）
  │   ├── Freestyle Project（自由风格项目）
  │   ├── Pipeline（流水线）
  │   └── Multibranch Pipeline（多分支流水线）
  ├── Build（构建）
  │   ├── 构建号 #1, #2, #3...
  │   ├── 构建参数
  │   ├── 构建日志
  │   └── 构建产物
  ├── Node/Agent（节点/代理）
  │   ├── Master
  │   └── Slave nodes
  ├── View（视图）
  └── Credentials（凭据）
```

### 3.2 API 认证方式

```python
# 方式 1: 用户名 + API Token（推荐）
server = jenkins.Jenkins('http://jenkins:8080', username='admin', password='api-token')

# 方式 2: 用户名 + 密码（不推荐）
server = jenkins.Jenkins('http://jenkins:8080', username='admin', password='password')

# 方式 3: 匿名访问（仅允许公开操作）
server = jenkins.Jenkins('http://jenkins:8080')
```

## 4. 基础用法

### 4.1 连接与基本操作

```python
#!/usr/bin/env python3
"""Jenkins 连接与基本操作"""

import jenkins
import json

# === 连接 Jenkins ===
# 注意：请替换为实际的 Jenkins 地址和凭据
JENKINS_URL = "http://localhost:8080"
JENKINS_USER = "admin"
JENKINS_TOKEN = "your-api-token"

def get_jenkins_client():
    """获取 Jenkins 客户端"""
    try:
        server = jenkins.Jenkins(
            JENKINS_URL,
            username=JENKINS_USER,
            password=JENKINS_TOKEN,
            timeout=30,
        )
        # 验证连接
        user = server.get_whoami()
        version = server.get_version()
        print(f"Jenkins 版本: {version}")
        print(f"当前用户: {user['fullName']}")
        return server
    except Exception as e:
        print(f"连接失败: {e}")
        return None

# === 模拟 Jenkins 客户端（用于演示） ===
class MockJenkins:
    """模拟 Jenkins 客户端，用于无 Jenkins 环境的演示"""

    def __init__(self):
        self.jobs = {
            'web-app-build': {
                'name': 'web-app-build', 'url': 'http://jenkins:8080/job/web-app-build/',
                'color': 'blue', 'nextBuildNumber': 42,
                'lastBuild': {'number': 41, 'result': 'SUCCESS', 'duration': 120000},
            },
            'api-deploy-staging': {
                'name': 'api-deploy-staging', 'url': 'http://jenkins:8080/job/api-deploy-staging/',
                'color': 'red', 'nextBuildNumber': 15,
                'lastBuild': {'number': 14, 'result': 'FAILURE', 'duration': 300000},
            },
            'infra-terraform-plan': {
                'name': 'infra-terraform-plan', 'url': 'http://jenkins:8080/job/infra-terraform-plan/',
                'color': 'blue', 'nextBuildNumber': 88,
                'lastBuild': {'number': 87, 'result': 'SUCCESS', 'duration': 60000},
            },
        }

    def get_whoami(self):
        return {'fullName': 'Admin User'}

    def get_version(self):
        return '2.426.1'

    def get_jobs(self):
        return [{'name': k, 'url': v['url'], 'color': v['color']} for k, v in self.jobs.items()]

    def get_job_info(self, name):
        return self.jobs.get(name, {})

    def get_build_info(self, name, number):
        job = self.jobs.get(name, {})
        return {
            'number': number,
            'result': 'SUCCESS',
            'duration': 120000,
            'timestamp': 1705286400000,
            'building': False,
            'url': f"{job.get('url', '')}{number}/",
        }

    def build_job(self, name, parameters=None):
        print(f"  [模拟] 触发构建: {name}, 参数: {parameters}")
        return 42

    def get_queue_info(self):
        return []

# 使用模拟客户端演示
server = MockJenkins()

# === 获取所有 Job ===
print("\n=== Jenkins Job 列表 ===")
jobs = server.get_jobs()
for job in jobs:
    status = {'blue': '成功', 'red': '失败', 'yellow': '不稳定', 'disabled': '禁用',
              'notbuilt': '未构建', 'aborted': '中止'}.get(job['color'], job['color'])
    print(f"  [{status}] {job['name']}")

# === 获取 Job 详情 ===
print("\n=== Job 详情 ===")
job_info = server.get_job_info('web-app-build')
print(f"  名称: {job_info['name']}")
print(f"  下次构建号: {job_info['nextBuildNumber']}")
print(f"  最后构建: #{job_info['lastBuild']['number']} "
      f"({job_info['lastBuild']['result']})")
```

### 4.2 Job 管理（创建/修改/删除）

```python
#!/usr/bin/env python3
"""Jenkins Job 管理"""

# === 创建 Freestyle Job ===
FREESTYLE_CONFIG = """<?xml version='1.0' encoding='UTF-8'?>
<project>
  <description>SRE 自动化创建的 Job</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <hudson.model.ParametersDefinitionProperty>
      <parameterDefinitions>
        <hudson.model.StringParameterDefinition>
          <name>ENVIRONMENT</name>
          <description>部署环境</description>
          <defaultValue>staging</defaultValue>
        </hudson.model.StringParameterDefinition>
        <hudson.model.ChoiceParameterDefinition>
          <name>ACTION</name>
          <description>操作类型</description>
          <choices>
            <string>deploy</string>
            <string>rollback</string>
            <string>restart</string>
          </choices>
        </hudson.model.ChoiceParameterDefinition>
      </parameterDefinitions>
    </hudson.model.ParametersDefinitionProperty>
  </properties>
  <builders>
    <hudson.tasks.Shell>
      <command>
echo "环境: ${ENVIRONMENT}"
echo "操作: ${ACTION}"
echo "开始执行..."
# 实际部署脚本
      </command>
    </hudson.tasks.Shell>
  </builders>
</project>"""

# === 创建 Pipeline Job ===
PIPELINE_CONFIG = """<?xml version='1.0' encoding='UTF-8'?>
<flow-definition>
  <description>SRE Pipeline 示例</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <hudson.model.ParametersDefinitionProperty>
      <parameterDefinitions>
        <hudson.model.StringParameterDefinition>
          <name>GIT_BRANCH</name>
          <defaultValue>main</defaultValue>
        </hudson.model.StringParameterDefinition>
      </parameterDefinitions>
    </hudson.model.ParametersDefinitionProperty>
  </properties>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition">
    <script>
pipeline {
    agent any
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main')
    }
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out ${params.GIT_BRANCH}"
            }
        }
        stage('Build') {
            steps {
                echo "Building..."
                sh 'echo "Build complete"'
            }
        }
        stage('Test') {
            steps {
                echo "Testing..."
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploying..."
            }
        }
    }
    post {
        failure {
            echo "构建失败！需要处理"
        }
        success {
            echo "构建成功！"
        }
    }
}
    </script>
    <sandbox>true</sandbox>
  </definition>
</flow-definition>"""

print("=== Job 配置模板已准备 ===")
print(f"  Freestyle Job XML: {len(FREESTYLE_CONFIG)} 字符")
print(f"  Pipeline Job XML: {len(PIPELINE_CONFIG)} 字符")

# 实际使用：
# server.create_job('my-freestyle-job', FREESTYLE_CONFIG)
# server.create_job('my-pipeline-job', PIPELINE_CONFIG)
# server.reconfig_job('my-job', new_config)  # 修改
# server.delete_job('my-job')                 # 删除
# server.enable_job('my-job')                 # 启用
# server.disable_job('my-job')                # 禁用

# === 批量创建 Job ===
def batch_create_jobs(server, jobs_config: list):
    """批量创建 Jenkins Job"""
    results = []
    for job in jobs_config:
        name = job['name']
        config = job['config']
        try:
            if name in [j['name'] for j in server.get_jobs()]:
                print(f"  [跳过] {name} 已存在")
                results.append({'name': name, 'status': 'exists'})
            else:
                server.create_job(name, config)
                print(f"  [创建] {name}")
                results.append({'name': name, 'status': 'created'})
        except Exception as e:
            print(f"  [失败] {name}: {e}")
            results.append({'name': name, 'status': f'error: {e}'})
    return results

# 批量创建示例
print("\n=== 批量创建 Job（模拟） ===")
environments = ['dev', 'staging', 'prod']
for env in environments:
    print(f"  [模拟创建] deploy-{env}")
```

### 4.3 触发构建与监控

```python
#!/usr/bin/env python3
"""触发构建与状态监控"""

import time
from datetime import datetime

# 使用模拟客户端
class MockJenkinsExtended:
    """扩展模拟客户端"""

    def build_job(self, name, parameters=None):
        """触发构建"""
        print(f"  触发构建: {name}")
        if parameters:
            for k, v in parameters.items():
                print(f"    参数: {k} = {v}")
        return 42  # 返回队列 ID

    def get_build_info(self, name, number):
        return {
            'number': number,
            'result': 'SUCCESS',
            'building': False,
            'duration': 125000,
            'timestamp': int(time.time() * 1000),
            'url': f'http://jenkins:8080/job/{name}/{number}/',
        }

    def get_build_console_output(self, name, number):
        return f"""Started by user Admin
Building in workspace /var/jenkins/workspace/{name}
[{name}] $ /bin/sh -xe /tmp/script.sh
+ echo "部署开始..."
部署开始...
+ echo "环境: staging"
环境: staging
+ echo "部署完成"
部署完成
Finished: SUCCESS"""

server = MockJenkinsExtended()

# === 触发参数化构建 ===
def trigger_build(server, job_name: str, parameters: dict = None,
                  wait: bool = True, timeout: int = 600) -> dict:
    """触发构建并等待完成"""
    print(f"\n{'='*50}")
    print(f"触发构建: {job_name}")
    print(f"{'='*50}")

    # 触发构建
    queue_id = server.build_job(job_name, parameters=parameters)
    print(f"队列 ID: {queue_id}")

    if not wait:
        return {'status': 'triggered', 'queue_id': queue_id}

    # 等待构建完成（模拟）
    print("等待构建完成...")
    build_number = 42  # 模拟

    # 获取构建结果
    build_info = server.get_build_info(job_name, build_number)
    result = build_info.get('result', 'UNKNOWN')
    duration = build_info.get('duration', 0) / 1000

    print(f"\n构建结果:")
    print(f"  构建号: #{build_number}")
    print(f"  状态: {result}")
    print(f"  耗时: {duration:.1f}s")
    print(f"  URL: {build_info['url']}")

    return {
        'job_name': job_name,
        'build_number': build_number,
        'result': result,
        'duration_seconds': duration,
        'url': build_info['url'],
    }

# 触发构建
result = trigger_build(server, 'web-app-deploy', parameters={
    'ENVIRONMENT': 'staging',
    'GIT_BRANCH': 'release/v2.0',
    'ROLLBACK': 'false',
})

# === 获取构建日志 ===
print(f"\n=== 构建日志 ===")
console = server.get_build_console_output('web-app-deploy', 42)
print(console)
```

## 5. 进阶用法

### 5.1 构建历史分析

```python
#!/usr/bin/env python3
"""构建历史分析"""

from collections import Counter
from datetime import datetime

# 模拟构建历史
mock_builds = [
    {'number': i, 'result': r, 'duration': d * 1000,
     'timestamp': 1705286400000 + i * 3600000}
    for i, (r, d) in enumerate([
        ('SUCCESS', 120), ('SUCCESS', 115), ('FAILURE', 300),
        ('SUCCESS', 125), ('SUCCESS', 110), ('UNSTABLE', 200),
        ('SUCCESS', 130), ('FAILURE', 45), ('SUCCESS', 118),
        ('SUCCESS', 122), ('SUCCESS', 128), ('SUCCESS', 135),
        ('FAILURE', 60), ('SUCCESS', 119), ('SUCCESS', 121),
    ], start=28)
]

def analyze_build_history(builds: list, job_name: str):
    """分析构建历史"""
    print(f"\n{'='*50}")
    print(f"构建历史分析: {job_name}")
    print(f"{'='*50}")

    total = len(builds)
    results = Counter(b['result'] for b in builds)
    durations = [b['duration'] / 1000 for b in builds]  # 转为秒

    # 成功率
    success_count = results.get('SUCCESS', 0)
    success_rate = success_count / total * 100 if total else 0

    print(f"\n--- 基础统计 ---")
    print(f"  总构建数: {total}")
    print(f"  成功率: {success_rate:.1f}%")
    for result, count in results.most_common():
        print(f"  {result}: {count} ({count/total*100:.1f}%)")

    print(f"\n--- 耗时统计 ---")
    print(f"  平均耗时: {sum(durations)/len(durations):.1f}s")
    print(f"  最短耗时: {min(durations):.1f}s")
    print(f"  最长耗时: {max(durations):.1f}s")

    # 失败趋势
    print(f"\n--- 最近构建 ---")
    for b in builds[-5:]:
        ts = datetime.fromtimestamp(b['timestamp'] / 1000).strftime('%Y-%m-%d %H:%M')
        icon = "+" if b['result'] == 'SUCCESS' else "X" if b['result'] == 'FAILURE' else "!"
        print(f"  [{icon}] #{b['number']} {ts} {b['result']} ({b['duration']/1000:.0f}s)")

analyze_build_history(mock_builds, "web-app-build")
```

## 6. SRE 实战案例：Jenkins 流水线自动化管理

```python
#!/usr/bin/env python3
"""
SRE 实战：Jenkins 流水线自动化管理系统
功能：
1. 批量创建/管理 Job
2. 自动触发构建流水线
3. 构建监控与告警
4. 构建历史统计
5. Job 配置备份
6. 流水线编排（顺序/并行）
"""

import json
import time
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional
from dataclasses import dataclass, field, asdict
from collections import Counter, defaultdict
from concurrent.futures import ThreadPoolExecutor

# ============================================================
# 数据模型
# ============================================================
@dataclass
class BuildResult:
    """构建结果"""
    job_name: str
    build_number: int
    status: str        # SUCCESS / FAILURE / UNSTABLE / ABORTED
    duration_seconds: float
    timestamp: str
    parameters: Dict = field(default_factory=dict)
    url: str = ""

@dataclass
class PipelineStage:
    """流水线阶段"""
    name: str
    job_name: str
    parameters: Dict = field(default_factory=dict)
    depends_on: List[str] = field(default_factory=list)
    allow_failure: bool = False

# ============================================================
# Jenkins 管理器
# ============================================================
class JenkinsManager:
    """Jenkins 流水线自动化管理"""

    def __init__(self, url: str, username: str, token: str):
        self.url = url
        self.username = username
        self.token = token
        self.build_history: List[BuildResult] = []
        # 实际使用时:
        # self.server = jenkins.Jenkins(url, username=username, password=token)

    def create_pipeline_job(self, name: str, git_url: str, branch: str = 'main',
                            jenkinsfile_path: str = 'Jenkinsfile') -> bool:
        """创建 Pipeline Job（从 SCM）"""
        config = f"""<?xml version='1.0' encoding='UTF-8'?>
<flow-definition>
  <description>Auto-created by SRE toolkit</description>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition">
    <scm class="hudson.plugins.git.GitSCM">
      <userRemoteConfigs>
        <hudson.plugins.git.UserRemoteConfig>
          <url>{git_url}</url>
        </hudson.plugins.git.UserRemoteConfig>
      </userRemoteConfigs>
      <branches>
        <hudson.plugins.git.BranchSpec>
          <name>*/{branch}</name>
        </hudson.plugins.git.BranchSpec>
      </branches>
    </scm>
    <scriptPath>{jenkinsfile_path}</scriptPath>
  </definition>
</flow-definition>"""
        print(f"  [创建] Pipeline Job: {name} (git: {git_url}, branch: {branch})")
        return True

    def trigger_and_wait(self, job_name: str, parameters: dict = None,
                         timeout: int = 600) -> BuildResult:
        """触发构建并等待结果"""
        import random

        print(f"  [触发] {job_name} ...")
        start = time.time()

        # 模拟构建执行
        time.sleep(0.1)  # 实际中会轮询等待

        # 模拟结果
        status = random.choice(['SUCCESS', 'SUCCESS', 'SUCCESS', 'FAILURE'])
        duration = random.uniform(30, 180)

        result = BuildResult(
            job_name=job_name,
            build_number=random.randint(1, 100),
            status=status,
            duration_seconds=round(duration, 1),
            timestamp=datetime.now().isoformat(),
            parameters=parameters or {},
        )
        self.build_history.append(result)

        icon = "+" if status == 'SUCCESS' else "X"
        print(f"  [{icon}] {job_name} #{result.build_number} -> {status} ({duration:.0f}s)")
        return result

    def run_pipeline(self, stages: List[PipelineStage]) -> Dict:
        """执行流水线编排"""
        print(f"\n{'='*60}")
        print(f"执行流水线 ({len(stages)} 个阶段)")
        print(f"{'='*60}")

        completed = {}
        failed = []

        for stage in stages:
            # 检查依赖
            if stage.depends_on:
                unmet = [d for d in stage.depends_on if d not in completed]
                if unmet:
                    print(f"  [跳过] {stage.name}: 依赖未满足 {unmet}")
                    continue

                # 检查依赖是否成功
                dep_failures = [d for d in stage.depends_on
                              if completed.get(d, {}).get('status') != 'SUCCESS']
                if dep_failures and not stage.allow_failure:
                    print(f"  [跳过] {stage.name}: 依赖 {dep_failures} 失败")
                    continue

            # 执行阶段
            print(f"\n--- 阶段: {stage.name} ---")
            result = self.trigger_and_wait(stage.job_name, stage.parameters)
            completed[stage.name] = asdict(result)

            if result.status != 'SUCCESS':
                failed.append(stage.name)
                if not stage.allow_failure:
                    print(f"  [中止] 阶段 {stage.name} 失败，流水线中止")
                    break

        # 汇总
        total = len(completed)
        success = sum(1 for v in completed.values() if v['status'] == 'SUCCESS')

        print(f"\n{'='*60}")
        print(f"流水线完成: {success}/{total} 阶段成功")
        if failed:
            print(f"失败阶段: {', '.join(failed)}")
        print(f"{'='*60}")

        return {
            'total_stages': len(stages),
            'completed': total,
            'succeeded': success,
            'failed': failed,
            'results': completed,
        }

    def backup_jobs(self, output_dir: str, job_names: List[str] = None) -> int:
        """备份 Job 配置"""
        output_path = Path(output_dir)
        output_path.mkdir(parents=True, exist_ok=True)

        # 模拟备份
        backed_up = 0
        jobs = job_names or ['web-app-build', 'api-deploy-staging', 'infra-terraform-plan']

        for job_name in jobs:
            config_file = output_path / f"{job_name}.xml"
            config_file.write_text(f"<!-- Jenkins Job 配置备份: {job_name} -->\n<project/>")
            backed_up += 1
            print(f"  [备份] {job_name} → {config_file}")

        return backed_up

    def build_statistics(self) -> Dict:
        """构建统计"""
        if not self.build_history:
            return {}

        stats = {
            'total_builds': len(self.build_history),
            'by_status': Counter(b.status for b in self.build_history),
            'by_job': defaultdict(lambda: {'total': 0, 'success': 0, 'avg_duration': 0}),
        }

        # 按 Job 统计
        job_builds = defaultdict(list)
        for b in self.build_history:
            job_builds[b.job_name].append(b)

        for job, builds in job_builds.items():
            total = len(builds)
            success = sum(1 for b in builds if b.status == 'SUCCESS')
            avg_dur = sum(b.duration_seconds for b in builds) / total
            stats['by_job'][job] = {
                'total': total,
                'success': success,
                'success_rate': f"{success/total*100:.1f}%",
                'avg_duration': f"{avg_dur:.1f}s",
            }

        return stats


# ============================================================
# 演示
# ============================================================
def main():
    # 初始化管理器
    mgr = JenkinsManager(
        url="http://jenkins.example.com:8080",
        username="admin",
        token="demo-token"
    )

    # 1. 批量创建 Pipeline Jobs
    print("=== 1. 批量创建 Pipeline Jobs ===")
    services = ['web-frontend', 'api-backend', 'worker-service']
    for svc in services:
        mgr.create_pipeline_job(
            name=f"{svc}-pipeline",
            git_url=f"https://github.com/company/{svc}.git",
            branch="main",
        )

    # 2. 执行部署流水线
    print("\n=== 2. 执行部署流水线 ===")
    pipeline_stages = [
        PipelineStage(name="代码检查", job_name="code-lint", parameters={"BRANCH": "main"}),
        PipelineStage(name="单元测试", job_name="unit-test", depends_on=["代码检查"]),
        PipelineStage(name="构建镜像", job_name="docker-build", depends_on=["单元测试"]),
        PipelineStage(name="部署 Staging", job_name="deploy-staging", depends_on=["构建镜像"]),
        PipelineStage(name="集成测试", job_name="integration-test", depends_on=["部署 Staging"],
                      allow_failure=True),
        PipelineStage(name="部署 Production", job_name="deploy-prod", depends_on=["集成测试"]),
    ]
    pipeline_result = mgr.run_pipeline(pipeline_stages)

    # 3. 构建统计
    print("\n=== 3. 构建统计 ===")
    stats = mgr.build_statistics()
    print(f"总构建数: {stats['total_builds']}")
    print(f"状态分布: {dict(stats['by_status'])}")
    print(f"\n按 Job 统计:")
    for job, info in stats['by_job'].items():
        print(f"  {job}: {info}")

    # 4. 备份 Job 配置
    print("\n=== 4. 备份 Job 配置 ===")
    count = mgr.backup_jobs("/tmp/jenkins-backup")
    print(f"已备份 {count} 个 Job")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **API Token 过期**：定期更新 Token，使用专用服务账户
2. **并发构建冲突**：同一 Job 并发触发可能排队
3. **XML 配置格式**：Job 配置是 XML，修改时注意转义
4. **超时处理**：构建可能长时间运行，设置合理超时
5. **权限问题**：API 用户需要足够的 Jenkins 权限

### 最佳实践

```python
# 1. 使用 API Token 而非密码
# 2. Job 配置纳入版本控制（JCasC / Job DSL）
# 3. 构建触发加入超时和重试机制
# 4. 敏感参数使用 Jenkins Credentials
# 5. 定期备份 Job 配置
```

## 8. 速查表

```python
import jenkins

# === 连接 ===
server = jenkins.Jenkins(url, username=user, password=token)
server.get_whoami()
server.get_version()

# === Job 管理 ===
server.get_jobs()                        # 列出所有 Job
server.get_job_info(name)                # Job 详情
server.create_job(name, xml_config)      # 创建
server.reconfig_job(name, xml_config)    # 修改配置
server.delete_job(name)                  # 删除
server.enable_job(name)                  # 启用
server.disable_job(name)                 # 禁用
server.get_job_config(name)              # 获取 XML 配置

# === 构建 ===
server.build_job(name)                   # 触发构建
server.build_job(name, {'KEY': 'val'})   # 参数化构建
server.get_build_info(name, number)      # 构建详情
server.get_build_console_output(name, n) # 构建日志
server.stop_build(name, number)          # 停止构建

# === 视图 ===
server.get_views()
server.create_view(name, config)

# === 节点 ===
server.get_nodes()
server.get_node_info(name)
```
