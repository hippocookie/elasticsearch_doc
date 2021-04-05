# ElasticSearch

## 数据录入和查询
#### 更新不存在的文档报错，使用upsert
**upsert**
```json
POST /website/pageviews/1/_update
{
    "script" : "ctx._source.views+=1",
    "upsert": {
        "views": 1
    }
}
```
#### 多线程更新冲突时，可配置使用重试
**retry_on_conflict**
```json
POST /website/pageviews/1/_update?retry_on_conflict=5
{
    "script" : "ctx._source.views+=1",
    "upsert": {
        "views": 0
    }
}
```
#### 查询多个文档
多个查询可合并为一个查询，减少多个请求耗时
```json
GET / _mget 
{
	"docs": [{
			"_index": "website",
			"_type": "blog",
			"_id": 2
		},
		{
			"_index": "website",
			"_type": "pageviews",
			"_id": 1,
			"_source": "views"
		}
	]
}
```
返回体中，多个请求结果分别存放在一个json对象中
```json
{
	"docs": [{
			"_index": "website",
			"_id": "2",
			"_type": "blog",
			"found": true,
			"_source": {
				"text": "This is a piece of cake...",
				"title": "My first external blog entry"
			},
			"_version": 10
		},
		{
			"_index": "website",
			"_id": "1",
			"_type": "pageviews",
			"found": true,
			"_version": 2,
			"_source": {
				"views": 2
			}
		}
	]
}
```
如对应请求未查询到数据，相应的返回体标识未查询到，第二个请求未查询到结果不会影响第一个查询，两个请求结果分开存放
```json
{
	"docs": [{
			"_index": "website",
			"_type": "blog",
			"_id": "2",
			"_version": 10,
			"found": true,
			"_source": {
				"title": "My first external blog entry",
				"text": "This is a piece of cake..."
			}
		},
		{
			"_index": "website",
			"_type": "blog",
			"_id": "1",
			"found": false
		}
	]
}
```
#### 高效的批处理
批处理允许一次执行多个相同的操作

- 每行数据已换行符(\n)作为结尾，包括最后一行也许已换行符结尾
- 每行数据中不能包含未转义的换行符，即pretty-printed的json格式
```json
{ action: { metadata }}\n
{ request body }\n
{ action: { metadata }}\n
{ request body }\n
```

##### Action
**create**
当文档不存在时创建文档，否则抛出异常
**index**
创建新的文档，或替换已存在的文档
**update**
更新一个文档的部分字段
**delete**
删除一个文档

##### 请求中Metadata
需包含文档的_index, _type, _id
```json
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }}
```

返回体中包含各请求处理结果
```json
{
	"took": 4,
	"errors": false,
	"items": [{
			"delete": {
				"_index": "website",
				"_type": "blog",
				"_id": "123",
				"_version": 2,
				"status": 200,
				"found": true
			}
		},
		{
			"create": {
				"_index": "website",
				"_type": "blog",
				"_id": "123",
				"_version": 3,
				"status": 201
			}
		},
		{
			"create": {
				"_index": "website",
				"_type": "blog",
				"_id": "EiwfApScQiiy7TIKFxRCTw",
				"_version": 1,
				"status": 201
			}
		},
		{
			"update": {
				"_index": "website",
				"_type": "blog",
				"_id": "123",
				"_version": 4,
				"status": 200
			}
		}
	]
}
}
```
各子请求相互独立，其中一个失败不会影响其他子请求执行
```json
{
	"took": 3,
	"errors": true,
	"items": [{
			"create": {
				"_index": "website",
				"_type": "blog",
				"_id": "123",
				"status": 409,
				"error": "DocumentAlreadyExistsException [[website][4][blog][123]:document already exists]"
			}
		},
		{
			"index": {
				"_index": "website",
				"_type": "blog",
				"_id": "123",
				"_version": 5,
				"status": 200
			}
		}
	]
}
```

#### 批处理非原子性
各子请求独立执行，不支持事务性，一个子请求失败不会影响其他子请求执行

#### 每批请求大小
批处理文档需全部加载到节点内存中，单批请求大小和硬件、文档数、复杂度、索引和查询负载等相关，一般每批介于1000至5000个文档，大小介于5至15MB，如果单个文档比较大，可以减小文档数。

### 分布式文档存储
#### 文档路由至节点
**路由策略**
*shard = hash(routing) % number_of_primary_shards*

**执行create，index，delete操作**
> 请求至Master Node -> Primary Shard Node -> Replica Node
当主分片和副本执行成功后，返回成功

可配置策略修改这一执行方式

#### *replication*
default=sync: Primary Shard会等待replica shareds更新成功后返回
async: Primary Shard成功后即刻返回，仍会发送请求至replica，但无法知道是否成功，可能会导致ES请求堆积，不建议使用

#### *consistency*
默认情况下，需要满足法定分片数量(quorum)可用，才可以写入，需满足：
*int( (primary + number_of_replicas) / 2 ) + 1*

#### *timeout*
当可用shard数量不足时，ES将会等待，默认为1min，可进行配置， e.g. 100(ms), 30s

### 查询文档
文档可以从Primary或Replica shard中获取
> 请求至Master Node -> 路由请求至对应Primary/Replica Shard

在索引文档过程中，可能存在Primary/Replica节点暂时不一致，Primary存在文档但Replica还不存在，当索引请求成功返回后，Primary/Replica节点恢复一致。

### 更新部分文档
> 请求至Master Node -> 路由请求至对应Primary Shard -> 尝试更新对应文档，如存在冲突重试retry_on_conflict次 -> 发送至Replica Shard节点更新
> Primary Shard发送更新时，发送整个文档而非只有更新字段，发送过程为异步，可能存在乱序情况导致Replica节点文档破坏

### 多文档操作
mget与bulk API与单个请求操作流程类似，不同的是其根据shard对请求进行拆分，发送请求至对应节点处理，当接收到各节点返回后，合并为一个返回结果

## 搜索
### 不带条件搜索
- 返回体中默认按相关评分_score排序，max_score为搜索结果中最相关数据的得分
- took: 表示查询耗时
- shards: 表示查询对应分片是否有失败，如存在失败，导致对应分片数据查询不到
- timed_out: 查询是否超时，默认情况下不超时，如响应实现比完整结果更重要，可在查询时进行设定(GET /_search?timeout=10ms)

```json
GET /_search
{
	"hits": {
		"total": 14,
		"hits": [{
				"_index": "us",
				"_type": "tweet",
				"_id": "7",
				"_score": 1,
				"_source": {
					"date": "2014-09-17",
					"name": "John Smith",
					"tweet": "The Query DSL is really powerful and flexible",
					"user_id": 2
				}
			},
			...9 RESULTS REMOVED...
		],
		"max_score": 1
	},
	"took": 4,
	"_shards": {
		"failed": 0,
		"successful": 10,
		"total": 10
	},
	"timed_out": false
}
```

### 多索引、类型查询
查询中没有带有_index, _type条件，ES回搜索全部索引和类型，并行发送请求至Primary/Replica Shards，合并结果后返回top 10

查询时可指定索引
```
/_search
Search all types in all indices

/gb/_search
Search all types in the gb index

/gb,us/_search
Search all types in the gb and us indices

/g*,u*/_search
Search all types in any indices beginning with g or beginning with u

/gb/user/_search
Search type user in the gb index

/gb,us/user,tweet/_search
Search types user and tweet in the gb and us indices

/_all/user,tweet/_search
Search types user and tweet in all indices
```
### 分页
- size: Indicates the number of results that should be returned, defaults to 10
- from: Indicates the number of initial results that should be skipped, defaults to 0

如分页靠后，需查询各分片中数据，然后汇总排序后进行返回，确保顺序正确。例如，查询1至10条数据，获取每个分片top 10，返回至接受请求的节点，将50条数据排序后返回top 10。在分布式系统中分页查询代价随翻页成倍增加。对于任何查询，搜索引擎返回结果最好不超过1000。

### Search Lite
- query-string: 参数通过查询请求字符串传入
> GET /_all/tweet/_search?q=tweet:elasticsearch
> 
> +name:john +tweet:mary
> +:必须满足，-:必须不满足
> GET /_search?q=%2Bname%3Ajohn+%2Btweet%3Amary

- request-body: 参数通过查询Json结构体传入，使用查询DSL

### _all字段
查询包含mary字段
> GET /_search?q=mary

文档内容如下:
```json
{
    "tweet": "However did I manage before Elasticsearch?",
    "date": "2014-09-14",
    "name": "Mary Jones",
    "user_id": 1
}
```
如另添加一个_all字段，除非特别声明搜索字段，否则会使用这个_all字段进行搜索
```json
{
    "_all": "However did I manage before Elasticsearch? 2014-09-14 Mary Jones 1"
}
```

#### 更复杂的查询
- The name field contains mary or john
- The date is greater than 2014-09-10
- The _all field contains either of the words aggregations or geo
> +name:(mary john) +date:>2014-09-10 +(aggregations geo)
>
> ?q=%2Bname%3A(mary+john)+%2Bdate%3A%3E2014-09-10+%2B(aggregations+geo)

## Mapping和Analysis
### 精确值和全文匹配
- Exact values: 精确值匹配需完全一致对应，Foo与foo，2014与2014-09-15被认为是不同的值
- Full text: 一般是语言相关

### 倒排索引
- inverted index: 倒排索引包含单词与对应出现的多个文档映射
例如有如下两个文档
> 1. The quick brown fox jumped over the lazy dog
> 2. Quick brown foxes leap over lazy dogs in summer

|Term| Doc_1 |Doc_2|
|----|----|----|
|Quick | | X|
|The | X |
|brown | X | X|
|dog | X ||
|dogs | | X|
|fox | X ||
|foxes | | X|
|in | | X|
|jumped | X ||
|lazy | X | X|
|leap | | X|
|over | X | X|
|quick | X ||
|summer | | X|
|the | X ||
|----|----|----|

但该索引存在如下问题
- Quick and quick appear as separate terms, while the user probably thinks of
them as the same word.
- fox and foxes are pretty similar, as are dog and dogs; They share the same root
word.
- jumped and leap, while not from the same root word, are similar in meaning.
They are synonyms.

可进行归一化normalize处理，查询请求参数和索引字段都需进行归一化处理
- Quick can be lowercased to become quick.
- foxes can be stemmed--reduced to its root form—to become fox. Similarly, dogs
could be stemmed to dog.
- jumped and leap are synonyms and can be indexed as just the single term jump.

### Analysis和Analyzers
分析过程进行如下操作:
- 将一段文本拆分为适合进行倒排索引的单个短语
- 然后将短语进行归一化，变为更易搜索的字段

这个过程通过分词器来完成，分词器中一般包含三个过程
- Character filters: 首先将文本中特殊字符进行除去和转换，如去除HTML标签，或将&转换为and
- Tokenizer: 将文本拆分为单个短语，简单的分词器一般使用空格或标点符号进行拆分
- Token filters: 拆分后的短语进行归一化，如转为小写、去除a，and，the，或转换同义词

### 内置分词器
- Standard analyzer: 这个是ES默认使用的分词器，其根据词边界进行划分，去除标点符号，转换为小写
- Simple analyzer: 根据非单词字符进行分词，然后转换为小写
- Whitespace analyzer: 根据空格进行分词，不会进行归一化转换为小写
- Language analyzers: 不用语言有对应分词器

#### 何时使用分词器
当索引一个文档时，其对应文本字段通过分词器分析后建立倒排索引，当查询时，查询参数同样经过分词器后再进行查询，确保查询参数与索引字段一致

- Full text: 当查询full-text字段时，查询参数会通过相同分词器
- Exact value: 当使用精确匹配时，查询参数不会经过分词器
因此，当索引2014-09-15字段值时，date类型为精确匹配2014-09-15，_all字段进行分词后变为2014，09，15

### 测试分词器
可以通过调用analyze API来查看文档分词和存储过程
```json
GET /_analyze?analyzer=standard
Text to analyze
{
	"tokens": [{
			"token": "text",
			"start_offset": 0,
			"end_offset": 4,
			"type": "<ALPHANUM>",
			"position": 1
		},
		{
			"token": "to",
			"start_offset": 5,
			"end_offset": 7,
			"type": "<ALPHANUM>",
			"position": 2
		},
		{
			"token": "analyze",
			"start_offset": 8,
			"end_offset": 15,
			"type": "<ALPHANUM>",
			"position": 3
		}
	]
}
```
- token: 表示真实会存储在索引中的字段
- position: 表示该字段在文本中出现的位置
- start_offset, end_offset: 该字段在文本中字符位置
- ALPHANUM: 不用分词器含义不同，可忽略，唯一用处为keep_types token filter

#### 指定分词器
当ES检测到文档某个字段为string类型时，自动配置其为全文检索string，并使用standard analyzer对齐进行分词。

当不需要使用该默认特性时，可以使用mapping指定对应字段类型和分词器。

### Mapping
index中每个type均可设定自己的Mapping(Schema Definition)，Mapping定义文档字段的类型，以及ES如何处理对应字段。

#### 字段类型
- 字符串: string
- 整数: byte, short, integer, long
- 浮点数: float, double
- 布尔值: boolean
- 日期: date

当新添加一个未定义的字段时，ES会使用dynamic mapping尝试猜测对应字段类型，规则如下:
> JSON type -> Field type
> Boolean: true or false -> boolean
> Whole number: 123 -> long
> Floating point: 123.45 -> double
> String, valid date: 2014-09-15 -> date
> String: foo bar -> string
>
> "123"会被当做字符串，而非long类型，如已经定义Mapping为long，ES会先尝试转换该字段，如果失败抛出异常

#### 查看已定义Mapping
查看索引下多个类型Mapping，可使用/_mapping后缀
```json 
GET /gb/_mapping/tweet
{
	"gb": {
		"mappings": {
			"tweet": {
				"properties": {
					"date": {
						"type": "date",
						"format": "dateOptionalTime"
					},
					"name": {
						"type": "string"
					},
					"tweet": {
						"type": "string"
					},
					"user_id": {
						"type": "long"
					}
				}
			}
		}
	}
}
```

#### 配置字段Mapping
**not string***
定义非string类型字段时，一般只需字段类型type
```json
{
	"number_of_clicks": {
		"type": "integer"
	}
}
```
可设置index类型

**string**
string类型在建立索引前会使用分词器进行分析，同样查询条件在进行检索前，也需经过分词器

string类型两个最重要的配置是index和analyzer
*index*
- analyzed(default): 对字符串先进行分析再索引
- not_analyzed: 对字符串添加索引，可以进行搜索，但不进行分词，精确匹配
- no: 不索引该字段，无法进行搜索

```json
{
    "tag": {
        "type": "string",
        "index": "not_analyzed"
    }
}
```

#### analyzer
默认使用standard分词器，可以声明使用其他内置分词器(whitespace, simple, english)
```json
{
    "tweet": {
        "type": "string",
        "analyzer": "english"
    }
}
```

#### 更新Mapping
可以使用/_mapping后缀增加新的Mapping字段，但无法修改已存在的Mapping类型

### 复杂核心类型
#### 数组
属性可包含多个数值，数组没有要求对应的Mapping类型，可包含0至多个数值，与全文本类型一样，可以进行分词
- 数组中的数值必须是相同的类型，当创建新的数组字段时，ES会使用数组中的第一个元素类型决定这个字段的类型
- 在查询数组字段时，数组中元素返回顺序与索引字段时顺序一致
- 在搜索时，数组中的元素是无序的，无法使用first/last方式获取元素
```json
{ "tag": [ "search", "nosql" ]}
```

#### 空值
Lucene中无法设置null值，因此null值都会被认为是空来处理
```json
{
    "null_value": null,
    "empty_array": [],
    "array_with_null_value": [ null ]
}
```

#### 内部对象
文档中可包含Inner Object
```json
{
	"tweet": "Elasticsearch is very flexible",
	"user": {
		"id": "@johnsmith",
		"gender": "male",
		"age": 26,
		"name": {
			"full": "John Smith",
			"first": "John",
			"last": "Smith"
		}
	}
}
```
ES会自动检测内部对象Mapping类型
```json
{
	"gb": {
		"tweet": {
			"properties": {
				"tweet": {
					"type": "string"
				},
				"user": {
					"type": "object",
					"properties": {
						"id": {
							"type": "string"
						},
						"gender": {
							"type": "string"
						},
						"age": {
							"type": "long"
						},
						"name": {
							"type": "object",
							"properties": {
								"full": {
									"type": "string"
								},
								"first": {
									"type": "string"
								},
								"last": {
									"type": "string"
								}
							}
						}
					}
				}
			}
		}
	}
}
```

#### 内部对象是如何被索引的
Lucene不支持Inner Object类型，在索引时会进行转换
```json
{
    "tweet": [elasticsearch, flexible, very],
    "user.id": [@johnsmith],
    "user.gender": [male],
    "user.age": [26],
    "user.name.full": [john, smith],
    "user.name.first": [john],
    "user.name.last": [smith]
}
```
#### 数组内部对象
当属性值为数组对象时
```json
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
```
文档会扁平化转换为以下格式
```json
{
	"followers.age": [19, 26, 35],
	"followers.name": [alex, jones, lisa, smith, mary, white]
}
```

## 全文检索(Full-Body Search)
### 空搜索
ES认为GET更好的描述了所进行的操作，因此使用GET加请求体方式，但不是所有HTTP请求都支持GET请求体，ES也提供了POST对应的查询方式
```json
GET /_search
{}

GET /_search
{
	"from": 30,
	"size": 10
}

POST /_search
{
	"from": 30,
	"size": 10
}
```

### Query DSL
```json
GET /_search
{
	"query": {
		"match_all": {}
	}
}

GET /_search
{
	"query": {
		"match": {
			"tweet": "elasticsearch"
		}
	}
}
```

#### 组合多个查询语句
查询语句有以下类型:
- Leaf clauses(match): 用于比较字段
- Compound clauses: 用于组合其他查询字段，与bool从句衔接的只能是must, must_not, should
```json
{
	"bool": {
		"must": { "match": { "tweet": "elasticsearch" }},
		"must_not": { "match": { "name": "mary" }},
		"should": { "match": { "tweet": "full text" }}
	}
}
```
Compound clauses还可进行嵌套，复合成更复杂的查询条件
```json
{
	"bool": {
		"must": { "match": { "email": "business opportunity" }},
		"should": [
			{ "match": { "starred": true }},
			{ "bool": {
				"must": { "folder": "inbox" }},
				"must_not": { "spam": true }}
		],
		"minimum_should_match": 1
	}
}
```

### 查询与过滤
ES DSL语句分为两种，query DSL和filter DSL，这两种本质相似，但用途不同
- filter: yes|no 问题查询，以及某个字段是否包含对应精确值
- query: 用于查询文档匹配查询条件程度

#### 查询性能
使用filter查询结果可以将每个文档转换为1bit大小的缓存，在下次查询时进行复用，而query不仅需查找符合请求条件的文档，还需进行相关性计算，比filter更消耗资源，且不能缓存

缓存的filter性能高于query，其目的是减少query查询的文档数

#### 何时使用query/filter
一个基本的原则，使用query做全文检索查询可能影响文档相关性评分的字段，其他则使用filter

### 常用query/filter
#### term Filter
term用于查询精确匹配值，字段为numbers，dates，booleans或者设置为not_analyzed的精确字符串
```json
{ "term": { "age": 26 }}
{ "term": { "date": "2014-09-01" }}
{ "term": { "public": true }}
{ "term": { "tag": "full_text" }}
```
#### terms Filter
terms与term相同，可以声明多个字段用于匹配，如果文档包含任意查询的值即为匹配
```json
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

#### range Filter
用于查询number或date在特定区间的文档, gt/gte/lt/lte
```json
{
	"range": {
		"age": {
			"gte": 20,
			"lt": 30
		}
	}
}
```

#### exists and missing Filters
与SQL中IS_NULL(missing)，NOT IS_NULL(exists)相似，用于查询字段中是否包含或不包含特定值
```json
{
	"exists": {
		"field": "title"
	}
}
```

#### bool Filter
bool为逻辑操作符，用于合并多个bool逻辑，must/must_not/should
```json
{
	"bool": {
		"must": { "term": { "folder": "inbox" }},
		"must_not": { "term": { "tag": "spam" }},
		"should": [
			{ "term": { "starred": true }},
			{ "term": { "unread": true }}
		]
	}
}
```

#### match_all Query
经常用于与filter结合查询，用于查询所有数据，所有文档的相关性都认为是相同的_score=1
```json
{ "match_all": {}}
```

#### match Query
match是进行全文检索的标准查询，基本查询所有字段都可以使用，当使用match时，查询条件会使用对应的分词器进行解析后再执行搜索，当用于查询精确匹配时，则不会使用分词器
```json
{ "match": { "tweet": "About Search" }}
{ "match": { "age": 26 }}
{ "match": { "date": "2014-09-01" }}
{ "match": { "public": true }}
{ "match": { "tag": "full_text" }}
```

#### multi_match Query
multi_match是对多个字段使用match查询
```json
{
	"multi_match": {
		"query": "full text search",
		"fields": [ "title", "body" ]
	}
}
```

#### bool Query
像bool Filter一样，组合多个bool Query，不用的是filter验证yes|no，query则验证各从句相关性得分_score，must/must_not/should
```json
{
	"bool": {
		"must": { "match": { "title": "how to make millions" }},
		"must_not": { "match": { "tag": "spam" }},
		"should": [
			{ "match": { "tag": "starred" }},
			{ "range": { "date": { "gte": "2014-01-01" }}}
		]
	}
}
```

### Query与Filter组合查询
#### Filtering a Query
```json
GET /_search
{
	"query": {
		"filtered": {
			"query": { "match": { "email": "business opportunity" }},
			"filter": { "term": { "folder": "inbox" }}
		}
	}
}
```

#### Just a Filter
```json
GET /_search
{
	"query": {
		"filtered": {
			"filter": { "term": { "folder": "inbox" }}
		}
	}
}

GET /_search
{
	"query": {
		"filtered": {
			"query": { "match_all": {}},
			"filter": { "term": { "folder": "inbox" }}
		}
	}
}
```

#### A Query as a Filter
当在使用filter语句时，也可以嵌套query从句进行查询
```json
{
	"query": {
		"filtered": {
			"filter": {
				"bool": {
					"must": { "term": { "folder": "inbox" }},
					"must_not": {
						"query": {
							"match": { "email": "urgent business proposal" }
						}
					}
				}
			}
		}
	}
}
```
### 校验query
可以使用validate-query API对查询条件进行校验
```json
GET /gb/tweet/_validate/query
{
	"query": {
		"tweet" : {
			"match" : "really powerful"
		}
	}
}

response
{
	"valid" : false,
	"_shards" : {
		"total" : 1,
		"successful" : 1,
		"failed" : 0
	}
}
```
可以使用explain查看query校验失败的原因
```json
GET /gb/tweet/_validate/query?explain
{
	"query": {
		"tweet" : {
			"match" : "really powerful"
		}
	}
}

response
{
	"valid" : false,
	"_shards" : { ... },
	"explanations" : [ {
	"index" : "gb",
	"valid" : false,
	"error" : "org.elasticsearch.index.query.QueryParsingException:
	[gb] No query registered for [tweet]"
	} ]
}
```
#### 理解query
当查询条件合法时，explain会返回以索引为粒度查询条件对应的解析结果
```json
{
	"valid" : true,
	"_shards" : { ... },
	"explanations" : [ {
		"index" : "us",
		"valid" : true,
		"explanation" : "tweet:really tweet:powerful"
	}, {
		"index" : "gb",
		"valid" : true,
		"explanation" : "tweet:realli tweet:power"
	} ]
}
```

## 排序和相关性
### 排序
查询结果中_score字段表示结果与查询条件的相关性，为浮点数类型，默认查询排序结果为按_score降序排序

当使用filter和match_all query时，所有文档相关性_score=1

### 字段排序
```json
GET /_search
{
	"query" : {
	"filtered" : {
			"filter" : { "term" : { "user_id" : 1 }}
		}
	},
	"sort": { "date": { "order": "desc" }}
}

response
"hits" : {
	"total" : 6,
	"max_score" : null,
	"hits" : [ {
		"_index" : "us",
		"_type" : "tweet",
		"_id" : "14",
		"_score" : null,
		"_source" : {
		"date": "2014-09-24",
		...
	},
	"sort" : [ 1411516800000 ]
	},
	...
}
```
返回体中每个文档包含一个新字段*sort*，值为用于排序的字段值，并且_score和max_score字段值均为null，多数情况下这个两个值仅用来排序，而计算这两个值比较耗资源，因此如果不是按照相关性来排序的话，就不需计算这个值

如果仍想计算_score值得话，可传递参数*track_scores=true*

使用简写的话，默认使用升序排序，_score使用降序排序
```json
"sort": "number_of_children"
```

#### 多层排序
多个排序条件时，会先按第一个字段排序，然后第二个
```json
GET /_search
{
	"query" : {
		"filtered" : {
			"query": { "match": { "tweet": "manage text search" }},
			"filter" : { "term" : { "user_id" : 2 }}
		}
	},
	"sort": [
		{ "date": { "order": "desc" }},
		{ "_score": { "order": "desc" }}
	]
}
```

#### 数组字段排序
当字段为数组时，无法直接进行排序，需先将数组转换为可排序的值，可使用*mode:min/max/avg/sum*等排序模式
```json
"sort": {
	"dates": {
		"order": "asc",
		"mode": "min"
	}
}
```

### 字符串和多字段排序
进行分词查询的字段无法进行排序，要使用string进行排序只能对整个字符串字段排序(keyword/not_analyzed)。可以通过传递*fields*配置，对同一个字段使用两种不同的索引。
```json
"tweet": {
	"type": "string",
	"analyzer": "english",
	"fields": {
		"raw": {
			"type": "string",
			"index": "not_analyzed"
		}
	}
}
```
- tweet字段使用分词器进行分词
- tweet.raw字段不进行分词

查询时使用tweet.raw进行排序
```json
GET /_search
{
	"query": {
		"match": {
			"tweet": "elasticsearch"
		}
	},
	"sort": "tweet.raw"
}
```

### 什么是相关性
搜索结果中使用_score字段表示每个文档的相关性，ES中用于计算相关性的算法有:
*Term frequency*
查询的短语在文档中出现的次数越多，相关性越高

*Inverse document frequency*
一个词出现在文档的数量越多，这个词的相关性权重就越小

*Field-length norm*
一个词出现在较短的字段里，权重大于出现在长字段里

不仅是分词字段，像bool字段一样可以影响_score相关性得分

#### 理解相关性
可以通过explain查看搜索结果
```json
GET /_search?explain
GET /_search?explain&format=yaml
{
"query" : { "match" : { "tweet" : "honeymoon" }}
}


"_explanation": {
	"description": "weight(tweet:honeymoon in 0) [PerFieldSimilarity], result of:",
	"value": 0.076713204,
	"details": [
		{
			"description": "fieldWeight in 0, product of:",
			"value": 0.076713204,
			"details": [
				{
					"description": "tf(freq=1.0), with freq of:",
					"value": 1,
					"details": [
					{
						"description": "termFreq=1.0",
						"value": 1
					}
					]
				},
				{
				"description": "idf(docFreq=1, maxDocs=1)",
				"value": 0.30685282
				},
				{
				"description": "fieldNorm(doc=0)",
				"value": 0.25,
				}
			]
		}
	]
}
```

#### 理解文档如何匹配
使用/index/type/id/_explain API可以查看文档没有匹配的原因
```json
GET /us/tweet/12/_explain
{
	"query" : {
		"filtered" : {
			"filter" : { "term" : { "user_id" : 2 }},
			"query" : { "match" : { "tweet" : "honeymoon" }}
		}
	}
}
```

### fielddata
Inverted index是一个理想的搜索数据结构，但不善于排序，当需要排序的时候，ES需要加载所有文档对应字段至内存中进行排序，这部分数据成为fielddata。

排序时，ES不仅加载匹配查询条件的文档字段，会加载索引下全部文档的字段，不论是文档属于什么类型。这个过程会消耗很多内存，特别是具有很多值string字段。

Fielddata在ES中的使用场景:
- 对一个字段进行排序
- 对一个字段进行聚合
- 特定的filter，e.g. geolocation filters
- Scripts引用的字段


## 分布式搜索
一个查询请求会查询每个shard查看是否有匹配的文档，这只是第一步，查询后需要将多个shard的结果进行聚合，这种两阶段执行称作*query-then-fetch*

### Query阶段
查询阶段会广播查询请求至一个shard(primary or replica)，每个shard在本地执行搜索请求，然后使用匹配的文档构建一个priority queue。

*priority queue*是一个有序的top-n匹配文档列表，其大小取决于分页条件from/size
```json
GET /_search
{
	"from": 90,
	"size": 10
}
```

当一个节点接受到请求后，这个节点为*coordinating node*，负责向其他节点发送查询请求，并聚合各节点查询结果。

各分片将搜索的结果在本地*priority queue*进行排序，返回from + size的数据，仅有ID和_score字段。

*coordinating node*随后会将各分片结果进行汇聚得出全局有序结果。

### Fetch阶段
*coordinating node*将需要获取的文档对应shard发送GET请求，各shard加载文档对应字段(_source)并返回结果，最后由*coordinatin node*返回。

#### 深度翻页
from + size的翻页方式具有局限性，每个shard都需创建一个大小为from + size的priority queue，*coordinating node*需要排序number_of_shards * (from + size)数量的文档以便找到对应的结果。

如需要查询大量文档，可以使用scan查询类型。

### 查询选项
#### preference
可用于控制查询请求访问的shards，选项有: _primary, _primary_first, _local, _only_node:xyz, _prefer_node:xyz, _shards:2,3

最常用的值是任意字符串，以避免出现bouncing results问题:
在搜索时是使用一个字段进行排序，例如timestamp，当两个文档的timestamp具有相同的值时，因为搜索请求时轮询方式发送值各可用shard的，这两个节点在请求不同shard的时候可能以不同的顺序返回。可以设置preference值为任意一字符串例如用户的session ID。

#### timeout
通过设置timeout可以告诉coordinating节点应该等待多久进行返回，当时间到时，返回当前已获取到的结果，避免应为个别节点响应延迟而导致整个查询变慢。

查询结果中会显示有多少个节点成功返回了请求
```json
...
"timed_out": true,
"_shards": {
	"total": 5,
	"successful": 4,
	"failed": 1
},
...
```

#### routing
可以在创建文档索引时，指定routing字段，来让相同文档分配到同一个shard中，在查询中可以使用routing来限制查询数据范围在对应的shards中
> GET /_search?routing=user_1,user2

#### search_type
> GET /_search?search_type=count
- count: 只有query阶段
- query_and_fetch: 默认配置，一般不需指明，当有条件限制请求到一个shard的时候(routing)，会被优化为一步查询
- dfs_query_then_fetch/dfs_query_and_fetch: dfs类型的查询会有个*prequery*阶段，加载会将所有相关节点的词频，用于计算一个全局词频
- scan: 与scroll API配合，用于读取大量的文档，会禁用排序选项

#### scan and scroll
scan scroll的方式用于获取大量文档，并且没有深度翻页效率问题。
*scroll*
类似于关系数据库的cursor，可以初始化一个查询并持续获取一批文档，直至所有匹配的文档已被获取完。当开始scroll的时候，ES会创建一个快照，确保在查询的过程中数据不会变化。

*scan*
深度翻页最大的开销是需要一个全局有序的结果集，如果我们不使用排序的话，可以很轻松的获取到对应文档，scan操作不使用排序，仅从各shard中获取文档。

可传递scan参数开启，并传递scroll参数配置scroll保持时间
```json
GET /old_index/_search?search_type=scan&scroll=1m
{
	"query": { "match_all": {}},
	"size": 1000
}
```

结果中会返回Base-64编码的_scroll_id，用于进行翻页操作，当再次调用传递scroll=1m的时候，对应过期时间延长1m，每次调用需将上次返回的_scroll_id传入。

**每次查询可能得到的文档数=size * number_of_primary_shards**
```
GET /_search/scroll?scroll=1m
c2Nhbjs1OzExODpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExOTpRNV9aY1VyUVM4U0
NMd2pjWlJ3YWlBOzExNjpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExNzpRNV9aY1Vy
UVM4U0NMd2pjWlJ3YWlBOzEyMDpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzE7dG90YW
xfaGl0czoxOw==
```

## 索引管理
### 创建索引
当传入索引文档时，ES可以使用dynamic mapping自动创建索引字段配置，同时我们也可以自己传入索引配置进行创建。
```json
PUT /my_index
{
	"settings": { ... any settings ... },
	"mappings": {
		"type_one": { ... any mappings ... },
		"type_two": { ... any mappings ... },
		...
	}
}
```

可以在config/elasticsearch.yml文件中加入如下配置项，关闭dynamic mapping
> action.auto_create_index: false

### 删除索引
可通过如下方式删除索引:
> DELETE /my_index
> DELETE /index_one,index_two
> DELETE /index_*
> DELETE /_all

### 索引配置
*number_of_shards*
一个索引所包含primary shards的数量，默认值为5，索引创建后不可修改。

*number_of_replicas*
每个primary shard对应replica shard的数量，默认值为1，索引创建后可以修改。

### 配置分词器
#### Character filters
一个分词器可以有0到多个字符过滤器，在进行分词前，对字段进行处理，例如，使用html_strip去除字段中HTML标签

#### Tokenizers
一个分词器只有一个tokenizer，将字符串拆分为多个独立的词组

#### Token filters
对分词后的词组进行转换、添加或删除，例如lowercase、stop token filters

### 创建自定义分词器
可以通过在*analysis*属性中，配置character filters、tokenizers和token filters
```json
PUT /my_index
{
	"settings": {
		"analysis": {
			"char_filter": { ... custom character filters ... },
			"tokenizer": { ... custom tokenizers ... },
			"filter": { ... custom token filters ... },
			"analyzer": { ... custom analyzers ... }
		}
	}
}
```
如下，我们配置一个自定义的分词器
- 使用html_strip去除HTML标签
- 使用自定义mapping character filter替换&为and
- 使用standard tokenizer进行分词
- 使用lowercase token filter转为小写
- 使用stop token filter去除停止词
用以上定义的filter配置自定义analyzer
```json
PUT /my_index
{
	"settings": {
		"analysis": {
			"char_filter": {
			"&_to_and": {
			"type": "mapping",
			"mappings": [ "&=> and "]
		}},
		"filter": {
			"my_stopwords": {
				"type": "stop",
				"stopwords": [ "the", "a" ]
		}},
		"analyzer": {
			"my_analyzer": {
				"type": "custom",
				"char_filter": [ "html_strip", "&_to_and" ],
				"tokenizer": "standard",
				"filter": [ "lowercase", "my_stopwords" ]
				}
			}
		}
	}
}

PUT /my_index/_mapping/my_type
{
	"properties": {
		"title": {
			"type": "string",
			"analyzer": "my_analyzer"
		}
	}
}
```

### Types and Mappings
*Type*表示一类相似的文档，ES新版本中取消配置type，默认使用_doc

Lucene中文档时一组field-value组合，不论是string、number或date类型，都会被转为*opaque bytes*。

#### Type是如何实现的
Lucene中没有type这个概念，对应的值是存储在文档_type字段中的，当搜索特定类型的文档时，ES用_type字段进行过滤获取到匹配的文档。

Lucene中也没有Mappings的概念，Mappings是ES中用来将Lucene中的简单flat document映射到复杂JSON结构体的。

### Root对象
Mapping中包含的顶层属性为root对象。

#### Properties
*type*
字段数据类型
*index*
字段是否需要分词analyzed/not_analyzed
*analyzer*
需要分词场景下，使用的分词器

#### Metadata: _source Field
默认情况下，ES将文档的JSON存储在_source字段中，压缩完成后存储至磁盘中。好处如下:
- 整个文档可以从查询结果中直接访问，不用从其他存储中再次fetch文档
- 没有_source字段，部分更新操作将无法使用
- 当修改了mapping需要reindex数据时，不需要从其他存储中获取，可以直接从ES中获取到所有文档
- 更容易debug查询，可以清楚的看到每个文档中包含的字段

可以通过如下方式关闭_source字段
```json
PUT /my_index
{
	"mappings": {
		"my_type": {
			"_source": {
				"enabled": false
			}
		}
	}
}
```
可以在查询时声明_source字段，指定对应查询的字段
```json
GET /_search
{
	"query": { "match_all": {}},
	"_source": [ "title", "created" ]
}
```

**Store Fields**
Lucene中可以使用stored field来选择需要返回的字段，在ES中即为_source，整个文档在ES中都存储为_source字段，查询过程中最好使用_source明确指明需要返回的数据字段。

### Metadata: _all Field
将其他字段合成一个长字符串，如查询时未声明搜索的字段，默认将使用_all字段进行查询。_all字段对应的是字符串，使用默认分词器进行处理。
```json
GET /_search
{
	"match": {
		"_all": "john smith marketing"
	}
}
```
可以通过如下设置来禁止_all字段，或通过设置include_in_all来禁用
```json
PUT /my_index/_mapping/my_type
{
	"my_type": {
		"_all": { "enabled": false }
	}
}

PUT /my_index/my_type/_mapping
{
	"my_type": {
		"include_in_all": false,
		"properties": {
			"title": {
				"type": "string",
				"include_in_all": true
			},
		...
		}
	}
}
```
也可以配置_all字段使用的分词器
```json
PUT /my_index/my_type/_mapping
{
	"my_type": {
		"_all": { "analyzer": "whitespace" }
	}
}
```

### Metadata: Document Identity
*_id*
文档string ID，不索引且不存储

*_type*
文档类型，索引字段，不存储

*_index*
文档索引，不索引且不存储

*_uid*
_type和_id组合(type#id)，存储且索引的字段，ES使用_uid来获取_id

可通过配置path字段来声明_id从哪个字段取值，此操作在使用bulk处理时存在性能损失，节点不能对bulk处理进行优化，原本只需解析文档metadata数据可知道对应路由节点，配置后需解析完整文档，才能获取到_id。
```json
PUT /my_index
{
	"mappings": {
		"my_type": {
			"_id": {
				"path": "doc_id"
			},
			"properties": {
				"doc_id": {
					"type": "string",
					"index": "not_analyzed"
				}
			}
		}
	}
}
```

### Dynamic Mapping
*true*
默认配置，对没有配置Mapping的字段自动生成类型

*false*
忽略未配置Mapping字段，配置为false并不会影响_source字段，_source仍然会包含整个完整的文档，但对应字段不会生成Mapping，也不可搜索

*strict*
遇到未配置Mapping字段抛出异常

可在创建索引Mapping时进行配置
```json
PUT /my_index
{
	"mappings": {
		"my_type": {
			"dynamic": "strict",
				"properties": {
				"title": { "type": "string"},
				"stash": {
					"type": "object",
					"dynamic": true
				}
			}
		}
	}
}
```

### 自定义Dynamic Mapping 147







## 结构化搜索
### 精确查询
#### term Filter with Numbers
*filtered query*可以同时接收query和filter，filter并不参与结果中_score分数计算
```json
GET /my_store/products/_search
{
	"query" : {
		"filtered" : {
			"query" : {
				"match_all" : {}
			},
			"filter" : {
				"term" : {
				"price" : 20
				}
			}
		}
	}
}
```

#### term Filter with Text
类似于执行SQL
```sql
SELECT product
FROM products
WHERE productID = "XHDK-A-1293-#fJ3"
```

*term*查询字符串与数字方式相同
```json
GET /my_store/products/_search
{
	"query" : {
		"filtered" : {
			"filter" : {
				"term" : {
					"productID" : "XHDK-A-1293-#fJ3"
				}
			}
		}
	}
}
```

但查询返回结果为空，因为对productID字段使用了分词器，使其值在索引中被分解为多个token，当我们尝试匹配完整的值时，在索引中没有对应的结果，可查看分析如下:
```json
GET /my_store/_analyze?field=productID
XHDK-A-1293-#fJ3
{
	"tokens" : [ 
		{
			"token" : "xhdk",
			"start_offset" : 0,
			"end_offset" : 4,
			"type" : "<ALPHANUM>",
			"position" : 1
		}, {
			"token" : "a",
			"start_offset" : 5,
			"end_offset" : 6,
			"type" : "<ALPHANUM>",
			"position" : 2
		}, {
			"token" : "1293",
			"start_offset" : 7,
			"end_offset" : 11,
			"type" : "<NUM>",
			"position" : 3
		}, {
			"token" : "fj3",
			"start_offset" : 13,
			"end_offset" : 16,
			"type" : "<ALPHANUM>",
			"position" : 4
		}
	]
}
```
- 索引值被拆分为4个token
- 所有字母被转换为小写
- 拆分后特殊字符丢失(-#)

要使用精确查询的话，我们需告诉ES该字段不需进行分词
```json
DELETE /my_store

PUT /my_store
{
	"mappings" : {
		"products" : {
			"properties" : {
				"productID" : {
					"type" : "string",
					"index" : "not_analyzed"
				}
			}
		}
	}
}
```

#### Internal Filter Operation
ES内部执行filter时有一下操作:
*Find matching docs*
term filter在倒排索引中查询包含该字段值得文档

*Build a bitset*
Filter创建一个bitset(一个包含0/1的数组)，匹配的文档标识1bit

*Cache the bitset*
最后，这个bitset会缓存在内存中，在后续的查询中可以直接跳过以上两个步骤来查询，提升filter查询的性能。

当执行*filtered query*时，filter优先于query执行，query会在bitset查询结果集中执行，缩小了query执行的范围。

### Combining Filter
类似于执行SQL
```sql
SELECT product
FROM products
WHERE (price = 20 OR productID = "XHDK-A-1293-#fJ3")
AND (price != 30)
```
#### Bool Filter
*bool filter*包含三种类型:
```json
{
	"bool" : {
		"must" : [],
		"should" : [],
		"must_not" : []
	}
}
```
*must*
所有条件都需要满足，与AND相同

*must_not*
所有条件都需要不满足，与NOT相同

*should*
至少一个条件需要满足，与OR相同

bool查询条件可以多个同时使用，我们使用filtered将过滤条件包起来
```json
GET /my_store/products/_search
{
	"query" : {
		"filtered" : {
			"filter" : {
				"bool" : {
					"should" : [
						{ "term" : {"price" : 20}},
						{ "term" : {"productID" : "XHDK-A-1293-#fJ3"}}
					],
					"must_not" : {
						"term" : {"price" : 30}
					}
				}
			}
		}
	}
}
```


#### Nesting Boolean Filter
我们可以在一个bool filter内部嵌套另一个bool filter，例如样例SQL如下
```sql
SELECT document
FROM products
WHERE productID = "KDKE-B-9947-#kL5"
OR ( productID = "JODL-X-1937-#pV7" AND price = 30 )
```
使用ES查询条件如下，在should中嵌套的多个条件，至少有一个需要满足，must中的条件都需满足。
```json
GET /my_store/products/_search
{
	"query" : {
		"filtered" : {
			"filter" : {
				"bool" : {
					"should" : [
						{ "term" : {"productID" : "KDKE-B-9947-#kL5"}},
						{ "bool" : {
							"must" : [
								{ "term" : {"productID" : "JODL-X-1937-#pV7"}},
								{ "term" : {"price" : 30}}
							]
						}}
					]
				}
			}
		}
	}
}
```

### 精确查找多个值
*term*用于查找单个精确值，*terms*可以用来查找多个精确值，查找方式为contains，不是equals，当查找的字段有多个值时，有一个满足即可匹配。
```json
GET /my_index/my_type/_search
{
	"query": {
		"filtered" : {
			"filter" : {
				"bool" : {
					"must" : [
						{ "term" : { "tags" : "search" } },
						{ "term" : { "tag_count" : 1 } }
					]
				}
			}
		}
	}
}
```

### Ranges
样例SQL如下
```sql
SELECT document
FROM products
WHERE price BETWEEN 20 AND 40
```

ES使用操作符如下:
- gt: > greater than
- lt: < less than
- gte: >= greater than or equal to
- lte: <= less than or equal to

```json
GET /my_store/products/_search
{
	"query" : {
		"filtered" : {
			"filter" : {
				"range" : {
					"price" : {
						"gte" : 20,
						"lt" : 40
					}
				}
			}
		}
	}
}
```

#### Ranges on Dates
可以对date类型字段进行range操作
```json
{
	"range" : {
		"timestamp" : {
			"gt" : "2014-01-01 00:00:00",
			"lt" : "2014-01-07 00:00:00"
		}
	}
}
```
range支持date math操作
```json
{
	"range" : {
		"timestamp" : {
			"gt" : "now-1h"
		}
	}
}
```

#### Ranges on Strings
range也可以对string类型进行操作，使用的是字典排序。

> terms在倒排索引中使用的也是字典排序

```json
{
	"range" : {
		"title" : {
			"gte" : "a",
			"lt" : "b"
		}
	}
}
```

> ES对数字和date索引方式可以有效的使用range操作，但string类型不行，当对string类型使用range操作时，类似于对range范围内每个组合使用term过滤条件，效率低于数字和date类型

















## 监控
### Cluster Health
```json
GET _cluster/health
{
	"cluster_name": "elasticsearch_zach",
	"status": "green",
	"timed_out": false,
	"number_of_nodes": 1,
	"number_of_data_nodes": 1,
	"active_primary_shards": 10,
	"active_shards": 10,
	"relocating_shards": 0,
	"initializing_shards": 0,
	"unassigned_shards": 0
}
```
*status*
- green: 所有Primary/Replica Shards都已完成分配，集群100%可用
- yellow: 所有Primary Shards已完成分配，至少有一个Replica Shard没有分配，此时没有数据丢失，查询的结果集是完整的，集群可用性有一些损失，可认为是一种警示
- red: 至少有一个Primary Shards丢失，此时查询有数据缺失

*number_of_nodes/number_of_data_nodes*
节点数

*active_primary_shards*
集群中全部可用Primary Shards，包含所有索引

*active_shards*
集群中全部shards，包含Primary/Replica

*relocating_shards*
迁移中的shards数量，一般为0，当节点分布不均时，或有新节点加入、节点不可用时产生

*initializing_shards*
新创建的shards，或节点重启时，仅是中间过程

*unassigned_shards*
常见为unassigned replicas

### 查看故障索引
node总数为10，如下显示缺少两个节点，以及20个shards未分配
```json
{
	"cluster_name": "elasticsearch_zach",
	"status": "red",
	"timed_out": false,
	"number_of_nodes": 8,
	"number_of_data_nodes": 8,
	"active_primary_shards": 90,
	"active_shards": 180,
	"relocating_shards": 0,
	"initializing_shards": 0,
	"unassigned_shards": 20
}
```
进一步查看索引信息
```json
GET _cluster/health?level=indices
{
	"cluster_name": "elasticsearch_zach",
	"status": "red",
	"timed_out": false,
	"number_of_nodes": 8,
	"number_of_data_nodes": 8,
	"active_primary_shards": 90,
	"active_shards": 180,
	"relocating_shards": 0,
	"initializing_shards": 0,
	"unassigned_shards": 20
	"indices": {
		"v1": {
			"status": "green",
			"number_of_shards": 10,
			"number_of_replicas": 1,
			"active_primary_shards": 10,
			"active_shards": 20,
			"relocating_shards": 0,
			"initializing_shards": 0,
			"unassigned_shards": 0
		},
		"v2": {
			"status": "red",
			"number_of_shards": 10,
			"number_of_replicas": 1,
			"active_primary_shards": 0,
			"active_shards": 0,
			"relocating_shards": 0,
			"initializing_shards": 0,
			"unassigned_shards": 20
		},
		"v3": {
			"status": "green",
			"number_of_shards": 10,
			"number_of_replicas": 1,
			"active_primary_shards": 10,
			"active_shards": 20,
			"relocating_shards": 0,
			"initializing_shards": 0,
			"unassigned_shards": 0
		},
	....
	}
}
```
可用如下请求查看具体分片信息，但不是很好理解，知道具体问题索引后，可以使用后续其他方式排查
> GET _cluster/health?level=shards

#### 等待状态变化
cluster-health API可以进行调试或执行脚本
> 等待集群状态为green
>
> GET _cluster/health?wait_for_status=green

例如创建索引后，立即添加文档，此时可调用以上接口阻塞等待，待索引创建完毕后继续执行

#### 监控单个节点
可用于查看节点信息是否存在错误
```json
GET _nodes/stats
{
	"cluster_name": "elasticsearch_zach",
	"nodes": {
	"UNr6ZMf5Qk-YCPA_L18BOQ": {
		"timestamp": 1408474151742,
		"name": "Zach",
		"transport_address": "inet[zacharys-air/192.168.1.131:9300]",
		"host": "zacharys-air",
		"ip": [
		"inet[zacharys-air/192.168.1.131:9300]",
		"NONE"
		],
...
```

#### indices Section
```json
"indices": {
	"docs": {
		"count": 6163666,
		"deleted": 0
	},
	"store": {
		"size_in_bytes": 2301398179,
		"throttle_time_in_millis": 122850
	},
```
*docs*
当前文档数量，以及标记为删除文档数量

*store*
节点消耗存储大小

```json
"indexing": {
	"index_total": 803441,
	"index_time_in_millis": 367654,
	"index_current": 99,
	"delete_total": 0,
	"delete_time_in_millis": 0,
	"delete_current": 0
},
"get": {
	"total": 6,
	"time_in_millis": 2,
	"exists_total": 5,
	"exists_time_in_millis": 2,
	"missing_total": 1,
	"missing_time_in_millis": 0,
	"current": 0
},
"search": {
	"open_contexts": 0,
	"query_total": 123,
	"query_time_in_millis": 531,
	"query_current": 0,
	"fetch_total": 3,
	"fetch_time_in_millis": 55,
	"fetch_current": 0
},
"merges": {
	"current": 0,
	"current_docs": 0,
	"current_size_in_bytes": 0,
	"total": 1128,
	"total_time_in_millis": 21338523,
	"total_docs": 7241313,
	"total_size_in_bytes": 5724869463
},
```

*indexing*
索引文档数量，单向递增，文档删除后也不会减小，索引或更新操作会导致增长

*get*
get-by-ID统计，单个文档GET/HEAD查询

*search*
当前查询数量(open_contexts)，查询总量，以及查询耗时
fetch表明从磁盘读取数据耗时，如果fetch耗时高于query的话，表明磁盘读取效率低，或从磁盘中加载过大量文档，或者查询分页数据量太大

*merges*
正在进行merges数量、参与的文档数量、累计merged segments数量、以及消耗总时间。如果集群写操作很多，merges统计就十分重要，merges会消耗大量磁盘IO和CPU资源。另外，删除更新和删除操作会导致segment片段化，也会导致大量merge。
```json
{
	"filter_cache": {
		"memory_size_in_bytes": 48,
		"evictions": 0
	},
	"id_cache": {
		"memory_size_in_bytes": 0
	},
	"fielddata": {
		"memory_size_in_bytes": 0,
		"evictions": 0
	},
	"segments": {
		"count": 319,
		"memory_in_bytes": 65812120
	},
	...
}
```

*filter_cache*
统计filter bitsets使用的内存，以及filter缓存被老化的次数，如果老化次数过高，可以考虑增加filter缓存大小，否则缓存无法有效命中。
但是缓存老化比较难以量化，因为缓存是以segment进行存储的，老化小的segment影响远小于老化大的segment。
使用eviction metric作为一个粗略指导，如果看到存在大量eviction，查看你的filter确保缓存命中率。

*id_cache*
统计parent/child mappings使用的内存，当使用parent/child映射时，id_cache在内存中保留了join table用来维护这个关系，存放在heap中，统计值可以显示有多少内存被使用了，这部分占用与parent/child文档数量相关，受其他方面影响较小。

*field_data*
显示fielddata使用内存大小，一般用于聚合、排序等，这里同样有eviction统计，与filter_cache不同的是，任何的evict都是昂贵的，应该被避免。如果出现了，你应该关注内存使用情况、fielddata limit、查询条件等。

*segments*
这部分表示Lucene segments，即使存储TB级的数据，一般也介于50-150个之间，如果存在大量segments会导致merging，此处包含当前节点所有索引的统计信息。

### OS and Process Sections
#### JVM
在node-stats API中可以看到JVM相关信息，包含JVM堆使用信息
```jvm
"jvm": {
	"timestamp": 1408556438203,
	"uptime_in_millis": 14457,
	"mem": {
	"heap_used_in_bytes": 457252160,
	"heap_used_percent": 44,
	"heap_committed_in_bytes": 1038876672,
	"heap_max_in_bytes": 1038876672,
	"non_heap_used_in_bytes": 38680680,
	"non_heap_committed_in_bytes": 38993920,
	...
"pools": {
	"young": {
		"used_in_bytes": 138467752,
		"max_in_bytes": 279183360,
		"peak_used_in_bytes": 279183360,
		"peak_max_in_bytes": 279183360
	},
	"survivor": {
		"used_in_bytes": 34865152,
		"max_in_bytes": 34865152,
		"peak_used_in_bytes": 34865152,
		"peak_max_in_bytes": 34865152
	},
	"old": {
		"used_in_bytes": 283919256,
		"max_in_bytes": 724828160,
		"peak_used_in_bytes": 283919256,
		"peak_max_in_bytes": 724828160
	}
}

"gc": {
	"collectors": {
		"young": {
			"collection_count": 13,
			"collection_time_in_millis": 923
		},
		"old": {
			"collection_count": 0,
			"collection_time_in_millis": 0
		}
	}
}
```

### Threadpool Section
一般情况并不需要调试线程池，但有时可以用来观察集群状态
```json
"index": {
	"threads": 1,
	"queue": 0,
	"active": 0,
	"rejected": 0,
	"largest": 1,
	"completed": 1
}
```

#### Bulk Rejections
最常见的是bulk indexing请求，每个节点都有吞吐量上线，如果超过这个限制，批量请求就会被拒绝。当遇到bulk rejection时，可做如下处理:
1. 暂停导入线程3-5秒
2. 解析拒绝响应，查看具体失败的信息
3. 发送新的bulk请求添加失败的文档
4. 如果再次遇到bulk rejection，重复步骤1

拒绝请求并非是错误，仅是提醒需要稍后重试。

以下线程池需要进行观察:
*indexing*
用于处理索引文档请求

*bulk*
用于处理批量请求

*get*
处理Get-by-ID请求

*search*
所有查询请求

*merging*
用于处理Lucene merge

### FS and Network Sections
node-stats API还包含磁盘信息，包括剩余空间、数据路径、以及磁盘IO。

同时也包含网络信息:
```json
"transport": {
	"server_open": 13,
	"rx_count": 11696,
	"rx_size_in_bytes": 1525774,
	"tx_count": 10282,
	"tx_size_in_bytes": 1440101928
},
"http": {
	"current_open": 4,
	"total_opened": 23
},
```

*transport*
包含节点之间通信信息(通常端口为9300)，ES节点之间通常有大量链接

*http*
如果看到total_opened数量持续增长，可以确定是HTTP clients没有使用keep-alive链接，使用keep-alive可以提高性能。

### Circuit Breaker
这里可以查看circuit-breaker最大数量，以及tripped次数。如果数量很大且持续上涨，表明你的查询条件需要进行优化或者需要更大的内存空间。
```json
"fielddata_breaker": {
	"maximum_size_in_bytes": 623326003,
	"maximum_size": "594.4mb",
	"estimated_size_in_bytes": 0,
	"estimated_size": "0b",
	"overhead": 1.03,
	"tripped": 0
}
```

### Cluster Stats
Cluster-stats API提供了监控集群中所有节点状态的视图
> GET _cluster/stats

### Index Stats
通过index-stats API可进行查看
> my_index/_stats
> my_index,another_index/_stats
> _all/_stats

返回的内容与node-stats相似，可用于识别集群中热点索引，或用于鉴别个别索引查询效率，在实际中node-stats更为实用。

### Pending Tasks
有些任务仅master节点可以执行，例如创建索引、移动集群内分片。集群中只有一个master节点，可以处理集群级别元模型变更。在少数场景下，模型变更速度超过master节点处理速度，导致任务积压。

可以通过pending-task API进行查看
```json
GET _cluster/pending_tasks
{
	"tasks": [
		{
			"insert_order": 101,
			"priority": "URGENT",
			"source": "create-index [foo_9], cause [api]",
			"time_in_queue_millis": 86,
			"time_in_queue": "86ms"
		},
		{
			"insert_order": 46,
			"priority": "HIGH",
			"source": "shard-started ([foo_2][1], node[tMTocMvQQgGCkj7QDHl3OA], [P],s[INITIALIZING]), reason [after recovery from gateway]",
			"time_in_queue_millis": 842,
			"time_in_queue": "842ms"
		},
		{
			"insert_order": 45,
			"priority": "HIGH",
			"source": "shard-started ([foo_2][0], node[tMTocMvQQgGCkj7QDHl3OA], [P],s[INITIALIZING]), reason [after recovery from gateway]",
			"time_in_queue_millis": 858,
			"time_in_queue": "858ms"
		}
	]
}
```

#### 合适需顾虑Pending Tasks
当集群cluster-stat很大且变更很频繁时，可能导致Pending Tasks，此时可以考虑如下方法:
- 提高master节点性能，水平扩展只是延迟了这一现象
- 限制文档的鼎泰变化，从而限制cluster-state大小
- 当超过集群阈值后，切换至另一个集群

### cat API
cat API提供与之前API相似的内容，可以查看如下内容
```
GET /_cat
=^.^=
/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
/_cat/indices
/_cat/indices/{index}
/_cat/segments
/_cat/segments/{index}
/_cat/count
/_cat/count/{index}
/_cat/recovery
/_cat/recovery/{index}
/_cat/health
/_cat/pending_tasks
/_cat/aliases
/_cat/aliases/{alias}
/_cat/thread_pool
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
```

查询结果中没有包含行首，可以追加?v参数进行打印
> GET /_cat/health?v
> epoch time cluster status node.total node.data shards pri relo init
> 1408[..] 12[..] el[..] 1 1 114 114 0 0 114 unassign

cat API可以向Linux工具一样，使用*sort grep awk*等操作
> curl 'localhost:9200/_cat/indices?bytes=b' | sort -rnk8
> curl 'localhost:9200/_cat/indices?bytes=b' | sort -rnk8 | grep -v marvel



