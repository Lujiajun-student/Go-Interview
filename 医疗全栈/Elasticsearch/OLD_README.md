下面按“从概念到原理、从使用到优化、从开发到面试”的方式系统讲解 Elasticsearch，简称 ES。严格来说官方名称是 **Elasticsearch**，不是 ElasticSearch，不过日常面试和开发中常写 ES。

# 1. Elasticsearch 是什么

Elasticsearch 是一个基于 **Apache Lucene** 构建的分布式搜索与分析引擎。它擅长解决的问题不是“事务型数据存储”，而是：

搜索、检索、模糊匹配、全文检索、多字段组合查询、日志检索、聚合分析、近实时查询、地理位置搜索、自动补全、向量检索等。

例如医疗系统中可以用 ES 做：

病历全文检索：搜索“糖尿病 高血压 冠心病”。

患者信息模糊搜索：输入姓名拼音、手机号后四位、身份证部分信息等。

文档检索：检索检查报告、出院小结、医嘱、影像报告。

日志检索：查询某个接口在某个时间段内的异常日志。

统计分析：统计不同科室、不同病种、不同时间段的就诊数量。

但是 ES 不适合做强事务业务主库。比如订单扣库存、财务转账、医疗收费、处方开具等强一致性业务，仍然应该以 MySQL、Oracle、PostgreSQL 等关系型数据库为主，ES 通常作为搜索副本或分析副本。

# 2. ES 的整体架构

ES 是一个分布式系统，核心概念包括：

Cluster，集群。多个 ES 节点组成一个集群，共同对外提供搜索和写入能力。

Node，节点。一个 ES 实例就是一个节点。节点可以承担不同角色，例如 master node、data node、coordinating node、ingest node、machine learning node 等。

Index，索引。类似数据库中的表，但不能完全等同。一个 index 存放一类结构相似的文档，例如 `patient_index`、`medical_record_index`、`log-2026.06.17`。

Document，文档。ES 中最小的可搜索数据单元，通常是一个 JSON 对象。类似关系型数据库中的一行记录。

Field，字段。文档中的属性，例如 `patient_name`、`age`、`diagnosis`、`created_time`。

Mapping，映射。定义字段类型、分词方式、索引方式、是否可聚合、是否存储等。类似表结构，但 ES 的 mapping 更关注搜索行为。

Shard，分片。一个索引会被拆分成多个主分片，每个分片本质上是一个 Lucene 索引。分片使 ES 可以横向扩展。

Replica，副本。主分片的复制品，用于高可用和提高查询吞吐。主分片挂了，副本可以提升为主分片。

Segment，段。Lucene 底层不可变的小索引文件。ES 写入数据后会不断产生 segment，后续通过 merge 合并。

倒排索引、分词器、mapping、shard、replica、refresh、segment，这些是理解 ES 的核心。

# 3. ES、MySQL、Oracle 的区别

ES 和关系型数据库有明显差异。

| 对比项     | MySQL / Oracle       | Elasticsearch          |
| ---------- | -------------------- | ---------------------- |
| 核心定位   | 事务型数据库 OLTP    | 搜索与分析引擎         |
| 数据模型   | 表、行、列           | 索引、文档、字段       |
| 查询方式   | SQL                  | Query DSL              |
| 强事务     | 支持 ACID            | 不适合作为强事务主库   |
| 多表 Join  | 擅长                 | 不擅长，应尽量反范式化 |
| 全文搜索   | 较弱或需要额外插件   | 非常强                 |
| 模糊搜索   | 较弱                 | 强                     |
| 聚合分析   | 支持但大数据量压力大 | 擅长近实时聚合         |
| 扩展方式   | 分库分表、读写分离   | 原生分布式分片         |
| 数据一致性 | 强一致事务能力强     | 通常是近实时、最终一致 |

所以 ES 常见架构是：

业务数据先写入 MySQL / Oracle，作为权威数据源。

通过 Canal、Debezium、Logstash、MQ、定时任务或业务异步写入，将数据同步到 ES。

查询搜索类接口走 ES。

涉及准确交易、状态变更、强一致判断的接口走数据库。

一句话概括：**数据库负责正确地存，ES 负责快速地搜。**

# 4. 倒排索引

倒排索引是 ES 的核心。

传统数据库通常使用正排结构：

```text
文档1：张三患有高血压和糖尿病
文档2：李四患有高血压
文档3：王五体检正常
```

如果搜索“高血压”，普通方式要扫描每篇文档。

倒排索引会反过来建立“词 → 文档”的映射：

```text
高血压 -> 文档1, 文档2
糖尿病 -> 文档1
体检   -> 文档3
正常   -> 文档3
```

这样搜索“高血压”时，不需要扫描所有文档，直接通过词项找到对应文档。

Lucene 的倒排索引通常包含：

Term Dictionary：词典，保存所有词项。

Term Index：词典索引，用于快速定位词项。

Posting List：倒排列表，记录某个词出现在哪些文档中。

Position：词在文档中的位置，用于短语查询，例如 `"高血压 糖尿病"`。

Offset：词在原文中的字符偏移，用于高亮显示。

Term Frequency：词频，用于相关性评分。

Doc Values：列式存储，用于排序、聚合、脚本计算。

Norms：字段长度等评分信息。

倒排索引适合全文检索，但不等于万能索引。数值范围查询、日期范围查询、地理位置查询等会使用不同的数据结构，例如 BKD Tree、Point 类型索引等。

# 5. 分词器 Analyzer

分词器决定文本如何被拆成词。它直接影响搜索结果。

ES 的 Analyzer 通常由三部分组成：

Char Filter：字符过滤器。先对原始文本做预处理，例如去 HTML 标签、字符替换。

Tokenizer：分词器。把文本切成 token。例如按空格切、按词典切、按 ngram 切。

Token Filter：词元过滤器。对 token 做加工，例如小写化、去停用词、同义词扩展、拼音转换、词干提取。

例如英文：

```text
The patient has diabetes.
```

可能被分成：

```text
patient, has, diabetes
```

中文更复杂，因为中文天然没有空格：

```text
患者患有高血压和糖尿病
```

如果分词不正确，搜索效果会非常差。

常见分词器：

standard：标准分词器，英文效果较好，中文通常按单字切分，不适合复杂中文搜索。

keyword：不分词，整段内容作为一个词。适合精确匹配，例如身份证号、手机号、订单号、状态码。

ik_smart：IK 中文分词的智能模式，切分较粗，搜索召回较少但更精确。

ik_max_word：IK 中文分词的细粒度模式，尽可能多地切词，召回更高。

pinyin analyzer：拼音分词，适合姓名、药品名、机构名拼音搜索。

ngram / edge_ngram：适合前缀搜索、自动补全、模糊输入。

synonym：同义词过滤器，例如“高血压”和“血压高”可以扩展为同义表达。

分词器有两个重要阶段：

index analyzer：写入文档时怎么分词。

search analyzer：查询时怎么分词。

例如字段 `diagnosis` 写入时用 `ik_max_word` 提高召回，查询时用 `ik_smart` 提高精度，是中文搜索中常见做法。

查看分词效果可以用 `_analyze`：

```json
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "患者患有高血压和糖尿病"
}
```

分词器设计是 ES 搜索效果优化的核心之一。

# 6. Index、Document、Mapping

## 6.1 Index

Index 是一类文档的集合。比如：

```text
patient_index
medical_record_index
prescription_index
log-2026.06.17
```

日志场景通常按时间建索引，例如每天一个索引：

```text
app-log-2026.06.17
app-log-2026.06.18
```

业务搜索场景通常按业务对象建索引，例如：

```text
patient
doctor
department
medical_record
```

Index 的设计要考虑数据量、查询模式、生命周期、分片数、权限隔离、租户隔离等。

## 6.2 Document

文档是 JSON，例如：

```json
{
  "patient_id": "P10001",
  "name": "张三",
  "gender": "male",
  "age": 56,
  "phone": "13800001111",
  "diagnosis": "高血压、2型糖尿病",
  "department": "内分泌科",
  "visit_time": "2026-06-17 09:30:00"
}
```

ES 文档是 schema-flexible 的，但生产环境不应该完全依赖动态 mapping。动态 mapping 容易导致字段类型误判，例如把字符串日期识别错误，或者把本该是 keyword 的字段识别成 text。

## 6.3 Mapping

Mapping 定义字段类型和索引方式，例如：

```json
PUT /patient_index
{
  "mappings": {
    "properties": {
      "patient_id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "age": {
        "type": "integer"
      },
      "phone": {
        "type": "keyword"
      },
      "diagnosis": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "department": {
        "type": "keyword"
      },
      "visit_time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||strict_date_optional_time||epoch_millis"
      }
    }
  }
}
```

注意这里的 `name` 同时有两个字段：

`name` 是 text 类型，用于全文检索。

`name.keyword` 是 keyword 类型，用于精确匹配、排序、聚合。

这就是 multi-fields，多字段映射。它是 ES 中非常常见的设计。

# 7. Text 和 Keyword 的区别

这是 ES 面试和开发中最高频的知识点之一。

## 7.1 text

`text` 会被分词，适合全文检索。

例如：

```json
"diagnosis": {
  "type": "text",
  "analyzer": "ik_max_word"
}
```

文本：

```text
患者患有高血压和糖尿病
```

可能被分成：

```text
患者, 患有, 高血压, 糖尿病
```

适合：

```text
病历内容
文章正文
商品标题
医生简介
诊断描述
日志 message
```

通常用 `match` 查询。

## 7.2 keyword

`keyword` 不分词，整体作为一个词，适合精确匹配、过滤、聚合、排序。

例如：

```json
"department": {
  "type": "keyword"
}
```

适合：

```text
订单号
患者ID
手机号
身份证号
状态
枚举值
科室名
标签
城市
日志级别 ERROR / INFO
```

通常用 `term` 查询。

## 7.3 常见错误

错误一：用 `term` 查询 text 字段。

```json
{
  "term": {
    "diagnosis": "高血压糖尿病"
  }
}
```

如果 `diagnosis` 被分词了，索引中可能没有完整的 `"高血压糖尿病"` 这个 term，查询可能查不到。

错误二：用 `match` 查询 keyword 字段。

```json
{
  "match": {
    "patient_id": "P10001"
  }
}
```

虽然有时能查到，但语义上不准确。ID、状态、枚举值应该用 `term`。

简单记忆：

全文检索：`text + match`

精确匹配：`keyword + term`

排序聚合：优先使用 `keyword / numeric / date`，不要直接对 `text` 聚合。

# 8. 精确查询

精确查询通常不分析查询文本，而是直接匹配倒排索引中的 term。

常见精确查询包括：

`term`：查单个精确值。

```json
GET /patient_index/_search
{
  "query": {
    "term": {
      "patient_id": "P10001"
    }
  }
}
```

`terms`：匹配多个值。

```json
{
  "query": {
    "terms": {
      "department": ["内分泌科", "心内科"]
    }
  }
}
```

`ids`：根据 `_id` 查询。

```json
{
  "query": {
    "ids": {
      "values": ["1", "2", "3"]
    }
  }
}
```

`exists`：判断字段是否存在。

```json
{
  "query": {
    "exists": {
      "field": "phone"
    }
  }
}
```

`prefix`：前缀查询。

```json
{
  "query": {
    "prefix": {
      "phone": "138"
    }
  }
}
```

`wildcard`：通配符查询。

```json
{
  "query": {
    "wildcard": {
      "name.keyword": "张*"
    }
  }
}
```

注意：`wildcard`、`regexp`、前置通配符如 `*abc` 可能非常慢，生产环境要谨慎使用。

# 9. 全文查询和模糊查询

全文查询会对查询内容进行分词，然后拿分词结果去倒排索引中检索。

## 9.1 match 查询

```json
GET /medical_record_index/_search
{
  "query": {
    "match": {
      "content": "高血压 糖尿病"
    }
  }
}
```

它会对 `"高血压 糖尿病"` 分词，再查询相关文档。

## 9.2 match_phrase 短语查询

要求词项顺序和距离更严格。

```json
{
  "query": {
    "match_phrase": {
      "content": "高血压 糖尿病"
    }
  }
}
```

可以加 `slop` 允许中间隔几个词：

```json
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "高血压 糖尿病",
        "slop": 2
      }
    }
  }
}
```

## 9.3 multi_match 多字段查询

例如同时搜索姓名、诊断、病历内容：

```json
{
  "query": {
    "multi_match": {
      "query": "张三 高血压",
      "fields": ["name^3", "diagnosis^2", "content"]
    }
  }
}
```

`name^3` 表示 name 字段权重更高。

## 9.4 fuzzy 模糊查询

fuzzy 可以容忍编辑距离错误。例如拼错、少字、多字。

```json
{
  "query": {
    "match": {
      "name": {
        "query": "章三",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

但 fuzzy 不是万能的中文纠错。中文模糊搜索通常还会结合：

拼音分词。

同义词。

ngram。

自定义词典。

业务规则召回。

function_score 权重调整。

# 10. 范围查询

范围查询常用于数值、日期、价格、年龄、时间范围等。

```json
GET /patient_index/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 40,
        "lte": 70
      }
    }
  }
}
```

日期范围：

```json
{
  "query": {
    "range": {
      "visit_time": {
        "gte": "2026-06-01 00:00:00",
        "lte": "2026-06-17 23:59:59"
      }
    }
  }
}
```

常见操作符：

`gt`：大于。

`gte`：大于等于。

`lt`：小于。

`lte`：小于等于。

范围字段不要用 text，要用 integer、long、double、date 等合适类型。

# 11. Bool 查询

`bool` 是 ES 查询中最重要的组合查询。

它包含：

`must`：必须匹配，参与评分。

`should`：应该匹配，通常用于提升相关性。

`filter`：必须匹配，但不参与评分，适合过滤条件。

`must_not`：必须不匹配。

例如查询“内分泌科，年龄 40 到 70，病历中包含糖尿病，排除已删除数据”：

```json
GET /patient_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "diagnosis": "糖尿病"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "department": "内分泌科"
          }
        },
        {
          "range": {
            "age": {
              "gte": 40,
              "lte": 70
            }
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "deleted": true
          }
        }
      ]
    }
  }
}
```

开发中要区分 `must` 和 `filter`：

和相关性排序有关的放 `must`。

只是筛选条件的放 `filter`。

`filter` 不计算 `_score`，性能通常更好，也更容易被缓存。

# 12. 搜索相关性和权重排序

ES 默认使用 BM25 相关性算法。BM25 可以粗略理解为：

词在文档中出现越多，相关性越高。

词越稀有，区分度越高。

字段越短，命中词越集中，相关性越高。

但实际业务搜索不能只依赖默认 `_score`。常见优化方式包括：

字段权重。比如姓名命中比正文命中更重要。

```json
{
  "multi_match": {
    "query": "张三 高血压",
    "fields": ["name^5", "diagnosis^3", "content"]
  }
}
```

should 提权。比如精确命中 ID、手机号、姓名时提高分数。

```json
{
  "bool": {
    "must": [
      {
        "match": {
          "content": "糖尿病"
        }
      }
    ],
    "should": [
      {
        "term": {
          "department": {
            "value": "内分泌科",
            "boost": 2
          }
        }
      }
    ]
  }
}
```

function_score。根据业务字段调整分数，例如最近就诊时间、医院等级、医生评分、点击量、热度等。

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "content": "糖尿病"
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "visit_count",
            "factor": 0.1,
            "modifier": "log1p"
          }
        }
      ],
      "boost_mode": "sum",
      "score_mode": "sum"
    }
  }
}
```

时间衰减。越新的文档权重越高。

```json
{
  "gauss": {
    "visit_time": {
      "origin": "now",
      "scale": "30d",
      "decay": 0.5
    }
  }
}
```

相关性优化通常包括：

分词优化。

字段权重优化。

同义词优化。

业务排序特征引入。

召回和排序分层。

使用 `_explain` 查看评分原因。

使用 `_profile` 分析查询耗时。

# 13. 聚合查询 Aggregation

聚合用于统计分析，类似 SQL 的 `GROUP BY`、`COUNT`、`AVG`、`MAX`、`MIN` 等，但 ES 聚合能力更适合搜索结果上的实时分析。

聚合分三类：

Metric Aggregation：指标聚合，例如 count、avg、sum、min、max、stats、cardinality。

Bucket Aggregation：桶聚合，例如 terms、range、date_histogram、filter、histogram。

Pipeline Aggregation：管道聚合，对前面聚合结果再次计算。

## 13.1 按科室统计患者数

```json
GET /patient_index/_search
{
  "size": 0,
  "aggs": {
    "by_department": {
      "terms": {
        "field": "department",
        "size": 10
      }
    }
  }
}
```

`size: 0` 表示不返回文档，只返回聚合结果。

## 13.2 统计平均年龄

```json
{
  "size": 0,
  "aggs": {
    "avg_age": {
      "avg": {
        "field": "age"
      }
    }
  }
}
```

## 13.3 按日期统计就诊量

```json
{
  "size": 0,
  "aggs": {
    "visits_per_day": {
      "date_histogram": {
        "field": "visit_time",
        "calendar_interval": "day"
      }
    }
  }
}
```

## 13.4 先按科室分组，再统计平均年龄

```json
{
  "size": 0,
  "aggs": {
    "by_department": {
      "terms": {
        "field": "department"
      },
      "aggs": {
        "avg_age": {
          "avg": {
            "field": "age"
          }
        }
      }
    }
  }
}
```

聚合注意事项：

聚合字段通常要用 keyword、numeric、date。

不要直接对 text 字段做 terms 聚合，否则可能触发 fielddata，导致大量堆内存消耗。

高基数字段，例如用户 ID、订单 ID、身份证号，不适合随意 terms 聚合。

大聚合要关注内存、shard_size、size、cardinality 近似误差等问题。

# 14. 高亮查询 Highlight

高亮用于在搜索结果中突出显示命中的词。

```json
GET /medical_record_index/_search
{
  "query": {
    "match": {
      "content": "糖尿病"
    }
  },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "content": {}
    }
  }
}
```

返回结果中可能包含：

```json
"highlight": {
  "content": [
    "患者患有<em>糖尿病</em>多年"
  ]
}
```

高亮依赖分词、offset、字段配置。大字段高亮可能消耗较高，日志系统或长文本检索中要谨慎。

# 15. 分页查询

ES 支持多种分页方式。

## 15.1 from + size

普通分页：

```json
GET /patient_index/_search
{
  "from": 0,
  "size": 10,
  "query": {
    "match": {
      "diagnosis": "糖尿病"
    }
  }
}
```

第二页：

```json
{
  "from": 10,
  "size": 10
}
```

缺点是深分页性能差。

比如：

```json
{
  "from": 100000,
  "size": 10
}
```

ES 不是直接跳到第 100001 条，而是每个分片都要取大量候选结果，再在协调节点合并排序，内存和 CPU 压力很大。

默认 `index.max_result_window` 通常限制为 10000，不建议简单调大。

## 15.2 Scroll

Scroll 适合后台批量导出、全量遍历，不适合用户实时翻页。

```json
GET /patient_index/_search?scroll=1m
{
  "size": 1000,
  "query": {
    "match_all": {}
  }
}
```

Scroll 会保留搜索上下文，占用资源。用完要清理。

## 15.3 search_after

`search_after` 适合深分页和“下一页”场景，但不支持随机跳页。

要求有稳定排序字段，例如时间 + id：

```json
GET /patient_index/_search
{
  "size": 10,
  "sort": [
    {
      "visit_time": "desc"
    },
    {
      "patient_id": "asc"
    }
  ],
  "query": {
    "match": {
      "diagnosis": "糖尿病"
    }
  }
}
```

下一页使用上一页最后一条结果的 sort 值：

```json
GET /patient_index/_search
{
  "size": 10,
  "search_after": ["2026-06-17 09:30:00", "P10001"],
  "sort": [
    {
      "visit_time": "desc"
    },
    {
      "patient_id": "asc"
    }
  ],
  "query": {
    "match": {
      "diagnosis": "糖尿病"
    }
  }
}
```

## 15.4 PIT

PIT，Point In Time，用于创建一个查询时刻的快照视图，避免分页过程中数据变化导致重复或漏查。

生产中深分页推荐：

```text
PIT + search_after
```

面试中可以这样回答：

普通分页用 `from + size`。

深分页不要直接增大 `from`。

批量导出用 `scroll`。

实时深分页用 `search_after + PIT`。

聚合分页用 `composite aggregation`。

# 16. 批量写入 Bulk

ES 单条写入开销较大，生产中大量数据写入应该使用 Bulk API。

```json
POST /_bulk
{ "index": { "_index": "patient_index", "_id": "P10001" } }
{ "patient_id": "P10001", "name": "张三", "age": 56 }
{ "index": { "_index": "patient_index", "_id": "P10002" } }
{ "patient_id": "P10002", "name": "李四", "age": 63 }
```

Bulk 注意事项：

每批不要过大，也不要过小。常见做法是按文档数量或请求体大小控制，例如几千条或数 MB 到十几 MB 级别，需要压测确定。

使用稳定 `_id` 保证幂等写入。例如患者 ID、订单 ID、业务主键。

失败要逐条检查，因为 Bulk 可能部分成功、部分失败。

高吞吐写入时可以临时调大 `refresh_interval`。

大批量初始化导入时可以临时降低 replica 数，导入完成后恢复。

写入端要控制并发，避免把 ES 写爆。

# 17. ES 写入过程

ES 的写入不是“立刻可搜索”的强实时，而是近实时。

简化流程：

客户端写入文档。

请求根据路由规则进入目标主分片。

主分片写入内存 buffer。

同时写入 translog，保证故障恢复。

主分片复制到副本分片。

refresh 后生成新的 segment，文档变得可搜索。

flush 时将数据持久化并清理旧 translog。

后台 merge 合并小 segment。

几个关键概念：

refresh：让新写入数据对搜索可见。默认大约周期性 refresh。手动 `refresh=true` 会增加写入成本。

flush：将内存和 translog 状态持久化，减少恢复成本。

merge：合并 segment，清理删除标记，提高查询效率，但会消耗 IO 和 CPU。

delete：不是立即物理删除，而是先打删除标记，等 segment merge 时真正清理。

update：本质上是删除旧文档，再写入新文档。

这解释了为什么 ES 是近实时搜索引擎，而不是强事务数据库。

# 18. 分片与副本

创建索引时需要考虑主分片和副本数。

```json
PUT /patient_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

`number_of_shards: 3` 表示 3 个主分片。

`number_of_replicas: 1` 表示每个主分片有 1 个副本。

总分片数 = 主分片数 × (1 + 副本数)

也就是：

```text
3 × (1 + 1) = 6
```

分片设计的影响：

主分片数影响写入和数据分布。

副本数影响高可用和查询吞吐。

分片太少，扩展能力不足。

分片太多，集群元数据、文件句柄、内存、调度开销变大。

单个 shard 过大，迁移和恢复慢。

单个 shard 过小，管理开销高。

常见原则：

不要为小索引创建过多分片。

日志类索引可以结合 ILM 和 rollover 自动控制索引大小。

时间序列数据适合按日期或 rollover 拆分。

业务索引要结合数据量、增长速度、查询 QPS、节点数量设计。

# 19. 路由 Routing

ES 默认根据文档 `_id` 计算 hash，决定文档落到哪个 shard。

```text
shard = hash(_routing) % number_of_primary_shards
```

默认 `_routing` 是 `_id`。

可以自定义 routing，例如按患者 ID、租户 ID、医院 ID 路由：

```json
PUT /medical_record_index/_doc/1?routing=HOSPITAL_001
{
  "hospital_id": "HOSPITAL_001",
  "content": "患者患有高血压"
}
```

查询时也带 routing：

```json
GET /medical_record_index/_search?routing=HOSPITAL_001
{
  "query": {
    "match": {
      "content": "高血压"
    }
  }
}
```

好处是查询只打到相关 shard，减少分片扫描。

风险是数据倾斜。如果某个医院或租户数据特别多，会导致某些 shard 过热。

# 20. ES 数据建模

ES 不擅长 Join，所以建模思想和关系型数据库不同。

关系型数据库强调范式化，减少冗余。

ES 更强调查询效率，常使用反范式化、宽文档、冗余字段。

例如关系型数据库中可能有：

```text
patient 表
department 表
doctor 表
medical_record 表
```

ES 中可能把常用查询字段冗余到一个文档中：

```json
{
  "record_id": "R10001",
  "patient_id": "P10001",
  "patient_name": "张三",
  "patient_age": 56,
  "department_id": "D01",
  "department_name": "内分泌科",
  "doctor_id": "DR01",
  "doctor_name": "王医生",
  "diagnosis": "2型糖尿病，高血压",
  "record_content": "..."
}
```

这样搜索病历时不需要 Join。

常见数据模型：

扁平文档：最常用，性能好。

Object：对象字段，会被扁平化处理。

Nested：嵌套对象，适合数组对象内部需要保持关联关系的场景。

Parent-Child：父子关系，适合更新频率差异大、不适合冗余的场景，但性能成本较高，应谨慎使用。

Flattened：适合动态 key 的对象，例如日志 labels、扩展属性。

## Nested 的必要性

假设文档：

```json
{
  "patient": "张三",
  "diagnoses": [
    {
      "name": "高血压",
      "level": "重度"
    },
    {
      "name": "糖尿病",
      "level": "轻度"
    }
  ]
}
```

如果不用 nested，查询：

```text
name = 高血压 AND level = 轻度
```

可能错误命中，因为 ES 扁平化后会丢失数组对象内部关联。

使用 nested 可以保持每个对象的独立性。

# 21. ES 与数据库数据同步

生产中最常见架构是：

```text
业务系统 -> MySQL/Oracle -> Binlog/CDC/MQ -> Elasticsearch
```

常见同步方案：

业务代码双写：写数据库后再写 ES。简单，但容易出现部分失败、一致性问题。

异步 MQ：业务写数据库后发送消息，由消费者写 ES。吞吐好，但需要处理消息丢失、重复、乱序。

Canal：监听 MySQL binlog，同步到 ES。

Debezium：CDC 工具，支持多种数据库，常配合 Kafka。

Logstash JDBC input：定时查询数据库同步到 ES，适合简单场景。

定时任务全量/增量同步：简单但实时性差。

Outbox Pattern：业务库中写 outbox 表，再异步投递，保证本地事务内事件不丢。

同步要解决的问题：

新增、更新、删除如何同步。

失败重试。

重复消息幂等。

乱序消息处理。

数据库事务提交后再同步。

ES mapping 变更和重建索引。

全量重建期间如何不停机切换。

数据一致性校验。

典型可靠方案：

数据库作为主库。

监听 binlog 或 outbox 事件。

消息进入 MQ。

消费者批量写 ES。

使用业务主键作为 ES `_id`，保证幂等。

更新事件带版本号或更新时间，避免旧消息覆盖新消息。

定期做数据校验和补偿。

# 22. 索引别名 Alias 与零停机重建

ES 的 mapping 中很多字段类型不能直接修改。例如一个字段从 text 改成 keyword，通常需要创建新索引并 reindex。

生产中不要让业务直接访问物理索引名，而是访问 alias。

例如：

```text
patient_index_v1
patient_index_v2
patient_index_alias
```

业务查询：

```text
patient_index_alias
```

重建流程：

创建 `patient_index_v2`。

从数据库或旧索引重建数据到 v2。

校验数据。

将 alias 从 v1 切到 v2。

删除旧索引或保留回滚。

切换 alias 示例：

```json
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "patient_index_v1",
        "alias": "patient_index"
      }
    },
    {
      "add": {
        "index": "patient_index_v2",
        "alias": "patient_index"
      }
    }
  ]
}
```

这就是 ES 中常用的零停机索引重建方式。

# 23. 索引模板与 ILM

日志、监控、时间序列数据会不断产生新索引，不可能每次手动创建 mapping 和 settings。

Index Template 用来定义一类索引的模板：

```text
log-* 使用同一套 mapping/settings
```

ILM，Index Lifecycle Management，用于管理索引生命周期：

hot：热数据，频繁写入和查询。

warm：温数据，较少写入，仍可能查询。

cold：冷数据，低频查询。

frozen：冻结数据，极低频访问。

delete：过期删除。

日志场景常见策略：

当天日志写入 hot index。

索引达到一定大小或时间后 rollover。

7 天后进入 warm。

30 天后进入 cold。

90 天后删除。

这可以防止日志索引无限增长。

# 24. 高可用与集群状态

ES 集群状态常见三种：

green：所有主分片和副本分片正常。

yellow：主分片正常，但部分副本不可用。数据可用，但高可用不足。

red：部分主分片不可用，数据可能不可查或丢失风险高。

常见原因：

磁盘满。

节点宕机。

分片无法分配。

副本数大于可用节点数。

mapping 爆炸。

查询或写入压力过大。

JVM 堆内存不足。

集群高可用要考虑：

至少多个 master-eligible 节点。

主分片和副本不要分配在同一节点。

监控磁盘水位线。

控制 shard 数量。

定期快照备份。

使用安全认证和访问控制。

# 25. ES 性能优化

## 25.1 写入优化

使用 Bulk 批量写。

合理设置 refresh_interval。

初始化导入时可以临时设置 `number_of_replicas: 0`，导入完成后恢复。

避免频繁 update，update 本质是 delete + index。

使用稳定 `_id`，但注意自定义 `_id` 写入时可能需要检查旧文档，性能可能不如自动 ID。

控制字段数量，避免 mapping 爆炸。

不需要搜索的字段设置 `index: false`。

不需要保存原文的特殊字段可以考虑关闭存储，但 `_source` 通常建议保留，便于重建和排查。

## 25.2 查询优化

过滤条件放 filter，不要都放 must。

避免深分页。

避免前置 wildcard，例如 `*abc`。

避免对 text 字段聚合。

用 keyword 字段做排序和聚合。

使用 routing 缩小查询分片范围。

减少返回字段，使用 `_source` includes/excludes。

使用 `terminate_after` 或 timeout 控制极端查询。

用 profile 分析慢查询。

## 25.3 聚合优化

控制 terms aggregation 的 size。

高基数字段慎用 terms。

大数据分页聚合使用 composite aggregation。

避免在超大范围上做复杂多层嵌套聚合。

尽量使用 doc_values 字段。

## 25.4 mapping 优化

字段类型要准确。

能用 keyword 不用 text。

需要全文检索才用 text。

需要排序聚合的字符串字段加 `.keyword`。

动态字段过多时使用 flattened。

关闭不必要字段的索引。

日期、数值不要存成字符串。

## 25.5 集群优化

控制 shard 数量。

保证磁盘、CPU、内存、IO 充足。

JVM heap 不宜过大，常见经验是不要超过压缩对象指针阈值级别，生产中需结合版本和机器配置调优。

冷热数据分层。

定期 snapshot。

监控 slowlog、GC、thread pool、segment、merge、refresh、search latency、indexing latency。

# 26. ES 查询 DSL 总览

ES 使用 JSON DSL 查询。

常见查询可以分为：

全文查询：

```text
match
match_phrase
multi_match
query_string
simple_query_string
```

精确查询：

```text
term
terms
ids
range
exists
prefix
wildcard
regexp
```

组合查询：

```text
bool
dis_max
constant_score
function_score
boosting
```

结构化查询：

```text
nested
has_child
has_parent
```

地理查询：

```text
geo_distance
geo_bounding_box
geo_shape
```

特殊查询：

```text
script
more_like_this
percolate
rank_feature
knn / vector search
```

最常用的是：

```text
bool + match + term + range + filter + sort + aggs
```

掌握这些就能覆盖绝大多数业务搜索场景。

# 27. 排序 Sort

根据字段排序：

```json
GET /patient_index/_search
{
  "query": {
    "match": {
      "diagnosis": "糖尿病"
    }
  },
  "sort": [
    {
      "visit_time": {
        "order": "desc"
      }
    },
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

注意：

排序字段应使用 keyword、numeric、date。

不要直接对 text 排序。

排序会影响性能，尤其是跨多个分片排序。

搜索相关性排序默认按 `_score`。

业务排序常常是 `_score + 时间 + 热度 + 权重` 的组合。

# 28. 自动补全

自动补全常见方案：

completion suggester：性能好，适合专门的补全字段。

edge_ngram：适合前缀匹配和搜索框联想。

search_as_you_type：适合输入即搜索场景。

拼音补全：中文姓名、药品名、机构名常用。

例如边缘 ngram：

```json
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "my_edge_ngram_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "my_autocomplete_analyzer": {
          "tokenizer": "my_edge_ngram_tokenizer",
          "filter": ["lowercase"]
        }
      }
    }
  }
}
```

自动补全要注意控制索引膨胀。ngram 会生成大量 token，可能显著增加索引体积。

# 29. 同义词

同义词用于提升召回。

例如医疗场景：

```text
高血压, 血压高
糖尿病, DM
心肌梗死, 心梗
冠状动脉粥样硬化性心脏病, 冠心病
```

同义词可以在索引阶段扩展，也可以在搜索阶段扩展。

索引阶段扩展：

优点是查询快。

缺点是同义词变更后通常要重建索引。

搜索阶段扩展：

优点是同义词更新更灵活。

缺点是查询时成本更高。

实际生产中，医疗、法律、电商等专业领域往往需要维护领域词典和同义词库，否则搜索效果会明显不足。

# 30. 中文搜索优化

中文 ES 搜索常见问题：

分词不准。

专业词无法识别。

同义词缺失。

拼音搜索不支持。

短文本召回不足。

长文本噪音过多。

精确匹配和全文匹配混用。

常见优化方案：

使用 IK、jieba、自研词典或业务词典。

维护专业词词库，例如药品名、疾病名、手术名、科室名。

使用 synonym 做同义词扩展。

姓名、医院、药品支持拼音字段。

短字段用 keyword + text 多字段。

核心字段增加权重。

结合 function_score 引入业务热度、时间、点击率。

日志搜索和病历搜索应使用不同 mapping，不要一套 mapping 解决所有场景。

# 31. ES 安全

生产环境不能裸奔。

需要考虑：

用户认证。

角色权限控制。

索引级权限。

字段级权限。

文档级权限。

HTTPS/TLS。

审计日志。

敏感字段脱敏。

最小权限原则。

医疗系统尤其要注意隐私数据，例如患者姓名、身份证、手机号、诊断、病历内容等。ES 中的数据权限和脱敏不能弱于数据库。

# 32. Snapshot 备份与恢复

ES 应该定期做 snapshot。副本不是备份，副本只能防节点故障，不能防误删、逻辑错误、集群级故障。

Snapshot 可以备份到共享文件系统、对象存储等。

常见用途：

灾备。

误删除恢复。

跨集群迁移。

版本升级前备份。

测试环境复制数据。

# 33. ES 常见开发流程

一个典型搜索功能的开发流程是：

明确业务查询需求。

设计字段和 mapping。

确定哪些字段 text，哪些 keyword，哪些 numeric/date。

设计分词器、同义词、拼音字段。

创建 index template 或具体 index。

全量导入历史数据。

建立增量同步机制。

开发查询 DSL。

做相关性调优。

压测写入和查询性能。

增加监控、告警、慢查询分析。

上线后根据用户搜索行为继续优化。

不要一上来就写 DSL。ES 项目成败很大程度取决于 mapping 和数据建模设计。

# 34. 医疗病历搜索示例

假设要做一个病历检索系统，支持：

按患者姓名搜索。

按诊断搜索。

按病历正文搜索。

按科室过滤。

按就诊时间过滤。

高亮命中内容。

按相关性和时间排序。

可以设计索引：

```json
PUT /medical_record_v1
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "record_id": {
        "type": "keyword"
      },
      "patient_id": {
        "type": "keyword"
      },
      "patient_name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "department": {
        "type": "keyword"
      },
      "diagnosis": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "visit_time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||strict_date_optional_time"
      },
      "deleted": {
        "type": "boolean"
      }
    }
  }
}
```

查询：

```json
GET /medical_record_v1/_search
{
  "from": 0,
  "size": 10,
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "张三 糖尿病",
            "fields": [
              "patient_name^5",
              "diagnosis^3",
              "content"
            ]
          }
        }
      ],
      "filter": [
        {
          "term": {
            "department": "内分泌科"
          }
        },
        {
          "range": {
            "visit_time": {
              "gte": "2026-01-01 00:00:00",
              "lte": "2026-06-17 23:59:59"
            }
          }
        },
        {
          "term": {
            "deleted": false
          }
        }
      ]
    }
  },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "patient_name": {},
      "diagnosis": {},
      "content": {}
    }
  },
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    },
    {
      "visit_time": {
        "order": "desc"
      }
    }
  ]
}
```

这个例子基本覆盖了全文检索、权重排序、过滤、范围查询、高亮、分页。

# 35. Java 中使用 ES 的基本思路

Java 后端通常不会在业务代码里到处拼 JSON 字符串，而是封装搜索条件对象和查询构造器。

常见调用方式：

Elasticsearch Java API Client。

Spring Data Elasticsearch。

RestHighLevelClient，旧项目中常见，但新版本中逐渐被新 Java Client 替代。

伪代码思路：

```java
public SearchResult<MedicalRecordDTO> search(MedicalRecordSearchRequest request) {
    // 1. 构建 bool query
    // 2. 关键词走 multi_match
    // 3. 科室、删除状态走 filter
    // 4. 时间范围走 range
    // 5. 设置分页、排序、高亮
    // 6. 执行查询
    // 7. 解析 hits 和 highlight
    // 8. 返回业务 DTO
}
```

实际项目中要注意：

不要让前端直接传 ES DSL，除非是内部系统。

搜索条件要做白名单校验，防止昂贵查询。

分页 size 要限制。

查询超时要限制。

高亮字段要限制。

ES 异常要降级处理，必要时回退数据库基础查询。

# 36. ES 为什么不是关系型数据库替代品

这是重点。

ES 不适合替代 Oracle/MySQL 做强事务主库，原因包括：

ES 不擅长多表 Join。

ES 没有传统关系型数据库那样完整的强事务能力。

ES 更新是近实时可见，不是严格实时。

ES 的 update/delete 成本相对更高。

ES 更适合按查询模型设计宽文档，而不是按范式化模型存储。

ES 的一致性模型、refresh 机制、分片复制机制和数据库事务模型不同。

ES 更适合搜索、过滤、聚合、日志分析，不适合作为订单、支付、库存、账务、处方等核心业务的唯一事实来源。

正确架构应该是：

```text
MySQL / Oracle：事实数据源、强事务、核心业务状态
Elasticsearch：搜索副本、查询加速、聚合分析、日志检索
```

# 37. 常见面试题与回答要点

## 37.1 ES 为什么查询快？

核心是倒排索引。ES 会把文本分词后建立 term 到 doc 的映射，查询时可以直接根据词项定位文档，而不是全表扫描。同时 ES 基于 Lucene，有 segment、跳表、压缩、缓存、doc values、BKD Tree 等底层优化，并且通过分片并行查询提升吞吐。

## 37.2 倒排索引是什么？

倒排索引是“词 → 文档列表”的结构。例如“糖尿病”对应文档 1、文档 3、文档 10。搜索时先找词，再找包含词的文档。

## 37.3 text 和 keyword 的区别？

text 会分词，适合全文检索，用 match。

keyword 不分词，适合精确匹配、过滤、排序、聚合，用 term。

## 37.4 match 和 term 的区别？

match 会对查询文本分词，适合 text 字段。

term 不分词，直接精确匹配倒排索引中的 term，适合 keyword、numeric、boolean 等字段。

## 37.5 must 和 filter 的区别？

must 必须匹配，并参与相关性评分。

filter 必须匹配，但不参与评分，适合结构化过滤，性能通常更好，也更容易缓存。

## 37.6 ES 深分页为什么慢？

因为 from + size 深分页时，每个分片都要取大量候选结果，再由协调节点排序合并。比如 `from=100000,size=10`，不是只取 10 条，而是要处理前面大量数据，内存和 CPU 压力很大。

解决方案：

浅分页用 from + size。

深分页用 search_after + PIT。

批量导出用 scroll。

聚合分页用 composite aggregation。

## 37.7 ES 如何保证数据一致性？

ES 不适合作为强一致主库。通常数据库是主数据源，ES 是搜索副本。通过 binlog、MQ、CDC、定时校验等方式保证最终一致。写入 ES 时用业务主键作为 `_id` 保证幂等，并通过版本号或更新时间避免乱序覆盖。

## 37.8 ES 写入后为什么马上查不到？

因为 ES 是近实时搜索。文档写入后要等 refresh 生成可搜索 segment，才会被搜索到。可以手动 refresh，但会增加性能开销，不建议高频使用。

## 37.9 ES update 的底层原理是什么？

ES 文档所在的 Lucene segment 是不可变的。update 本质上是标记旧文档删除，再写入新文档。真正物理删除要等 segment merge。

## 37.10 ES 如何做高可用？

通过主分片和副本分片实现。主分片负责写入，副本分片提供冗余和查询能力。节点故障后，副本可以提升为主分片。还要合理配置 master 节点、分片分配、磁盘水位线、快照备份和监控告警。

# 38. 学习 ES 的推荐顺序

第一阶段，掌握基础概念：

```text
index
document
mapping
field
shard
replica
cluster
node
倒排索引
分词器
text / keyword
```

第二阶段，掌握常用查询：

```text
match
term
range
bool
filter
sort
highlight
aggregation
pagination
```

第三阶段，掌握中文搜索：

```text
IK 分词
自定义词典
同义词
拼音搜索
ngram
相关性调优
```

第四阶段，掌握工程实践：

```text
bulk
数据库同步
索引别名
reindex
scroll
search_after
PIT
mapping 设计
数据建模
```

第五阶段，掌握集群和性能：

```text
shard 规划
replica
refresh
flush
merge
segment
缓存
慢查询
ILM
snapshot
监控告警
```

第六阶段，掌握高级能力：

```text
nested
parent-child
function_score
script_score
completion suggester
地理位置查询
向量检索
混合检索
冷热分层
跨集群搜索
```

# 39. 最核心总结

ES 的本质是基于 Lucene 的分布式搜索引擎。

倒排索引是 ES 快速全文检索的核心。

分词器决定文本如何被索引和搜索，中文场景尤其重要。

mapping 决定字段类型和搜索方式，设计错了后期经常要重建索引。

text 用于全文检索，keyword 用于精确匹配、过滤、排序、聚合。

match 会分词，term 不分词。

bool 查询是业务搜索的核心组合方式。

filter 不参与评分，适合结构化过滤。

BM25 是默认相关性算法，但真实业务通常要结合字段权重和 function_score。

聚合适合实时统计分析，但要注意高基数字段和内存消耗。

深分页不能简单用 from + size，应该使用 search_after + PIT 或 scroll。

Bulk 是高吞吐写入的基础。

ES 是近实时系统，不是强事务数据库。

生产架构中，MySQL/Oracle 作为主库，ES 作为搜索和分析副本。

ES 最重要的工程能力不是会写几个查询语句，而是会设计 mapping、分词、索引结构、同步链路、分页方案、相关性排序和集群容量。