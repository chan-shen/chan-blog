---
title: Elasticsearch介绍与Python实现
date: 2019-06-10
lastmod: 2019-06-10
description: ""

tags:
 - web
 - python
categories: 
 - 2019
 - 技术
---

## 简介
* 分布式的实时文档存储，每个字段可以被索引与搜索
* 分布式实时分析搜索引擎
* 胜任上百个服务节点的扩展，支持 PB 级别的结构化或者非结构化数据

## Elasticsearch 的安装
[Elasticsearch官方网站](https://www.elastic.co/downloads/elasticsearch)。

下载安装包并解压，然后运行 bin/elasticsearch（Mac 或 Linux）

或者 bin\elasticsearch.bat (Windows) 即可启动 Elasticsearch。

Mac 下推荐使用 Homebrew 安装：

[安装Homebrew问题](http://xshanshan.cn/blogs/2020/20200401_macOS%E5%AE%89%E8%A3%85Homebrew%E9%97%AE%E9%A2%98.html)

``` shell
brew install elasticsearch
```

Elasticsearch 默认会在 9200 端口上运行，我们打开浏览器访问
http://localhost:9200/ 就可以看到类似内容
``` json
{
  "name" : "atntrTf",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "e64hkjGtTp6_G2h1Xxdv5g",
  "version" : {
    "number": "6.2.4",
    "build_hash": "ccec39f",
    "build_date": "2018-04-12T20:37:28.497551Z",
    "build_snapshot": false,
    "lucene_version": "7.2.1",
    "minimum_wire_compatibility_version": "5.6.0",
    "minimum_index_compatibility_version": "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
**版本很重要，以后安装一些插件都要做到版本对应才可以。**

## Elasticsearch 相关概念
在 Elasticsearch 中有几个基本的概念，如节点、索引、文档等。

### Node 和 Cluster
Elasticsearch 本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 Elasticsearch 实例。

单个 Elasticsearch 实例称为一个节点（Node）。一组节点构成一个集群（Cluster）。

### Index
Elasticsearch 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

所以，Elasticsearch 数据管理的顶层单位就叫做 Index（索引），其实就相当于 MySQL、MongoDB 等里面的数据库的概念。另外值得注意的是，每个 Index （即数据库）的名字必须是小写。

### Document
Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

Document 使用 JSON 格式表示，下面是一个例子。

同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

### Type
Document 可以分组，比如 weather 这个 Index 里面，可以按城市分组（北京和上海），也可以按气候分组（晴天和雨天）。这种分组就叫做 Type，它是虚拟的逻辑分组，用来过滤 Document，类似 MySQL 中的数据表，MongoDB 中的 Collection。

不同的 Type 应该有相似的结构（Schema），举例来说，id 字段不能在这个组是字符串，在另一个组是数值。这是与关系型数据库的表的一个区别。性质完全不同的数据（比如 products 和 logs）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。

根据规划，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。

### Fields
即字段，每个 Document 都类似一个 JSON 结构，它包含了许多字段，每个字段都有其对应的值，多个字段组成了一个 Document，其实就可以类比 MySQL 数据表中的字段。

**在 Elasticsearch 中，文档归属于一种类型（Type），而这些类型存在于索引（Index）中，些简单的对比图来类比传统关系型数据库：**
``` shell
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

## Python 对接 Elasticsearch
Elasticsearch 提供了一系列 Restful API 来进行存取和查询操作，可以使用 curl 等命令来进行操作。这里直接利用 Python 来对接 Elasticsearch 的相关方法。

Python 中对接 Elasticsearch 使用的就是一个同名的库：
``` py
pip3 install elasticsearch
```

[官方文档](https://elasticsearch-py.readthedocs.io/)，后面的内容也是基于官方文档。

### 创建 Index
创建一个名为 news 的索引：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
result = es.indices.create(index='news', ignore=400)
print(result)
```

创建成功，会返回如下结果：
``` shell
{'acknowledged': True, 'shards_acknowledged': True, 'index': 'news'}
```
返回结果是 JSON 格式，其中的 acknowledged 字段表示创建操作执行成功。

如果把代码执行一次的话，就会返回如下结果：
``` py
{'error': {'root_cause': [{'type': 'resource_already_exists_exception', 'reason': 'index [news/QM6yz2W8QE-bflKhc5oThw] already exists', 'index_uuid': 'QM6yz2W8QE-bflKhc5oThw', 'index': 'news'}], 'type': 'resource_already_exists_exception', 'reason': 'index [news/QM6yz2W8QE-bflKhc5oThw] already exists', 'index_uuid': 'QM6yz2W8QE-bflKhc5oThw', 'index': 'news'}, 'status': 400}
```
它提示创建失败，status 状态码是 400，错误原因是 Index 已经存在了。

注意这里代码里面使用了 ignore 参数为 400，这说明如果返回结果是 400 的话，就忽略这个错误不会报错，程序不会执行抛出异常。

假如不加 ignore 这个参数的话：
``` py
es = Elasticsearch()
result = es.indices.create(index='news')
print(result)
```

再次执行就会报错了：
``` py
raise HTTP_EXCEPTIONS.get(status_code, TransportError)(status_code, error_message, additional_info)
elasticsearch.exceptions.RequestError: TransportError(400, 'resource_already_exists_exception', 'index [news/QM6yz2W8QE-bflKhc5oThw] already exists')
```

这样程序的执行就会出现问题，所以需要善用 ignore 参数，把一些意外情况排除，这样可以保证程序的正常执行而不会中断。

删除 Index
删除 Index 也是类似的，代码如下：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
result = es.indices.delete(index='news', ignore=[400, 404])
print(result)
```

这里也是使用了 ignore 参数，来忽略 Index 不存在而删除失败导致程序中断的问题。

如果删除成功，会输出如下结果：
``` py
{'acknowledged': True}
```

如果 Index 已经被删除，再执行删除则会输出如下结果：
``` py
{'error': {'root_cause': [{'type': 'index_not_found_exception', 'reason': 'no such index', 'resource.type': 'index_or_alias', 'resource.id': 'news', 'index_uuid': '_na_', 'index': 'news'}], 'type': 'index_not_found_exception', 'reason': 'no such index', 'resource.type': 'index_or_alias', 'resource.id': 'news', 'index_uuid': '_na_', 'index': 'news'}, 'status': 404}
```

这个结果表明当前 Index 不存在，删除失败，返回的结果同样是 JSON，状态码是 400，但是由于我们添加了 ignore 参数，忽略了 400 状态码，因此程序正常执行输出 JSON 结果，而不是抛出异常。

### 插入数据
Elasticsearch 就像 MongoDB 一样，在插入数据的时候可以直接插入结构化字典数据，插入数据可以调用 create() 方法，插入一条新闻数据：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
es.indices.create(index='news', ignore=400)

data = {'title': '美国留给伊拉克的是个烂摊子吗', 'url': 'http://view.news.qq.com/zt2011/usa_iraq/index.htm'}
result = es.create(index='news', doc_type='politics', id=1, body=data)
print(result)
```

这里首先声明了一条新闻数据，包括标题和链接，然后通过调用 create() 方法插入了这条数据，在调用 create() 方法时，我们传入了四个参数，index 参数代表了索引名称，doc_type 代表了文档类型，body 则代表了文档具体内容，id 则是数据的唯一标识 ID。

运行结果如下：
``` py
{'_index': 'news', '_type': 'politics', '_id': '1', '_version': 1, 'result': 'created', '_shards': {'total': 2, 'successful': 1, 'failed': 0}, '_seq_no': 0, '_primary_term': 1}
```

结果中 result 字段为 created，代表该数据插入成功。

也可以使用 index() 方法来插入数据，但与 create() 不同的是，create() 方法需要指定 id 字段来唯一标识该条数据，而 index() 方法则不需要，如果不指定 id，会自动生成一个 id，调用 index() 方法的写法如下：
``` py
es.index(index='news', doc_type='politics', body=data)
```

**create() 方法内部其实也是调用了 index() 方法，是对 index() 方法的封装。**

### 更新数据
更新数据同样需要指定数据的 id 和内容，调用 update() 方法即可，代码如下：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
data = {
    'title': '美国留给伊拉克的是个烂摊子吗',
    'url': 'http://view.news.qq.com/zt2011/usa_iraq/index.htm',
    'date': '2011-12-16'
}
result = es.update(index='news', doc_type='politics', body=data, id=1)
print(result)
```

为数据增加了一个日期字段，然后调用了 update() 方法，结果如下：
``` py
{'_index': 'news', '_type': 'politics', '_id': '1', '_version': 2, 'result': 'updated', '_shards': {'total': 2, 'successful': 1, 'failed': 0}, '_seq_no': 1, '_primary_term': 1}
```

返回结果中，result 字段为 updated，即表示更新成功，注意字段 _version，这代表更新后的版本号数，2 代表这是第二个版本，因为之前已经插入过一次数据，所以第一次插入的数据是版本 1，可以参见上例的运行结果，这次更新之后版本号就变成了 2，以后每更新一次，版本号都会加 1。

另外更新操作其实利用 index() 方法同样可以做到，写法如下：
``` py
es.index(index='news', doc_type='politics', body=data, id=1)
```

index() 方法可以代完成两个操作，如果数据不存在，那就执行插入操作，如果已经存在，那就执行更新操作，非常方便。

### 删除数据
如果想删除一条数据可以调用 delete() 方法，指定需要删除的数据 id 即可，写法如下：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
result = es.delete(index='news', doc_type='politics', id=1)
print(result)
```

运行结果如下：
``` py
{'_index': 'news', '_type': 'politics', '_id': '1', '_version': 3, 'result': 'deleted', '_shards': {'total': 2, 'successful': 1, 'failed': 0}, '_seq_no': 2, '_primary_term': 1}
```

可以看到运行结果中 result 字段为 deleted，代表删除成功，_version 变成了 3，又增加了 1。

### 查询数据
上面的几个操作都是非常简单的操作，普通的数据库如 MongoDB 都是可以完成的，Elasticsearch 更特殊的地方在于其异常强大的检索功能。

对于中文来说，需要安装一个分词插件，这里使用的是 elasticsearch-analysis-ik，GitHub 链接为：[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)，这里我们使用 Elasticsearch 的另一个命令行工具 elasticsearch-plugin 来安装，这里安装的版本是 6.2.4，请确保和 Elasticsearch 的版本对应起来，命令如下：
``` shell
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.2.4/elasticsearch-analysis-ik-6.2.4.zip
```

**这里的版本号请替换成你的 Elasticsearch 的版本号。**

安装之后重新启动 Elasticsearch 就可以了，它会自动加载安装好的插件。

首先新建一个索引并指定需要分词的字段，代码如下：
``` py
from elasticsearch import Elasticsearch

es = Elasticsearch()
mapping = {
    'properties': {
        'title': {
            'type': 'text',
            'analyzer': 'ik_max_word',
            'search_analyzer': 'ik_max_word'
        }
    }
}
es.indices.delete(index='news', ignore=[400, 404])
es.indices.create(index='news', ignore=400)
result = es.indices.put_mapping(index='news', doc_type='politics', body=mapping)
print(result)
```

这里先将之前的索引删除了，然后新建了一个索引，然后更新了它的 mapping 信息，mapping 信息中指定了分词的字段，指定了字段的类型 type 为 text，分词器 analyzer 和 搜索分词器 search_analyzer 为 ik_max_word，即使用我们刚才安装的中文分词插件。如果不指定的话则使用默认的英文分词器。

插入几条新的数据：
``` py
datas = [
    {
        'title': '美国留给伊拉克的是个烂摊子吗',
        'url': 'http://view.news.qq.com/zt2011/usa_iraq/index.htm',
        'date': '2011-12-16'
    },
    {
        'title': '公安部：各地校车将享最高路权',
        'url': 'http://www.chinanews.com/gn/2011/12-16/3536077.shtml',
        'date': '2011-12-16'
    },
    {
        'title': '中韩渔警冲突调查：韩警平均每天扣1艘中国渔船',
        'url': 'https://news.qq.com/a/20111216/001044.htm',
        'date': '2011-12-17'
    },
    {
        'title': '中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首',
        'url': 'http://news.ifeng.com/world/detail_2011_12/16/11372558_0.shtml',
        'date': '2011-12-18'
    }
]

for data in datas:
    es.index(index='news', doc_type='politics', body=data)
```

指定了四条数据，都带有 title、url、date 字段，然后通过 index() 方法将其插入 Elasticsearch 中，索引名称为 news，类型为 politics。

根据关键词查询一下相关内容：
``` py
result = es.search(index='news', doc_type='politics')
print(result)
```

查询出了所有插入的四条数据：
``` py
{
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 1.0,
    "hits": [
      {
        "_index": "news",
        "_type": "politics",
        "_id": "c05G9mQBD9BuE5fdHOUT",
        "_score": 1.0,
        "_source": {
          "title": "美国留给伊拉克的是个烂摊子吗",
          "url": "http://view.news.qq.com/zt2011/usa_iraq/index.htm",
          "date": "2011-12-16"
        }
      },
      {
        "_index": "news",
        "_type": "politics",
        "_id": "dk5G9mQBD9BuE5fdHOUm",
        "_score": 1.0,
        "_source": {
          "title": "中国驻洛杉矶领事馆遭亚裔男子枪击，嫌犯已自首",
          "url": "http://news.ifeng.com/world/detail_2011_12/16/11372558_0.shtml",
          "date": "2011-12-18"
        }
      },
      {
        "_index": "news",
        "_type": "politics",
        "_id": "dU5G9mQBD9BuE5fdHOUj",
        "_score": 1.0,
        "_source": {
          "title": "中韩渔警冲突调查：韩警平均每天扣1艘中国渔船",
          "url": "https://news.qq.com/a/20111216/001044.htm",
          "date": "2011-12-17"
        }
      },
      {
        "_index": "news",
        "_type": "politics",
        "_id": "dE5G9mQBD9BuE5fdHOUf",
        "_score": 1.0,
        "_source": {
          "title": "公安部：各地校车将享最高路权",
          "url": "http://www.chinanews.com/gn/2011/12-16/3536077.shtml",
          "date": "2011-12-16"
        }
      }
    ]
  }
}
```

返回结果会出现在 hits 字段里面，然后其中有 total 字段标明了查询的结果条目数，还有 max_score 代表了最大匹配分数。

**全文检索**，Elasticsearch 搜索引擎特性的地方：
``` py
dsl = {
    'query': {
        'match': {
            'title': '中国 领事馆'
        }
    }
}

es = Elasticsearch()
result = es.search(index='news', doc_type='politics', body=dsl)
print(json.dumps(result, indent=2, ensure_ascii=False))
```

使用 Elasticsearch 支持的 DSL 语句来进行查询，使用 match 指定全文检索，检索的字段是 title，内容是“中国领事馆”，搜索结果如下：
``` py
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 2.546152,
    "hits": [
      {
        "_index": "news",
        "_type": "politics",
        "_id": "dk5G9mQBD9BuE5fdHOUm",
        "_score": 2.546152,
        "_source": {
          "title": "中国驻洛杉矶领事馆遭亚裔男子枪击，嫌犯已自首",
          "url": "http://news.ifeng.com/world/detail_2011_12/16/11372558_0.shtml",
          "date": "2011-12-18"
        }
      },
      {
        "_index": "news",
        "_type": "politics",
        "_id": "dU5G9mQBD9BuE5fdHOUj",
        "_score": 0.2876821,
        "_source": {
          "title": "中韩渔警冲突调查：韩警平均每天扣1艘中国渔船",
          "url": "https://news.qq.com/a/20111216/001044.htm",
          "date": "2011-12-17"
        }
      }
    ]
  }
}
```
匹配的结果有两条，第一条的分数为 2.54，第二条的分数为 0.28，这是因为第一条匹配的数据中含有“中国”和“领事馆”两个词，第二条匹配的数据中不包含“领事馆”，但是包含了“中国”这个词，所以也被检索出来了，但是分数比较低。

检索时会对对应的字段全文检索，结果还会按照检索关键词的相关性进行排序，这就是一个基本的搜索引擎雏形。

Elasticsearch 支持非常多的查询方式，详情可以参考 [官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/query-dsl.html)
