# GitPython 仓库操作 — 用 Python 自动化 Git 管理与审计

## 1. 是什么 & 为什么用它

GitPython 是 Python 操作 Git 仓库的库，提供了对 Git 仓库的完整操作能力。
对 SRE 来说，自动化 Git 操作是 CI/CD 和代码审计的基础：

- **代码变更审计**：自动分析谁改了什么、何时改的
- **配置仓库管理**：自动化 GitOps 配置管理
- **自动化提交**：脚本生成配置后自动提交
- **仓库健康检查**：分支清理、大文件检测

对比命令行 `git`：
- Shell 脚本中解析 git 输出容易出错
- GitPython 提供结构化的 Python 对象
- 更容易集成到 Python 自动化流程中

## 2. 安装与环境

```bash
# 安装 GitPython
pip install GitPython

# 验证
python3 -c "import git; print(git.__version__)"

# 需要系统安装 git
git --version
```

## 3. 核心概念与原理

### 3.1 GitPython 核心对象

```
Repo（仓库）
  ├── heads（本地分支列表）
  ├── remotes（远程仓库列表）
  ├── tags（标签列表）
  ├── index（暂存区）
  ├── head（当前 HEAD）
  ├── active_branch（当前分支）
  └── working_dir（工作目录）

Commit（提交）
  ├── hexsha（提交哈希）
  ├── message（提交消息）
  ├── author / committer（作者/提交者）
  ├── authored_date / committed_date（时间）
  ├── parents（父提交列表）
  ├── tree（文件树）
  └── diff()（差异比较）

Tree / Blob（目录树 / 文件）
  ├── tree → 目录
  └── blob → 文件内容
```

### 3.2 操作映射

| Git 命令 | GitPython |
|----------|-----------|
| `git init` | `Repo.init()` |
| `git clone` | `Repo.clone_from()` |
| `git status` | `repo.is_dirty()`, `repo.untracked_files` |
| `git add` | `repo.index.add()` |
| `git commit` | `repo.index.commit()` |
| `git log` | `repo.iter_commits()` |
| `git diff` | `commit.diff()` |
| `git branch` | `repo.heads` |
| `git checkout` | `repo.heads.branch.checkout()` |
| `git pull` | `repo.remotes.origin.pull()` |
| `git push` | `repo.remotes.origin.push()` |

## 4. 基础用法

### 4.1 仓库初始化与基本操作

```python
#!/usr/bin/env python3
"""GitPython 基础操作"""

import git
from pathlib import Path
import os

# === 初始化仓库 ===
repo_path = Path("/tmp/test-git-repo")
if repo_path.exists():
    import shutil
    shutil.rmtree(repo_path)

repo = git.Repo.init(repo_path)
print(f"仓库已初始化: {repo.working_dir}")
print(f"是否空仓库: {repo.bare}")

# === 创建文件并提交 ===
# 创建文件
readme = repo_path / "README.md"
readme.write_text("# SRE 工具箱\n\n自动化运维工具集合\n")

config = repo_path / "config.yaml"
config.write_text("server:\n  host: 0.0.0.0\n  port: 8080\n")

# 添加到暂存区
repo.index.add(["README.md", "config.yaml"])

# 提交
commit = repo.index.commit(
    "初始提交: 添加 README 和配置文件",
    author=git.Actor("SRE Bot", "sre@example.com"),
    committer=git.Actor("SRE Bot", "sre@example.com"),
)
print(f"\n提交成功: {commit.hexsha[:8]} - {commit.message}")

# === 查看仓库状态 ===
print(f"\n=== 仓库状态 ===")
print(f"当前分支: {repo.active_branch.name}")
print(f"是否有未提交更改: {repo.is_dirty()}")
print(f"未跟踪文件: {repo.untracked_files}")

# === 修改文件并提交 ===
config.write_text("server:\n  host: 0.0.0.0\n  port: 9090\n  workers: 4\n")
repo.index.add(["config.yaml"])
commit2 = repo.index.commit("更新配置: 修改端口和添加 workers 参数")
print(f"\n第二次提交: {commit2.hexsha[:8]} - {commit2.message}")

# === 查看提交日志 ===
print(f"\n=== 提交历史 ===")
for c in repo.iter_commits(max_count=10):
    print(f"  {c.hexsha[:8]} {c.authored_datetime.strftime('%Y-%m-%d %H:%M')} "
          f"[{c.author.name}] {c.message.strip()}")
```

### 4.2 分支管理

```python
#!/usr/bin/env python3
"""分支管理操作"""

import git
from pathlib import Path

# 使用上面创建的仓库
repo_path = "/tmp/test-git-repo"
repo = git.Repo(repo_path)

# === 创建分支 ===
feature_branch = repo.create_head("feature/add-monitoring")
print(f"创建分支: {feature_branch.name}")

# === 切换分支 ===
feature_branch.checkout()
print(f"当前分支: {repo.active_branch.name}")

# 在新分支上提交
monitoring_config = Path(repo_path) / "monitoring.yaml"
monitoring_config.write_text("prometheus:\n  port: 9090\n  scrape_interval: 15s\n")
repo.index.add(["monitoring.yaml"])
repo.index.commit("添加监控配置")

# === 列出所有分支 ===
print(f"\n=== 所有分支 ===")
for branch in repo.heads:
    is_current = "*" if branch == repo.active_branch else " "
    print(f"  {is_current} {branch.name} -> {branch.commit.hexsha[:8]}")

# === 切回主分支 ===
# 检测主分支名称（可能是 master 或 main）
main_branch = repo.heads[0]  # 第一个分支通常是主分支
main_branch.checkout()
print(f"\n切换到: {repo.active_branch.name}")

# === 合并分支 ===
repo.git.merge("feature/add-monitoring")
print(f"合并完成: feature/add-monitoring → {repo.active_branch.name}")

# === 标签管理 ===
tag = repo.create_tag("v1.0.0", message="Release v1.0.0: 初始版本")
print(f"\n创建标签: {tag.name}")

print(f"\n=== 所有标签 ===")
for t in repo.tags:
    print(f"  {t.name} -> {t.commit.hexsha[:8]}")

# === 删除分支 ===
repo.delete_head("feature/add-monitoring", force=True)
print(f"\n已删除分支: feature/add-monitoring")
```

### 4.3 Diff 比较

```python
#!/usr/bin/env python3
"""查看代码变更（diff）"""

import git

repo = git.Repo("/tmp/test-git-repo")

# === 查看两次提交之间的差异 ===
commits = list(repo.iter_commits(max_count=3))

if len(commits) >= 2:
    latest = commits[0]
    previous = commits[1]

    print(f"比较: {previous.hexsha[:8]} → {latest.hexsha[:8]}")
    diffs = previous.diff(latest)

    for diff in diffs:
        change_type = {
            'A': '新增',
            'D': '删除',
            'M': '修改',
            'R': '重命名',
        }.get(diff.change_type, diff.change_type)

        print(f"\n  [{change_type}] {diff.a_path or diff.b_path}")

        # 显示具体变更内容
        if diff.a_blob and diff.b_blob:
            try:
                old_content = diff.a_blob.data_stream.read().decode('utf-8')
                new_content = diff.b_blob.data_stream.read().decode('utf-8')
                print(f"    旧内容 ({len(old_content)} bytes)")
                print(f"    新内容 ({len(new_content)} bytes)")
            except UnicodeDecodeError:
                print(f"    (二进制文件)")

# === 工作区与暂存区的差异 ===
print(f"\n=== 工作区修改 ===")
if repo.is_dirty():
    for diff in repo.index.diff(None):
        print(f"  [{diff.change_type}] {diff.a_path}")
else:
    print("  无修改")
```

## 5. 进阶用法

### 5.1 克隆远程仓库

```python
#!/usr/bin/env python3
"""克隆远程仓库"""

import git
from pathlib import Path
import shutil

def clone_repo(url: str, local_path: str, branch: str = None, depth: int = None) -> git.Repo:
    """克隆远程仓库"""
    path = Path(local_path)
    if path.exists():
        shutil.rmtree(path)

    kwargs = {}
    if branch:
        kwargs['branch'] = branch
    if depth:
        kwargs['depth'] = depth  # 浅克隆

    print(f"克隆: {url} → {local_path}")
    repo = git.Repo.clone_from(url, local_path, **kwargs)
    print(f"完成: {repo.head.commit.hexsha[:8]} [{repo.active_branch.name}]")
    return repo

# 示例（浅克隆，只获取最近 1 次提交）
# repo = clone_repo(
#     "https://github.com/user/repo.git",
#     "/tmp/cloned-repo",
#     branch="main",
#     depth=1,
# )

# 打印仓库信息
def repo_info(repo: git.Repo):
    """显示仓库信息"""
    print(f"\n=== 仓库信息 ===")
    print(f"  路径: {repo.working_dir}")
    print(f"  当前分支: {repo.active_branch.name}")
    print(f"  HEAD: {repo.head.commit.hexsha[:8]}")
    print(f"  分支数: {len(repo.heads)}")
    print(f"  标签数: {len(repo.tags)}")
    print(f"  远程仓库: {[r.name for r in repo.remotes]}")
    if repo.remotes:
        for remote in repo.remotes:
            print(f"    {remote.name}: {list(remote.urls)}")

# 对已有仓库查看信息
repo_info(git.Repo("/tmp/test-git-repo"))
```

### 5.2 搜索与统计

```python
#!/usr/bin/env python3
"""Git 历史搜索与统计"""

import git
from collections import defaultdict, Counter
from datetime import datetime

repo = git.Repo("/tmp/test-git-repo")

# === 搜索包含特定关键词的提交 ===
def search_commits(repo: git.Repo, keyword: str, max_count: int = 100):
    """搜索提交消息中包含关键词的提交"""
    results = []
    for commit in repo.iter_commits(max_count=max_count):
        if keyword.lower() in commit.message.lower():
            results.append({
                'hash': commit.hexsha[:8],
                'date': datetime.fromtimestamp(commit.authored_date).strftime('%Y-%m-%d'),
                'author': commit.author.name,
                'message': commit.message.strip(),
            })
    return results

print("=== 搜索提交 ===")
results = search_commits(repo, "配置")
for r in results:
    print(f"  {r['hash']} [{r['date']}] {r['author']}: {r['message']}")

# === 按作者统计提交 ===
def commit_stats_by_author(repo: git.Repo, max_count: int = 1000):
    """按作者统计提交数"""
    author_stats = Counter()
    for commit in repo.iter_commits(max_count=max_count):
        author_stats[commit.author.name] += 1
    return author_stats.most_common()

print(f"\n=== 作者提交统计 ===")
for author, count in commit_stats_by_author(repo):
    print(f"  {author}: {count} 次")

# === 文件变更频率统计 ===
def file_change_frequency(repo: git.Repo, max_count: int = 100):
    """统计文件变更频率"""
    file_stats = Counter()
    commits = list(repo.iter_commits(max_count=max_count))

    for i, commit in enumerate(commits):
        if commit.parents:
            diffs = commit.parents[0].diff(commit)
            for diff in diffs:
                path = diff.b_path or diff.a_path
                file_stats[path] += 1

    return file_stats.most_common()

print(f"\n=== 文件变更频率 ===")
for filepath, count in file_change_frequency(repo):
    print(f"  {filepath}: {count} 次")
```

## 6. SRE 实战案例：代码变更自动审计工具

```python
#!/usr/bin/env python3
"""
SRE 实战：代码变更自动审计工具
功能：
1. 分析指定时间范围的代码变更
2. 检测敏感文件修改（配置文件、密钥等）
3. 检测大文件提交
4. 统计提交活跃度
5. 生成审计报告
6. 检测危险操作（force push 等）
"""

import git
import re
import json
from pathlib import Path
from datetime import datetime, timedelta
from collections import defaultdict, Counter
from typing import Dict, List, Optional, Set
from dataclasses import dataclass, field, asdict

# ============================================================
# 配置
# ============================================================
AUDIT_CONFIG = {
    # 敏感文件模式
    'sensitive_patterns': [
        r'\.env$',
        r'\.key$',
        r'\.pem$',
        r'credentials',
        r'secret',
        r'password',
        r'\.kube/config',
        r'id_rsa',
        r'\.p12$',
        r'\.pfx$',
    ],
    # 危险内容模式
    'dangerous_content_patterns': [
        r'password\s*=\s*["\'][^"\']+["\']',
        r'api[_-]?key\s*=\s*["\'][^"\']+["\']',
        r'secret\s*=\s*["\'][^"\']+["\']',
        r'token\s*=\s*["\'][^"\']+["\']',
        r'-----BEGIN.*PRIVATE KEY-----',
    ],
    # 配置文件模式
    'config_patterns': [
        r'\.ya?ml$',
        r'\.json$',
        r'\.toml$',
        r'\.conf$',
        r'\.ini$',
        r'nginx\.conf',
        r'prometheus',
        r'Dockerfile',
        r'docker-compose',
    ],
    # 大文件阈值（字节）
    'large_file_threshold': 1 * 1024 * 1024,  # 1MB
}

# ============================================================
# 数据模型
# ============================================================
@dataclass
class AuditFinding:
    """审计发现"""
    severity: str         # info / warning / critical
    category: str         # sensitive_file / large_file / dangerous_content / config_change
    commit_hash: str
    author: str
    date: str
    file_path: str
    message: str
    details: str = ""

@dataclass
class CommitAuditResult:
    """提交审计结果"""
    commit_hash: str
    author: str
    date: str
    message: str
    files_changed: int
    insertions: int
    deletions: int
    findings: List[AuditFinding] = field(default_factory=list)

# ============================================================
# Git 审计器
# ============================================================
class GitAuditor:
    """Git 代码变更审计器"""

    def __init__(self, repo_path: str, config: dict = None):
        self.repo = git.Repo(repo_path)
        self.config = config or AUDIT_CONFIG
        self._compile_patterns()
        self.findings: List[AuditFinding] = []
        self.commit_results: List[CommitAuditResult] = []

    def _compile_patterns(self):
        """预编译正则模式"""
        self.sensitive_re = [re.compile(p, re.I) for p in self.config['sensitive_patterns']]
        self.dangerous_re = [re.compile(p, re.I) for p in self.config['dangerous_content_patterns']]
        self.config_re = [re.compile(p, re.I) for p in self.config['config_patterns']]

    def _is_sensitive_file(self, filepath: str) -> bool:
        """检查是否是敏感文件"""
        return any(p.search(filepath) for p in self.sensitive_re)

    def _is_config_file(self, filepath: str) -> bool:
        """检查是否是配置文件"""
        return any(p.search(filepath) for p in self.config_re)

    def _check_dangerous_content(self, content: str) -> List[str]:
        """检查内容中的危险模式"""
        matches = []
        for pattern in self.dangerous_re:
            found = pattern.findall(content)
            if found:
                matches.extend(found)
        return matches

    def audit_commit(self, commit: git.Commit) -> CommitAuditResult:
        """审计单个提交"""
        result = CommitAuditResult(
            commit_hash=commit.hexsha[:8],
            author=commit.author.name,
            date=datetime.fromtimestamp(commit.authored_date).strftime('%Y-%m-%d %H:%M'),
            message=commit.message.strip()[:100],
            files_changed=0,
            insertions=0,
            deletions=0,
        )

        # 获取变更的文件
        if commit.parents:
            diffs = commit.parents[0].diff(commit)
        else:
            diffs = commit.diff(git.NULL_TREE)

        result.files_changed = len(diffs)

        for diff in diffs:
            filepath = diff.b_path or diff.a_path or ""

            # 检查 1: 敏感文件
            if self._is_sensitive_file(filepath):
                finding = AuditFinding(
                    severity='critical',
                    category='sensitive_file',
                    commit_hash=result.commit_hash,
                    author=result.author,
                    date=result.date,
                    file_path=filepath,
                    message=f"敏感文件被修改: {filepath}",
                    details=f"变更类型: {diff.change_type}",
                )
                result.findings.append(finding)
                self.findings.append(finding)

            # 检查 2: 配置文件变更
            if self._is_config_file(filepath):
                finding = AuditFinding(
                    severity='info',
                    category='config_change',
                    commit_hash=result.commit_hash,
                    author=result.author,
                    date=result.date,
                    file_path=filepath,
                    message=f"配置文件变更: {filepath}",
                    details=f"变更类型: {diff.change_type}",
                )
                result.findings.append(finding)
                self.findings.append(finding)

            # 检查 3: 大文件
            if diff.b_blob:
                try:
                    file_size = diff.b_blob.size
                    if file_size > self.config['large_file_threshold']:
                        finding = AuditFinding(
                            severity='warning',
                            category='large_file',
                            commit_hash=result.commit_hash,
                            author=result.author,
                            date=result.date,
                            file_path=filepath,
                            message=f"大文件提交: {filepath} ({file_size/1024/1024:.1f}MB)",
                        )
                        result.findings.append(finding)
                        self.findings.append(finding)
                except Exception:
                    pass

            # 检查 4: 危险内容
            if diff.b_blob and diff.change_type in ('A', 'M'):
                try:
                    content = diff.b_blob.data_stream.read().decode('utf-8', errors='ignore')
                    dangers = self._check_dangerous_content(content)
                    if dangers:
                        finding = AuditFinding(
                            severity='critical',
                            category='dangerous_content',
                            commit_hash=result.commit_hash,
                            author=result.author,
                            date=result.date,
                            file_path=filepath,
                            message=f"疑似敏感信息泄露: {filepath}",
                            details=f"匹配模式: {len(dangers)} 处",
                        )
                        result.findings.append(finding)
                        self.findings.append(finding)
                except Exception:
                    pass

        return result

    def audit_range(self, since: str = None, until: str = None,
                    max_count: int = 100, branch: str = None):
        """审计指定范围的提交"""
        kwargs = {'max_count': max_count}
        if since:
            kwargs['since'] = since
        if until:
            kwargs['until'] = until

        commits = list(self.repo.iter_commits(branch or self.repo.active_branch, **kwargs))
        print(f"\n审计 {len(commits)} 个提交...")

        for commit in commits:
            result = self.audit_commit(commit)
            self.commit_results.append(result)

            # 有发现时输出
            if result.findings:
                for f in result.findings:
                    icon = {'critical': '!!', 'warning': '!', 'info': 'i'}.get(f.severity, '?')
                    print(f"  [{icon}] {f.commit_hash} {f.message}")

    def statistics(self) -> Dict:
        """统计信息"""
        stats = {
            'total_commits': len(self.commit_results),
            'total_findings': len(self.findings),
            'by_severity': Counter(f.severity for f in self.findings),
            'by_category': Counter(f.category for f in self.findings),
            'by_author': Counter(r.author for r in self.commit_results),
            'top_changed_files': Counter(),
        }

        for r in self.commit_results:
            for f in r.findings:
                stats['top_changed_files'][f.file_path] += 1

        return stats

    def generate_report(self) -> str:
        """生成审计报告"""
        stats = self.statistics()
        lines = [
            f"{'='*60}",
            f"Git 代码变更审计报告",
            f"{'='*60}",
            f"仓库: {self.repo.working_dir}",
            f"分支: {self.repo.active_branch.name}",
            f"时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"",
            f"--- 汇总 ---",
            f"  审计提交数: {stats['total_commits']}",
            f"  发现问题数: {stats['total_findings']}",
        ]

        # 按严重级别
        lines.append(f"\n--- 按严重级别 ---")
        for sev in ['critical', 'warning', 'info']:
            count = stats['by_severity'].get(sev, 0)
            if count:
                lines.append(f"  {sev}: {count}")

        # 按类别
        lines.append(f"\n--- 按类别 ---")
        category_names = {
            'sensitive_file': '敏感文件',
            'large_file': '大文件',
            'dangerous_content': '危险内容',
            'config_change': '配置变更',
        }
        for cat, count in stats['by_category'].most_common():
            lines.append(f"  {category_names.get(cat, cat)}: {count}")

        # 按作者
        lines.append(f"\n--- 按作者 ---")
        for author, count in stats['by_author'].most_common(10):
            lines.append(f"  {author}: {count} 次提交")

        # 重要发现详情
        critical_findings = [f for f in self.findings if f.severity == 'critical']
        if critical_findings:
            lines.append(f"\n--- 严重问题详情 ---")
            for f in critical_findings:
                lines.append(f"  [{f.severity.upper()}] {f.commit_hash} [{f.date}] {f.author}")
                lines.append(f"    {f.message}")
                if f.details:
                    lines.append(f"    {f.details}")

        return '\n'.join(lines)


# ============================================================
# 演示
# ============================================================
def main():
    # 准备测试仓库（添加一些敏感文件模拟）
    repo_path = "/tmp/test-git-repo"
    repo = git.Repo(repo_path)

    # 添加模拟的敏感文件
    env_file = Path(repo_path) / ".env"
    env_file.write_text("DB_PASSWORD=secret123\nAPI_KEY=abc-xyz-123\n")
    repo.index.add([".env"])
    repo.index.commit(
        "添加环境配置（不应提交到仓库）",
        author=git.Actor("Dev A", "deva@example.com"),
    )

    # 添加配置文件变更
    config = Path(repo_path) / "config.yaml"
    config.write_text("server:\n  host: 0.0.0.0\n  port: 9090\n  secret: my_token_123\n")
    repo.index.add(["config.yaml"])
    repo.index.commit(
        "更新服务器配置",
        author=git.Actor("Dev B", "devb@example.com"),
    )

    # 执行审计
    auditor = GitAuditor(repo_path)
    auditor.audit_range(max_count=50)

    # 生成报告
    report = auditor.generate_report()
    print(report)

    # 保存报告
    report_path = Path("/tmp/git_audit_report.txt")
    report_path.write_text(report)
    print(f"\n报告已保存: {report_path}")

if __name__ == '__main__':
    main()
```

## 7. 常见坑点与最佳实践

### 坑点

1. **空仓库操作**：新初始化的仓库没有 HEAD，某些操作会报错
2. **分离 HEAD 状态**：`repo.active_branch` 在 detached HEAD 时会异常
3. **二进制文件**：读取 blob 内容时需处理编码异常
4. **大仓库性能**：`iter_commits` 不要全量遍历，用 `max_count` 限制
5. **权限问题**：SSH 克隆需要配置密钥

### 最佳实践

```python
# 1. 使用 iter_commits 的 max_count 限制
for commit in repo.iter_commits(max_count=100):
    ...

# 2. 异常处理
try:
    branch = repo.active_branch
except TypeError:
    print("HEAD 处于分离状态")

# 3. 使用 depth 做浅克隆
Repo.clone_from(url, path, depth=1)

# 4. 使用 with 上下文管理器
# 5. 敏感信息检查加入 CI/CD
```

## 8. 速查表

```python
import git

# === 仓库 ===
repo = git.Repo('.')                          # 打开仓库
repo = git.Repo.init('/path')                 # 初始化
repo = git.Repo.clone_from(url, path)         # 克隆

# === 状态 ===
repo.is_dirty()                               # 有未提交修改
repo.untracked_files                          # 未跟踪文件
repo.active_branch                            # 当前分支

# === 提交 ===
repo.index.add(['file.txt'])                  # 暂存
repo.index.commit('message')                  # 提交
repo.iter_commits(max_count=10)               # 遍历提交

# === 分支 ===
repo.create_head('branch-name')               # 创建分支
repo.heads                                     # 所有分支
branch.checkout()                              # 切换

# === Diff ===
commit.diff(other_commit)                     # 两次提交间差异
repo.index.diff(None)                         # 工作区 vs 暂存区

# === 远程 ===
repo.remotes.origin.pull()                    # 拉取
repo.remotes.origin.push()                    # 推送
```
