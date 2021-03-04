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