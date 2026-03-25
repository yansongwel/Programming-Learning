# MySQL 与 SQLAlchemy — Python SRE 的数据库操作全指南

> 一句话说明：从 `pymysql` 原生 SQL 到 `SQLAlchemy` ORM，掌握 Python 操作 MySQL 的各种姿势，实战数据库巡检脚本（慢查询/连接数/表空间监控）。

---

## 1. 是什么 & 为什么用它

### 对比 Shell 操作 MySQL

```bash
# Shell：用 mysql 命令行执行查询
mysql -h 127.0.0.1 -u root -p'password' -e "SELECT COUNT(*) FROM users;"

# Shell：查看慢查询
mysql -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';"

# Shell：导出查询结果
mysql -N -B -e "SELECT host,user FROM mysql.user" > /tmp/users.txt

# Shell：检查连接数
mysql -e "SHOW PROCESSLIST;" | wc -l

# 缺点：
# 1. 密码明文暴露在命令行/脚本中
# 2. 结果解析困难（纯文本）
# 3. 没有连接池，每次都新建连接
# 4. 无法参数化查询（SQL 注入风险）
# 5. 复杂逻辑难以实现
```

### Python 方案对比

| 特性 | Shell + mysql | pymysql | SQLAlchemy |
|------|-------------|---------|------------|
| 连接池 | 无 | 需借助 DBUtils | 内置 |
| 参数化查询 | 无 | 支持 `%s` | 完全支持 |
| SQL 注入防护 | 无 | 参数化可防 | ORM 天然防护 |
| 结果格式 | 文本 | 元组/字典 | Python 对象 |
| 事务管理 | 手动 | 手动 | 自动/手动 |
| 代码可维护性 | 差 | 中 | 好 |

---

## 2. 安装与环境

```bash
# 安装 MySQL 客户端库
pip install pymysql          # 纯 Python MySQL 驱动
pip install cryptography     # pymysql SSL 支持

# 安装连接池
pip install DBUtils          # 通用数据库连接池

# 安装 SQLAlchemy ORM
pip install sqlalchemy       # ORM 框架
pip install alembic          # 数据库迁移工具

# 验证安装
python3 -c "import pymysql; print(f'pymysql: {pymysql.__version__}')"
python3 -c "import sqlalchemy; print(f'SQLAlchemy: {sqlalchemy.__version__}')"

# 如果本地没有 MySQL，用 Docker 快速启动
# docker run -d --name mysql-test -p 3306:3306 \
#   -e MYSQL_ROOT_PASSWORD=root123 \
#   -e MYSQL_DATABASE=sre_db \
#   mysql:8.0
```

---

## 3. 核心概念与原理

### 数据库连接生命周期

```
应用程序
    |
    v
连接池（Pool）
    +-- 空闲连接队列
    +-- 活动连接计数
    +-- 最大/最小连接数
    |
    v
MySQL 协议通信
    +-- 认证握手
    +-- 查询请求/响应
    +-- 预编译语句
    |
    v
MySQL Server
```

### SQLAlchemy 架构层次

```
应用代码
    |
    v
ORM 层（Session、Model、Query）
    |                    ← 你可以只用这一层
    v
Core 层（Table、Column、select()、insert()）
    |                    ← 或直接用 SQL 表达式
    v
Engine 层（连接池、方言 Dialect）
    |
    v
DBAPI 层（pymysql、mysqlclient 等驱动）
    |
    v
数据库服务器
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 pymysql 直连

```python
#!/usr/bin/env python3
"""pymysql 基础操作"""

import pymysql

# ========== 连接数据库 ==========
# 等价于: mysql -h 127.0.0.1 -u root -p'root123' sre_db
connection = pymysql.connect(
    host="127.0.0.1",
    port=3306,
    user="root",
    password="root123",
    database="sre_db",
    charset="utf8mb4",
    cursorclass=pymysql.cursors.DictCursor,  # 返回字典格式
    connect_timeout=5,
    read_timeout=10,
    autocommit=True  # 自动提交（查询场景推荐）
)

try:
    with connection.cursor() as cursor:
        # === 创建表 ===
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS servers (
                id INT AUTO_INCREMENT PRIMARY KEY,
                hostname VARCHAR(100) NOT NULL,
                ip VARCHAR(15) NOT NULL,
                status ENUM('online', 'offline', 'maintenance') DEFAULT 'online',
                cpu_cores INT DEFAULT 4,
                memory_gb INT DEFAULT 8,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        """)
        print("表创建成功")

        # === INSERT 插入 ===
        # 等价于: mysql -e "INSERT INTO servers ..."
        # 使用参数化查询防止 SQL 注入（%s 占位符）
        sql = "INSERT INTO servers (hostname, ip, cpu_cores, memory_gb) VALUES (%s, %s, %s, %s)"
        cursor.execute(sql, ("web-01", "10.0.1.1", 8, 16))
        print(f"插入成功，ID: {cursor.lastrowid}")

        # 批量插入
        servers = [
            ("web-02", "10.0.1.2", 8, 16),
            ("db-01", "10.0.2.1", 16, 64),
            ("cache-01", "10.0.3.1", 4, 32),
        ]
        cursor.executemany(sql, servers)
        print(f"批量插入 {cursor.rowcount} 条")

        # === SELECT 查询 ===
        # 等价于: mysql -e "SELECT * FROM servers"
        cursor.execute("SELECT * FROM servers")
        rows = cursor.fetchall()  # 获取所有结果
        for row in rows:
            print(f"  {row['hostname']} ({row['ip']}) - {row['status']}")

        # 条件查询（参数化）
        cursor.execute(
            "SELECT * FROM servers WHERE cpu_cores >= %s AND status = %s",
            (8, "online")
        )
        print(f"\nCPU >= 8 核的在线服务器: {cursor.rowcount} 台")

        # === UPDATE 更新 ===
        cursor.execute(
            "UPDATE servers SET status = %s WHERE hostname = %s",
            ("maintenance", "web-01")
        )
        print(f"更新 {cursor.rowcount} 条记录")

        # === DELETE 删除 ===
        cursor.execute(
            "DELETE FROM servers WHERE hostname = %s",
            ("cache-01",)
        )
        print(f"删除 {cursor.rowcount} 条记录")

finally:
    connection.close()
    print("连接已关闭")
```

### 4.2 pymysql 事务管理

```python
#!/usr/bin/env python3
"""pymysql 事务操作"""

import pymysql

connection = pymysql.connect(
    host="127.0.0.1",
    user="root",
    password="root123",
    database="sre_db",
    charset="utf8mb4",
    cursorclass=pymysql.cursors.DictCursor,
    autocommit=False  # 关闭自动提交，手动管理事务
)

try:
    with connection.cursor() as cursor:
        # 开始事务（pymysql 自动开始）
        # 操作 1: 更新服务器状态
        cursor.execute(
            "UPDATE servers SET status = 'maintenance' WHERE hostname = %s",
            ("web-01",)
        )

        # 操作 2: 记录变更日志
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS change_log (
                id INT AUTO_INCREMENT PRIMARY KEY,
                target VARCHAR(100),
                action VARCHAR(200),
                operator VARCHAR(50),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        cursor.execute(
            "INSERT INTO change_log (target, action, operator) VALUES (%s, %s, %s)",
            ("web-01", "设置为维护模式", "sre-bot")
        )

        # 提交事务（两个操作要么都成功，要么都失败）
        connection.commit()
        print("事务提交成功")

except Exception as e:
    # 出错时回滚
    connection.rollback()
    print(f"事务回滚: {e}")

finally:
    connection.close()
```

### 4.3 连接池（DBUtils）

```python
#!/usr/bin/env python3
"""使用 DBUtils 实现连接池"""

import pymysql
from dbutils.pooled_db import PooledDB

# 创建连接池
pool = PooledDB(
    creator=pymysql,           # 使用 pymysql 作为数据库驱动
    maxconnections=20,         # 连接池最大连接数
    mincached=5,               # 初始化时最少空闲连接数
    maxcached=10,              # 最多空闲连接数
    maxshared=0,               # 不共享连接（0 = 所有连接独占）
    blocking=True,             # 连接池满时是否阻塞等待
    maxusage=None,             # 单个连接最大使用次数（None=无限）
    setsession=[],             # 每次连接初始化执行的 SQL
    ping=1,                    # 每次获取连接时 ping 检测
    host="127.0.0.1",
    port=3306,
    user="root",
    password="root123",
    database="sre_db",
    charset="utf8mb4",
    cursorclass=pymysql.cursors.DictCursor,
)

def query_with_pool(sql, params=None):
    """从连接池获取连接执行查询"""
    conn = pool.connection()  # 从池中获取连接
    try:
        with conn.cursor() as cursor:
            cursor.execute(sql, params)
            return cursor.fetchall()
    finally:
        conn.close()  # 归还连接到池（不是真的关闭）

# 使用示例
results = query_with_pool("SELECT * FROM servers WHERE status = %s", ("online",))
for row in results:
    print(f"{row['hostname']}: {row['ip']}")

# 连接池状态
print(f"连接池中空闲连接数: {pool._cache.qsize() if hasattr(pool, '_cache') else 'N/A'}")
```

---

## 5. 进阶用法：SQLAlchemy

### 5.1 SQLAlchemy Core（SQL 表达式）

```python
#!/usr/bin/env python3
"""SQLAlchemy Core 层使用"""

from sqlalchemy import create_engine, text, MetaData, Table, Column
from sqlalchemy import Integer, String, DateTime, Enum
from sqlalchemy import select, insert, update, delete, func
from datetime import datetime

# 创建引擎（内置连接池）
engine = create_engine(
    "mysql+pymysql://root:root123@127.0.0.1:3306/sre_db",
    pool_size=10,           # 连接池大小
    max_overflow=20,        # 超出 pool_size 后最多额外创建的连接
    pool_timeout=30,        # 获取连接的超时时间
    pool_recycle=3600,      # 连接回收时间（秒）— 防止 MySQL 8h 断连
    pool_pre_ping=True,     # 每次使用前 ping 检测连接是否有效
    echo=False              # True 则打印所有 SQL（调试用）
)

# 使用 text() 执行原生 SQL
with engine.connect() as conn:
    # 原生查询
    result = conn.execute(text("SELECT * FROM servers LIMIT 5"))
    for row in result:
        print(dict(row._mapping))

    # 参数化查询
    result = conn.execute(
        text("SELECT * FROM servers WHERE status = :status"),
        {"status": "online"}
    )
    print(f"在线服务器: {result.rowcount} 台")
```

### 5.2 SQLAlchemy ORM

```python
#!/usr/bin/env python3
"""SQLAlchemy ORM 完整示例"""

from sqlalchemy import create_engine, Column, Integer, String, DateTime, Enum, Float
from sqlalchemy.orm import declarative_base, sessionmaker, Session
from datetime import datetime

# 创建基类
Base = declarative_base()

# ========== 定义 ORM 模型 ==========

class Server(Base):
    """服务器信息表"""
    __tablename__ = "servers_orm"

    id = Column(Integer, primary_key=True, autoincrement=True)
    hostname = Column(String(100), nullable=False, unique=True, comment="主机名")
    ip = Column(String(15), nullable=False, comment="IP 地址")
    status = Column(
        Enum("online", "offline", "maintenance"),
        default="online",
        comment="状态"
    )
    cpu_cores = Column(Integer, default=4, comment="CPU 核数")
    memory_gb = Column(Integer, default=8, comment="内存 GB")
    region = Column(String(50), default="cn-east", comment="区域")
    created_at = Column(DateTime, default=datetime.now, comment="创建时间")
    updated_at = Column(DateTime, default=datetime.now, onupdate=datetime.now)

    def __repr__(self):
        return f"<Server {self.hostname} ({self.ip}) [{self.status}]>"

    def to_dict(self):
        return {
            "id": self.id,
            "hostname": self.hostname,
            "ip": self.ip,
            "status": self.status,
            "cpu_cores": self.cpu_cores,
            "memory_gb": self.memory_gb,
        }

class Alert(Base):
    """告警记录表"""
    __tablename__ = "alerts"

    id = Column(Integer, primary_key=True, autoincrement=True)
    server_id = Column(Integer, nullable=False, comment="服务器 ID")
    level = Column(Enum("info", "warning", "critical"), comment="级别")
    message = Column(String(500), comment="告警信息")
    resolved = Column(Integer, default=0, comment="是否已解决")
    created_at = Column(DateTime, default=datetime.now)

# ========== 初始化 ==========

engine = create_engine(
    "mysql+pymysql://root:root123@127.0.0.1:3306/sre_db",
    pool_recycle=3600,
    pool_pre_ping=True,
)

# 创建所有表（如果不存在）
Base.metadata.create_all(engine)

# 创建 Session 工厂
SessionLocal = sessionmaker(bind=engine)

# ========== CRUD 操作 ==========

def add_servers():
    """新增服务器"""
    session = SessionLocal()
    try:
        servers = [
            Server(hostname="web-01", ip="10.0.1.1", cpu_cores=8, memory_gb=16),
            Server(hostname="web-02", ip="10.0.1.2", cpu_cores=8, memory_gb=16),
            Server(hostname="db-01", ip="10.0.2.1", cpu_cores=16, memory_gb=64,
                   region="cn-north"),
        ]
        session.add_all(servers)
        session.commit()
        print(f"添加 {len(servers)} 台服务器")
    except Exception as e:
        session.rollback()
        print(f"添加失败: {e}")
    finally:
        session.close()

def query_servers():
    """查询服务器"""
    session = SessionLocal()
    try:
        # 查询所有
        all_servers = session.query(Server).all()
        print(f"总服务器数: {len(all_servers)}")

        # 条件查询
        online = session.query(Server).filter(
            Server.status == "online",
            Server.cpu_cores >= 8
        ).all()
        print(f"在线 & CPU>=8: {len(online)} 台")
        for s in online:
            print(f"  {s}")

        # 排序 + 限制
        top3 = session.query(Server).order_by(
            Server.memory_gb.desc()
        ).limit(3).all()
        print(f"内存 Top 3: {[s.hostname for s in top3]}")

        # 聚合查询
        from sqlalchemy import func
        stats = session.query(
            func.count(Server.id).label("total"),
            func.avg(Server.cpu_cores).label("avg_cpu"),
            func.sum(Server.memory_gb).label("total_mem")
        ).first()
        print(f"统计: 总数={stats.total}, 平均CPU={stats.avg_cpu:.1f}, 总内存={stats.total_mem}GB")

    finally:
        session.close()

def update_server():
    """更新服务器"""
    session = SessionLocal()
    try:
        server = session.query(Server).filter(Server.hostname == "web-01").first()
        if server:
            server.status = "maintenance"
            server.cpu_cores = 16
            session.commit()
            print(f"更新成功: {server}")
    except Exception as e:
        session.rollback()
        print(f"更新失败: {e}")
    finally:
        session.close()

# 运行
if __name__ == "__main__":
    add_servers()
    query_servers()
    update_server()
```

### 5.3 Session 管理最佳实践

```python
#!/usr/bin/env python3
"""Session 管理最佳实践"""

from contextlib import contextmanager
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("mysql+pymysql://root:root123@127.0.0.1:3306/sre_db")
SessionLocal = sessionmaker(bind=engine)

@contextmanager
def get_session():
    """
    安全的 Session 上下文管理器
    自动处理 commit/rollback/close
    """
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# 使用方式
# with get_session() as session:
#     servers = session.query(Server).all()
#     session.add(Server(hostname="new-01", ip="10.0.1.100"))
#     # 退出 with 块时自动 commit
```

### 5.4 Alembic 数据库迁移简介

```bash
# 初始化 Alembic
alembic init alembic

# 配置 alembic.ini 中的数据库连接
# sqlalchemy.url = mysql+pymysql://root:root123@127.0.0.1:3306/sre_db

# 生成迁移脚本
alembic revision --autogenerate -m "add servers table"

# 执行迁移
alembic upgrade head

# 查看历史
alembic history

# 回滚
alembic downgrade -1
```

```python
# alembic/env.py 中配置 target_metadata
# from models import Base
# target_metadata = Base.metadata
```

---

## 6. SRE 实战案例：数据库巡检脚本

```python
#!/usr/bin/env python3
"""
SRE 实战：MySQL 数据库巡检脚本
功能：
  1. 连接数监控
  2. 慢查询统计
  3. 表空间使用情况
  4. 复制状态检查
  5. 锁等待检测
  6. 关键参数检查
  7. 生成巡检报告
"""

import pymysql
from datetime import datetime
from typing import Dict, List, Any
import json


class MySQLInspector:
    """MySQL 数据库巡检器"""

    def __init__(self, host: str, port: int, user: str, password: str):
        """
        初始化巡检器

        参数:
            host: 数据库主机
            port: 端口
            user: 用户名
            password: 密码
        """
        self.conn_params = {
            "host": host,
            "port": port,
            "user": user,
            "password": password,
            "charset": "utf8mb4",
            "cursorclass": pymysql.cursors.DictCursor,
            "connect_timeout": 5,
            "read_timeout": 30,
        }
        self.results = {}

    def _execute(self, sql: str, params=None) -> List[Dict]:
        """执行 SQL 查询"""
        conn = pymysql.connect(**self.conn_params)
        try:
            with conn.cursor() as cursor:
                cursor.execute(sql, params)
                return cursor.fetchall()
        finally:
            conn.close()

    def _execute_one(self, sql: str) -> Dict:
        """执行查询并返回第一行"""
        rows = self._execute(sql)
        return rows[0] if rows else {}

    # ========== 巡检项 ==========

    def check_connections(self) -> Dict:
        """
        检查连接数
        等价于: mysql -e "SHOW GLOBAL STATUS LIKE 'Threads%';
                         SHOW GLOBAL VARIABLES LIKE 'max_connections';"
        """
        # 当前连接数
        threads = {}
        for row in self._execute("SHOW GLOBAL STATUS LIKE 'Threads%'"):
            threads[row["Variable_name"]] = int(row["Value"])

        # 最大连接数
        max_conn_row = self._execute_one(
            "SHOW GLOBAL VARIABLES LIKE 'max_connections'"
        )
        max_connections = int(max_conn_row.get("Value", 0))

        # 连接使用率
        current = threads.get("Threads_connected", 0)
        usage_pct = round(current / max_connections * 100, 1) if max_connections else 0

        result = {
            "max_connections": max_connections,
            "current_connections": current,
            "running_threads": threads.get("Threads_running", 0),
            "cached_threads": threads.get("Threads_cached", 0),
            "usage_percent": usage_pct,
            "status": "WARNING" if usage_pct > 80 else "OK"
        }

        self.results["connections"] = result
        return result

    def check_slow_queries(self) -> Dict:
        """
        检查慢查询
        等价于: mysql -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';
                         SHOW GLOBAL VARIABLES LIKE 'long_query_time';"
        """
        # 慢查询总数
        slow = self._execute_one("SHOW GLOBAL STATUS LIKE 'Slow_queries'")
        slow_count = int(slow.get("Value", 0))

        # 慢查询阈值
        threshold = self._execute_one(
            "SHOW GLOBAL VARIABLES LIKE 'long_query_time'"
        )
        threshold_sec = float(threshold.get("Value", 10))

        # 是否开启慢查询日志
        log_status = self._execute_one(
            "SHOW GLOBAL VARIABLES LIKE 'slow_query_log'"
        )
        log_enabled = log_status.get("Value", "OFF") == "ON"

        # 查询总次数（计算慢查询比例）
        questions = self._execute_one("SHOW GLOBAL STATUS LIKE 'Questions'")
        total_queries = int(questions.get("Value", 1))
        slow_pct = round(slow_count / total_queries * 100, 4)

        result = {
            "slow_query_count": slow_count,
            "total_queries": total_queries,
            "slow_percent": slow_pct,
            "threshold_seconds": threshold_sec,
            "log_enabled": log_enabled,
            "status": "WARNING" if slow_pct > 0.1 else "OK"
        }

        self.results["slow_queries"] = result
        return result

    def check_table_sizes(self, database: str = None) -> List[Dict]:
        """
        检查表空间使用
        等价于: mysql -e "SELECT table_name, data_length, index_length
                         FROM information_schema.tables ..."
        """
        sql = """
            SELECT
                table_schema AS db_name,
                table_name,
                ROUND(data_length / 1024 / 1024, 2) AS data_mb,
                ROUND(index_length / 1024 / 1024, 2) AS index_mb,
                ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb,
                table_rows AS row_count,
                engine,
                ROUND(data_free / 1024 / 1024, 2) AS fragmented_mb
            FROM information_schema.tables
            WHERE table_schema NOT IN ('information_schema', 'performance_schema',
                                        'mysql', 'sys')
        """
        if database:
            sql += f" AND table_schema = %s"
            tables = self._execute(sql + " ORDER BY total_mb DESC LIMIT 20",
                                   (database,))
        else:
            tables = self._execute(sql + " ORDER BY total_mb DESC LIMIT 20")

        # 统计
        total_size = sum(t.get("total_mb", 0) or 0 for t in tables)

        result = {
            "top_tables": tables,
            "total_size_mb": round(total_size, 2),
            "table_count": len(tables)
        }

        self.results["table_sizes"] = result
        return tables

    def check_replication(self) -> Dict:
        """
        检查复制状态（Slave）
        等价于: mysql -e "SHOW SLAVE STATUS\G"
        """
        try:
            rows = self._execute("SHOW SLAVE STATUS")
            if not rows:
                result = {"role": "master_or_standalone", "status": "N/A"}
                self.results["replication"] = result
                return result

            slave = rows[0]
            io_running = slave.get("Slave_IO_Running", "No")
            sql_running = slave.get("Slave_SQL_Running", "No")
            seconds_behind = slave.get("Seconds_Behind_Master")

            is_healthy = io_running == "Yes" and sql_running == "Yes"

            result = {
                "role": "slave",
                "io_thread": io_running,
                "sql_thread": sql_running,
                "seconds_behind_master": seconds_behind,
                "master_host": slave.get("Master_Host"),
                "last_error": slave.get("Last_Error", ""),
                "status": "OK" if is_healthy else "CRITICAL"
            }

            self.results["replication"] = result
            return result

        except Exception as e:
            result = {"error": str(e), "status": "ERROR"}
            self.results["replication"] = result
            return result

    def check_locks(self) -> Dict:
        """
        检查锁等待
        等价于: mysql -e "SELECT * FROM information_schema.innodb_lock_waits"
        """
        try:
            # MySQL 8.0+ 使用 performance_schema
            locks = self._execute("""
                SELECT
                    r.trx_id AS waiting_trx_id,
                    r.trx_mysql_thread_id AS waiting_thread,
                    r.trx_query AS waiting_query,
                    b.trx_id AS blocking_trx_id,
                    b.trx_mysql_thread_id AS blocking_thread,
                    b.trx_query AS blocking_query
                FROM information_schema.innodb_lock_waits w
                JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
                JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
                LIMIT 10
            """)
        except Exception:
            # 兼容不同版本
            locks = []

        result = {
            "lock_waits": len(locks),
            "details": locks[:5],
            "status": "WARNING" if locks else "OK"
        }

        self.results["locks"] = result
        return result

    def check_key_variables(self) -> Dict:
        """
        检查关键参数
        等价于: mysql -e "SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_size';"
        """
        key_vars = [
            "innodb_buffer_pool_size",
            "innodb_log_file_size",
            "max_connections",
            "query_cache_size",
            "tmp_table_size",
            "max_heap_table_size",
            "innodb_flush_log_at_trx_commit",
            "sync_binlog",
            "binlog_format",
            "server_id",
            "read_only",
        ]

        variables = {}
        for var in key_vars:
            try:
                row = self._execute_one(
                    f"SHOW GLOBAL VARIABLES LIKE '{var}'"
                )
                variables[var] = row.get("Value", "N/A")
            except Exception:
                variables[var] = "获取失败"

        # 获取 Buffer Pool 命中率
        try:
            reads = self._execute_one(
                "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests'"
            )
            disk_reads = self._execute_one(
                "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads'"
            )
            total_reads = int(reads.get("Value", 1))
            disk = int(disk_reads.get("Value", 0))
            hit_rate = round((1 - disk / total_reads) * 100, 2) if total_reads else 0
            variables["buffer_pool_hit_rate"] = f"{hit_rate}%"
        except Exception:
            variables["buffer_pool_hit_rate"] = "N/A"

        self.results["variables"] = variables
        return variables

    def check_processlist(self) -> Dict:
        """
        检查当前进程列表
        等价于: mysql -e "SHOW FULL PROCESSLIST"
        """
        processes = self._execute("SHOW FULL PROCESSLIST")

        # 统计各状态的连接数
        state_count = {}
        long_queries = []

        for p in processes:
            state = p.get("State", "unknown") or "idle"
            state_count[state] = state_count.get(state, 0) + 1

            # 找出执行时间超过 10 秒的查询
            exec_time = p.get("Time", 0) or 0
            if exec_time > 10 and p.get("Command") != "Sleep":
                long_queries.append({
                    "id": p["Id"],
                    "user": p.get("User"),
                    "host": p.get("Host"),
                    "time": exec_time,
                    "state": state,
                    "query": (p.get("Info") or "")[:200]
                })

        result = {
            "total_processes": len(processes),
            "state_distribution": state_count,
            "long_running_queries": long_queries,
            "status": "WARNING" if long_queries else "OK"
        }

        self.results["processlist"] = result
        return result

    # ========== 生成报告 ==========

    def run_full_inspection(self, database: str = None) -> str:
        """执行完整巡检并生成报告"""
        print("开始数据库巡检...\n")

        # 执行所有检查
        checks = [
            ("连接数检查", self.check_connections),
            ("慢查询检查", self.check_slow_queries),
            ("表空间检查", lambda: self.check_table_sizes(database)),
            ("复制状态检查", self.check_replication),
            ("锁等待检查", self.check_locks),
            ("关键参数检查", self.check_key_variables),
            ("进程列表检查", self.check_processlist),
        ]

        report = []
        report.append("=" * 65)
        report.append("        MySQL 数据库巡检报告")
        report.append("=" * 65)
        report.append(f"巡检时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append(f"目标主机: {self.conn_params['host']}:{self.conn_params['port']}")
        report.append("")

        for name, check_func in checks:
            try:
                result = check_func()
                status = result.get("status", "OK") if isinstance(result, dict) else "OK"
                icon = "[OK]" if status == "OK" else "[!!]"
                report.append(f"{icon} {name}")

                # 根据检查项输出关键信息
                if isinstance(result, dict):
                    for k, v in result.items():
                        if k not in ("status", "details", "top_tables",
                                     "state_distribution", "long_running_queries"):
                            report.append(f"    {k}: {v}")
                report.append("")

            except Exception as e:
                report.append(f"[ERR] {name}: {str(e)}")
                report.append("")

        report.append("=" * 65)

        report_text = "\n".join(report)
        print(report_text)

        # 导出 JSON
        output = {
            "inspection_time": datetime.now().isoformat(),
            "host": f"{self.conn_params['host']}:{self.conn_params['port']}",
            "results": self.results
        }
        output_path = "/tmp/mysql_inspection.json"
        with open(output_path, "w", encoding="utf-8") as f:
            json.dump(output, f, ensure_ascii=False, indent=2, default=str)
        print(f"\n详细报告已导出: {output_path}")

        return report_text


# ========== 主函数 ==========

if __name__ == "__main__":
    # 配置数据库连接（实际使用时从配置文件或环境变量读取）
    inspector = MySQLInspector(
        host="127.0.0.1",
        port=3306,
        user="root",
        password="root123"
    )

    try:
        inspector.run_full_inspection(database="sre_db")
    except pymysql.OperationalError as e:
        print(f"连接数据库失败: {e}")
        print("请确认 MySQL 服务是否运行，连接信息是否正确")
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：SQL 注入

```python
# 错误：字符串拼接（SQL 注入漏洞！）
username = "admin' OR '1'='1"
cursor.execute(f"SELECT * FROM users WHERE name = '{username}'")

# 正确：参数化查询
cursor.execute("SELECT * FROM users WHERE name = %s", (username,))
```

### 坑点 2：MySQL 8 小时断连

```python
# 错误：长期持有连接不处理断连
conn = pymysql.connect(...)  # 8 小时后自动断开

# 正确：使用连接池 + pool_recycle
engine = create_engine(url, pool_recycle=3600, pool_pre_ping=True)
```

### 坑点 3：忘记关闭连接/Session

```python
# 错误：
session = SessionLocal()
data = session.query(Model).all()
# 忘记 close()，连接泄漏！

# 正确：用 with 或 try/finally
with get_session() as session:
    data = session.query(Model).all()
```

### 坑点 4：大量数据一次性加载

```python
# 错误：一次加载百万行
all_rows = session.query(BigTable).all()  # 内存爆炸

# 正确：分页查询
page_size = 1000
offset = 0
while True:
    batch = session.query(BigTable).offset(offset).limit(page_size).all()
    if not batch:
        break
    process(batch)
    offset += page_size
```

---

## 8. 速查表

### pymysql 速查

```python
import pymysql

# 连接
conn = pymysql.connect(host="127.0.0.1", user="root", password="pass", database="db")

# 查询
with conn.cursor(pymysql.cursors.DictCursor) as cur:
    cur.execute("SELECT * FROM t WHERE id = %s", (1,))
    rows = cur.fetchall()       # 所有行
    row = cur.fetchone()        # 一行
    cur.fetchmany(10)           # N 行

# 写入
with conn.cursor() as cur:
    cur.execute("INSERT INTO t (name) VALUES (%s)", ("val",))
    cur.executemany("INSERT INTO t (name) VALUES (%s)", [("a",), ("b",)])
    conn.commit()

# 事务
conn.begin()
try:
    cur.execute(...)
    conn.commit()
except:
    conn.rollback()
finally:
    conn.close()
```

### SQLAlchemy ORM 速查

```python
from sqlalchemy.orm import Session

# 查询
session.query(Model).all()                          # 全部
session.query(Model).filter(Model.id == 1).first()  # 条件
session.query(Model).filter_by(name="x").all()      # 关键字条件
session.query(Model).order_by(Model.id.desc())      # 排序
session.query(Model).limit(10).offset(20)           # 分页
session.query(func.count(Model.id))                 # 聚合

# 写入
session.add(obj)              # 添加单个
session.add_all([o1, o2])     # 批量添加
session.commit()              # 提交
session.rollback()            # 回滚

# 更新
obj.name = "new"
session.commit()

# 删除
session.delete(obj)
session.commit()
```

### 常用 MySQL 巡检 SQL

```sql
-- 连接数
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL VARIABLES LIKE 'max_connections';

-- 慢查询
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL VARIABLES LIKE 'long_query_time';

-- 表空间
SELECT table_schema, table_name,
       ROUND((data_length+index_length)/1024/1024, 2) AS size_mb
FROM information_schema.tables ORDER BY size_mb DESC LIMIT 20;

-- Buffer Pool 命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- 当前运行查询
SHOW FULL PROCESSLIST;

-- 复制状态
SHOW SLAVE STATUS\G
```
