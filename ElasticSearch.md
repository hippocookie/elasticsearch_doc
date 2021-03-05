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

