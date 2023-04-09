[toc]
作者：阮一峰

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本系列文章将整理到我的个人博客

> www.how2playlife.com

更多Java技术文章会更新在我的微信公众号【Java技术江湖】上，欢迎关注
该系列博文会介绍常见的后端技术，这对后端工程师来说是一种综合能力，我们会逐步了解搜索技术，云计算相关技术、大数据研发等常见的技术喜提，以便让你更完整地了解后端技术栈的全貌，为后续参与分布式应用的开发和学习做好准备。


如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系我，欢迎你参与本系列博文的创作和修订。

<!-- more -->
原文链接：www.ruanyifeng.com


它可以快速地储存、搜索和分析海量数据。维基百科、Stack Overflow、Github 都采用它。

Elastic 的底层是开源库Lucene
但是，你没法直接用 Lucene，必须自己写代码去调用它的接口。
Elastic 是 Lucene 的封装，提供了 REST API 的操作接口，开箱即用。

本文从零开始，讲解如何使用 Elastic 搭建自己的全文搜索引擎。每一步都有详细的说明，大家跟着做就能学会。

## 一、安装

Elastic 需要 Java 8 环境。注意要保证环境变量`JAVA_HOME`正确设置。

安装完 Java，就可以跟着官方文档安装 Elastic。直接下载压缩包比较简单。

> ```
> $ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.zip$ unzip elasticsearch-5.5.1.zip$ cd elasticsearch-5.5.1/ 
> ```

接着，进入解压后的目录，运行下面的命令，启动 Elastic。

> ```
> $ ./bin/elasticsearch
> ```


> ```
> $ sudo sysctl -w vm.max_map_count=262144
> ```

如果一切正常，Elastic 就会在默认的9200端口运行。这时，打开另一个命令行窗口，请求该端口，会得到说明信息。

```
> $ curl localhost:9200
{
"name": "atntrTf",
"cluster_name": "elasticsearch",
"cluster_uuid": "tf9250XhQ6ee4h7YI11anA",
"version": {
"number": "5.5.1",
"build_hash": "19c13d0",
"build_date": "2017-07-18T20:44:24.823Z",
"build_snapshot": false,
"lucene_version": "6.6.0"
},
"tagline": "You Know, for Search"
}
```

上面代码中，请求9200端口，Elastic 返回一个 JSON 对象，包含当前节点、集群、版本等信息。

按下 Ctrl + C，Elastic 就会停止运行。

默认情况下，Elastic 只允许本机访问，如果需要远程访问，可以修改 Elastic 安装目录的`config/elasticsearch.yml`文件，去掉`network.host`的注释，将它的值改成`0.0.0.0`，然后重新启动 Elastic。

> ```
> network.host: 0.0.0.0
> ```

上面代码中，设成`0.0.0.0`让任何人都可以访问。线上服务不要这样设置，要设成具体的 IP。

## 二、基本概念

### 2.1 Node 与 Cluster

Elastic 本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 Elastic 实例。

单个 Elastic 实例称为一个节点（node）。一组节点构成一个集群（cluster）。

### 2.2 Index

Elastic 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。

下面的命令可以查看当前节点的所有 Index。

> ```
> $ curl -X GET 'http://localhost:9200/_cat/indices?v'
> ```

### 2.3 Document

Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

Document 使用 JSON 格式表示，下面是一个例子。

> ```
> {  "user": "张三",  "title": "工程师",  "desc": "数据库管理"}
> ```

同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

### 2.4 Type

Document 可以分组，比如`weather`这个 Index 里面，可以按城市分组（北京和上海），也可以按气候分组（晴天和雨天）。这种分组就叫做 Type，它是虚拟的逻辑分组，用来过滤 Document。

不同的 Type 应该有相似的结构（schema），举例来说，`id`字段不能在这个组是字符串，在另一个组是数值。
这是与关系型数据库的表的一个区别。
性质完全不同的数据（比如`products`和`logs`）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。

下面的命令可以列出每个 Index 所包含的 Type。

> ```
> $ curl 'localhost:9200/_mapping?pretty=true'
> ```

根据规划Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。

## 三、新建和删除 Index

新建 Index，可以直接向 Elastic 服务器发出 PUT 请求。下面的例子是新建一个名叫`weather`的 Index。

> ```
> $ curl -X PUT 'localhost:9200/weather'
> ```

服务器返回一个 JSON 对象，里面的`acknowledged`字段表示操作成功。

> ```
> {  "acknowledged":true,  "shards_acknowledged":true}
> ```

然后，我们发出 DELETE 请求，删除这个 Index。

> ```
> $ curl -X DELETE 'localhost:9200/weather'
> ```

## 四、中文分词设置

首先，安装中文分词插件。这里使用的是ik，也可以考虑其他插件（比如smartcn）。

> ```
> $ ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
> ```

上面代码安装的是5.5.1版的插件，与 Elastic 5.5.1 配合使用。

接着，重新启动 Elastic，就会自动加载这个新安装的插件。

然后，新建一个 Index，指定需要分词的字段。这一步根据数据结构而异，下面的命令只针对本文。基本上，凡是需要搜索的中文字段，都要单独设置一下。

```
> $ curl -X PUT 'localhost:9200/accounts' -d 
{
	"mappings": {
		"person": {
			"properties": {
				"user": {
					"type": "text",
					"analyzer": "ik_max_word",
					"search_analyzer": "ik_max_word"
				},
				"title": {
					"type": "text",
					"analyzer": "ik_max_word",
					"search_analyzer": "ik_max_word"
				},
				"desc": {
					"type": "text",
					"analyzer": "ik_max_word",
					"search_analyzer": "ik_max_word"
				}
			}
		}
	}
}```

上面代码中，首先新建一个名称为`accounts`的 Index，里面有一个名称为`person`的 Type。`person`有三个字段。

> *   user
> *   title
> *   desc

这三个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

Elastic 的分词器称为analyzer。我们对每个字段指定分词器。

> ```
> "user": {  "type": "text",  "analyzer": "ik_max_word",  "search_analyzer": "ik_max_word"}
> ```

上面代码中，`analyzer`是字段文本的分词器，`search_analyzer`是搜索词的分词器。`ik_max_word`分词器是插件`ik`提供的，可以对文本进行最大数量的分词。

## 五、数据操作

### 5.1 新增记录

向指定的 /Index/Type 发送 PUT 请求，就可以在 Index 里面新增一条记录。比如，向`/accounts/person`发送请求，就可以新增一条人员记录。

> ```
> $ curl -X PUT 'localhost:9200/accounts/person/1' -d '{  "user": "张三",  "title": "工程师",  "desc": "数据库管理"}' 
> ```

服务器返回的 JSON 对象，会给出 Index、Type、Id、Version 等信息。

> ```
> {  "_index":"accounts",  "_type":"person",  "_id":"1",  "_version":1,  "result":"created",  "_shards":{"total":2,"successful":1,"failed":0},  "created":true}
> ```

如果你仔细看，会发现请求路径是`/accounts/person/1`，最后的`1`是该条记录的 Id。它不一定是数字，任意字符串（比如`abc`）都可以。

新增记录的时候，也可以不指定 Id，这时要改成 POST 请求。

> ```
> $ curl -X POST 'localhost:9200/accounts/person' -d '{  "user": "李四",  "title": "工程师",  "desc": "系统管理"}'
> ```

上面代码中，向`/accounts/person`发出一个 POST 请求，添加一个记录。这时，服务器返回的 JSON 对象里面，`_id`字段就是一个随机字符串。

```
{
	"_index": "accounts",
	"_type": "person",
	"_id": "AV3qGfrC6jMbsbXb6k1p",
	"_version": 1,
	"result": "created",
	"_shards": {
		"total": 2,
		"successful": 1,
		"failed": 0
	},
	"created": true
}
```

注意，如果没有先创建 Index（这个例子是`accounts`），直接执行上面的命令，Elastic 也不会报错，而是直接生成指定的 Index。所以，打字的时候要小心，不要写错 Index 的名称。

### 5.2 查看记录

向`/Index/Type/Id`发出 GET 请求，就可以查看这条记录。

> ```
> $ curl 'localhost:9200/accounts/person/1?pretty=true'
> ```

上面代码请求查看`/accounts/person/1`这条记录，URL 的参数`pretty=true`表示以易读的格式返回。

返回的数据中，`found`字段表示查询成功，`_source`字段返回原始记录。

```
{
	"_index": "accounts",
	"_type": "person",
	"_id": "1",
	"_version": 1,
	"found": true,
	"_source": {
		"user": "张三",
		"title": "工程师",
		"desc": "数据库管理"
	}
}
```

如果 Id 不正确，就查不到数据，`found`字段就是`false`。

> ```
> $ curl 'localhost:9200/weather/beijing/abc?pretty=true' {  "_index" : "accounts",  "_type" : "person",  "_id" : "abc",  "found" : false}
> ```

### 5.3 删除记录

删除记录就是发出 DELETE 请求。

> ```
> $ curl -X DELETE 'localhost:9200/accounts/person/1'
> ```

这里先不要删除这条记录，后面还要用到。

### 5.4 更新记录

更新记录就是使用 PUT 请求，重新发送一次数据。

>```
> $ curl -X PUT 'localhost:9200/accounts/person/1' -d 
> '{    "user" : "张三",    "title" : "工程师",    "desc" : "数据库管理，软件开发"}'  
>```

上面代码中，我们将原始数据从"数据库管理"改成"数据库管理，软件开发"。 返回结果里面，有几个字段发生了变化。

> ```
> "_version" : 2,"result" : "updated","created" : false
> ```

可以看到，记录的 Id 没变，但是版本（version）从`1`变成`2`，操作类型（result）从`created`变成`updated`，`created`字段变成`false`，因为这次不是新建记录。

## 六、数据查询

### 6.1 返回所有记录

使用 GET 方法，直接请求`/Index/Type/_search`，就会返回所有记录。

```
> $ curl 'localhost:9200/accounts/person/_search' 
{
	"took": 2,
	"timed_out": false,
	"_shards": {
		"total": 5,
		"successful": 5,
		"failed": 0
	},
	"hits": {
		"total": 2,
		"max_score": 1.0,
		"hits": [{
			"_index": "accounts",
			"_type": "person",
			"_id": "AV3qGfrC6jMbsbXb6k1p",
			"_score": 1.0,
			"_source": {
				"user": "李四",
				"title": "工程师",
				"desc": "系统管理"
			}
		}, {
			"_index": "accounts",
			"_type": "person",
			"_id": "1",
			"_score": 1.0,
			"_source": {
				"user": "张三",
				"title": "工程师",
				"desc": "数据库管理，软件开发"
			}
		}]
	}
}
```

上面代码中，返回结果的`took`字段表示该操作的耗时（单位为毫秒），`timed_out`字段表示是否超时，`hits`字段表示命中的记录，里面子字段的含义如下。

> *   `total`：返回记录数，本例是2条。
> *   `max_score`：最高的匹配程度，本例是`1.0`。
> *   `hits`：返回的记录组成的数组。

返回的记录中，每条记录都有一个`_score`字段，表示匹配的程序，默认是按照这个字段降序排列。

### 6.2 全文搜索

Elastic 的查询非常特别，使用自己的查询语法，要求 GET 请求带有数据体。

```
> $ curl 'localhost:9200/accounts/person/_search'  -d 
'{  "query" : { "match" : { "desc" : "软件" }}}'
```

上面代码使用Match 查询，指定的匹配条件是`desc`字段里面包含"软件"这个词。返回结果如下。

```
{
	"took": 3,
	"timed_out": false,
	"_shards": {
		"total": 5,
		"successful": 5,
		"failed": 0
	},
	"hits": {
		"total": 1,
		"max_score": 0.28582606,
		"hits": [{
			"_index": "accounts",
			"_type": "person",
			"_id": "1",
			"_score": 0.28582606,
			"_source": {
				"user": "张三",
				"title": "工程师",
				"desc": "数据库管理，软件开发"
			}
		}]
	}
}
```

Elastic 默认一次返回10条结果，可以通过`size`字段改变这个设置。

```
> $ curl 'localhost:9200/accounts/person/_search'  -d 
'{  "query" : { "match" : { "desc" : "管理" }},  "size": 1}'
```

上面代码指定，每次只返回一条结果。

还可以通过`from`字段，指定位移。

 ```
> $ curl 'localhost:9200/accounts/person/_search'  -d 
'{  "query" : { "match" : { "desc" : "管理" }},  "from": 1,  "size": 1}'
 ```

上面代码指定，从位置1开始（默认是从位置0开始），只返回一条结果。

### 6.3 逻辑运算

如果有多个搜索关键字， Elastic 认为它们是`or`关系。

```
> $ curl 'localhost:9200/accounts/person/_search'  -d 
'{  "query" : { "match" : { "desc" : "软件 系统" }}}'
```

上面代码搜索的是`软件 or 系统`。

如果要执行多个关键词的`and`搜索，必须使用[布尔查询](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2F5.5%2Fquery-dsl-bool-query.html)。

```
> $ curl 'localhost:9200/accounts/person/_search'  -d 
{
	"query": {
		"bool": {
			"must": [{
				"match": {
					"desc": "软件"
				}
			}, {
				"match": {
					"desc": "系统"
				}
			}]
		}
	}
}
```

## 七、参考链接

*   [ElasticSearch 官方手册](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fgetting-started.html)
*   [A Practical Introduction to Elasticsearch](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fblog%2Fa-practical-introduction-to-elasticsearch)

（完）









![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-3a3865f474573947.png)





#### 一、前言

在开发网站/App项目的时候，通常需要搭建搜索服务。比如，新闻类应用需要检索标题/内容，社区类应用需要检索用户/帖子。

对于简单的需求，可以使用数据库的 LIKE 模糊搜索，示例：

> SELECT * FROM news WHERE title LIKE '%法拉利跑车%'

可以查询到所有标题含有 "法拉利跑车" 关键词的新闻，但是这种方式有明显的弊端：

> 1、模糊查询性能极低，当数据量庞大的时候，往往会使数据库服务中断；
>
> 2、无法查询相关的数据，只能严格在标题中匹配关键词。

因此，需要搭建专门提供搜索功能的服务，具备分词、全文检索等高级功能。 Solr 就是这样一款搜索引擎，可以让你快速搭建适用于自己业务的搜索服务。

#### 二、安装

到官网[http://lucene.apache.org/solr/](https://link.jianshu.com/?t=http://lucene.apache.org/solr/)下载安装包，解压并进入 Solr 目录：

> wget 'http://apache.website-solution.net/lucene/solr/6.2.0/solr-6.2.0.tgz'
>
> tar xvf solr-6.2.0.tgz
>
> cd solr-6.2.0

目录结构如下：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-ddbb880dd1a7bcb0.png)



Solr 6.2 目录结构



启动 Solr 服务之前，确认已经安装 Java 1.8 ：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-049501dade838caf.png)



查看 Java 版本



启动 Solr 服务：

> ./bin/solr start -m 1g

Solr 将默认监听 8983 端口，其中 -m 1g 指定分配给 JVM 的内存为 1 G。

在浏览器中访问 Solr 管理后台：

> http://127.0.0.1:8983/solr/#/





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-19bdf6ec1077db99.png)



Solr 管理后台



创建 Solr 应用：

> ./bin/solr create -c my_news

可以在 solr-6.2.0/server/solr 目录下生成 my_news 文件夹，结构如下：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-9911b7416917ca06.png)



my_news 目录结构



同时，可以在管理后台看到 my_news：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-81af0fb0b5d89edd.png)



管理后台



#### 三、创建索引

我们将从 MySQL 数据库中导入数据到 Solr 并建立索引。

首先，需要了解 Solr 中的两个概念： 字段(field) 和 字段类型(fieldType)，配置示例如下：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-cbc2ba3d84087319.png)



schema.xml 示例



field 指定一个字段的名称、是否索引/存储和字段类型。

fieldType 指定一个字段类型的名称以及在查询/索引的时候可能用到的分词插件。

将 solr-6.2.0\server\solr\my_news\conf 目录下默认的配置文件 managed-schema 重命名为 schema.xml 并加入新的 fieldType：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-2657cfb3507d1bae.png)



分词类型



在 my_news 目录下创建 lib 目录，将用到的分词插件 ik-analyzer-solr5-5.x.jar 加到 lib 目录，结构如下：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-3a3e436e33fa9311.png)



my_news 目录结构



在 Solr 安装目录下重启服务：

> ./bin/solr restart

可以在管理后台看到新加的类型：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-5609a84930ed96f0.png)



text_ik 类型



接下来创建和我们数据库字段对应的 field：title 和 content，类型选为 text_ik：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-a46bba01779c0701.png)



新建字段 title



将要导入数据的 MySQL 数据库表结构：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-ab4dec5179c0f5c3.png)





编辑 conf/solrconfig.xml 文件，加入类库和数据库配置：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-e3dc609b92f395a1.png)



类库







![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-7a145baf9aa36599.png)



dataimport config



同时新建数据库连接配置文件 conf/db-mysql-config.xml ，内容如下：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-edc3bb352c36e8c2.png)



数据库配置文件



将数据库连接组件 mysql-connector-java-5.1.39-bin.jar 放到 lib 目录下，重启 Solr，访问管理后台，执行全量导入数据：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-a4462a20df0716a2.png)



全量导入数据



创建定时更新脚本：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-f437d561069eedd2.png)



定时更新脚本



加入到定时任务，每5分钟增量更新一次索引：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-73d93e996f0a132c.png)



定时任务



在 Solr 管理后台测试搜索结果：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-9f003409af70ae7a.png)



分词搜索结果



至此，基本的搜索引擎搭建完毕，外部应用只需通过 http 协议提供查询参数，就可以获取搜索结果。

#### 四、搜索干预

通常需要对搜索结果进行人工干预，比如编辑推荐、竞价排名或者屏蔽搜索结果。Solr 已经内置了 QueryElevationComponent 插件，可以从配置文件中获取搜索关键词对应的干预列表，并将干预结果排在搜索结果的前面。

在 solrconfig.xml 文件中，可以看到：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-09494ec9437338cd.png)



干预其请求配置



定义了搜索组件 elevator，应用在 /elevate 的搜索请求中，干预结果的配置文件在 solrconfig.xml 同目录下的 elevate.xml 中，干预配置示例：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-3f2587b4bb0dcee3.png)





重启 Solr ，当搜索 "关键词" 的时候，id 为 1和 4 的文档将出现在前面，同时 id = 3 的文档被排除在结果之外，可以看到，没有干预的时候，搜索结果为：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-b12a6ec2234beaef.png)



无干预结果



当有搜索干预的时候：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-f57a54656abc2f62.png)



干预结果



通过配置文件干预搜索结果，虽然简单，但是每次更新都要重启 Solr 才能生效，稍显麻烦，我们可以仿照 QueryElevationComponent 类，开发自己的干预组件，例如:从 Redis 中读取干预配置。

#### 五、中文分词

中文的搜索质量，和分词的效果息息相关，可以在 Solr 管理后台测试分词：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-bc4dfa9a4801846f.png)



分词结果测试



上例可以看到，使用[IKAnalyzer](https://blog.csdn.net/a724888/article/details/80993677)分词插件，对 “北京科技大学” 分词的测试结果。当用户搜索 “北京”、“科技大学”、“科技大”、“科技”、“大学” 这些关键词的时候，都会搜索到文本内容含 “北京科技大学” 的文档。

常用的中文分词插件有 IKAnalyzer、mmseg4j和 Solr 自带的 smartcn 等，分词效果各有优劣，具体选择哪个，可以根据自己的业务场景，分别测试效果再选择。

分词插件一般都有自己的默认词库和扩展词库，默认词库包含了绝大多数常用的中文词语。如果默认词库无法满足你的需求，比如某些专业领域的词汇，可以在扩展词库中手动添加，这样分词插件就能识别新词语了。





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-ac9e935a3b98661c.png)



分词插件扩展词库配置示例



分词插件还可以指定停止词库，将某些无意义的词汇剔出分词结果，比如：“的”、“哼” 等，例如：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/19687-34e025db9e4db451.png)



去除无意义的词



#### 六、总结

以上介绍了 Solr 最常用的一些功能，Solr 本身还有很多其他丰富的功能，比如分布式部署。

希望对你有所帮助。

#### 七、附录

1、参考资料：

[https://wiki.apache.org/solr/](https://link.jianshu.com/?t=https://wiki.apache.org/solr/)

[http://lucene.apache.org/solr/quickstart.html](https://link.jianshu.com/?t=http://lucene.apache.org/solr/quickstart.html)

[https://cwiki.apache.org/confluence/display/solr/Apache+Solr+Reference+Guide](https://link.jianshu.com/?t=https://cwiki.apache.org/confluence/display/solr/Apache+Solr+Reference+Guide)

2、上述 Demo 中用到的所有配置文件、Jar 包：

[https://github.com/Ceelog/OpenSchool/blob/master/my_news.zip](https://link.jianshu.com/?t=https://github.com/Ceelog/OpenSchool/blob/master/my_news.zip)

3、还有疑问？联系作者微博/微信[@Ceelog](https://link.jianshu.com/?t=http://weibo.com/ceelog/)



# 搜索引擎选型整理：Elasticsearch vs Solr



> 本文首发于[我的博客](https://link.juejin.im/?target=https%3A%2F%2Fblog.kittypanic.com%2F)
> 原文链接：[Elasticsearch 与 Solr 的比较](https://link.juejin.im/?target=https%3A%2F%2Fblog.kittypanic.com%2Felastic_vs_solr%2F)

## Elasticsearch简介

Elasticsearch是一个实时的分布式搜索和分析引擎。它可以帮助你用前所未有的速度去处理大规模数据。

它可以用于全文搜索，结构化搜索以及分析，当然你也可以将这三者进行组合。

Elasticsearch是一个建立在全文搜索引擎 Apache Lucene(TM) 基础上的搜索引擎，可以说Lucene是当今最先进，最高效的全功能开源搜索引擎框架。

但是Lucene只是一个框架，要充分利用它的功能，需要使用JAVA，并且在程序中集成Lucene。需要很多的学习了解，才能明白它是如何运行的，Lucene确实非常复杂。

Elasticsearch使用Lucene作为内部引擎，但是在使用它做全文搜索时，只需要使用统一开发好的API即可，而不需要了解其背后复杂的Lucene的运行原理。

当然Elasticsearch并不仅仅是Lucene这么简单，它不但包括了全文搜索功能，还可以进行以下工作:

*   分布式实时文件存储，并将每一个字段都编入索引，使其可以被搜索。

*   实时分析的分布式搜索引擎。

*   可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据。

这么多的功能被集成到一台服务器上，你可以轻松地通过客户端或者任何你喜欢的程序语言与ES的RESTful API进行交流。

Elasticsearch的上手是非常简单的。它附带了很多非常合理的默认值，这让初学者很好地避免一上手就要面对复杂的理论，

它安装好了就可以使用了，用很小的学习成本就可以变得很有生产力。

随着越学越深入，还可以利用Elasticsearch更多高级的功能，整个引擎可以很灵活地进行配置。可以根据自身需求来定制属于自己的Elasticsearch。

使用案例：

*   维基百科使用Elasticsearch来进行全文搜做并高亮显示关键词，以及提供search-as-you-type、did-you-mean等搜索建议功能。

*   英国卫报使用Elasticsearch来处理访客日志，以便能将公众对不同文章的反应实时地反馈给各位编辑。

*   StackOverflow将全文搜索与地理位置和相关信息进行结合，以提供more-like-this相关问题的展现。

*   GitHub使用Elasticsearch来检索超过1300亿行代码。

*   每天，Goldman Sachs使用它来处理5TB数据的索引，还有很多投行使用它来分析股票市场的变动。

但是Elasticsearch并不只是面向大型企业的，它还帮助了很多类似DataDog以及Klout的创业公司进行了功能的扩展。

## Elasticsearch的优缺点

### 优点

1.  Elasticsearch是分布式的。不需要其他组件，分发是实时的，被叫做"Push replication"。

*   Elasticsearch 完全支持 Apache Lucene 的接近实时的搜索。
*   处理多租户（[multitenancy](https://link.juejin.im/?target=http%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMultitenancy)）不需要特殊配置，而Solr则需要更多的高级设置。
*   Elasticsearch 采用 Gateway 的概念，使得完备份更加简单。
*   各节点组成对等的网络结构，某些节点出现故障时会自动分配其他节点代替其进行工作。

### 缺点

1.  只有一名开发者（当前Elasticsearch GitHub组织已经不只如此，已经有了相当活跃的维护者）

*   还不够自动（不适合当前新的Index Warmup API）

## Solr简介

Solr（读作“solar”）是Apache Lucene项目的开源企业搜索平台。其主要功能包括全文检索、命中标示、分面搜索、动态聚类、数据库集成，以及富文本（如Word、PDF）的处理。Solr是高度可扩展的，并提供了分布式搜索和索引复制。Solr是最流行的企业级搜索引擎，Solr4 还增加了NoSQL支持。

Solr是用Java编写、运行在Servlet容器（如 Apache Tomcat 或Jetty）的一个独立的全文搜索服务器。 Solr采用了 Lucene Java 搜索库为核心的全文索引和搜索，并具有类似REST的HTTP/XML和JSON的API。Solr强大的外部配置功能使得无需进行Java编码，便可对其进行调整以适应多种类型的应用程序。Solr有一个插件架构，以支持更多的高级定制。

因为2010年 Apache Lucene 和 Apache Solr 项目合并，两个项目是由同一个Apache软件基金会开发团队制作实现的。提到技术或产品时，Lucene/Solr或Solr/Lucene是一样的。

## Solr的优缺点

### 优点

1.  Solr有一个更大、更成熟的用户、开发和贡献者社区。

*   支持添加多种格式的索引，如：HTML、PDF、微软 Office 系列软件格式以及 JSON、XML、CSV 等纯文本格式。
*   Solr比较成熟、稳定。
*   不考虑建索引的同时进行搜索，速度更快。

### 缺点

1.  建立索引时，搜索效率下降，实时索引搜索效率不高。

## Elasticsearch与Solr的比较

当单纯的对已有数据进行搜索时，Solr更快。

![](https://user-gold-cdn.xitu.io/2016/12/30/d5944021d5ad35ab6c62e4e56ae21e22.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)Search Fesh Index While Idle

当实时建立索引时, Solr会产生io阻塞，查询性能较差, Elasticsearch具有明显的优势。

![](https://user-gold-cdn.xitu.io/2016/12/30/8c908279adf11197d4631c95915fc167.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)search_fresh_index_while_indexing

随着数据量的增加，Solr的搜索效率会变得更低，而Elasticsearch却没有明显的变化。

![](https://user-gold-cdn.xitu.io/2016/12/30/931c2f218ae2c0c145279507b4eb0e7b.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)search_fresh_index_while_indexing

综上所述，Solr的架构不适合实时搜索的应用。

## 实际生产环境测试

下图为将搜索引擎从Solr转到Elasticsearch以后的平均查询速度有了50倍的提升。

![](https://user-gold-cdn.xitu.io/2016/12/30/76c108b2590ef4835b114dec4a018b8a.jpg?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)average_execution_time

## Elasticsearch 与 Solr 的比较总结

*   二者安装都很简单；

*   Solr 利用 Zookeeper 进行分布式管理，而 Elasticsearch 自身带有分布式协调管理功能;
*   Solr 支持更多格式的数据，而 Elasticsearch 仅支持json文件格式；
*   Solr 官方提供的功能更多，而 Elasticsearch 本身更注重于核心功能，高级功能多有第三方插件提供；
*   Solr 在传统的搜索应用中表现好于 Elasticsearch，但在处理实时搜索应用时效率明显低于 Elasticsearch。

Solr 是传统搜索应用的有力解决方案，但 Elasticsearch 更适用于新兴的实时搜索应用。

## 其他基于Lucene的开源搜索引擎解决方案

1. 直接使用[Lucene](https://link.juejin.im/?target=http%3A%2F%2Flucene.apache.org)

   说明：Lucene 是一个 JAVA 搜索类库，它本身并不是一个完整的解决方案，需要额外的开发工作。

   优点：成熟的解决方案，有很多的成功案例。apache 顶级项目，正在持续快速的进步。庞大而活跃的开发社区，大量的开发人员。它只是一个类库，有足够的定制和优化空间：经过简单定制，就可以满足绝大部分常见的需求；经过优化，可以支持 10亿+ 量级的搜索。

   缺点：需要额外的开发工作。所有的扩展，分布式，可靠性等都需要自己实现；非实时，从建索引到可以搜索中间有一个时间延迟，而当前的“近实时”(Lucene Near Real Time search)搜索方案的可扩展性有待进一步完善

* [Katta](https://link.juejin.im/?target=http%3A%2F%2Fkatta.sourceforge.net)

  说明：基于 Lucene 的，支持分布式，可扩展，具有容错功能，准实时的搜索方案。

  优点：开箱即用，可以与 Hadoop 配合实现分布式。具备扩展和容错机制。

  缺点：只是搜索方案，建索引部分还是需要自己实现。在搜索功能上，只实现了最基本的需求。成功案例较少，项目的成熟度稍微差一些。因为需要支持分布式，对于一些复杂的查询需求，定制的难度会比较大。

* [Hadoop contrib/index](https://link.juejin.im/?target=http%3A%2F%2Fsvn.apache.org%2Frepos%2Fasf%2Fhadoop%2Fmapreduce%2Ftrunk%2Fsrc%2Fcontrib%2Findex%2FREADME)

  说明：Map/Reduce 模式的，分布式建索引方案，可以跟 Katta 配合使用。

  优点：分布式建索引，具备可扩展性。

  缺点：只是建索引方案，不包括搜索实现。工作在批处理模式，对实时搜索的支持不佳。

* [LinkedIn 的开源方案](https://link.juejin.im/?target=http%3A%2F%2Fsna-projects.com)

  说明：基于 Lucene 的一系列解决方案，包括 准实时搜索 zoie ，facet 搜索实现 bobo ，机器学习算法 decomposer ，摘要存储库 krati ，数据库模式包装 sensei 等等

  优点：经过验证的解决方案，支持分布式，可扩展，丰富的功能实现

  缺点：与 linkedin 公司的联系太紧密，可定制性比较差

* [Lucandra](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Ftjake%2FLucandra)

  说明：基于 Lucene，索引存在 cassandra 数据库中

  优点：参考 cassandra 的优点

  缺点：参考 cassandra 的缺点。另外，这只是一个 demo，没有经过大量验证

* [HBasene](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fakkumar%2Fhbasene)

  说明：基于 Lucene，索引存在 HBase 数据库中

  优点：参考 HBase 的优点

  缺点：参考 HBase 的缺点。另外，在实现中，lucene terms 是存成行，但每个 term 对应的 posting lists 是以列的方式存储的。随着单个 term 的 posting lists 的增大，查询时的速度受到的影响会非常大





