# Elasticsearch 日志查询 — Python SRE 的日志搜索与分析利器

> 一句话说明：用 `elasticsearch-py` 操作 Elasticsearch，掌握索引管理、DSL 查询、聚合分析、scroll 大量数据查询，实战日志搜索与错误统计分析工具。

---

## 1. 是什么 & 为什么用它

### 对比 Shell/curl 操作 ES

```bash
# Shell：用 curl 查询 ES
curl -s "http://localhost:9200/logs-*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"match": {"message": "error"}},
    "size": 10
  }' | jq '.hits.hits[]._source'

# Shell：查看索引列表
curl -s "http://localhost:9200/_cat/indices?v"

# Shell：查看集群健康
curl -s "http://localhost:9200/_cluster/health" | jq .

# 缺点：
# 1. DSL 查询 JSON 手写困难且易出错
# 2. 分页/scroll 逻辑需要手动处理
# 3. 聚合结果解析复杂
# 4. 批量操作（bulk）需要构造 NDJSON 格式
# 5. 错误处理不方便
```

### Python elasticsearch-py 优势

| 特性 | curl/Shell | elasticsearch-py |
|------|-----------|-----------------|
| DSL 构建 | 手写 JSON | 字典 + 辅助库 |
| 分页 | 手动管理 from/size | 内置 scroll/search_after |
| Bulk 操作 | NDJSON 手动拼接 | helpers.bulk() |
| 错误处理 | 解析 JSON | 结构化异常 |
| 连接管理 | 无 | 连接池 + 嗅探 |
| 重试 | 自己写 | 内置重试机制 |

---

## 2. 安装与环境

```bash
# 安装 elasticsearch-py
pip install elasticsearch

# 如果用 ES 7.x
pip install elasticsearch==7.17.9

# 如果用 ES 8.x
pip install elasticsearch==8.12.0

# 验证安装
python3 -c "import elasticsearch; print(f'elasticsearch-py: {elasticsearch.__version__}')"

# Docker 快速启动 ES（测试用）
# docker run -d --name es-test -p 9200:9200 -p 9300:9300 \
#   -e "discovery.type=single-node" \
#   -e "xpack.security.enabled=false" \
#   elasticsearch:8.12.0
```

---

## 3. 核心概念与原理

### ES 基本概念对照

```
关系数据库        Elasticsearch
────────────     ────────────
Database         Index（索引）
Table            Type（7.x 已废弃，一个 Index 一个 Type）
Row              Document（文档）
Column           Field（字段）
SQL              Query DSL
```

### 查询 DSL 结构

```json
{
  "query": {          // 查询条件
    "bool": {
      "must": [],     // AND 条件
      "should": [],   // OR 条件
      "must_not": [], // NOT 条件
      "filter": []    // 过滤（不影响评分，性能更好）
    }
  },
  "sort": [],         // 排序
  "from": 0,          // 偏移量
  "size": 10,         // 返回数量
  "aggs": {},         // 聚合分析
  "_source": []       // 返回字段
}
```

### 日志索引命名惯例

```
logs-nginx-2024.01.15     # 按天分索引
logs-app-2024.01          # 按月分索引
logs-*                    # 通配符匹配所有日志索引
```

---

## 4. 基础用法（最小可运行示例）

### 4.1 连接与基本操作

```python
#!/usr/bin/env python3
"""Elasticsearch 连接与基础操作"""

from elasticsearch import Elasticsearch
from datetime import datetime

# ========== 连接 ES ==========

# 方式 1：简单连接
es = Elasticsearch(
    ["http://localhost:9200"],
    request_timeout=30,
    retry_on_timeout=True,
    max_retries=3,
)

# 方式 2：带认证
# es = Elasticsearch(
#     ["https://es-host:9200"],
#     basic_auth=("elastic", "password"),
#     ca_certs="/path/to/ca.crt",
#     verify_certs=True,
# )

# 方式 3：API Key 认证
# es = Elasticsearch(
#     ["https://es-host:9200"],
#     api_key=("id", "api_key"),
# )

# 测试连接
info = es.info()
print(f"ES 版本: {info['version']['number']}")
print(f"集群名: {info['cluster_name']}")

# 集群健康
health = es.cluster.health()
print(f"集群状态: {health['status']}")  # green/yellow/red
print(f"节点数: {health['number_of_nodes']}")
print(f"分片数: {health['active_shards']}")
```

### 4.2 索引管理

```python
#!/usr/bin/env python3
"""索引管理：创建、配置、删除"""

from elasticsearch import Elasticsearch

es = Elasticsearch(["http://localhost:9200"])

INDEX_NAME = "sre-logs-test"

# ========== 创建索引 ==========
# 等价于: curl -X PUT localhost:9200/sre-logs-test -d '{...}'

index_settings = {
    "settings": {
        "number_of_shards": 1,          # 主分片数
        "number_of_replicas": 0,         # 副本数（测试环境 0）
        "refresh_interval": "5s",        # 刷新间隔
        "index.mapping.total_fields.limit": 2000,  # 字段数上限
    },
    "mappings": {
        "properties": {
            "timestamp": {
                "type": "date",
                "format": "yyyy-MM-dd HH:mm:ss||epoch_millis"
            },
            "level": {
                "type": "keyword"       # 精确匹配（不分词）
            },
            "service": {
                "type": "keyword"
            },
            "host": {
                "type": "keyword"
            },
            "message": {
                "type": "text",         # 全文搜索（分词）
                "analyzer": "standard",
                "fields": {
                    "keyword": {        # 多字段：同时支持全文和精确匹配
                        "type": "keyword",
                        "ignore_above": 512
                    }
                }
            },
            "status_code": {
                "type": "integer"
            },
            "response_time": {
                "type": "float"
            },
            "request_path": {
                "type": "keyword"
            },
            "client_ip": {
                "type": "ip"
            }
        }
    }
}

# 创建索引
if not es.indices.exists(index=INDEX_NAME):
    es.indices.create(index=INDEX_NAME, body=index_settings)
    print(f"索引 {INDEX_NAME} 创建成功")
else:
    print(f"索引 {INDEX_NAME} 已存在")

# 查看索引信息
# 等价于: curl localhost:9200/sre-logs-test/_settings
settings = es.indices.get_settings(index=INDEX_NAME)
print(f"索引设置: {settings}")

# 查看 Mapping
mapping = es.indices.get_mapping(index=INDEX_NAME)
print(f"字段映射: {list(mapping[INDEX_NAME]['mappings']['properties'].keys())}")

# 列出所有索引
# 等价于: curl localhost:9200/_cat/indices?v
indices = es.cat.indices(format="json")
for idx in indices:
    print(f"  {idx['index']}: {idx['docs.count']} docs, {idx['store.size']}")
```

### 4.3 文档 CRUD

```python
#!/usr/bin/env python3
"""文档增删改查"""

from elasticsearch import Elasticsearch
from datetime import datetime, timedelta
import random

es = Elasticsearch(["http://localhost:9200"])
INDEX = "sre-logs-test"

# ========== 插入文档 ==========

# 单条插入
# 等价于: curl -X POST localhost:9200/sre-logs-test/_doc -d '{...}'
doc = {
    "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    "level": "ERROR",
    "service": "nginx",
    "host": "web-01",
    "message": "upstream connect timeout (110: Connection timed out)",
    "status_code": 504,
    "response_time": 30.5,
    "request_path": "/api/users",
    "client_ip": "10.0.1.100"
}
result = es.index(index=INDEX, document=doc)
print(f"插入文档 ID: {result['_id']}, 结果: {result['result']}")

# ========== 批量插入 ==========
from elasticsearch.helpers import bulk

def generate_test_logs(count=100):
    """生成测试日志数据"""
    services = ["nginx", "app-server", "mysql", "redis", "kafka"]
    levels = ["INFO", "WARNING", "ERROR", "CRITICAL"]
    hosts = ["web-01", "web-02", "db-01", "cache-01"]
    paths = ["/api/users", "/api/orders", "/api/products", "/health", "/metrics"]
    messages = {
        "INFO": ["请求处理成功", "连接已建立", "缓存命中"],
        "WARNING": ["响应时间超过阈值", "内存使用率 > 80%", "连接池接近上限"],
        "ERROR": ["数据库连接失败", "上游超时", "认证失败"],
        "CRITICAL": ["服务不可用", "磁盘空间不足", "OOM Killer 触发"],
    }

    for i in range(count):
        level = random.choices(levels, weights=[60, 20, 15, 5])[0]
        timestamp = datetime.now() - timedelta(
            hours=random.randint(0, 48),
            minutes=random.randint(0, 59)
        )

        yield {
            "_index": INDEX,
            "_source": {
                "timestamp": timestamp.strftime("%Y-%m-%d %H:%M:%S"),
                "level": level,
                "service": random.choice(services),
                "host": random.choice(hosts),
                "message": random.choice(messages[level]),
                "status_code": random.choice([200, 201, 301, 400, 401, 404, 500, 502, 503, 504]),
                "response_time": round(random.uniform(0.01, 30.0), 3),
                "request_path": random.choice(paths),
                "client_ip": f"10.0.{random.randint(1,10)}.{random.randint(1,254)}"
            }
        }

# 执行批量插入
success, errors = bulk(es, generate_test_logs(500), stats_only=True)
print(f"批量插入: 成功 {success}, 失败 {errors}")

# 刷新索引（使文档可搜索）
es.indices.refresh(index=INDEX)

# ========== 查询文档 ==========

# 根据 ID 查询
# doc = es.get(index=INDEX, id="your-doc-id")

# ========== 更新文档 ==========
# es.update(index=INDEX, id="doc-id", body={
#     "doc": {"level": "WARNING"}
# })

# ========== 删除文档 ==========
# es.delete(index=INDEX, id="doc-id")

# 按条件删除
# es.delete_by_query(index=INDEX, body={
#     "query": {"match": {"level": "DEBUG"}}
# })
```

---

## 5. 进阶用法

### 5.1 DSL 查询详解

```python
#!/usr/bin/env python3
"""ES DSL 查询详解"""

from elasticsearch import Elasticsearch

es = Elasticsearch(["http://localhost:9200"])
INDEX = "sre-logs-test"

# ========== 1. 全文搜索 ==========
# 等价于: curl 'localhost:9200/sre-logs-test/_search' -d '{"query":{"match":{"message":"超时"}}}'
result = es.search(index=INDEX, body={
    "query": {
        "match": {
            "message": "超时"  # 全文搜索，会分词
        }
    },
    "size": 5,
    "_source": ["timestamp", "level", "message", "service"]
})
print(f"全文搜索: 命中 {result['hits']['total']['value']} 条")
for hit in result["hits"]["hits"]:
    print(f"  [{hit['_source']['level']}] {hit['_source']['message']}")

# ========== 2. 精确匹配 ==========
result = es.search(index=INDEX, body={
    "query": {
        "term": {
            "level": "ERROR"  # 精确匹配，不分词
        }
    },
    "size": 5
})
print(f"\n精确匹配 ERROR: {result['hits']['total']['value']} 条")

# ========== 3. 复合查询（Bool Query） ==========
result = es.search(index=INDEX, body={
    "query": {
        "bool": {
            "must": [
                {"term": {"service": "nginx"}},       # 必须是 nginx
                {"match": {"message": "超时"}}         # 消息包含"超时"
            ],
            "filter": [
                {"range": {                            # 时间范围过滤
                    "timestamp": {
                        "gte": "2024-01-01 00:00:00",
                        "lte": "2026-12-31 23:59:59"
                    }
                }},
                {"terms": {                            # 状态码在指定范围
                    "status_code": [500, 502, 503, 504]
                }}
            ],
            "must_not": [
                {"term": {"host": "web-02"}}           # 排除 web-02
            ]
        }
    },
    "sort": [
        {"timestamp": {"order": "desc"}}               # 按时间倒序
    ],
    "size": 10,
    "_source": ["timestamp", "level", "service", "host", "message", "status_code"]
})
print(f"\n复合查询: {result['hits']['total']['value']} 条")
for hit in result["hits"]["hits"]:
    src = hit["_source"]
    print(f"  {src['timestamp']} [{src['level']}] {src['service']}@{src['host']}: {src['message']}")

# ========== 4. 范围查询 ==========
result = es.search(index=INDEX, body={
    "query": {
        "range": {
            "response_time": {
                "gte": 5.0,    # 响应时间 >= 5 秒
                "lte": 30.0    # 响应时间 <= 30 秒
            }
        }
    },
    "sort": [{"response_time": "desc"}],
    "size": 5
})
print(f"\n慢请求（>=5s）: {result['hits']['total']['value']} 条")

# ========== 5. 通配符/正则查询 ==========
result = es.search(index=INDEX, body={
    "query": {
        "wildcard": {
            "request_path": "/api/*"   # 通配符
        }
    },
    "size": 5
})
print(f"\nAPI 路径: {result['hits']['total']['value']} 条")

# ========== 6. 多字段搜索 ==========
result = es.search(index=INDEX, body={
    "query": {
        "multi_match": {
            "query": "失败",
            "fields": ["message", "service"]
        }
    }
})
print(f"\n多字段搜索: {result['hits']['total']['value']} 条")
```

### 5.2 聚合分析

```python
#!/usr/bin/env python3
"""ES 聚合分析 — 日志统计与分析"""

from elasticsearch import Elasticsearch

es = Elasticsearch(["http://localhost:9200"])
INDEX = "sre-logs-test"

# ========== 1. 按日志级别统计 ==========
# 等价于: SELECT level, COUNT(*) FROM logs GROUP BY level
result = es.search(index=INDEX, body={
    "size": 0,  # 不返回文档，只要聚合结果
    "aggs": {
        "level_distribution": {
            "terms": {
                "field": "level",
                "size": 10,
                "order": {"_count": "desc"}
            }
        }
    }
})

print("日志级别分布:")
for bucket in result["aggregations"]["level_distribution"]["buckets"]:
    print(f"  {bucket['key']}: {bucket['doc_count']} 条")

# ========== 2. 按服务 + 级别交叉统计 ==========
result = es.search(index=INDEX, body={
    "size": 0,
    "aggs": {
        "by_service": {
            "terms": {"field": "service", "size": 20},
            "aggs": {
                "by_level": {
                    "terms": {"field": "level"}
                },
                "avg_response_time": {
                    "avg": {"field": "response_time"}
                }
            }
        }
    }
})

print("\n服务维度统计:")
for svc in result["aggregations"]["by_service"]["buckets"]:
    levels = {b["key"]: b["doc_count"] for b in svc["by_level"]["buckets"]}
    avg_rt = svc["avg_response_time"]["value"]
    print(f"  {svc['key']}: 总计 {svc['doc_count']} 条, "
          f"ERROR={levels.get('ERROR', 0)}, "
          f"平均响应={avg_rt:.3f}s" if avg_rt else "")

# ========== 3. 时间直方图（按小时统计） ==========
result = es.search(index=INDEX, body={
    "size": 0,
    "aggs": {
        "logs_over_time": {
            "date_histogram": {
                "field": "timestamp",
                "calendar_interval": "hour",  # 按小时
                "format": "yyyy-MM-dd HH:mm",
                "min_doc_count": 0
            },
            "aggs": {
                "error_count": {
                    "filter": {
                        "terms": {"level": ["ERROR", "CRITICAL"]}
                    }
                }
            }
        }
    }
})

print("\n每小时日志趋势:")
for bucket in result["aggregations"]["logs_over_time"]["buckets"][-12:]:  # 最近 12 小时
    total = bucket["doc_count"]
    errors = bucket["error_count"]["doc_count"]
    bar = "#" * min(total // 2, 40)
    print(f"  {bucket['key_as_string']}: {total:>4} 条 (ERR: {errors:>2}) {bar}")

# ========== 4. 响应时间百分位数 ==========
result = es.search(index=INDEX, body={
    "size": 0,
    "aggs": {
        "response_time_stats": {
            "extended_stats": {"field": "response_time"}
        },
        "response_time_percentiles": {
            "percentiles": {
                "field": "response_time",
                "percents": [50, 75, 90, 95, 99]
            }
        }
    }
})

stats = result["aggregations"]["response_time_stats"]
pcts = result["aggregations"]["response_time_percentiles"]["values"]
print(f"\n响应时间统计:")
print(f"  最小: {stats['min']:.3f}s")
print(f"  最大: {stats['max']:.3f}s")
print(f"  平均: {stats['avg']:.3f}s")
print(f"  P50: {pcts['50.0']:.3f}s")
print(f"  P95: {pcts['95.0']:.3f}s")
print(f"  P99: {pcts['99.0']:.3f}s")

# ========== 5. Top N 错误路径 ==========
result = es.search(index=INDEX, body={
    "size": 0,
    "query": {
        "terms": {"status_code": [500, 502, 503, 504]}
    },
    "aggs": {
        "top_error_paths": {
            "terms": {
                "field": "request_path",
                "size": 10,
                "order": {"_count": "desc"}
            }
        }
    }
})

print("\n5xx 错误 Top 路径:")
for bucket in result["aggregations"]["top_error_paths"]["buckets"]:
    print(f"  {bucket['key']}: {bucket['doc_count']} 次")
```

### 5.3 Scroll 大量数据查询

```python
#!/usr/bin/env python3
"""Scroll API — 查询大量数据（类似数据库游标）"""

from elasticsearch import Elasticsearch
from elasticsearch.helpers import scan

es = Elasticsearch(["http://localhost:9200"])
INDEX = "sre-logs-test"

# ========== 方式 1：使用 helpers.scan（推荐） ==========

def export_error_logs():
    """导出所有 ERROR 级别日志"""

    query = {
        "query": {
            "bool": {
                "filter": [
                    {"terms": {"level": ["ERROR", "CRITICAL"]}}
                ]
            }
        },
        "sort": [{"timestamp": "desc"}]
    }

    # scan() 自动处理 scroll，逐条返回
    count = 0
    for doc in scan(
        es,
        index=INDEX,
        query=query,
        scroll="5m",       # scroll 上下文保持 5 分钟
        size=1000,          # 每批 1000 条
        request_timeout=60
    ):
        source = doc["_source"]
        count += 1
        if count <= 5:  # 只打印前 5 条
            print(f"  {source['timestamp']} [{source['level']}] "
                  f"{source['service']}: {source['message']}")

    print(f"总共导出 {count} 条错误日志")

export_error_logs()

# ========== 方式 2：手动 Scroll ==========

def manual_scroll_demo():
    """手动管理 scroll（了解原理）"""

    # 第一次搜索，创建 scroll 上下文
    result = es.search(
        index=INDEX,
        body={"query": {"match_all": {}}, "size": 100},
        scroll="2m"
    )

    scroll_id = result["_scroll_id"]
    total = result["hits"]["total"]["value"]
    hits = result["hits"]["hits"]
    fetched = len(hits)

    print(f"总文档数: {total}")

    # 继续滚动获取
    while len(hits) > 0:
        result = es.scroll(scroll_id=scroll_id, scroll="2m")
        scroll_id = result["_scroll_id"]
        hits = result["hits"]["hits"]
        fetched += len(hits)

    print(f"已获取: {fetched} 条")

    # 清理 scroll 上下文（重要！释放服务端资源）
    es.clear_scroll(scroll_id=scroll_id)

manual_scroll_demo()
```

---

## 6. SRE 实战案例：日志搜索与错误统计分析工具

```python
#!/usr/bin/env python3
"""
SRE 实战：ES 日志搜索与错误统计分析工具
功能：
  1. 日志搜索（关键词、级别、服务、时间范围）
  2. 错误统计（按服务、按路径、按时间趋势）
  3. 异常检测（突增告警）
  4. 生成分析报告
"""

from elasticsearch import Elasticsearch
from elasticsearch.helpers import scan
from datetime import datetime, timedelta
from typing import Optional, List, Dict
import json
import sys


class LogAnalyzer:
    """ES 日志分析器"""

    def __init__(self, es_hosts: list, index_pattern: str = "sre-logs-*"):
        """
        初始化分析器

        参数:
            es_hosts: ES 节点列表
            index_pattern: 索引模式
        """
        self.es = Elasticsearch(
            es_hosts,
            request_timeout=30,
            retry_on_timeout=True,
            max_retries=3,
        )
        self.index_pattern = index_pattern

        # 验证连接
        if not self.es.ping():
            raise ConnectionError("无法连接到 Elasticsearch")

    # ========== 日志搜索 ==========

    def search_logs(
        self,
        keyword: str = None,
        level: str = None,
        service: str = None,
        host: str = None,
        time_range: str = "24h",
        size: int = 50,
    ) -> Dict:
        """
        多条件日志搜索

        参数:
            keyword: 搜索关键词（全文搜索 message 字段）
            level: 日志级别过滤
            service: 服务名过滤
            host: 主机名过滤
            time_range: 时间范围（如 "1h", "24h", "7d"）
            size: 返回条数

        返回:
            dict: 搜索结果
        """
        # 构建 Bool 查询
        must = []
        filters = []

        if keyword:
            must.append({"match": {"message": keyword}})

        if level:
            filters.append({"term": {"level": level.upper()}})

        if service:
            filters.append({"term": {"service": service}})

        if host:
            filters.append({"term": {"host": host}})

        # 时间范围过滤
        if time_range:
            now = datetime.now()
            unit = time_range[-1]
            value = int(time_range[:-1])
            if unit == "h":
                start = now - timedelta(hours=value)
            elif unit == "d":
                start = now - timedelta(days=value)
            elif unit == "m":
                start = now - timedelta(minutes=value)
            else:
                start = now - timedelta(hours=24)

            filters.append({
                "range": {
                    "timestamp": {
                        "gte": start.strftime("%Y-%m-%d %H:%M:%S"),
                        "lte": now.strftime("%Y-%m-%d %H:%M:%S")
                    }
                }
            })

        # 构建完整查询
        query = {
            "query": {
                "bool": {
                    "must": must if must else [{"match_all": {}}],
                    "filter": filters
                }
            },
            "sort": [{"timestamp": {"order": "desc"}}],
            "size": size,
            "_source": ["timestamp", "level", "service", "host",
                        "message", "status_code", "response_time"]
        }

        result = self.es.search(index=self.index_pattern, body=query)

        return {
            "total": result["hits"]["total"]["value"],
            "returned": len(result["hits"]["hits"]),
            "logs": [hit["_source"] for hit in result["hits"]["hits"]]
        }

    # ========== 错误统计 ==========

    def error_statistics(self, time_range: str = "24h") -> Dict:
        """
        错误统计分析

        参数:
            time_range: 时间范围

        返回:
            dict: 统计结果
        """
        now = datetime.now()
        unit = time_range[-1]
        value = int(time_range[:-1])
        if unit == "h":
            start = now - timedelta(hours=value)
        elif unit == "d":
            start = now - timedelta(days=value)
        else:
            start = now - timedelta(hours=24)

        query = {
            "size": 0,
            "query": {
                "bool": {
                    "filter": [
                        {"terms": {"level": ["ERROR", "CRITICAL"]}},
                        {"range": {"timestamp": {
                            "gte": start.strftime("%Y-%m-%d %H:%M:%S"),
                            "lte": now.strftime("%Y-%m-%d %H:%M:%S")
                        }}}
                    ]
                }
            },
            "aggs": {
                # 按服务统计
                "by_service": {
                    "terms": {"field": "service", "size": 20},
                    "aggs": {
                        "by_level": {"terms": {"field": "level"}},
                        "avg_response": {"avg": {"field": "response_time"}}
                    }
                },
                # 按主机统计
                "by_host": {
                    "terms": {"field": "host", "size": 20}
                },
                # 按错误消息统计（Top 错误）
                "top_errors": {
                    "terms": {
                        "field": "message.keyword",
                        "size": 10,
                        "order": {"_count": "desc"}
                    }
                },
                # 按 HTTP 状态码统计
                "by_status_code": {
                    "terms": {"field": "status_code", "size": 20}
                },
                # 时间趋势
                "error_trend": {
                    "date_histogram": {
                        "field": "timestamp",
                        "calendar_interval": "hour",
                        "format": "MM-dd HH:mm"
                    }
                },
                # 总错误数
                "total_errors": {
                    "value_count": {"field": "level"}
                }
            }
        }

        result = self.es.search(index=self.index_pattern, body=query)
        aggs = result["aggregations"]

        return {
            "time_range": time_range,
            "total_errors": result["hits"]["total"]["value"],
            "by_service": [
                {
                    "service": b["key"],
                    "count": b["doc_count"],
                    "levels": {l["key"]: l["doc_count"] for l in b["by_level"]["buckets"]},
                    "avg_response": round(b["avg_response"]["value"] or 0, 3)
                }
                for b in aggs["by_service"]["buckets"]
            ],
            "by_host": [
                {"host": b["key"], "count": b["doc_count"]}
                for b in aggs["by_host"]["buckets"]
            ],
            "top_errors": [
                {"message": b["key"], "count": b["doc_count"]}
                for b in aggs["top_errors"]["buckets"]
            ],
            "by_status_code": [
                {"code": b["key"], "count": b["doc_count"]}
                for b in aggs["by_status_code"]["buckets"]
            ],
            "trend": [
                {"time": b["key_as_string"], "count": b["doc_count"]}
                for b in aggs["error_trend"]["buckets"]
            ]
        }

    # ========== 异常检测 ==========

    def detect_anomalies(self, window_minutes: int = 10) -> Dict:
        """
        检测错误突增

        逻辑：比较最近 N 分钟与前 N 分钟的错误数，
              如果增长超过 200% 则报警

        参数:
            window_minutes: 检测窗口（分钟）

        返回:
            dict: 异常检测结果
        """
        now = datetime.now()
        recent_start = now - timedelta(minutes=window_minutes)
        prev_start = now - timedelta(minutes=window_minutes * 2)

        def count_errors(start, end):
            result = self.es.count(index=self.index_pattern, body={
                "query": {
                    "bool": {
                        "filter": [
                            {"terms": {"level": ["ERROR", "CRITICAL"]}},
                            {"range": {"timestamp": {
                                "gte": start.strftime("%Y-%m-%d %H:%M:%S"),
                                "lte": end.strftime("%Y-%m-%d %H:%M:%S")
                            }}}
                        ]
                    }
                }
            })
            return result["count"]

        recent_count = count_errors(recent_start, now)
        prev_count = count_errors(prev_start, recent_start)

        # 计算增长率
        if prev_count > 0:
            growth_rate = (recent_count - prev_count) / prev_count * 100
        elif recent_count > 0:
            growth_rate = 100.0
        else:
            growth_rate = 0

        is_anomaly = growth_rate > 200 and recent_count > 5  # 增长超200%且数量>5

        return {
            "window_minutes": window_minutes,
            "recent_errors": recent_count,
            "previous_errors": prev_count,
            "growth_rate": round(growth_rate, 1),
            "is_anomaly": is_anomaly,
            "severity": "CRITICAL" if growth_rate > 500 else
                        "WARNING" if growth_rate > 200 else "OK"
        }

    # ========== 生成报告 ==========

    def generate_report(self, time_range: str = "24h") -> str:
        """生成分析报告"""
        report = []
        report.append("=" * 65)
        report.append("        ES 日志分析报告")
        report.append("=" * 65)
        report.append(f"分析时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append(f"时间范围: 最近 {time_range}")
        report.append(f"索引模式: {self.index_pattern}")

        # 错误统计
        try:
            stats = self.error_statistics(time_range)
            report.append(f"\n总错误数: {stats['total_errors']}")

            report.append("\n--- 按服务统计 ---")
            for svc in stats["by_service"]:
                report.append(
                    f"  {svc['service']:<15} | 错误: {svc['count']:>5} | "
                    f"平均响应: {svc['avg_response']:.3f}s"
                )

            report.append("\n--- 按主机统计 ---")
            for h in stats["by_host"]:
                report.append(f"  {h['host']:<15} | 错误: {h['count']:>5}")

            report.append("\n--- Top 错误消息 ---")
            for err in stats["top_errors"][:5]:
                report.append(f"  [{err['count']:>4}次] {err['message'][:60]}")

            report.append("\n--- HTTP 状态码 ---")
            for sc in stats["by_status_code"]:
                report.append(f"  HTTP {sc['code']}: {sc['count']} 次")

            # 趋势图（ASCII）
            if stats["trend"]:
                report.append("\n--- 错误趋势 ---")
                max_count = max(b["count"] for b in stats["trend"]) or 1
                for b in stats["trend"][-12:]:
                    bar_len = int(b["count"] / max_count * 30)
                    bar = "#" * bar_len
                    report.append(f"  {b['time']} | {b['count']:>4} {bar}")

        except Exception as e:
            report.append(f"\n错误统计失败: {e}")

        # 异常检测
        try:
            anomaly = self.detect_anomalies(window_minutes=10)
            report.append(f"\n--- 异常检测 ---")
            report.append(f"  最近 10 分钟错误: {anomaly['recent_errors']}")
            report.append(f"  前 10 分钟错误: {anomaly['previous_errors']}")
            report.append(f"  增长率: {anomaly['growth_rate']:.1f}%")
            report.append(f"  状态: {anomaly['severity']}")
        except Exception as e:
            report.append(f"\n异常检测失败: {e}")

        report.append("\n" + "=" * 65)
        return "\n".join(report)


# ========== 主函数 ==========

def main():
    """主入口"""
    try:
        analyzer = LogAnalyzer(
            es_hosts=["http://localhost:9200"],
            index_pattern="sre-logs-test"
        )
    except Exception as e:
        print(f"连接 ES 失败: {e}")
        print("请确认 ES 服务已启动")
        sys.exit(1)

    # 1. 搜索日志
    print("===== 日志搜索 =====")
    result = analyzer.search_logs(
        keyword="失败",
        level="ERROR",
        time_range="48h",
        size=5
    )
    print(f"搜索结果: {result['total']} 条 (显示 {result['returned']} 条)")
    for log in result["logs"]:
        print(f"  {log.get('timestamp')} [{log.get('level')}] "
              f"{log.get('service')}: {log.get('message')}")

    # 2. 生成报告
    print("\n")
    report = analyzer.generate_report("48h")
    print(report)

    # 3. 导出报告
    output_path = "/tmp/es_log_report.json"
    stats = analyzer.error_statistics("48h")
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(stats, f, ensure_ascii=False, indent=2)
    print(f"\n详细数据已导出: {output_path}")


if __name__ == "__main__":
    main()
```

---

## 7. 常见坑点与最佳实践

### 坑点 1：from + size 深分页

```python
# 错误：深分页（from > 10000 会报错）
es.search(index="logs", body={"from": 50000, "size": 100})

# 正确：用 scroll 或 search_after
from elasticsearch.helpers import scan
for doc in scan(es, index="logs", query={"query": {"match_all": {}}}):
    process(doc)
```

### 坑点 2：Mapping 爆炸

```python
# 错误：动态 Mapping 导致字段数爆炸
# 如果日志中有不固定的 JSON 字段，ES 会为每个字段创建 Mapping

# 正确：关闭动态 Mapping 或设置上限
index_settings = {
    "mappings": {
        "dynamic": "strict",  # 只接受预定义的字段
        # 或 "dynamic": false  # 忽略未定义字段
    }
}
```

### 坑点 3：不清理 Scroll 上下文

```python
# 错误：scroll 用完不清理，浪费服务端资源
result = es.search(index="logs", scroll="5m", ...)
# 没有 clear_scroll！

# 正确：用完及时清理
es.clear_scroll(scroll_id=scroll_id)

# 或者用 helpers.scan（自动管理）
```

### 坑点 4：Bulk 操作不处理错误

```python
# 错误：bulk 部分失败但不知道
bulk(es, actions)

# 正确：检查错误
success, errors = bulk(es, actions, raise_on_error=False, stats_only=False)
if errors:
    for err in errors:
        print(f"Bulk 错误: {err}")
```

---

## 8. 速查表

### elasticsearch-py 操作速查

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk, scan

es = Elasticsearch(["http://localhost:9200"])

# --- 索引管理 ---
es.indices.create(index="name", body=settings)    # 创建索引
es.indices.delete(index="name")                    # 删除索引
es.indices.exists(index="name")                    # 索引是否存在
es.indices.refresh(index="name")                   # 刷新索引
es.indices.get_mapping(index="name")               # 查看映射
es.cat.indices(format="json")                      # 列出所有索引

# --- 文档操作 ---
es.index(index="name", document=doc)               # 插入文档
es.get(index="name", id="doc-id")                  # 获取文档
es.update(index="name", id="id", body={"doc": {}}) # 更新文档
es.delete(index="name", id="id")                   # 删除文档
bulk(es, actions)                                   # 批量操作

# --- 搜索 ---
es.search(index="name", body=query)                # 搜索
es.count(index="name", body=query)                 # 计数
scan(es, index="name", query=query)                # 滚动查询

# --- 集群 ---
es.cluster.health()                                # 集群健康
es.cluster.stats()                                 # 集群统计
es.info()                                          # 集群信息
```

### DSL 查询速查

```python
# 全文搜索
{"match": {"message": "error timeout"}}

# 精确匹配
{"term": {"level": "ERROR"}}

# 多值匹配
{"terms": {"status_code": [500, 502, 503]}}

# 范围查询
{"range": {"timestamp": {"gte": "2024-01-01", "lte": "2024-01-31"}}}

# Bool 组合
{"bool": {"must": [...], "filter": [...], "must_not": [...], "should": [...]}}

# 通配符
{"wildcard": {"path": "/api/*"}}

# 存在性检查
{"exists": {"field": "error_message"}}
```

### 对比 curl 命令

| curl | elasticsearch-py |
|------|-----------------|
| `GET /_cluster/health` | `es.cluster.health()` |
| `GET /_cat/indices?v` | `es.cat.indices(format="json")` |
| `PUT /index` | `es.indices.create(index="index")` |
| `POST /index/_doc` | `es.index(index="index", document=doc)` |
| `GET /index/_search` | `es.search(index="index", body=query)` |
| `POST /index/_count` | `es.count(index="index", body=query)` |
| `DELETE /index` | `es.indices.delete(index="index")` |
