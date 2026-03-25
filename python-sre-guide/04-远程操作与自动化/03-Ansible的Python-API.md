# Ansible Python API — 用 Python 代码调用 Ansible 实现配置管理自动化

## 1. 是什么 & 为什么用它

### 什么是 Ansible Python API

Ansible 不仅是一个命令行工具（`ansible`、`ansible-playbook`），它本身就是用 Python 编写的，提供了完整的 Python API。通过这些 API，你可以在 Python 代码中直接调用 Ansible 的功能，实现更灵活的自动化。

### 为什么 SRE 需要 Ansible Python API

| 场景 | 命令行方式 | Python API 方式 |
|---|---|---|
| 执行 Playbook | `ansible-playbook site.yml` | 代码中调用，可集成到 Web/API |
| 动态 Inventory | 写 Shell/Python 脚本输出 JSON | Python 类，原生集成 |
| 自定义模块 | 独立 Python 文件 | 可直接复用项目代码 |
| 结果处理 | 解析终端输出 | 结构化 Python 对象 |
| 流程控制 | Jinja2 条件判断 | Python 原生 if/else/循环 |

```
命令行方式的局限:
  $ ansible-playbook -i inventory.yml deploy.yml -e "version=2.0"
  # 结果只能从终端输出或日志文件中解析
  # 无法在代码中动态决定下一步操作
  # 集成到 Web 界面/API 困难

Python API 的优势:
  - 在代码中灵活控制执行逻辑
  - 结果以 Python 对象返回，便于处理
  - 可集成到 Flask/Django/FastAPI 等 Web 框架
  - 动态生成 Inventory 和变量
  - 自定义回调处理（发送通知、写数据库等）
```

## 2. 安装与环境

```bash
# 安装 Ansible
pip install ansible

# 验证安装
python3 -c "import ansible; print(ansible.__version__)"
ansible --version

# 如果需要特定版本
pip install ansible-core==2.15.0

# 确认 API 可用
python3 -c "
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.inventory.manager import InventoryManager
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
print('Ansible Python API 可用')
"
```

**注意：** Ansible Python API 在不同版本间可能有变化，本教程基于 ansible-core 2.14+。

## 3. 核心概念与原理

### Ansible 内部架构

```
+------------------------------------------------------------------+
|                     你的 Python SRE 脚本                           |
+------------------------------------------------------------------+
|                     Ansible Python API                            |
|                                                                   |
|  TaskQueueManager    PlaybookExecutor    AdHocCommandRunner       |
|  (任务队列管理器)     (Playbook 执行器)   (临时命令执行)            |
+------------------------------------------------------------------+
|                     核心组件                                       |
|                                                                   |
|  DataLoader     InventoryManager    VariableManager    Play       |
|  (数据加载)     (主机清单管理)      (变量管理)        (剧本定义)   |
+------------------------------------------------------------------+
|                     模块系统                                       |
|                                                                   |
|  ActionModule    CallbackPlugin     Connection        Module      |
|  (动作模块)      (回调插件)         (连接方式)        (功能模块)   |
+------------------------------------------------------------------+
|                     传输层                                         |
|                                                                   |
|  SSH (paramiko)     Local     Docker     WinRM                    |
+------------------------------------------------------------------+
```

### 核心类关系

```
执行流程:

  DataLoader (加载 YAML/JSON 数据)
       |
  InventoryManager (解析主机清单)
       |
  VariableManager (管理变量)
       |
  Play (定义任务)
       |
  TaskQueueManager (调度执行)
       |
       +-- Connection (SSH/Local 连接)
       |       |
       +-- ActionModule (执行动作)
       |       |
       +-- CallbackPlugin (处理结果)
               |
           结果返回给你的代码
```

### 对比命令行 vs Python API

```
命令行:
  ansible webservers -m ping
  ansible-playbook -i hosts.yml deploy.yml -e version=2.0

Python API 等效:
  # 见下面的代码示例
  # 更灵活: 可以在代码中决定执行什么、对哪些主机执行
```

## 4. 基础用法

### 4.1 Ad-Hoc 命令执行（对比 ansible 命令）

```python
#!/usr/bin/env python3
"""
Ansible Ad-Hoc 命令执行
对比: ansible all -i hosts -m shell -a "uptime"
"""

import json
import shutil
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.inventory.manager import InventoryManager
from ansible.module_utils.common.collections import ImmutableDict
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
from ansible.plugins.callback import CallbackBase
from ansible import context

# --- 自定义回调插件（收集结果）---
class ResultCallback(CallbackBase):
    """
    自定义回调：收集任务执行结果
    对比命令行: ansible 默认将结果输出到终端
    Python API 中我们可以自定义如何处理结果
    """
    CALLBACK_VERSION = 2.0

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.host_ok = {}       # 成功的主机
        self.host_unreachable = {}  # 不可达的主机
        self.host_failed = {}   # 执行失败的主机

    def v2_runner_on_ok(self, result, **kwargs):
        """任务成功时的回调"""
        host = result._host.get_name()
        self.host_ok[host] = result._result

    def v2_runner_on_unreachable(self, result):
        """主机不可达时的回调"""
        host = result._host.get_name()
        self.host_unreachable[host] = result._result

    def v2_runner_on_failed(self, result, ignore_errors=False):
        """任务失败时的回调"""
        host = result._host.get_name()
        self.host_failed[host] = result._result


def run_adhoc(hosts, module_name, module_args='', become=False,
              inventory_source=None):
    """
    执行 Ad-Hoc 命令
    对比: ansible <hosts> -m <module> -a "<args>"

    :param hosts: 目标主机模式（如 'all', 'webservers'）
    :param module_name: 模块名（如 'shell', 'ping', 'copy'）
    :param module_args: 模块参数
    :param become: 是否使用 sudo
    :param inventory_source: Inventory 来源
    :return: 回调结果对象
    """
    # 设置 Ansible 全局选项
    # 对比: ansible 命令行的各种 -o 选项
    context.CLIARGS = ImmutableDict(
        connection='ssh',           # 连接方式（ssh/local/docker）
        module_path=[],             # 自定义模块路径
        forks=10,                   # 并发数（对比 ansible -f 10）
        become=become,              # 是否 sudo
        become_method='sudo',
        become_user='root',
        check=False,                # 是否为检查模式（dry-run）
        diff=False,
        verbosity=0,
        syntax=None,
        start_at_task=None,
    )

    # 数据加载器
    loader = DataLoader()

    # 密码（如果需要）
    passwords = {}  # {'become_pass': 'sudo_password'}

    # 回调插件
    results_callback = ResultCallback()

    # Inventory 管理器
    # 对比: ansible -i hosts.yml 或 ansible -i 'host1,host2,'
    if inventory_source is None:
        # 默认使用逗号分隔的主机列表
        inventory_source = '127.0.0.1,'
    inventory = InventoryManager(loader=loader, sources=[inventory_source])

    # 变量管理器
    from ansible.vars.manager import VariableManager
    variable_manager = VariableManager(loader=loader, inventory=inventory)

    # 定义 Play（相当于一个最小的 Playbook）
    play_source = dict(
        name=f"Ad-Hoc: {module_name}",
        hosts=hosts,
        gather_facts='no',  # 不收集 facts（加快执行）
        tasks=[
            dict(
                action=dict(
                    module=module_name,
                    args=module_args,
                ),
            ),
        ],
    )
    play = Play().load(play_source, variable_manager=variable_manager,
                       loader=loader)

    # 执行
    tqm = None
    try:
        tqm = TaskQueueManager(
            inventory=inventory,
            variable_manager=variable_manager,
            loader=loader,
            passwords=passwords,
            stdout_callback=results_callback,
        )
        result = tqm.run(play)
    finally:
        if tqm is not None:
            tqm.cleanup()
        # 清理临时文件
        shutil.rmtree(loader.get_basedir(), True)

    return results_callback


# ============================================================
# 使用示例
# ============================================================

# 对比: ansible all -i '127.0.0.1,' -m ping
print("=== Ping 测试 ===")
callback = run_adhoc(
    hosts='all',
    module_name='ping',
    inventory_source='127.0.0.1,',
)

for host, result in callback.host_ok.items():
    print(f"  [OK] {host}: {result.get('ping', '')}")
for host, result in callback.host_unreachable.items():
    print(f"  [不可达] {host}: {result.get('msg', '')}")
for host, result in callback.host_failed.items():
    print(f"  [失败] {host}: {result.get('msg', '')}")

# 对比: ansible all -i '127.0.0.1,' -m shell -a "uptime"
print("\n=== 执行命令 ===")
callback = run_adhoc(
    hosts='all',
    module_name='shell',
    module_args='hostname && uptime && df -h /',
    inventory_source='127.0.0.1,',
)

for host, result in callback.host_ok.items():
    print(f"  [{host}]")
    print(f"  {result.get('stdout', '')}")
```

### 4.2 执行 Playbook（对比 ansible-playbook）

```python
#!/usr/bin/env python3
"""
执行 Ansible Playbook
对比: ansible-playbook -i inventory.yml playbook.yml -e "var=value"
"""

import os
import json
from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.inventory.manager import InventoryManager
from ansible.module_utils.common.collections import ImmutableDict
from ansible.parsing.dataloader import DataLoader
from ansible.plugins.callback import CallbackBase
from ansible.vars.manager import VariableManager
from ansible import context


class PlaybookResultCallback(CallbackBase):
    """Playbook 执行结果回调"""
    CALLBACK_VERSION = 2.0

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.task_results = []
        self.play_recap = {}

    def v2_runner_on_ok(self, result, **kwargs):
        host = result._host.get_name()
        task = result._task.get_name()
        self.task_results.append({
            'host': host,
            'task': task,
            'status': 'ok',
            'changed': result._result.get('changed', False),
            'result': result._result,
        })
        changed = "(changed)" if result._result.get('changed') else ""
        print(f"  ok: [{host}] {task} {changed}")

    def v2_runner_on_failed(self, result, ignore_errors=False):
        host = result._host.get_name()
        task = result._task.get_name()
        self.task_results.append({
            'host': host,
            'task': task,
            'status': 'failed',
            'result': result._result,
        })
        print(f"  FAILED: [{host}] {task}")
        print(f"    msg: {result._result.get('msg', '')}")

    def v2_runner_on_unreachable(self, result):
        host = result._host.get_name()
        self.task_results.append({
            'host': host,
            'task': 'connection',
            'status': 'unreachable',
            'result': result._result,
        })
        print(f"  UNREACHABLE: [{host}]")

    def v2_runner_on_skipped(self, result):
        host = result._host.get_name()
        task = result._task.get_name()
        print(f"  skipping: [{host}] {task}")

    def v2_playbook_on_play_start(self, play):
        print(f"\nPLAY [{play.get_name()}]")
        print("=" * 60)

    def v2_playbook_on_task_start(self, task, is_conditional):
        print(f"\nTASK [{task.get_name()}]")
        print("-" * 40)

    def v2_playbook_on_stats(self, stats):
        """Playbook 执行统计"""
        print(f"\nPLAY RECAP")
        print("=" * 60)
        hosts = sorted(stats.processed.keys())
        for host in hosts:
            s = stats.summarize(host)
            self.play_recap[host] = s
            print(f"  {host}: ok={s['ok']} changed={s['changed']} "
                  f"unreachable={s['unreachable']} failed={s['failures']} "
                  f"skipped={s['skipped']}")


def run_playbook(playbook_path, inventory_source, extra_vars=None,
                 tags=None, limit=None, become=False):
    """
    执行 Playbook
    对比: ansible-playbook -i <inventory> <playbook> [-e vars] [-t tags] [-l limit]

    :param playbook_path: Playbook YAML 文件路径
    :param inventory_source: Inventory 来源
    :param extra_vars: 额外变量字典
    :param tags: 要执行的标签列表
    :param limit: 限制执行的主机
    :param become: 是否使用 sudo
    """
    context.CLIARGS = ImmutableDict(
        connection='ssh',
        forks=10,
        become=become,
        become_method='sudo',
        become_user='root',
        check=False,
        diff=False,
        verbosity=0,
        syntax=None,
        start_at_task=None,
        listtags=False,
        listtasks=False,
        listhosts=False,
        tags=tags or [],
        skip_tags=[],
        module_path=[],
    )

    loader = DataLoader()
    inventory = InventoryManager(loader=loader, sources=[inventory_source])
    variable_manager = VariableManager(loader=loader, inventory=inventory)

    # 设置额外变量（对比 ansible-playbook -e "var=value"）
    if extra_vars:
        variable_manager._extra_vars = extra_vars

    # 设置限制（对比 ansible-playbook -l "host1,host2"）
    if limit:
        inventory.subset(limit)

    # 回调
    callback = PlaybookResultCallback()

    # 执行
    pbex = PlaybookExecutor(
        playbooks=[playbook_path],
        inventory=inventory,
        variable_manager=variable_manager,
        loader=loader,
        passwords={},
    )
    pbex._tqm._stdout_callback = callback

    result = pbex.run()

    return {
        'rc': result,  # 0 成功, 非 0 失败
        'task_results': callback.task_results,
        'recap': callback.play_recap,
    }


# 使用示例
# result = run_playbook(
#     playbook_path='/path/to/deploy.yml',
#     inventory_source='/path/to/inventory.yml',
#     extra_vars={'app_version': '2.0', 'env': 'production'},
#     tags=['deploy', 'restart'],
#     become=True,
# )
# print(f"执行结果: {'成功' if result['rc'] == 0 else '失败'}")
```

### 4.3 动态 Inventory

```python
#!/usr/bin/env python3
"""
动态 Inventory —— 从数据库/API/CMDB 动态生成主机清单
对比: ansible -i dynamic_inventory.py 或 inventory 插件
"""

import json
import os
import tempfile
from pathlib import Path


class DynamicInventory:
    """
    动态 Inventory 生成器
    可以从各种数据源获取主机信息

    Ansible Inventory JSON 格式:
    {
        "group_name": {
            "hosts": ["host1", "host2"],
            "vars": {"group_var": "value"},
            "children": ["child_group"]
        },
        "_meta": {
            "hostvars": {
                "host1": {"host_var": "value"}
            }
        }
    }
    """

    def __init__(self):
        self.inventory = {
            '_meta': {
                'hostvars': {}
            }
        }

    def add_group(self, name, vars=None):
        """添加主机组"""
        self.inventory[name] = {
            'hosts': [],
            'vars': vars or {},
            'children': [],
        }
        return self

    def add_host(self, group, hostname, vars=None):
        """添加主机到组"""
        if group not in self.inventory:
            self.add_group(group)

        self.inventory[group]['hosts'].append(hostname)

        if vars:
            self.inventory['_meta']['hostvars'][hostname] = vars

        return self

    def add_child_group(self, parent, child):
        """添加子组"""
        if parent not in self.inventory:
            self.add_group(parent)
        self.inventory[parent]['children'].append(child)
        return self

    def to_json(self):
        """输出 JSON 格式"""
        return json.dumps(self.inventory, indent=2)

    def to_file(self, filepath=None):
        """输出到文件"""
        if filepath is None:
            fd, filepath = tempfile.mkstemp(suffix='.json')
            os.close(fd)

        with open(filepath, 'w') as f:
            json.dump(self.inventory, f, indent=2)

        return filepath

    @classmethod
    def from_cmdb_api(cls, api_url, api_token=None):
        """从 CMDB API 获取主机信息（示例）"""
        inventory = cls()

        # 模拟从 API 获取数据
        # 实际项目中使用 requests.get(api_url, headers=...)
        mock_hosts = [
            {'hostname': '10.0.1.1', 'group': 'webservers',
             'env': 'prod', 'role': 'nginx'},
            {'hostname': '10.0.1.2', 'group': 'webservers',
             'env': 'prod', 'role': 'nginx'},
            {'hostname': '10.0.2.1', 'group': 'dbservers',
             'env': 'prod', 'role': 'mysql'},
            {'hostname': '10.0.3.1', 'group': 'monitoring',
             'env': 'prod', 'role': 'prometheus'},
        ]

        for host in mock_hosts:
            inventory.add_host(
                group=host['group'],
                hostname=host['hostname'],
                vars={
                    'ansible_user': 'deployer',
                    'env': host['env'],
                    'role': host['role'],
                },
            )

        # 创建父组
        inventory.add_group('production', vars={'env': 'prod'})
        inventory.add_child_group('production', 'webservers')
        inventory.add_child_group('production', 'dbservers')
        inventory.add_child_group('production', 'monitoring')

        return inventory

    @classmethod
    def from_dict(cls, hosts_dict):
        """
        从简单字典创建 Inventory
        格式: {'group': [{'host': 'ip', 'user': 'xx', ...}]}
        """
        inventory = cls()

        for group, hosts in hosts_dict.items():
            for host_info in hosts:
                hostname = host_info.pop('host')
                inventory.add_host(group, hostname, vars=host_info)

        return inventory


# ============================================================
# 使用示例
# ============================================================

# 手动构建
inv = DynamicInventory()
inv.add_group('webservers', vars={'http_port': 80})
inv.add_host('webservers', '192.168.1.101',
             vars={'ansible_user': 'root', 'ansible_password': 'xxx'})
inv.add_host('webservers', '192.168.1.102',
             vars={'ansible_user': 'root', 'ansible_password': 'xxx'})
inv.add_group('dbservers')
inv.add_host('dbservers', '192.168.1.201',
             vars={'ansible_user': 'dba', 'db_port': 3306})

print("=== 动态 Inventory ===")
print(inv.to_json())

# 从模拟 CMDB 获取
print("\n=== 从 CMDB 获取 ===")
cmdb_inv = DynamicInventory.from_cmdb_api('http://cmdb.example.com/api/hosts')
print(cmdb_inv.to_json())

# 保存到文件（供 Ansible API 使用）
inv_file = inv.to_file('/tmp/dynamic_inventory.json')
print(f"\nInventory 文件: {inv_file}")
```

## 5. 进阶用法

### 5.1 自定义回调插件

```python
#!/usr/bin/env python3
"""
自定义 Ansible 回调插件
功能: 将执行结果发送到 Webhook / 写入数据库 / 发送通知
"""

import json
import time
from datetime import datetime
from ansible.plugins.callback import CallbackBase


class WebhookCallback(CallbackBase):
    """
    将 Ansible 执行结果发送到 Webhook
    可用于: 钉钉/飞书/Slack 通知、写入 ELK、记录到数据库
    """
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'webhook'

    def __init__(self, webhook_url=None, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.webhook_url = webhook_url
        self.start_time = None
        self.results = []

    def _send_webhook(self, data):
        """发送 Webhook 通知"""
        # 实际项目中使用 requests.post
        # import requests
        # requests.post(self.webhook_url, json=data, timeout=5)
        print(f"  [Webhook] {json.dumps(data, ensure_ascii=False)[:200]}")

    def v2_playbook_on_start(self, playbook):
        self.start_time = time.time()
        self._send_webhook({
            'event': 'playbook_start',
            'playbook': playbook._file_name,
            'timestamp': datetime.now().isoformat(),
        })

    def v2_runner_on_ok(self, result, **kwargs):
        host = result._host.get_name()
        task = result._task.get_name()
        changed = result._result.get('changed', False)
        self.results.append({
            'host': host,
            'task': task,
            'status': 'changed' if changed else 'ok',
        })

    def v2_runner_on_failed(self, result, ignore_errors=False):
        host = result._host.get_name()
        task = result._task.get_name()
        msg = result._result.get('msg', '')

        self.results.append({
            'host': host,
            'task': task,
            'status': 'failed',
            'message': msg,
        })

        # 失败时立即通知
        self._send_webhook({
            'event': 'task_failed',
            'host': host,
            'task': task,
            'message': msg,
            'timestamp': datetime.now().isoformat(),
        })

    def v2_playbook_on_stats(self, stats):
        """Playbook 执行完成，发送汇总"""
        elapsed = time.time() - self.start_time if self.start_time else 0

        summary = {}
        for host in sorted(stats.processed.keys()):
            summary[host] = stats.summarize(host)

        total_ok = sum(s['ok'] for s in summary.values())
        total_changed = sum(s['changed'] for s in summary.values())
        total_failed = sum(s['failures'] for s in summary.values())
        total_unreachable = sum(s['unreachable'] for s in summary.values())

        overall_status = 'success' if (total_failed == 0 and total_unreachable == 0) else 'failure'

        self._send_webhook({
            'event': 'playbook_complete',
            'status': overall_status,
            'elapsed_seconds': round(elapsed, 1),
            'summary': {
                'ok': total_ok,
                'changed': total_changed,
                'failed': total_failed,
                'unreachable': total_unreachable,
            },
            'hosts': summary,
            'timestamp': datetime.now().isoformat(),
        })


class DatabaseCallback(CallbackBase):
    """将执行记录写入数据库（SQLite 示例）"""
    CALLBACK_VERSION = 2.0

    def __init__(self, db_path='/tmp/ansible_log.db', *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.db_path = db_path
        self._init_db()

    def _init_db(self):
        """初始化数据库"""
        import sqlite3
        conn = sqlite3.connect(self.db_path)
        conn.execute('''
            CREATE TABLE IF NOT EXISTS ansible_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp TEXT,
                host TEXT,
                task TEXT,
                status TEXT,
                changed BOOLEAN,
                message TEXT,
                result_json TEXT
            )
        ''')
        conn.commit()
        conn.close()

    def _insert_record(self, host, task, status, changed=False,
                       message='', result_data=None):
        """插入记录"""
        import sqlite3
        conn = sqlite3.connect(self.db_path)
        conn.execute(
            'INSERT INTO ansible_log '
            '(timestamp, host, task, status, changed, message, result_json) '
            'VALUES (?, ?, ?, ?, ?, ?, ?)',
            (
                datetime.now().isoformat(),
                host, task, status, changed, message,
                json.dumps(result_data or {}, default=str)
            )
        )
        conn.commit()
        conn.close()

    def v2_runner_on_ok(self, result, **kwargs):
        self._insert_record(
            host=result._host.get_name(),
            task=result._task.get_name(),
            status='ok',
            changed=result._result.get('changed', False),
            result_data=result._result,
        )

    def v2_runner_on_failed(self, result, ignore_errors=False):
        self._insert_record(
            host=result._host.get_name(),
            task=result._task.get_name(),
            status='failed',
            message=result._result.get('msg', ''),
            result_data=result._result,
        )

    def v2_runner_on_unreachable(self, result):
        self._insert_record(
            host=result._host.get_name(),
            task='connection',
            status='unreachable',
            message=result._result.get('msg', ''),
        )
```

### 5.2 自定义 Ansible 模块

```python
#!/usr/bin/env python3
"""
自定义 Ansible 模块 —— 用 Python 编写自己的模块
保存为: library/check_service.py
调用: ansible all -m check_service -a "name=nginx port=80"
"""

# ============================================================
# 这个文件放在 playbook 同目录的 library/ 下
# 或者放在 ANSIBLE_LIBRARY 环境变量指定的目录
# ============================================================

from ansible.module_utils.basic import AnsibleModule
import socket
import subprocess


def check_port(host, port, timeout=3):
    """检查端口是否可达"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(timeout)
    try:
        result = sock.connect_ex((host, port))
        return result == 0
    except Exception:
        return False
    finally:
        sock.close()


def check_process(name):
    """检查进程是否运行"""
    try:
        result = subprocess.run(
            ['pgrep', '-f', name],
            capture_output=True, text=True
        )
        return result.returncode == 0, result.stdout.strip().split('\n')
    except Exception:
        return False, []


def main():
    """模块入口"""
    # 定义模块参数
    module_args = dict(
        name=dict(type='str', required=True),     # 服务名
        port=dict(type='int', required=False),      # 端口号
        host=dict(type='str', default='127.0.0.1'), # 检查的主机
        timeout=dict(type='int', default=3),        # 超时
        state=dict(type='str', default='running',   # 期望状态
                   choices=['running', 'stopped']),
    )

    # 创建模块实例
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True  # 支持 --check 模式
    )

    name = module.params['name']
    port = module.params['port']
    host = module.params['host']
    timeout = module.params['timeout']
    expected_state = module.params['state']

    # 执行检查
    result = dict(
        changed=False,
        service=name,
        checks={},
    )

    # 检查进程
    process_running, pids = check_process(name)
    result['checks']['process'] = {
        'running': process_running,
        'pids': pids if process_running else [],
    }

    # 检查端口
    if port:
        port_open = check_port(host, port, timeout)
        result['checks']['port'] = {
            'port': port,
            'host': host,
            'open': port_open,
        }

    # 判断是否符合预期
    if expected_state == 'running':
        if not process_running:
            module.fail_json(
                msg=f"服务 {name} 未运行",
                **result
            )
        if port and not port_open:
            module.fail_json(
                msg=f"服务 {name} 运行中但端口 {port} 未监听",
                **result
            )
    elif expected_state == 'stopped':
        if process_running:
            module.fail_json(
                msg=f"服务 {name} 应该停止但仍在运行 (PIDs: {pids})",
                **result
            )

    # 成功
    result['msg'] = f"服务 {name} 状态符合预期 ({expected_state})"
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

## 6. SRE 实战案例：用 Python 代码调用 Ansible 执行配置管理

```python
#!/usr/bin/env python3
"""
SRE 实战：Python 驱动的配置管理系统
功能：
  - 动态 Inventory（从配置/API 获取主机）
  - 自动生成并执行 Playbook
  - 自定义回调（结果收集+通知）
  - 多环境支持
  - 执行报告

对比命令行:
  ansible-playbook -i inventory.yml site.yml \
    -e "env=prod app_version=2.0" \
    -t deploy --limit webservers

Python API 的优势:
  - 在代码中动态决定执行什么
  - 结果以对象返回，可做更多处理
  - 集成到 Web 界面或 API 服务
"""

import json
import os
import shutil
import tempfile
import time
import yaml
from datetime import datetime
from pathlib import Path

from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.inventory.manager import InventoryManager
from ansible.module_utils.common.collections import ImmutableDict
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
from ansible.plugins.callback import CallbackBase
from ansible.vars.manager import VariableManager
from ansible import context


class SRECallback(CallbackBase):
    """SRE 专用回调：收集结果 + 实时输出"""
    CALLBACK_VERSION = 2.0

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.results = {
            'ok': [],
            'failed': [],
            'unreachable': [],
            'skipped': [],
            'changed': [],
        }
        self.start_time = time.time()

    def v2_runner_on_ok(self, result, **kwargs):
        host = result._host.get_name()
        task = result._task.get_name()
        changed = result._result.get('changed', False)

        entry = {
            'host': host, 'task': task,
            'changed': changed, 'result': result._result,
        }
        if changed:
            self.results['changed'].append(entry)
        self.results['ok'].append(entry)

        status = "changed" if changed else "ok"
        print(f"  {status}: [{host}] {task}")

    def v2_runner_on_failed(self, result, ignore_errors=False):
        host = result._host.get_name()
        task = result._task.get_name()
        msg = result._result.get('msg', '')

        self.results['failed'].append({
            'host': host, 'task': task, 'message': msg,
        })
        print(f"  FAILED: [{host}] {task} => {msg[:100]}")

    def v2_runner_on_unreachable(self, result):
        host = result._host.get_name()
        self.results['unreachable'].append({
            'host': host, 'message': result._result.get('msg', ''),
        })
        print(f"  UNREACHABLE: [{host}]")

    def v2_runner_on_skipped(self, result):
        host = result._host.get_name()
        task = result._task.get_name()
        self.results['skipped'].append({'host': host, 'task': task})

    def v2_playbook_on_play_start(self, play):
        print(f"\nPLAY [{play.get_name()}] {'*' * 40}")

    def v2_playbook_on_task_start(self, task, is_conditional):
        print(f"\nTASK [{task.get_name()}] {'-' * 30}")

    def get_summary(self):
        elapsed = time.time() - self.start_time
        return {
            'elapsed': round(elapsed, 1),
            'ok': len(self.results['ok']),
            'changed': len(self.results['changed']),
            'failed': len(self.results['failed']),
            'unreachable': len(self.results['unreachable']),
            'skipped': len(self.results['skipped']),
        }


class AnsibleManager:
    """Ansible 配置管理器"""

    def __init__(self, work_dir=None):
        self.work_dir = Path(work_dir or tempfile.mkdtemp(prefix='ansible_'))
        self.work_dir.mkdir(parents=True, exist_ok=True)

    def _setup_context(self, become=False, forks=10, verbosity=0):
        """设置 Ansible 运行上下文"""
        context.CLIARGS = ImmutableDict(
            connection='ssh',
            module_path=[],
            forks=forks,
            become=become,
            become_method='sudo',
            become_user='root',
            check=False,
            diff=False,
            verbosity=verbosity,
            syntax=None,
            start_at_task=None,
            listtags=False,
            listtasks=False,
            listhosts=False,
            tags=[],
            skip_tags=[],
        )

    def run_tasks(self, hosts_config, tasks, become=False, extra_vars=None):
        """
        执行一组任务
        :param hosts_config: 主机配置 {'group': [{'host': 'ip', ...}]}
        :param tasks: 任务列表（Ansible 任务字典格式）
        :param become: 是否使用 sudo
        :param extra_vars: 额外变量
        :return: 执行结果
        """
        self._setup_context(become=become)

        loader = DataLoader()

        # 生成 Inventory 文件
        inv_data = self._generate_inventory(hosts_config)
        inv_file = self.work_dir / 'inventory.json'
        inv_file.write_text(json.dumps(inv_data, indent=2))

        inventory = InventoryManager(loader=loader, sources=[str(inv_file)])
        variable_manager = VariableManager(loader=loader, inventory=inventory)

        if extra_vars:
            variable_manager._extra_vars = extra_vars

        # 构建 Play
        play_source = dict(
            name='SRE Managed Tasks',
            hosts='all',
            gather_facts='no',
            tasks=tasks,
        )
        play = Play().load(play_source, variable_manager=variable_manager,
                          loader=loader)

        # 执行
        callback = SRECallback()
        tqm = None
        try:
            tqm = TaskQueueManager(
                inventory=inventory,
                variable_manager=variable_manager,
                loader=loader,
                passwords={},
                stdout_callback=callback,
            )
            rc = tqm.run(play)
        finally:
            if tqm:
                tqm.cleanup()

        return {
            'rc': rc,
            'summary': callback.get_summary(),
            'results': callback.results,
        }

    def _generate_inventory(self, hosts_config):
        """从配置生成 Ansible Inventory JSON"""
        inventory = {'_meta': {'hostvars': {}}}

        for group, hosts in hosts_config.items():
            inventory[group] = {'hosts': [], 'vars': {}}
            for host_info in hosts:
                hostname = host_info['host']
                inventory[group]['hosts'].append(hostname)

                # 主机变量
                host_vars = {}
                if 'user' in host_info:
                    host_vars['ansible_user'] = host_info['user']
                if 'password' in host_info:
                    host_vars['ansible_password'] = host_info['password']
                    host_vars['ansible_ssh_common_args'] = '-o StrictHostKeyChecking=no'
                if 'port' in host_info:
                    host_vars['ansible_port'] = host_info['port']
                if 'key_file' in host_info:
                    host_vars['ansible_ssh_private_key_file'] = host_info['key_file']

                if host_vars:
                    inventory['_meta']['hostvars'][hostname] = host_vars

        return inventory

    def run_playbook_file(self, playbook_path, inventory_source,
                          extra_vars=None, become=False):
        """
        执行 Playbook 文件
        对比: ansible-playbook -i inventory playbook.yml
        """
        self._setup_context(become=become)

        loader = DataLoader()
        inventory = InventoryManager(loader=loader, sources=[inventory_source])
        variable_manager = VariableManager(loader=loader, inventory=inventory)

        if extra_vars:
            variable_manager._extra_vars = extra_vars

        callback = SRECallback()

        pbex = PlaybookExecutor(
            playbooks=[playbook_path],
            inventory=inventory,
            variable_manager=variable_manager,
            loader=loader,
            passwords={},
        )
        pbex._tqm._stdout_callback = callback

        rc = pbex.run()

        return {
            'rc': rc,
            'summary': callback.get_summary(),
            'results': callback.results,
        }

    def generate_playbook(self, tasks_config, output_path=None):
        """
        动态生成 Playbook YAML 文件
        :param tasks_config: 任务配置
        :param output_path: 输出路径
        """
        if output_path is None:
            output_path = str(self.work_dir / 'generated_playbook.yml')

        playbook = [
            {
                'name': tasks_config.get('name', 'SRE Generated Playbook'),
                'hosts': tasks_config.get('hosts', 'all'),
                'become': tasks_config.get('become', False),
                'gather_facts': tasks_config.get('gather_facts', False),
                'tasks': tasks_config.get('tasks', []),
                'handlers': tasks_config.get('handlers', []),
            }
        ]

        with open(output_path, 'w') as f:
            yaml.dump(playbook, f, default_flow_style=False, allow_unicode=True)

        return output_path

    def cleanup(self):
        """清理临时文件"""
        shutil.rmtree(self.work_dir, ignore_errors=True)


def print_report(result):
    """打印执行报告"""
    summary = result['summary']
    print(f"\n{'=' * 60}")
    print(f"  执行报告")
    print(f"{'=' * 60}")
    print(f"  耗时: {summary['elapsed']} 秒")
    print(f"  成功: {summary['ok']} | 变更: {summary['changed']} | "
          f"失败: {summary['failed']} | 不可达: {summary['unreachable']} | "
          f"跳过: {summary['skipped']}")
    print(f"  总体: {'成功' if result['rc'] == 0 else '失败'}")

    if result['results']['failed']:
        print(f"\n  失败详情:")
        for item in result['results']['failed']:
            print(f"    [{item['host']}] {item['task']}: {item['message'][:100]}")

    if result['results']['unreachable']:
        print(f"\n  不可达主机:")
        for item in result['results']['unreachable']:
            print(f"    {item['host']}: {item['message'][:100]}")

    print(f"{'=' * 60}")


# ============================================================
# 使用示例
# ============================================================

if __name__ == '__main__':
    manager = AnsibleManager(work_dir='/tmp/ansible_sre')

    # 示例 1: 执行简单任务
    print("=== 示例 1: Ad-Hoc 任务 ===")

    hosts_config = {
        'webservers': [
            {'host': '127.0.0.1', 'user': 'root'},
        ],
    }

    tasks = [
        {
            'name': '检查主机名',
            'shell': 'hostname',
            'register': 'hostname_result',
        },
        {
            'name': '显示主机名',
            'debug': {'msg': '主机名: {{ hostname_result.stdout }}'},
        },
        {
            'name': '检查磁盘空间',
            'shell': 'df -h /',
            'register': 'disk_result',
        },
        {
            'name': '检查内存',
            'shell': 'free -m | head -2',
            'register': 'mem_result',
        },
    ]

    result = manager.run_tasks(hosts_config, tasks)
    print_report(result)

    # 示例 2: 动态生成并执行 Playbook
    print("\n=== 示例 2: 动态生成 Playbook ===")

    playbook_config = {
        'name': '系统巡检',
        'hosts': 'all',
        'gather_facts': True,
        'tasks': [
            {
                'name': '显示系统信息',
                'debug': {
                    'msg': [
                        '主机名: {{ ansible_hostname }}',
                        '系统: {{ ansible_distribution }} {{ ansible_distribution_version }}',
                        '内核: {{ ansible_kernel }}',
                        'CPU: {{ ansible_processor_count }} 核',
                        '内存: {{ ansible_memtotal_mb }} MB',
                    ]
                }
            },
            {
                'name': '检查关键服务',
                'shell': 'systemctl is-active {{ item }} || true',
                'loop': ['sshd', 'crond', 'rsyslog'],
                'register': 'service_status',
            },
        ],
    }

    playbook_path = manager.generate_playbook(playbook_config)
    print(f"Playbook 已生成: {playbook_path}")

    # 读取并展示生成的 Playbook
    with open(playbook_path) as f:
        print(f.read())

    # 清理
    # manager.cleanup()
    print("\n配置管理器已就绪")
```

## 7. 常见坑点与最佳实践

### Do's (推荐做法)

```python
# 1. 始终清理 TaskQueueManager
tqm = None
try:
    tqm = TaskQueueManager(...)
    tqm.run(play)
finally:
    if tqm:
        tqm.cleanup()

# 2. 使用自定义回调收集结构化结果
# 不要依赖终端输出解析

# 3. Inventory 使用临时文件，用完清理
import tempfile
fd, inv_path = tempfile.mkstemp(suffix='.json')
try:
    with os.fdopen(fd, 'w') as f:
        json.dump(inventory_data, f)
    # 使用 inv_path
finally:
    os.unlink(inv_path)

# 4. 敏感信息使用环境变量或 Ansible Vault
# 好：从环境变量获取
password = os.environ.get('ANSIBLE_SSH_PASS')

# 5. 版本兼容：检查 API 是否可用
try:
    from ansible.executor.task_queue_manager import TaskQueueManager
except ImportError:
    print("请安装 ansible: pip install ansible")

# 6. 使用 check 模式测试
context.CLIARGS = ImmutableDict(
    check=True,  # dry-run 模式，不会实际执行
    # ...
)
```

### Don'ts (避免做法)

```python
# 1. 不要在多线程中共享 Ansible 对象
# Ansible 的很多组件不是线程安全的
# 坏：多线程共享 TaskQueueManager

# 2. 不要忽略 context.CLIARGS 的设置
# Ansible API 依赖全局 context，必须设置

# 3. 不要硬编码 Inventory
# 坏：
inventory = InventoryManager(loader=loader, sources=['192.168.1.1,'])
# 好：使用动态 Inventory 或配置文件

# 4. 不要在循环中反复初始化 DataLoader
# DataLoader 可以复用
loader = DataLoader()  # 创建一次
for playbook in playbooks:
    # 复用 loader
    pass

# 5. 不要忽略返回码
rc = tqm.run(play)
if rc != 0:
    print("执行失败!")
    # 处理失败情况
```

## 8. 速查表

```
# ==================== Ansible Python API 速查表 ====================

# --- 安装 ---
pip install ansible
pip install ansible-core==2.15.0    # 指定版本

# --- 核心导入 ---
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.inventory.manager import InventoryManager
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
from ansible.vars.manager import VariableManager
from ansible.plugins.callback import CallbackBase
from ansible.module_utils.common.collections import ImmutableDict
from ansible import context

# --- 初始化流程 ---
# 1. 设置全局选项
context.CLIARGS = ImmutableDict(
    connection='ssh', forks=10, become=False, ...)

# 2. 创建核心组件
loader = DataLoader()
inventory = InventoryManager(loader=loader, sources=['inventory.yml'])
variable_manager = VariableManager(loader=loader, inventory=inventory)

# 3a. Ad-Hoc 任务
play_source = dict(name='...', hosts='all', tasks=[...])
play = Play().load(play_source, variable_manager=variable_manager,
                   loader=loader)
tqm = TaskQueueManager(inventory=inventory,
    variable_manager=variable_manager, loader=loader,
    passwords={}, stdout_callback=my_callback)
rc = tqm.run(play)

# 3b. 执行 Playbook
pbex = PlaybookExecutor(playbooks=['site.yml'], inventory=inventory,
    variable_manager=variable_manager, loader=loader, passwords={})
rc = pbex.run()

# --- 回调类 ---
class MyCallback(CallbackBase):
    def v2_runner_on_ok(self, result):       # 成功
    def v2_runner_on_failed(self, result):   # 失败
    def v2_runner_on_unreachable(self, result):  # 不可达
    def v2_runner_on_skipped(self, result):  # 跳过
    def v2_playbook_on_stats(self, stats):   # 统计

# --- 对比命令行 ---
# ansible all -m ping
#   → run_adhoc(hosts='all', module_name='ping')

# ansible all -m shell -a "uptime"
#   → run_adhoc(hosts='all', module_name='shell', module_args='uptime')

# ansible-playbook -i hosts.yml site.yml -e "version=2.0"
#   → run_playbook('site.yml', 'hosts.yml', extra_vars={'version':'2.0'})

# ansible-playbook site.yml --check
#   → context.CLIARGS = ImmutableDict(check=True, ...)

# ansible-playbook site.yml -t deploy
#   → context.CLIARGS = ImmutableDict(tags=['deploy'], ...)

# ansible-playbook site.yml -l webservers
#   → inventory.subset('webservers')
```
