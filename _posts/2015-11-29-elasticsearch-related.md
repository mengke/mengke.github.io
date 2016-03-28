---
layout: post
title: "Elasticsearch Related"
category: Lucene
tags: [lucene, elasticsearch, searching]
---

以下所有章节都基于Elasticsearch 2.0

## Elasticsearch Transport Wires

## 中文分词引入

## Indexes

### 创建一个索引

{% highlight %}
curl -XPUT 'localhost:9200/customer?pretty'

// 结果返回
{
  "acknowledged" : true
}
{% endhighlight %}

上面的命令会创建一个名为`customer`的索引, URL后面添加`pretty`, 结果返回的json会经过格式化

### 查看所有索引

{% highlight %}
curl 'localhost:9200/_cat/indices?v'

// 结果返回
health index    pri rep docs.count docs.deleted store.size pri.store.size
yellow customer   5   1          0            0       495b           495b
{% endhighlight %}

上面命令告诉我们现在总共有1个索引名字叫做`customer`, 其中包含的Document总数, 被删掉的Document数以及占用空间等信息.

### 向索引中添加/更新一个Document

* 显式指定ID

{% highlight %}
curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
{
  "name": "John Doe"
}'

// 结果返回
{
  "_index" : "customer",
  "_type" : "external",
  "_id" : "1",
  "_version" : 1,
  "created" : true
}
{% endhighlight %}

上面命令会向`customer`索引中添加一个类型为`external`的Document, 这个Document的ID为1, 而且从结果返回中也可以看到索引, 类型, ID, 版本等信息

<div class="bs-callout bs-callout-info">
	<p>如果需要更新这个ID为1的Document, 只需使用同样的命令, ES会将请求覆盖掉这个Document</p>
</div>

* 不指定ID

{% highlight %}
curl -XPOST 'localhost:9200/customer/external?pretty' -d '
{
  "name": "Jane Doe"
}'
{% endhighlight %}

上面的命令没有指定ID, ES会自动为这个Document生成一个ID, 这个ID会在结果返回中返回

<div class="bs-callout bs-callout-info">
	<p>注意该命令使用的是POST动词, 而不是PUT</p>
</div>

### 删除一个Document

{% highlight %}
curl -XDELETE 'localhost:9200/customer/external/2?pretty'
{% endhighlight %}

该命令会在`customer`索引的`external`类型删除ID为2的Document

### 根据ID获取一个Document

{% highlight %}
curl -XGET 'localhost:9200/customer/external/1?pretty'

// 结果返回
{
  "_index" : "customer",
  "_type" : "external",
  "_id" : "1",
  "_version" : 1,
  "found" : true, "_source" : { "name": "John Doe" }
}
{% endhighlight %}

通过上面结果返回的`found`字段表示该查询是否命中, `_source`字段中包含了Document的内容

### 删除一个索引

{% highlight %}
curl -XDELETE 'localhost:9200/customer?pretty'

// 结果返回
{
  "acknowledged" : true
}
{% endhighlight %}

该命令会删除整个`customer`索引

## ES索引切片(Shards)工作原理

在ES中, 每一个索引对应于一个或多个切片, 索引在ES中是一个逻辑空间, 而切片是真正的数据存放的物理空间, 它们散落在集群的各个节点中. 与ES交互的个体并不知道切片的存在, 他们直接与索引进行交互,

当我们索引一个文档时, 文档存储在一个单独的主切片中. 那么当我们索引一个文档时, 我们如何该文档处于哪个主切片呢?

它可以通过以下公式来决定

{% highlight %}
shard = hash(routing) % number_of_primary_shards
{% endhighlight %}

其中`routing`值默认是Document的`_id`, 当然也可以自己设置. 该值会被传到`hash`函数中获得一个hash值, 将该hash值与`number_of_primary_shards`取模, 得到的`shard`值介于0 ~ number_of_primary_shards-1之间, 该document会存储在该值对应的shard中.
所以, 一个索引的主切片数(`number_of_primary_shards`)必须在索引创建时指定, 并且不能在中途做变更, 因为一旦更新后, 之前的documents有可能无法被正常找到



