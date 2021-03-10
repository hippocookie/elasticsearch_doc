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







