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

{% highlight json %}
curl -XPUT 'localhost:9200/customer?pretty'

// 结果返回
{
  "acknowledged" : true
}
{% endhighlight %}

上面的命令会创建一个名为`customer`的索引, URL后面添加`pretty`, 结果返回的json会经过格式化

### 查看所有索引

{% highlight json %}
curl 'localhost:9200/_cat/indices?v'

// 结果返回
health index    pri rep docs.count docs.deleted store.size pri.store.size
yellow customer   5   1          0            0       495b           495b
{% endhighlight %}

上面命令告诉我们现在总共有1个索引名字叫做`customer`, 其中包含的Document总数, 被删掉的Document数以及占用空间等信息.

### 向索引中添加/更新一个Document

* 显式指定ID

{% highlight json %}
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

{% highlight json %}
curl -XPOST 'localhost:9200/customer/external?pretty' -d '
{
  "name": "John Doe"
}'
{% endhighlight %}

上面的命令没有指定ID, ES会自动为这个Document生成一个ID, 这个ID会在结果返回中返回

<div class="bs-callout bs-callout-info">
	<p>注意该命令使用的是POST动词, 而不是PUT</p>
</div>

### 删除一个Document

{% highlight json %}
curl -XDELETE 'localhost:9200/customer/external/2?pretty'
{% endhighlight %}

该命令会在`customer`索引的`external`类型删除ID为2的Document

### 根据ID获取一个Document

{% highlight json %}
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

{% highlight json %}
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

<div class="bs-callout bs-callout-info">
	<p>shard = hash(routing) % number_of_primary_shards</p>
</div>

其中`routing`值默认是Document的`_id`, 当然也可以自己设置. 该值会被传到`hash`函数中获得一个hash值, 将该hash值与`number_of_primary_shards`取模, 得到的`shard`值介于0 ~ number_of_primary_shards-1之间, 该document会存储在该值对应的shard中.
所以, 一个索引的主切片数(`number_of_primary_shards`)必须在索引创建时指定, 并且不能在中途做变更, 因为一旦更新后, 之前的documents有可能无法被正常找到

## 标题没想好, 后面看内容定(基于5.3.X)

### ES启动过程

* 创建PID文件
* 检测Lucene版本
* 如果是*nux, 检测是否是root账户执行
* 根据配置, 启用`system call filter`, `mlockall`, 设置启动ES进程用户的最大进程数和最大虚拟内存数等操作
* 做一些启动前环境检测
** 检测jvm启动时初始堆内存与最大堆内存是否一致 - 由于初始内存和最大堆内存不一致的话, 在系统运行过程中根据内存用量会重新分配jvm占用内存大小, 从而导致jvm暂停
** 检测File descriptors最大数
** 检测内存锁 - JVM在做GC时, 有可能会将堆内存中的数据换到硬盘上, 而在进行后续与数据相关的操作时必须再将硬盘中的数据换回内存, 导致大量的磁盘操作
** 最大线程数检测
** 最大虚拟内存检测 - ES和Lucene使用`mmap`来将索引文件映射到内存加速索引的访问
** 和上一点类似, 为了让ES更有效率的使用`mmap`, 需要检测ES进程启动用户所能控制的最大内存映射大小
** 检测jvm版本不是client版
** 检测GC不能使用`-XX:+UseSerialGC`, SerialGC适合用于单核, 或小内存的机器
** 检测是否启用`system call filter`, 出于考虑, 启用`system call filter`来避免系统通过ES执行一些非法系统调用
** 检测是否设置了JVM启动参数`OnError`和`OnOutOfMemoryError`, 这两个标识允许JVM遇到Error后执行任意的命令, 这与上一项有冲突, 所以必须保证这两项参数不能被启用
** jdk8早期版本中G1GC会导致索引损坏, 检测当G1GC启用后, 没有使用这些版本(prior to Update 40)
* 在每个节点根据该节点设置的数据目录`path.data`(可能有多个)尝试获取节点锁, 如果其中一个数据目录获取失败, 则重新建立另外一个锁目录, 直至超过配置数node.max_local_storage_nodes           NodeEnvironment
* 载入配置中指定的`${path.home}/plugins`和`${path.home}/modules`目录中的所有插件, 并实例化它们的插件实例, 例如`public class Netty4Plugin extends Plugin implements NetworkPlugin { ... }`            PluginsService
* 创建一个含有操作类型的线程池, 每种操作类型都会映射到一个独立的线程池, 这些线程池根据实际用途拥有不同的线程数, 队列长度, 任务舍弃策略等   ThreadPool
* 利用插件构建不同服务, 例如NetworkService, ClusterService, TransportService等, 并启动这些服务


NetworkPlugin启动时, 会根据配置(默认netty4)启动指定的TCP和HTTP服务   Netty4Transport.doStart

### Index操作/Bulk操作

index操作合并在bulk操作中执行, 
首先请求中携带了pipeline参数的话, 检测当前节点是否是ingest节点, 如果是, 则将请求中的doc使用pipeline参数指定的pipeline-id进行预处理, 如果不是则将请求转发到随机的ingest节点上.  TransportBulkAction#doExecute
如果集群中没有请求中的Indices, 并且设置了允许自动创建的话, 则对缺失的索引进行创建.
根据请求中doc的计算shardId, 并根据shardId将批量请求中的请求项进行分组, 将每一组的请求项列表组成一个新的BulkShardRequest, 并统一都转发给TransportShardBulkAction处理                   TransportBulkAction$BulkOperation#doRun
根据shardId找到对应的主切片所在的节点, 如果当前节点为主切片所在的节点, 则直接在本地处理请求, 否则向远程节点发送请求                   TransportReplicationAction$ReroutePhase#doRun
请求发送:
本地请求: 根据具体请求操作类型决定线程池类型(index对应index的线程池, bulk对应bulk的线程池), TransportService#sendLocalRequest
远程请求: 会将请求转发给主切片所在的节点, 通信使用TCP协议. 在ES启动时会根据transport插件(默认netty4)来决定使用哪种实现, 并且在启动时还会建立与各个节点的连接, 并存放在Map中, 以DiscoveryNode为Key, 发送请求时, 根据被请求节点的DiscoveryNode对象获取连接, 并在该连接写入请求     TcpTransport#connectToNode
请求处理:
本地请求: 直接将请求通过RequestHandlerRegistry中注册的handler进行处理;
远程请求: 将请求序列化为一个字节数组, 并将该字节数组使用比如Netty4的Channel对象发送出去, 在上述启动过程中, 请求会经过netty4插件使用RequestHandlerRegistry中注册的handler进行处理, 定位handler的action在请求字节数组中获取   TcpTransport#messageReceived -> TcpTransport#handleRequest

在主切片完成索引写入操作后, 需要将该切片对应的复制切片置为过期数据, 然后向这些复制切片所在的节点发送复制请求  ReplicationOperation#performOnReplicas

实际索引操作: TransportReplicationAction$AsyncPrimaryAction#doRun -> onResponse -> ReplicationOperation#execute -> TransportWriteAction#shardOperationOnPrimary -> TransportShardBulkAction#shardOperationOnPrimary -> TransportShardBulkAction#executeIndexRequestOnPrimary -> IndexShard#index -> Engine#index(调用lucene API写入索引IndexWriter#addDocuments|IndexWriter#updateDocuments)

### 搜索

拿非跨集群的搜索举例, 跨集群搜索与集群内搜索处理方式唯一的不同是跨集群搜索需要建立与其他集群的连接, 而这个连接可以看做集群内的一个到任意DiscoveryNode的连接   TransportSearchAction#doExecute -> RemoteClusterService#collectSearchShards

根据请求中indies参数解析出具体的索引列表, 解析indies中带有通配符或日期函数的表达式, 将它们映射成为一系列具体的索引, 检查是否有索引别名与参数中的表达式所匹配, 一一将它们解析出来   IndexNameExpressionResolver#concreteIndices
获取indies参数中包含的索引别名关联的Filters, 这些过滤器在执行搜索操作前合并到查询条件的filter上下文中                   TransportSearchAction#buildPerIndexAliasFilter
根据请求中indies参数以及routing参数与设置索引别名时指定的routing计算每个索引名对应的routing列表的关系              IndexNameExpressionResolver#resolveSearchRouting
通过上一步的结果计算每一个分片的迭代器, 这个分片迭代器会随机选择该分片中任意一个副本                                   OperationRouting#searchShards
针对每一个分片, 依次获取该分片(副本)所在节点的连接, 然后通过该连接发送分片查询请求                            SearchTransportService#sendExecuteQuery
节点在接收到查询请求后, 在该节点执行查询操作, 内部根据查询请求记录条数以及查询结果等做了一些优化, 这一步完成后, 会得到该分片下符合查询条件记录条数以及查询区间中docId以及分数                   QueryPhase#execute
当所有分片查询完成后, 根据分数将所有分片的返回结果进行合并, 重新排序   AbstractSearchAsyncAction$FetchPhase#run -> SearchPhaseController#sortDocs
然后分别统计每个分片下的docId列表, 再向这些分片所在节点发送fetch请求, 该请求包含位于这个分片下所有符合查询条件的docId              SearchPhaseController#fillDocIdsToLoad -> AbstractSearchAsyncAction$FetchPhase#executeFetch
该节点获取到fetch请求后, 调用lucene进行获取这些doc的实际内容                      FetchPhase#execute
最后, 在上述步骤完成后, 将query步骤产生的目标结果docIds,分数等和fetch步骤中获取的doc的实际内容做合并, 并返回给请求方    AbstractSearchAsyncAction#sendResponseAsync -> SearchPhaseController#merge

在上面步骤中假定search_type指定为默认的query_then_fetch, 除了该默认设置选项, 还有dfs_query_then_fetch项, 它和默认选项的唯一区别就是在执行query阶段之前向各个索引获取额外的一些信息用来进行计分(TF/IDF), 由于默认项query_then_fetch在计分时, 计算TF和IDF时都是在单分片范围内计算的, 使用dfs_query_then_fetch选项在整个索引下所有分片范围内可以获得精确的TF/IDF值

### Discovery

要理清Discovery的工作方式, 我们假定有一个集群里包含4个节点, 并且设置`discovery.zen.minimum_master_nodes`为3(4/2 + 1), `discovery.zen.ping.unicast.hosts`都为`["node1", "node2", "node3", "node4"]`

<div class="bs-callout bs-callout-info">
  <p>master node是唯一一个能够更新集群状态节点, 当master node将集群状态从A改为B以后, 会将B广播给其他节点, 其他节点在收到广播后, 会告知master node已收到, 但是不会马上更新自己的集群状态; 如果master node在一段时间<span class="text-danger">discovery.zen.commit_timeout</span>内未收到至少<span class="text-danger">discovery.zen.minimum_master_nodes</span>个节点的回馈的话, 此次更新会回退</p>
  <p>一旦收到足够的回馈, 此次变更会被提交, 并发送消息给所有节点, 所有节点修改它们内部的集群状态; master node在执行队列中下一次集群状态变更之前会等待所有节点的修改回复</p>
</div>

假设我们启动第一个ES节点叫做node1, 上面提到的线程池中的generic提供一个线程来开始discovery的过程      JoinThreadControl$JoinThreadControl#startNewThreadIfNotRunning
该节点依次向配置`discovery.zen.ping.unicast.hosts`中节点发送ping请求, 每个ping请求会先后发送3次, 对于当前的例子, node1接到ping请求返回后, 并进行过滤(排除自身)处理, node1没有感知到任何其他节点   UnicastZenPing#ping
node1检查ping请求返回, 没有发现master节点, 并将自身加到候选队列, 检查是否有足够的候选节点, 在这个例子中是3个, 由于目前并不满足, 所以会等待一定时间后, 继续后续其他的流程                        ZenDiscovery#findMaster
接下来启动第二个节点node2, 依然是按照配置依次发送ping请求, 不同于第一个节点的是, 这一次node2得到的ping返回并排除自身后为node1
node2节点依次遍历ping返回仍然没有master节点, 将node1和node2加入候选队列后, 仍然不够3个, 所以node2会等待一定时间后, 继续后续其他的流程
node3节点启动后得到的ping返回为node1, node2, node3, 这时将它们加入候选队列后, 满足了最小的master候选个数, 这时在node3上选择出一个节点作为master, 假定是node2, 其他两个节点目前均得到3个节点的返回, 均选择node2作为master
现在3个节点均继续执行, node1和node3未被选中, 所以会与node2建立连接并发送joining请求                                       ZenDiscovery#joinElectedMaster
node2接收到一个joining请求, 会维护自己内存的joins数加1, 直到有足够的joins(discovery.zen.minimum_master_nodes - 1), 该节点会和加入它的节点建立连接     NodeJoinController#handleJoinRequest
master节点(node2)将最新的集群状态(v1)发布给其余节点, 其余节点会返回ack消息给master, master接收到所有节点的ack消息后才会向其余节点发送commit消息, node1和node3接收到commit消息后, 会应用master请求的更新的集群状态v1, 并且会定期向master节点发送消息, 确认master节点正常工作  MasterFaultDetection#restart
之后master节点会每过一段时间向这些节点发送ping消息, 以确认这些节点是否正常工作            NodesFaultDetection#updateNodesAndPing
待上述步骤完成后, 假定node4开始启动, 同样需要向其配置的hosts节点依次发送ping消息, 与前3个节点不同的是, node4接收到的ping返回包含了node2是master的信息, 因此node4可以直接加入以node2为master的集群
node2接收到node4的joining请求后, 由于选master的过程已经结束了, 所以node2提交了一个集群状态更新的任务, 并提交给所有涉及到的节点(包括node4), 最后node2将node4纳入故障监测列表                                  NodeJoinController#handleJoinRequest

下图描述了Discovery过程的线程模型

<img src="/assets/img/es_discovery_thread_model.png" class="img-thumbnail">

从节点之间的角度来描述Discovery过程如下图

<img src="/assets/img/es_discovery_nodes.png" class="img-thumbnail">

## 值得注意的一些功能

### ingest node

### 

## 从ES集群看分布式系统

### 数据隔离(?)

ES集群中每个节点只保存单独的一份数据, 该节点也只负责处理自己负责的这份数据

## 对ES工作感到困惑的点

### 索引时如何将对应的文档分配到某个分片中, 并将其复制到对应的副本中

### 搜索时如何将若干个分片中符合条件的结果组合

### 默认ZenDiscovery如何工作, 如何能够感知到新节点, 当一个节点down掉后, 如何保证ES还能正常提供服务; 当master down掉后, 如何重新选举新master