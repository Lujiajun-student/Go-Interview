# 1. ES概念

Elasticsearch是面向文档的。

| 数据库   | Elasticsearch |
| -------- | ------------- |
| database | Index         |
| tables   | types         |
| rows     | documents     |
| columns  | fields        |

ES的一切都是JSON。

ES可以包含多个索引，每个索引包含多个类型，每个类型包含多个文档，每个文档包含多个字段。

而es会将每个索引划分为多个分片，每个分片可以在集群的不同服务器之间迁移。

每个索引默认会有5个分片和，而每个分片会有一个副本。而一个分片实际上就是一个Lucene索引。

## 1. 倒排索引

倒排索引能够用来检索全文。它会将文档进行分词，创建包含所有词的列表，列出每个词条出现在哪个文档。这样，检索某个词的时候不需要遍历所有文档，只需要遍历词列表，即可知道它在哪些文档出现过。

这样假如检索两个词，匹配到两个文档，其中一个文档命中了这两个词，另一个文档只命中一个，那么第一个文档的权重就会更高，因为更匹配。

而倒排索引也会包含其他信息。

Term Dictionary：词典，包含所有词。

Term Index：词典索引，用来快速定位词。

Posting List：倒排列表，记录某个词出现在那些文档中。

Position：记录词在某个文档中的位置。

Offset：词在原文中的偏移，用来进行高亮显示。

Term Frequency：词频，用于相关性评分。

## 2. 分词器

ES在存储和搜索的时候都会涉及分词。分词器决定文本怎么被拆分。

通常有三部分。

Char Filter。字符过滤器，对文档进行处理，去除无关的字符。

Tokenizer。分词器。将文本切分成token。

Token Filter。过滤器，对token加工，如小写化、同义词扩展等。

常用的分词器有很多种。

* standard。标准分词器，英文效果好。
* keyword。不分词，整段内容作为一个词。
* ik_smart。ik中文分词。
* ik_max_word。细粒度模式，尽可能地进行分词。
* pinyin analyzer。拼音分词，适合姓名的拼音搜索。
* ngram。适合前缀搜索。

分词器有两个阶段，如index analyzer，写入文档怎么分词。还有search analyzer，查询时怎么分词。

写入的时候可以用`ik_max_word`提高召回率，查询时用`ik_smart`提高精度。

查看分词效果可以用`_analyze`。

```json
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "患者患有高血压和糖尿病"
}
```

## 3. Index

Index是一类文档的集合。如订单集合、商品集合、用户集合等。

## 4. Document

Document是JSON类型的文档。如

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

根据key-value的格式保存业务数据。

## 5. Mapping

Mapping定义字段类型和索引方式。

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

这里指定了每个字段的类型。但Mapping除了显式指定，还可以动态应射。索引新文档的时候，es会自动检测字段类型并创建映射。

其中，name虽然是text类型，但其中还有keyword字段，检索的时候能够通过检索name来实现模糊检索，或者通过name.keyword来实现精确检索。

## 6. text

text会被分词，用于全文搜索。text类型在被索引时，会经过分词器处理，将长文本拆成token，根据token构建倒排索引。使用`match`搜索的时候，也会将text类型的字段分词，根据分好的token在倒排索引中匹配，得到相关的文档。

索引的时候，需要为text类型的字段配置分词器。`analyzer`是索引时的分词器，`search_analyzer`是检索时的分词器。

## 7. keyword

keyword不会分词，而是作为一个整体保存并检索。

在索引时，keyword类型的字段会作为整个Term存入倒排索引。如果使用term匹配，es就会根据检索内容来检索完整的Term。

索引的时候，有多个参数可以控制。

`ignore_above`用于置顶最大长度，如果字段值超过长度，那么这个字段就无法被索引。

index。控制是否将该字段存入倒排索引。如果只存不查，就不需要索引。

## 8. 精确查询

精确查询不分析文本，而是直接匹配倒排索引的term。

1. term查单个精确值。

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

2. terms查多个匹配值。

```json
{
  "query": {
    "terms": {
      "department": ["内分泌科", "心内科"]
    }
  }
}
```

3. ids根据id来查询。

```json
{
  "query": {
    "ids": {
      "values": ["1", "2", "3"]
    }
  }
}
```

4. exists判断字段是否存在。

```json
{
  "query": {
    "exists": {
      "field": "phone"
    }
  }
}
```

5. prefix前缀查询。

```json
{
  "query": {
    "prefix": {
      "phone": "138"
    }
  }
}
```

6. wildcard通配符查询。

```json
{
  "query": {
    "wildcard": {
      "name.keyword": "张*"
    }
  }
}
```

> `*`表示匹配任意多字符，`?`表示匹配单个任意字符。
>
> 其中prefix查询前缀`*abc`和单字符`*a`时，需要匹配大量term，性能非常差。但查询中间通配符`a*b`时性能好一点，但不多，而index_prefixes能够最大程度地保证性能。

index_prefixes用于辅助通配符查询。文档进入索引时，可以选择开启index_prefixes，例如词条`apple`，那么会为词条生成额外的索引词条。如`ap`, `app`, `appl`等，如`application`也会生成`ap`, `app`, `appl`, `appli`等索引。这样，在通配符检索的时候，可以直接查找到适合的词条，不需要全盘检索。

## 9. 全文查询和模糊查询

全文查询是在搜索时对输入文本进行分词。模糊查询是搜索时不分词，但允许词条有错误。

全文查询在分词后，对每个分词进行查询，返回查找到的文档。

1. match。

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

这里分词会分为`高血压`和`糖尿病`两个词条，根据这两个词条向倒排列表检索。

2. match_phrase

要求顺序和距离一致。

```json
{
  "query": {
    "match_phrase": {
      "content": "高血压 糖尿病"
    }
  }
}
```

这里要求高血压必须在糖尿病前，并且两者必须相邻。

添加slop能够允许中间出现距离。

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

这里高血压必须在糖尿病前，但两者之间可以存在0-2个词的存在。

3. multi-match

同时搜索多个字段。

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

这里可能检索到姓名包含高血压，content有张三的文档。如果要在检索的时候不同字段给定不同的检索条件，需要使用Bool查询。

4. 模糊查询。

可以使用fuzzy容忍错误。

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

fuzzy的原理是计算最大编辑距离。

也就是从一个词条变成另一个词条，至少需要多少次插入、替换、删除的操作。如张三变为张三，编辑距离为1，张三变为李四，编辑距离为2。

但上面的是关于英文的编辑距离逻辑。中文的能力极差。检索章三的时候，分词器会拆分为“章”和“三”两个词，分别对这两个字计算编辑距离，然后匹配倒排索引的“张三”，而倒排索引也可能将“张三“分词为”张“和”三“。这样会导致大量文档被召回，是因为匹配了”张“和”三“两个字，如“张飞三顾茅庐”等，而不仅仅是“张三”和原本的“章三”。

如果要实现中文模糊查询的话，需要使用拼音分词器。这样能够召回与检索内容拼音相同的所有文档。

或者使用match+fuzziness，和上面一样将“章”和“三”分开进行模糊匹配，召回大量文档。

## 10. 范围查询

范围查询就是通过大于小于来进行查找。

| 参数 | 含义     |
| ---- | -------- |
| gte  | 大于等于 |
| gt   | 大于     |
| lte  | 小于等于 |
| lt   | 小于     |

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

能够对数值、日期等进行范围查询。

## 11. Bool查询

Bool查询是最重要的组合查询。

主要包含下面四个部分。

* must：必须匹配，参与评分。
* filter：必须匹配，不参与评分。
* should：可以匹配，参与评分。
* must_not：必须不匹配。

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "multi_match": {            // 全文搜索，需要算分
            "query": "iPhone",
            "fields": ["name", "description"]
        }}
      ],
      "filter": [                     // 硬性过滤，不参与算分，会被缓存
        { "range": { "price": { "gte": 3000, "lte": 8000 } } },
        { "term": { "brand.keyword": "Apple" } }
      ],
      "should": [                     // 软性加分项
        { "match": { "name": "Pro" } },
        { "match": { "name": "Max" } }
      ],
      "must_not": [
        { "term": { "status.keyword": "deleted" } }
      ]
    }
  }
}
```

其中，must每次查询的时候需要重新计算评分，性能较差。而filter由于不需要评分，并且会缓存，所以一般的字段都尽可能用filter来做过滤。

使用filter时，es会遍历倒排索引，找到相关文档后组成位图，将位图放到缓冲区。这样下次相同的检索能够直接查询位图来获取文档。

must由于需要计算评分，所以找到词条相关的文档后，需要计算每个文档对于这个词的词频、逆向文档频率等。索引发生更新，这些都要重新计算，所以must每次检索都需要从倒排索引中计算，性能开销大。

其中，逆向文档频率IDF用于衡量词的罕见程度。如果一个词在越少的文档中出现，IDF越高，权重越大。

## 12. 搜索相关性和权重排序

es使用BM25来计算相关性。词在文档中出现频率越高，相关性越高。词月稀有，区分度越高。字段越短，命中词越集中，相关性越高。在实际业务中，由于字段之间的优先级不同，所以需要添加字段权重来优化权重。

```json
{
  "multi_match": {
    "query": "张三 高血压",
    "fields": ["name^5", "diagnosis^3", "content"]
  }
}
```

> 名字权重提高5倍，疾病权重提高3倍。

或者也可以使用should来提高权重，在should中使用boost来设置权重系数。

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

## 13. 聚合查询

主要有三大类。

1. Metric聚合，对字段值进行数学计算。
2. Bucket聚合，将文档按条件分组。
3. Pipeline聚合，对其他聚合结果进行二次计算。

### 1. Metric聚合

返回数值统计结果，不关心文档分类。

有avg, sum, min, max, value_count, stats等。stats能够直接返回min, max, sum, count, avg的数据。

```json
GET /products/_search
{
  "size": 0,  // 只要聚合结果，不要具体的文档列表
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "max_price": { "max": { "field": "price" } },
    "total_quantity": { "sum": { "field": "quantity" } }
  }
}
```

> 统计商品价格的平均值、最大值和总数。

### 2. Bucket聚合

将文档分组，每组是一个桶。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": {
        "field": "brand.keyword",  // 必须作用在 keyword 字段上
        "size": 10,                // 返回销量最高的前 10 个品牌
        "order": { "_count": "desc" }  // 按文档数降序排列
      }
    }
  }
}
// 返回结果：华为: 120件, 苹果: 100件, 小米: 80件...
```

或者按range聚合。

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "低价", "to": 100 },
          { "key": "中价", "from": 100, "to": 500 },
          { "key": "高价", "from": 500 }
        ]
      }
    }
  }
}
```

### 3. Pipeline聚合

不对文档进行计算，而是对前面聚合的结果进行二次处理。

derivative计算倒数，分析趋势。

moving_age计算移动平均值，用于平滑数据曲线。

bucket_sort对聚合桶进行排序或截取。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "orders_per_month": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total_orders": { "value_count": { "field": "order_id" } },
        "monthly_growth": {
          "derivative": { "buckets_path": "total_orders" }  // 计算环比差值
        }
      }
    }
  }
}
// 返回：1月 100单, 2月 120单(+20), 3月 150单(+30)...
```

在这里使用aggs来做查询，而query会先查询，后筛选，aggs的聚合只作用于筛选后的文档。

## 14. 高亮查询

高亮用于在搜索结果中凸显命中的词。

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

返回的结果如下。

```json
"highlight": {
  "content": [
    "患者患有<em>糖尿病</em>多年"
  ]
}
```

## 15. 分页查询

通过from+size能够指定获取某页的数据。

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

> 这里from为0表示从第一个数据开始，size为10表示取10个数据。第二页就是from为10，size还是为10。

深分页的性能较差。查询的时候，需要在每个分片中筛选大量结果，获取所有结果后进行排序，取到from表示的数据。

### 2. Scroll

Scroll是为大量数据的一次性遍历设计的。第一次请求的时候，es会生成快照，返回scroll_id。后续请求携带这个scroll_id，不断获取下一批数据。设置了超时时间时，如果时间内没有发生滚动，就会释放这个快照。

```json
# 第一次请求（初始化游标）
GET /products/_search?scroll=1m   // 游标存活 1 分钟
{
  "size": 1000,
  "query": { "match_all": {} }
}

# 后续请求（滚动游标）
GET /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gB..."
}
```

### 3. search_after

搜索后，每次请求时，需要携带上一页最后文档的排序值。通过sort字段，指明按照什么字段排序，是升序还是降序，指定后首次请求后，就会返回当前sort的值。下次请求携带这个sort值即可。

```json
GET /patient_index/_search
{
  "size": 10,
  "sort": [ // 设置sort，需要指定按什么字段怎么排序
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

这里通过`search_after`指定了从指定时间和指定id开始下一页。

### 4. PIT

PIT会创建查询的快照，能够避免数据更新的时候导致查询数据重复或丢失。通常搭配search_after使用。

```json
# 第一步：创建 PIT
POST /products/_pit?keep_alive=1m

# 第二步：第一页查询（带上 pit_id）
GET /_search
{
  "size": 10,
  "query": { "match": { "name": "手机" } },
  "pit": {
    "id": "46ToAwMD...", 
    "keep_alive": "1m"
  },
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }   // tiebreaker 确保唯一性
  ]
}

# 第三步：第二页查询（使用 search_after）
GET /_search
{
  "size": 10,
  "query": { "match": { "name": "手机" } },
  "pit": { "id": "46ToAwMD...", "keep_alive": "1m" },
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }
  ],
  "search_after": [99, "product_123"]   // 上一页最后一条的排序值
}
```

### 5. Scroll和search_after+PIT的快照区别

Scroll在请求时，会在内存中生成完整的快照，分配scroll_id来作为指针，后续的滚动都是从快照中获取一定量的数据返回。

search_after+PIT在创建PIT时，es不会复制任何数据，知识在底层索引中添加时间戳来进行排序，search_after会拿着上一页的排序值，查找下一条数据。

创建PIT时，会记录当前索引有哪些分段，以及分段的内部信息。后续的search_after会根据这里的信息来查找数据，因此新数据对PIT是不可见的，不会出现下一页时数据重复出现或丢失的情况。

## 16. Bulk写入

es单条写入开销很大，主要原因是每次都需要发送HTTP请求来检索，检索完毕后也需要请求来返回数据，网络延迟高。并且请求需要分配线程来处理，线程切换开销大。如果单跳写入，es会在内存中生成分段，刷到磁盘。频繁的小分段会导致磁盘随机写多，而批量能够解决这个问题。

通过Bulk批量写能够将大量数据打包成一个请求，减少大量的网络开销，线程开销也减少，并且es将文档存储到分段时，会将数据攒到一起，一次性写入磁盘。随机写变成顺序写，速度快。

```json
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "手机", "price": 5000 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "电脑", "price": 8000 }
{ "delete": { "_index": "products", "_id": "3" } }
{ "update": { "_index": "products", "_id": "1" } }
{ "doc": { "price": 4500 } }
```

> 这里能够看到，Bulk将数据和操作分开了。第一行需要告诉es需要进行什么操作，第二行再给出数据。如第一行的字段是index，就说明需要添加文档，第二行写文档即可。如果是delete，第二行可以省略。如果是update，第二行写需要更新的字段即可。

## 17. 写入过程

es写入后不是立刻可搜索。

1. 客户端写入文档。
2. 请求根据路由规则进入目标分片。
3. 主分片写入内存Buffer。
4. 写入translog，保证故障恢复。
5. 主分片复制到副本分片。
6. refresh后生成新的segment，文档可搜索。
7. flush将segment持久化，清理translog。
8. 后台将小segment进行合并。

## 18. 查询流程

首先协调节点会将查询请求广播到索引的所有分片。

每个分片独立查询，返回本地的Top N个结果给协调节点。

协调节点手机所有分片结果，进行全局排序，取最终的Top N返回给客户端。

## 19. 分片和副本

创建索引的时候，需要考虑分片和副本数。

```json
PUT /patient_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

分片是用来解决存储能力的。不同分片可以存储到不同的服务器上。

副本用来解决高可用问题。如果主分片丢失了，或者主分片压力大，那么可以读取副本来查询数据。

shards表示当前索引有多少个主分片，replicas表示每个主分片有多少个副本。其中，主分片是数据的读写入口。索引文档时，首先写入主分片，再由主分片同步给副本。

## 20. 路由

es会根据文档的`_id`来计算hash，决定文档落到哪个分片。

在索引的时候可以自定义routing。

```json
PUT /medical_record_index/_doc/1?routing=HOSPITAL_001
{
  "hospital_id": "HOSPITAL_001",
  "content": "患者患有高血压"
}
```

查询时，也可以带上routing。

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

这样，关于这个`HOSPITAL_001`的文档就会保存到特定的一个分片，能够减少检索开销，但可能导致某个分片被频繁使用，导致负载不均衡。

## 21. 数据建模

es有多种组织文档字段的方式。

1. 扁平文档。将所有相关数据放到同一层对象中，不使用任何嵌套或者父子关系。

```json
{
  "product_id": "P001",
  "name": "iPhone 15",
  "brand": "Apple",
  "price": 5999,
  "seller": {
    "seller_id": "S01",
    "name": "Apple 旗舰店"
  }
}
```

> 这里将经销商名也作为商品的文档字段来保存。

好处是查询快，但有数据冗余。如果某个子对象发生修改，需要更新所有文档。

2. 嵌套建模。如果文档里有对象数组，并且需要独立查询这个对象数组时，就需要使用嵌套建模。

```json
{
  "name": "T恤",
  "specs": [
    { "size": "L", "color": "红", "stock": 10 },
    { "size": "S", "color": "蓝", "stock": 5 }
  ]
}
```

在Mapping中必须显式声明。

```json
"specs": {
  "type": "nested",   // 必须显式声明
  "properties": {
    "size": { "type": "keyword" },
    "color": { "type": "keyword" },
    "stock": { "type": "integer" }
  }
}
```

查询时必须使用nested查询。

```json
{
  "query": {
    "nested": {
      "path": "specs",
      "query": {
        "bool": {
          "must": [ 
            { "term": { "specs.size": "L" } },
            { "term": { "specs.color": "红" } }
          ]
        }
      }
    }
  }
}
```

但nested对象会被索引为独立文档，读取时需要再进行查询，性能差。

3. 父子建模。

如果文档里有大量子文档，那么需要使用join数据类型。

```json
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { "type": "text" },
      "my_join": {
        "type": "join",
        "relations": {
          "parent": "child"   // 定义父子关系
        }
      }
    }
  }
}
```

```json
{ "id": "U1", "name": "张三", "my_join": "parent" }
```

```json
{ "id": "O1", "total": 100, "my_join": { "name": "child", "parent": "U1" } }
```

上面就定义了父子关系。

查询的时候，能够通过子查父，父查子来检索文档。

```json
// 查询下单金额 > 50 的用户
{
  "query": {
    "has_child": {
      "type": "child",
      "query": { "range": { "total": { "gt": 50 } } },
      "score_mode": "sum"
    }
  }
}
```

4. 反范式建模。

与其将数据拆分成多个索引，不如将子索引的数据嵌入父文档。例如在订单文档中存储用户名，这样在订单服务中想要展示数据时，不需要再根据订单文档的用户id在用户文档再次查询了。但缺点是如果用户名改了，那么所有订单文档都需要修改。

## 22. ES与数据库同步

通常的架构如下。

```json
业务系统-MySQL-MQ-ES
```

首先将数据写入到数据库，再通过MQ发送到ES写入。

## 23. 索引别名

es的mapping中很多字段的类型无法更改。如果需要更改，需要创建新的索引，使用新的mapping。因此，业务应该访问es的alias别名，这样的话在使用时能够用同一个别名，但底层通过修改别名的指向来指向不同的索引。

流程如下。

1. 一开始有`patient_index_v1`的索引，服务访问该索引下的分区。
2. 接下来需要创建`patient_index_v2`新索引，并配置新的mapping。
3. 需要检验数据，是否满足新的mapping，不满足需要转换类型。
4. 将alias使用原子操作从v1切换到v2。
5. 删除旧索引。

这里的原子操作是将删除旧索引和添加新索引放到同一个请求。

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add": { "index": "products_v2", "alias": "products" } }
  ]
}
```

> 这里的原子操作能够实现零停机重建，因为不需要停止服务，即可更换服务使用的索引。

如果别名指向多个索引，写入就会拒绝。但读操作可以在多个索引上同时查询，也可以设置权重。

如下，创建6月索引并设置别名。

```json
PUT /logs_2025_06
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs_2025_06", "alias": "logs_current" } },
    { "add": { "index": "logs_2025_06", "alias": "logs_query" } }
  ]
}
```

到了7月后，需要更换别名。

```json
PUT /logs_2025_07
POST /_aliases
{
  "actions": [
    { "remove": { "index": "logs_2025_06", "alias": "logs_current" } },
    { "add": { "index": "logs_2025_07", "alias": "logs_current" } },
    { "add": { "index": "logs_2025_07", "alias": "logs_query" } }
  ]
}
```

其中，没有删除掉6月的`logs_query`的别名，这样查询的时候，既能写入`logs_2025_6`的索引，又能写入`logs_2025_7`的索引。

## 24. 性能优化

### 1.  写入优化

首先可以使用Bulk批量写。

合理设置`refresh_interval`，太小导致缓冲区数据被刷成大量的小段，产生大量小文件。

`translog.durability`默认是request，每次写入都会写translog到磁盘。改为async后，会异步刷盘。

使用`_id`生成。如果业务不需要id，可以让es自动生成`_id`，这样比用户指定`_id`性能更高。

对不需要搜索的字段，设置`index: false`。

### 2. 查询优化

尽可能使用filter，避免must的运算和无缓存。

避免深分页，深分页需要使用PIT+search_after。

在keyword上聚合和排序。

使用routing减少分片查询范围。

用`_profile: true`分析慢查询。能够显示每个组件耗费的时间。

```json
{
  "profile": {
    "shards": [
      {
        "id": "[node_name][index][0]",
        "searches": [
          {
            "query": [
              {
                "type": "TermQuery",        // 查询类型
                "description": "status:active",
                "time": "2.5ms",            // 该查询总耗时
                "breakdown": {              // 微观时间分解（核心）
                  "create_weight": 12345,   // 构建查询权重对象（纳秒）
                  "build_scorer": 23456,    // 构建打分器
                  "next_doc": 34567,        // 遍历文档
                  "score": 45678,           // 计算相关性得分
                  "match": 56789,           // 匹配文档
                  "advance": 67890          // 跳转文档
                },
                "children": [ ... ]         // 嵌套查询（如 bool 下的子句）
              }
            ],
            "rewrite_time": 10000,          // 重写阶段耗时（纳秒）
            "collector": [ ... ]            // 收集器耗时
          }
        ]
      }
    ]
  }
}
```

能够看到查询类型、查询好事，还有每一步查询的耗时。

### 3. 聚合优化

1. 不使用text聚合，只对keyword进行聚合。如果对text聚合，就会将整个倒排索引加载到内存，导致内存不足。如果要将text聚合，就需要配置keyword字段。

2. 如果某个keyword不需要被聚合或排序，可以将`doc_values`设为false。

```json
"my_field": {
  "type": "keyword",
  "doc_values": false
}
```

3. 如果某个字段是需要频繁聚合的，可以在Mapping中用`eager_global_ordinals`预加载，将构建过程冲查询时变为写入或Refresh时。

```json
"brand": {
  "type": "keyword",
  "eager_global_ordinals": true
}
```

这样首次聚合查询会很快，但增加了写入延迟和refresh开销。

4. 如果深度分页，可以使用`composite`实现。terms不支持from翻页，但可以使用`after`来指定上一页的最后一个值来翻页。

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "brands": {
      "composite": {
        "size": 100,
        "after": { "brand": "Apple" },  // 上一页最后一个值
        "sources": [
          { "brand": { "terms": { "field": "brand.keyword" } } }
        ]
      }
    }
  }
}
```

### 4. mapping优化

字段类型需要填写准确。

能使用keyword就不用text。

只有全文索引才使用text。

需要排序和聚合的字段如果是text，需要添加`.keyword`字段。

关闭不必要字段的索引。

## 25. 排序

如果查询时不指定sort，就会根据相关性得分来降序排列。但可以指定sort来优先按指定字段排序，然后再按相关性得分排序。

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "price": "asc" },
    { "sales": "desc" }
  ]
}
```

排序字段应该使用keyword、numeric、date，不能对text排序。

排序会影响性能。

## 26. 自动补全

在搜索的时候，能够实时展示完整搜索词。es主要有三种方案。

### 1. completion suggester

字段定义为completion，es索引时会构建有限状态转移器FST保存到内存中。检索时输入前缀，直接在FST中查找，速度快。

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "suggest": {
            "type": "completion"   // 专门用于补全的字段
          }
        }
      }
    }
  }
}
```

查找的时候，使用suggest来查找，指定prefix前缀和completion的字段，就能返回符合前缀的多个结果。

```json
POST /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "iphon",          // 用户输入的前缀
      "completion": {
        "field": "name.suggest",
        "size": 10,               // 返回前 10 个建议
        "skip_duplicates": true
      }
    }
  }
}
```

这种方法性能高。

### 2. search_as_you_type

适合输入即搜索。需要在mapping指定type为`search_as_you_type`。在索引时，会为字段生成所有长度n-gram前缀。查询时，用户输入前面部分单词，es会直接匹配n-gram，返回完整短语。

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "search_as_you_type"   // 专用类型
      }
    }
  }
}
```

```json
POST /products/_search
{
  "query": {
    "multi_match": {
      "query": "苹果手",               // 用户输入
      "type": "bool_prefix",          // 前缀匹配模式
      "fields": ["name", "name._2gram", "name._3gram"]  // 自动生成的 n-gram 字段
    }
  }
}
```

### 3. prefix查询

针对keyword，可以使用prefix查询。但它是遍历倒排索引的词条列表，数据量大时性能差。

```json
GET /products/_search
{
  "query": {
    "prefix": {
      "name.keyword": "iphon"
    }
  }
}
```

### 4. edge_ngram

适合前缀匹配和搜索框联想。

`edge_ngram`是分词器，在索引时会对字段值处理。

假如检索的字段值是`iphone`，配置了`min_gram=2`， `max_gram=6`，会生成下面的词条。

```
"ip", "iph", "iphon", "iphone"
```

这些词条也都会存到倒排索引，用户搜索时会匹配到ip词条，找到包含iphone的文档。

## 27. 同义词

用户检索“手机”的同时，也会希望检索“电话”的商品。

同义词能够解决这个问题，处理发生在查询时或索引时。

在索引时，分词器将同义词展开写入到倒排索引。

查询时，搜索词被同义词扩展后再匹配。

同义词可能会发生变化，如果一个词的同义词修改了，那么必须Reindex整个索引。最好是在查询时使用同义词，不需要重建索引。

## 28. es流程

1. 明确业务需求。
2. 设计字段和mapping。
3. 确定哪些字段text，哪些字段keyword，哪些字段numeric。
4. 设计分词器、同义词、拼音字段。
5. 创建index。
6. 导入历史数据。
7. 建立增量同步机制。
8. 开发查询DSL。
9. 调优。
10. 压测写入和查询性能。
11. 监控、告警、慢查询分析。

## 29. Merge

es的数据是通过生成Lucene分段来写入。如每次refresh都会生成新的小分段，每次update、delete或者新的写入，都会产生新的分段或删除标记。

如果不merge，搜索就需要遍历所有分段，因为每个分段都是单独的倒排索引。分段越多，扫描文件越多。并且每个分段都相当于一个文件，要占用句柄，如果太多会导致操作系统报错。

Merge的作用就是将小文件合并成大文件，清理掉逻辑删除的文档。

### 1. 流程

1. 选择参与合并的分段。Merge会根据分段的大小选择多个分段来合并，通常大小相近。
2. 复制并重写数据。将选中的分段所有有效文档复制到新的分段中，带有删除标记的文档不会被复制。
3. 交换分段。新分段写入磁盘后，会原子性地将新分段添加到索引，移除旧分段。

## 30. 语义查询

语义查询需要使用`semantic_text`来实现。

一开始需要指定使用那个模型来生成向量。比如创建一个elser推理模型。

```json
PUT _inference/sparse_embedding/elser-endpoint
{
  "service": "elser",
  "service_settings": {
    "num_threads": 1,
    "adaptive_allocations": {"enabled": true}
  }
}
```

在mapping时，需要将语义搜索的字段类型设置为`semantic_text`，并关联到上面的推理模型。

```json
PUT my-semantic-index
{
  "mappings": {
    "properties": {
      "my_content": {
        "type": "text",
        "copy_to": "my_content_semantic" 
      },
      "my_content_semantic": {
        "type": "semantic_text", 
        "inference_id": "elser-endpoint" 
      }
    }
  }
}
```

在查询的时候，使用semantic查询即可。

```json
GET my-semantic-index/_search
{
  "query": {
    "semantic": {
      "field": "my_content_semantic", 
      "query": "你希望搜索的语义内容"
    }
  }
}
```

## 面试

### 1. es为什么查询快

核心是倒排索引。将文本分词后建立词条到文档的映射，查询时直接根据词条来定位文档，不需要扫描全部文档。同时通过分片并行查询来提高吞吐量。

### 2. 倒排索引是什么

普通的查询是根据搜索内容，在全部文档中查找。而倒排索引构建了从词条到文档的映射，在检索的时候只需要查看倒排索引，找到相关词条即可得知有哪些文档相关。

### 3. text和keyword的区别

text会分词，适合全文检索。使用match来匹配。

keyword不分词，适合精确匹配、过滤、排序和聚合，用term来检索。

### 4. match和term的区别

match会对查询文本分词，用分出来的每个词条对倒排索引进行匹配。适合text。

term不分词，直接匹配倒排索引的词典，适合keyword、numeric、boolean等字段。

### 5. must和filter的区别

must必须匹配，参与相关性评分。

filter必须匹配，但不参与评分，没有评分是静态检索，可以缓存，下次直接使用，性能更好。

### 6. es深分页为什么慢

from+size深分页时，每个分片都要选取大量的结果，再由协调节点排序合并。导致每个分片都要取出深分页的数量，性能很低。

深分页需要使用search_after+PIT，批量导出用scroll，聚合分页使用composite。

### 7. es怎么保证数据一致性

es不保证强一致性，保存数据用关系型数据库，es是一个用于搜索的副本。数据首先在关系型数据库中保存，通过MQ等方法保证最终一致性。

### 8. 为什么es写入后可能查不到

es写入后，首先会保存到当前分片的缓冲区，每隔1秒会执行一次refresh，将内存缓冲区的数据生成新的分段，这样才可检索。然后隔一段时间，会执行flush，将段持久化。

### 9. es的update原理是什么

es文档的段是不可变的。一旦写入磁盘，就不会修改。不能在已有的倒排索引中插入、删除、修改词条，只能追加新的分段。

更新的流程如下。

1. 读取旧文档。

通过`_id`获取完整文档。如果不存在就报错。

2. 合并更新。

将传入的文档和旧文档合并，生成新的文档。

3. 将旧文档所在的段添加删除标记，表示逻辑删除。
4. 将生成的新文档写入当前索引的内存缓冲区。这个新文档就会分配一个新的版本号，等待后续的refresh和flush。

### 10. 如何高可用

通过主分片和副本实现。主分片负责写入，写入后同步到副本，副本提供冗余和查询能力。如果主分片故障，那么副本会提升为主分片。

### 11. 如何实现集群的主节点选举

主节点用于管理集群状态、处理创建删除索引的请求、管理节点的加入和离开等。

es通过Raft来选举主节点。任何一个有资格的节点都可以发起选举。

在集群启动时，或者主节点故障时，都会触发新的选举。

当一个借点认为当前集群没有主节点时，就会发起选举，向其他节点拉票。

如果要称为主节点，必须获取一半以上的票数，能够防止脑裂。投票会按照版本和节点id来投票，更高版本号，id更小的优先。

### 12. 索引过程

首先客户端需要写入数据，发出请求。协调节点接收请求后，根据文档的id确定分片，请求转发到负责这个分片的节点。这个节点执行写操作，成功的话将请求转发到副本分片，等待副本写入。如果副本写入成功，就会向协调节点报告成功。

其中，获取分片的过程是使用路由算法。针对文档id计算哈希值，再对分片数取模即可。

### 13. 检索过程

加入有多个分片，那么每个分片都会在本地执行查询，返回结果放到本地，将这些结果都发送到协调节点，协调节点进行排序，获取部分数据来返回到客户端。