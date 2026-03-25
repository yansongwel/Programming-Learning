# GitHub 与 GitLab API — 用 Python 实现自动化代码管理

## 1. 是什么 & 为什么用它

`PyGithub` 和 `python-gitlab` 是操作 GitHub/GitLab 的 Python 客户端库。
对 SRE 来说，Git 平台 API 自动化可以：

- **自动化 PR/MR 管理**：自动 Review、合并、标签
- **Issue 自动化**：告警自动创建 Issue，自动分配
- **仓库管理**：批量设置分支保护、权限
- **CI/CD 管道管理**：监控 Pipeline 状态
- **Webhook 集成**：事件驱动的自动化

## 2. 安装与环境

```bash
# GitHub
pip install PyGithub

# GitLab
pip install python-gitlab

# 验证
python3 -c "from github import Github; print('PyGithub 可用')"
python3 -c "import gitlab; print('python-gitlab 可用')"

# 需要准备 Personal Access Token
# GitHub: Settings → Developer settings → Personal access tokens
# GitLab: User Settings → Access Tokens
```

## 3. 核心概念与原理

### 3.1 API 认证

```
GitHub:
  - Personal Access Token (PAT) — 最常用
  - GitHub App Installation Token — 适合组织级自动化
  - OAuth App — 适合第三方集成

GitLab:
  - Personal Access Token — 最常用
  - Project/Group Access Token — 限定范围
  - OAuth2 — 第三方集成

Token 权限（Scope）：
  - repo: 仓库读写
  - workflow: GitHub Actions
  - admin:repo_hook: Webhook 管理
  - api: GitLab 完整 API 访问
```

### 3.2 核心资源对象

```
Repository/Project
  ├── Branches（分支）
  ├── Commits（提交）
  ├── Pull Requests / Merge Requests
  ├── Issues
  ├── Releases / Tags
  ├── Actions / Pipelines（CI/CD）
  ├── Webhooks
  └── Settings（保护分支、权限等）
```

## 4. 基础用法

### 4.1 GitHub 操作（PyGithub）

```python
#!/usr/bin/env python3
"""PyGithub 基础操作"""

# 实际使用时导入并认证:
# from github import Github
# g = Github("your-access-token")

# === 模拟客户端用于演示 ===
class MockGithubRepo:
    """模拟 GitHub 仓库"""
    def __init__(self):
        self.full_name = "company/sre-toolkit"
        self.description = "SRE 运维工具箱"
        self.default_branch = "main"
        self.stargazers_count = 128
        self.forks_count = 32
        self.open_issues_count = 5
        self.language = "Python"
        self.private = False

class MockGithub:
    """模拟 PyGithub"""
    def get_user(self):
        class User:
            login = "sre-bot"
            name = "SRE Bot"
        return User()

    def get_repo(self, name):
        return MockGithubRepo()

    def search_repositories(self, query, **kwargs):
        return []

g = MockGithub()

# === 获取用户信息 ===
user = g.get_user()
print(f"当前用户: {user.login} ({user.name})")

# === 获取仓库信息 ===
repo = g.get_repo("company/sre-toolkit")
print(f"\n=== 仓库信息 ===")
print(f"  名称: {repo.full_name}")
print(f"  描述: {repo.description}")
print(f"  默认分支: {repo.default_branch}")
print(f"  Star: {repo.stargazers_count}")
print(f"  Fork: {repo.forks_count}")
print(f"  Open Issues: {repo.open_issues_count}")
print(f"  语言: {repo.language}")
print(f"  私有: {repo.private}")

# === 实际 PyGithub 用法示例 ===
print("""
# === 仓库操作 ===
from github import Github
g = Github("your-token")
repo = g.get_repo("owner/repo")

# 获取分支列表
for branch in repo.get_branches():
    print(f"  {branch.name} -> {branch.commit.sha[:8]}")

# 获取提交历史
for commit in repo.get_commits()[:10]:
    print(f"  {commit.sha[:8]} {commit.commit.message}")

# 获取文件内容
contents = repo.get_contents("README.md")
print(contents.decoded_content.decode())

# 创建文件
repo.create_file("config/app.yaml", "添加配置文件", "server:\\n  port: 8080")

# 更新文件
file = repo.get_contents("config/app.yaml")
repo.update_file("config/app.yaml", "更新配置", "server:\\n  port: 9090", file.sha)
""")
```

### 4.2 Pull Request 操作

```python
#!/usr/bin/env python3
"""GitHub Pull Request 操作"""

print("""
# === PR 操作 ===
from github import Github
g = Github("your-token")
repo = g.get_repo("owner/repo")

# 创建 PR
pr = repo.create_pull(
    title="修复: 服务器配置更新",
    body="## 变更说明\\n- 更新端口配置\\n- 添加健康检查",
    head="feature/update-config",
    base="main",
)
print(f"PR 创建: #{pr.number} {pr.title}")

# 获取 PR 列表
for pr in repo.get_pulls(state='open', sort='created', direction='desc'):
    print(f"  #{pr.number} [{pr.user.login}] {pr.title}")
    print(f"    状态: {pr.state}, 合并: {pr.merged}")
    print(f"    变更: +{pr.additions} -{pr.deletions} 文件:{pr.changed_files}")

# 获取 PR 详情
pr = repo.get_pull(123)
# PR Review
for review in pr.get_reviews():
    print(f"  Review: {review.user.login} -> {review.state}")

# PR 评论
pr.create_review(body="LGTM!", event="APPROVE")
pr.create_issue_comment("自动检查通过 ✓")

# 合并 PR
if pr.mergeable:
    pr.merge(
        commit_title=f"Merge #{pr.number}: {pr.title}",
        merge_method="squash"  # merge / squash / rebase
    )
""")

# === 模拟 PR Review Bot ===
class PRReviewBot:
    """自动化 PR Review 机器人"""

    def __init__(self):
        self.rules = {
            'max_files': 20,             # 单 PR 最大文件数
            'max_additions': 500,        # 最大新增行数
            'required_reviewers': 2,     # 最少审核人数
            'protected_paths': [         # 受保护路径（需要额外审核）
                'config/', 'deploy/', '.github/',
                'Dockerfile', 'docker-compose',
            ],
            'forbidden_patterns': [      # 禁止的内容模式
                r'password\s*=', r'secret\s*=',
                r'api[_-]?key\s*=', r'TODO',
            ],
        }

    def review_pr(self, pr_data: dict) -> dict:
        """审核 PR"""
        issues = []
        warnings = []

        # 检查文件数
        if pr_data['changed_files'] > self.rules['max_files']:
            issues.append(f"变更文件数({pr_data['changed_files']})超过限制({self.rules['max_files']})")

        # 检查行数
        if pr_data['additions'] > self.rules['max_additions']:
            warnings.append(f"新增行数({pr_data['additions']})较多，建议拆分 PR")

        # 检查受保护路径
        protected_changes = [f for f in pr_data.get('files', [])
                           if any(p in f for p in self.rules['protected_paths'])]
        if protected_changes:
            warnings.append(f"修改了受保护路径: {protected_changes}")

        # 判断结果
        if issues:
            action = 'REQUEST_CHANGES'
        elif warnings:
            action = 'COMMENT'
        else:
            action = 'APPROVE'

        return {
            'action': action,
            'issues': issues,
            'warnings': warnings,
            'comment': self._build_comment(action, issues, warnings),
        }

    def _build_comment(self, action, issues, warnings):
        """构建 Review 评论"""
        lines = ["## 自动化 Review 结果\n"]
        if action == 'APPROVE':
            lines.append("自动检查通过\n")
        if issues:
            lines.append("### 问题（需修复）")
            for i in issues:
                lines.append(f"- {i}")
        if warnings:
            lines.append("\n### 警告（建议关注）")
            for w in warnings:
                lines.append(f"- {w}")
        return '\n'.join(lines)

# 模拟使用
bot = PRReviewBot()
pr_data = {
    'number': 42,
    'title': '更新部署配置',
    'changed_files': 5,
    'additions': 120,
    'deletions': 30,
    'files': ['config/nginx.conf', 'src/app.py', 'deploy/k8s.yaml'],
}

result = bot.review_pr(pr_data)
print(f"=== PR #{pr_data['number']} Review 结果 ===")
print(f"操作: {result['action']}")
print(result['comment'])
```

### 4.3 GitLab 操作（python-gitlab）

```python
#!/usr/bin/env python3
"""python-gitlab 基础操作"""

print("""
# === GitLab 连接 ===
import gitlab

# 自建 GitLab
gl = gitlab.Gitlab('https://gitlab.example.com', private_token='your-token')
gl.auth()

# GitLab.com
gl = gitlab.Gitlab('https://gitlab.com', private_token='your-token')

# === 项目操作 ===
# 获取项目
project = gl.projects.get('group/project-name')  # 或使用项目 ID
print(f"项目: {project.name}")
print(f"描述: {project.description}")

# 列出用户的项目
projects = gl.projects.list(owned=True)
for p in projects:
    print(f"  {p.path_with_namespace}: {p.description}")

# === Merge Request 操作 ===
# 创建 MR
mr = project.mergerequests.create({
    'source_branch': 'feature/update',
    'target_branch': 'main',
    'title': '更新服务配置',
    'description': '修改端口和添加健康检查',
    'assignee_id': 1,
    'reviewer_ids': [2, 3],
})

# 列出 MR
mrs = project.mergerequests.list(state='opened')
for mr in mrs:
    print(f"  !{mr.iid} [{mr.author['username']}] {mr.title}")

# 合并 MR
mr = project.mergerequests.get(42)
mr.merge(
    should_remove_source_branch=True,
    squash=True,
)

# === Pipeline 操作 ===
# 获取 Pipeline 列表
pipelines = project.pipelines.list()
for p in pipelines:
    print(f"  #{p.id} {p.ref} {p.status} {p.created_at}")

# 获取 Pipeline 详情
pipeline = project.pipelines.get(123)
jobs = pipeline.jobs.list()
for job in jobs:
    print(f"  Stage: {job.stage}, Job: {job.name}, Status: {job.status}")

# 触发 Pipeline
pipeline = project.pipelines.create({
    'ref': 'main',
    'variables': [{'key': 'ENV', 'value': 'staging'}],
})

# 重试失败的 Pipeline
pipeline.retry()

# === Issue 操作 ===
# 创建 Issue
issue = project.issues.create({
    'title': '告警: CPU 使用率超过 90%',
    'description': '服务器 web-01 CPU 持续高于阈值',
    'labels': ['alert', 'urgent'],
    'assignee_ids': [1],
})

# === Webhook 管理 ===
hook = project.hooks.create({
    'url': 'https://webhook.example.com/gitlab',
    'push_events': True,
    'merge_requests_events': True,
    'pipeline_events': True,
    'token': 'webhook-secret',
})
""")
```

## 5. 进阶用法

### 5.1 Webhook 事件处理

```python
#!/usr/bin/env python3
"""Webhook 事件处理服务器"""

from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import hmac
import hashlib

class WebhookHandler(BaseHTTPRequestHandler):
    """GitHub/GitLab Webhook 事件处理"""

    SECRET = b"your-webhook-secret"

    def do_POST(self):
        # 读取请求体
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length)

        # 验证签名（GitHub）
        signature = self.headers.get('X-Hub-Signature-256', '')
        if signature:
            expected = 'sha256=' + hmac.new(self.SECRET, body, hashlib.sha256).hexdigest()
            if not hmac.compare_digest(signature, expected):
                self.send_response(403)
                self.end_headers()
                return

        # 解析事件
        event_type = self.headers.get('X-GitHub-Event', '')
        payload = json.loads(body)

        # 处理不同事件
        handlers = {
            'push': self.handle_push,
            'pull_request': self.handle_pr,
            'issues': self.handle_issue,
            'workflow_run': self.handle_workflow,
        }

        handler = handlers.get(event_type, self.handle_unknown)
        handler(payload)

        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')

    def handle_push(self, payload):
        """处理 Push 事件"""
        repo = payload.get('repository', {}).get('full_name', '')
        ref = payload.get('ref', '')
        pusher = payload.get('pusher', {}).get('name', '')
        commits = payload.get('commits', [])
        print(f"[Push] {repo} {ref} by {pusher} ({len(commits)} commits)")

    def handle_pr(self, payload):
        """处理 PR 事件"""
        action = payload.get('action', '')
        pr = payload.get('pull_request', {})
        print(f"[PR] #{pr.get('number')} {action}: {pr.get('title')}")

    def handle_issue(self, payload):
        """处理 Issue 事件"""
        action = payload.get('action', '')
        issue = payload.get('issue', {})
        print(f"[Issue] #{issue.get('number')} {action}: {issue.get('title')}")

    def handle_workflow(self, payload):
        """处理 Workflow 事件"""
        action = payload.get('action', '')
        workflow = payload.get('workflow_run', {})
        print(f"[Workflow] {workflow.get('name')} {action}: {workflow.get('conclusion')}")

    def handle_unknown(self, payload):
        """处理未知事件"""
        print(f"[Unknown] 收到未知事件")

print("=== Webhook 处理器 ===")
print("启动方式: HTTPServer(('0.0.0.0', 9000), WebhookHandler).serve_forever()")
print("注册 Webhook URL: http://your-server:9000/webhook")
```

## 6. SRE 实战案例：自动化 PR Review 与合并机器人

```python
#!/usr/bin/env python3
"""
SRE 实战：自动化 PR Review 与合并机器人
功能：
1. 自动检测新 PR
2. 代码规范检查
3. 敏感文件变更告警
4. 自动分配 Reviewer
5. 自动合并通过的 PR
6. 状态通知
"""

import json
import re
import time
from datetime import datetime
from pathlib import Path
from typing import Dict, List, Optional
from dataclasses import dataclass, field, asdict

# ============================================================
# 配置
# ============================================================
BOT_CONFIG = {
    'auto_merge': {
        'enabled': True,
        'required_approvals': 2,
        'required_checks': ['ci/test', 'ci/lint'],
        'merge_method': 'squash',
        'delete_branch': True,
    },
    'auto_assign': {
        'enabled': True,
        'reviewers_pool': ['dev-lead', 'senior-dev-1', 'senior-dev-2', 'sre-1'],
        'reviewers_count': 2,
        'by_path': {
            'infra/': ['sre-1', 'sre-2'],
            'config/': ['sre-1', 'ops-lead'],
            'src/': ['dev-lead', 'senior-dev-1'],
        },
    },
    'checks': {
        'max_files': 30,
        'max_additions': 1000,
        'forbidden_files': ['.env', 'credentials.json', 'id_rsa'],
        'protected_branches': ['main', 'release/*'],
        'sensitive_patterns': [
            r'password\s*[:=]',
            r'api[_-]?key\s*[:=]',
            r'secret\s*[:=]',
            r'BEGIN.*PRIVATE KEY',
        ],
    },
}

# ============================================================
# 数据模型
# ============================================================
@dataclass
class PRCheckResult:
    """PR 检查结果"""
    pr_number: int
    title: str
    author: str
    checks_passed: bool
    issues: List[str] = field(default_factory=list)
    warnings: List[str] = field(default_factory=list)
    assigned_reviewers: List[str] = field(default_factory=list)
    auto_merge_eligible: bool = False
    action_taken: str = ""
    timestamp: str = ""

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

# ============================================================
# PR Review Bot
# ============================================================
class PRReviewMergeBot:
    """PR 自动化 Review 与合并机器人"""

    def __init__(self, config: dict):
        self.config = config
        self.results: List[PRCheckResult] = []
        self._compile_patterns()

    def _compile_patterns(self):
        """编译敏感内容正则"""
        self.sensitive_re = [
            re.compile(p, re.I) for p in self.config['checks']['sensitive_patterns']
        ]

    def check_pr(self, pr: dict) -> PRCheckResult:
        """检查 PR"""
        result = PRCheckResult(
            pr_number=pr['number'],
            title=pr['title'],
            author=pr['author'],
        )

        # 检查 1: 文件数量
        if pr['changed_files'] > self.config['checks']['max_files']:
            result.issues.append(
                f"变更文件数({pr['changed_files']})超过限制"
                f"({self.config['checks']['max_files']}), 请拆分 PR"
            )

        # 检查 2: 新增行数
        if pr['additions'] > self.config['checks']['max_additions']:
            result.warnings.append(
                f"新增行数({pr['additions']})较多, 建议拆分"
            )

        # 检查 3: 禁止的文件
        forbidden_files = [
            f for f in pr.get('files', [])
            if any(fb in f for fb in self.config['checks']['forbidden_files'])
        ]
        if forbidden_files:
            result.issues.append(f"包含禁止提交的文件: {forbidden_files}")

        # 检查 4: 敏感内容
        for file_info in pr.get('file_contents', []):
            for pattern in self.sensitive_re:
                if pattern.search(file_info.get('content', '')):
                    result.issues.append(
                        f"文件 {file_info['name']} 中可能包含敏感信息"
                    )
                    break

        # 检查 5: PR 描述
        if not pr.get('body') or len(pr.get('body', '')) < 10:
            result.warnings.append("PR 描述过短, 请补充变更说明")

        # 汇总
        result.checks_passed = len(result.issues) == 0
        return result

    def assign_reviewers(self, pr: dict, result: PRCheckResult):
        """自动分配 Reviewer"""
        if not self.config['auto_assign']['enabled']:
            return

        reviewers = set()

        # 根据文件路径分配
        for filepath in pr.get('files', []):
            for path_prefix, path_reviewers in self.config['auto_assign']['by_path'].items():
                if filepath.startswith(path_prefix):
                    reviewers.update(path_reviewers)

        # 补充到要求数量
        pool = set(self.config['auto_assign']['reviewers_pool'])
        pool.discard(pr['author'])  # 排除 PR 作者
        remaining = pool - reviewers

        count = self.config['auto_assign']['reviewers_count']
        while len(reviewers) < count and remaining:
            reviewers.add(remaining.pop())

        result.assigned_reviewers = list(reviewers)[:count]

    def check_merge_eligibility(self, pr: dict, result: PRCheckResult):
        """检查是否可以自动合并"""
        if not self.config['auto_merge']['enabled']:
            return

        conditions = []

        # 检查通过
        if not result.checks_passed:
            conditions.append("自动检查未通过")

        # 审核通过数
        approvals = pr.get('approvals', 0)
        required = self.config['auto_merge']['required_approvals']
        if approvals < required:
            conditions.append(f"审核通过数({approvals})不足({required})")

        # CI 检查通过
        passed_checks = set(pr.get('passed_checks', []))
        required_checks = set(self.config['auto_merge']['required_checks'])
        missing = required_checks - passed_checks
        if missing:
            conditions.append(f"CI 检查未通过: {missing}")

        # 无合并冲突
        if not pr.get('mergeable', True):
            conditions.append("存在合并冲突")

        result.auto_merge_eligible = len(conditions) == 0
        if not result.auto_merge_eligible:
            for c in conditions:
                result.warnings.append(f"不可自动合并: {c}")

    def process_pr(self, pr: dict) -> PRCheckResult:
        """处理单个 PR"""
        print(f"\n处理 PR #{pr['number']}: {pr['title']}")

        # 1. 检查
        result = self.check_pr(pr)

        # 2. 分配 Reviewer
        self.assign_reviewers(pr, result)
        if result.assigned_reviewers:
            print(f"  分配 Reviewer: {result.assigned_reviewers}")

        # 3. 检查合并条件
        self.check_merge_eligibility(pr, result)

        # 4. 决定操作
        if result.issues:
            result.action_taken = "REQUEST_CHANGES"
            print(f"  [要求修改] 发现 {len(result.issues)} 个问题")
        elif result.auto_merge_eligible:
            result.action_taken = "AUTO_MERGE"
            print(f"  [自动合并] 所有检查通过, 准备合并")
        elif result.checks_passed:
            result.action_taken = "APPROVE"
            print(f"  [批准] 检查通过, 等待人工审核")
        else:
            result.action_taken = "COMMENT"
            print(f"  [评论] 有警告需要关注")

        # 输出详情
        for issue in result.issues:
            print(f"    [问题] {issue}")
        for warning in result.warnings:
            print(f"    [警告] {warning}")

        self.results.append(result)
        return result

    def generate_comment(self, result: PRCheckResult) -> str:
        """生成 PR 评论"""
        lines = ["## SRE Bot 自动化 Review\n"]

        if result.checks_passed:
            lines.append("所有自动化检查通过\n")
        else:
            lines.append("发现以下问题需要修复：\n")

        if result.issues:
            lines.append("### 必须修复")
            for issue in result.issues:
                lines.append(f"- {issue}")

        if result.warnings:
            lines.append("\n### 建议关注")
            for warning in result.warnings:
                lines.append(f"- {warning}")

        if result.assigned_reviewers:
            lines.append(f"\n**已分配 Reviewer:** {', '.join('@' + r for r in result.assigned_reviewers)}")

        if result.auto_merge_eligible:
            lines.append("\n**自动合并已就绪，将在检查通过后自动合并。**")

        lines.append(f"\n---\n_SRE Bot {datetime.now().strftime('%Y-%m-%d %H:%M')}_")
        return '\n'.join(lines)

    def report(self) -> str:
        """生成处理报告"""
        lines = [
            f"{'='*60}",
            f"PR Bot 处理报告",
            f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"{'='*60}",
            f"处理 PR 数: {len(self.results)}",
        ]

        actions = {}
        for r in self.results:
            actions[r.action_taken] = actions.get(r.action_taken, 0) + 1
        lines.append(f"操作统计: {actions}")

        lines.append(f"\n--- 详情 ---")
        for r in self.results:
            lines.append(
                f"  #{r.pr_number} [{r.action_taken}] {r.title} "
                f"(问题:{len(r.issues)}, 警告:{len(r.warnings)})"
            )

        return '\n'.join(lines)


# ============================================================
# 演示
# ============================================================
def main():
    # 初始化 Bot
    bot = PRReviewMergeBot(BOT_CONFIG)

    # 模拟 PR 列表
    mock_prs = [
        {
            'number': 101,
            'title': '修复: 更新 Nginx 配置',
            'author': 'dev-a',
            'body': '修改了 Nginx upstream 配置，增加了新的后端服务器',
            'changed_files': 3,
            'additions': 25,
            'deletions': 10,
            'files': ['config/nginx.conf', 'config/upstream.conf', 'docs/deploy.md'],
            'approvals': 2,
            'passed_checks': ['ci/test', 'ci/lint'],
            'mergeable': True,
        },
        {
            'number': 102,
            'title': '功能: 添加新的部署脚本',
            'author': 'dev-b',
            'body': '',  # 描述为空
            'changed_files': 35,  # 超过限制
            'additions': 1500,    # 超过限制
            'deletions': 0,
            'files': ['infra/deploy.sh', 'infra/rollback.sh', '.env'],  # 包含 .env
            'approvals': 0,
            'passed_checks': ['ci/test'],
            'mergeable': True,
            'file_contents': [
                {'name': 'infra/deploy.sh', 'content': 'DB_PASSWORD=secret123'},
            ],
        },
        {
            'number': 103,
            'title': '优化: 监控告警规则调整',
            'author': 'sre-1',
            'body': '调整 CPU 告警阈值从 80% 到 90%，减少误告警',
            'changed_files': 2,
            'additions': 15,
            'deletions': 12,
            'files': ['config/prometheus/rules.yaml', 'config/alertmanager.yaml'],
            'approvals': 1,  # 不够
            'passed_checks': ['ci/test', 'ci/lint'],
            'mergeable': True,
        },
    ]

    # 处理所有 PR
    print("=== 开始处理 PR ===")
    for pr in mock_prs:
        result = bot.process_pr(pr)
        comment = bot.generate_comment(result)
        print(f"\n  生成评论:\n{comment}")

    # 生成报告
    print(f"\n{bot.report()}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **API 限流**：GitHub 每小时 5000 请求，超限会被拒绝
2. **Token 权限**：权限不足会返回 404（不是 403）
3. **分页**：默认只返回 30 条记录，需要处理分页
4. **Webhook 验证**：务必验证 Webhook 签名防止伪造
5. **大仓库操作**：获取全部文件列表可能超时

### 最佳实践

```python
# 1. 使用 GitHub App 而非 PAT（更安全、更细粒度）
# 2. 处理 API 限流
# 3. Webhook Secret 使用强随机字符串
# 4. Bot Token 使用最小权限原则
# 5. 敏感操作加审批流程
```

## 8. 速查表

```python
# === PyGithub ===
from github import Github
g = Github("token")
repo = g.get_repo("owner/repo")
repo.get_pulls(state='open')         # PR 列表
repo.create_pull(title, body, head, base)  # 创建 PR
pr.merge(merge_method='squash')      # 合并
repo.get_issues(state='open')        # Issue 列表
repo.create_issue(title, body)       # 创建 Issue

# === python-gitlab ===
import gitlab
gl = gitlab.Gitlab(url, private_token=token)
project = gl.projects.get('group/project')
project.mergerequests.list(state='opened')  # MR 列表
project.mergerequests.create({...})         # 创建 MR
project.pipelines.list()                    # Pipeline 列表
project.issues.create({...})                # 创建 Issue
```
