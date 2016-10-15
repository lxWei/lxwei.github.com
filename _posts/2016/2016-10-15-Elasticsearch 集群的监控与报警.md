---
layout: post
comments: true
date: 2016-10-15 
title: Elasticsearch 集群的监控与报警
categories:  
- Elasticsearch
---
前段时间和运维小伙伴一起完善了公司 Elasticsearch 集群的监控和报警，整理成此文，涉及到公司架构和系统信息的不便透露的，都隐去了。

# 系统架构

谈监控之前，先看看我们的系统架构，下图是个简化的搜索架构：

![搜索系统架构][1]

系统分两部分：

1. 蓝色部分，表示用户查询部分，客户端请求到达Tengine，Tengine将请求转发到AS(Advanced Search)，AS将请求转发到Elasticsearch，Elasticsearch查询，然后将结果原路返回。其中，AS是我们自主研发的，用于Query验证、改写、记日志、相关性计算等等。在这里，AS和ES都是一个集群。
2. 绿色部分，表示实时同步部分，系统日志、DB数据等通过消息队列到到我们的Sync程序，Sync将数据实时写入到ES集群。

上图覆盖了最重要的两个部分——实时索引与查询，全量索引仅需周期性进行一次，并未体现在此图中。对搜索集群添加监控，主要是针对图中各组件添加监控。

# 具体监控

**基础监控**

基础监控主要是一些常用的运维监控，主要包括：

* CPU 使用率
* IO
* 内存
* load
* 磁盘空间、INode
* 重传率
* fd
* ntp
* coredump
* JVM
* ...

这部分，运维都有很丰富的经验，不用操心。

**网关监控**

网关监控，主要关注两个指标，RT和状态吗，当RT > 1s 或者出现5XX的时候，进行报警。

**AS监控**

1. 进程监控，当进程数发生变化时，报警；
2. 日志监控，当日志出现ERROR时，报警。

**Elasticsearch 进程监控**

1. 进程监控，进程数发生变化时，报警。
2. ES 产生ERROR日志时，报警。

**Elasticsearch指标监控**

Elasticsearch 提供了丰富的API接口以便于采集集群、node、index等状态指标，可以根据这些指标做监控和报警。我们主要在集群级别、node 级别、index 级别三个层次进行采集和监控报警。

下面是一些主要关注的指标。

一、集群级别监控，采用```GET _cluster/health```，返回数据如下：

```json
{
   "cluster_name": "cluster_name_1",
   "status": "green",
   "timed_out": false,
   "number_of_nodes": 6,
   "number_of_data_nodes": 4,
   "active_primary_shards": 38,
   "active_shards": 68,
   "relocating_shards": 0,
   "initializing_shards": 0,
   "unassigned_shards": 0
}
```

对于我们来说，一般重点关注**集群状态 *status***以及其中的node数量。当集群状态变为yellow或red时报警。

另外，Elasticsearch 提供API对集群进行修改，而且，Elasticsearch 没有提供有效的权限控制，从而，也给我们带来了风险，所以，我们还加了一层监控，当集群配置发生变化时，报警。

二、node级别的监控，采用```GET _nodes/stats```接口取回node相关监控数据。

![node_stats][2]

大部分数据在基础监控里已经做了，所以，这里主要关注红框部分——indices、thread_pool和fielddata_breaker三部分。

indices部分如下：

```json
       "indices": {
            "docs": {
               "count": 3344,
               "deleted": 983
            },
            "store": {
               "size_in_bytes": 6529920,
               "throttle_time_in_millis": 2263
            },
            "indexing": {
               "index_total": 114516,
               "index_time_in_millis": 66226,
               "index_current": 969,
               "delete_total": 802,
               "delete_time_in_millis": 93,
               "delete_current": 0
            },
            "get": {
               "total": 449,
               "time_in_millis": 13,
               "exists_total": 571,
               "exists_time_in_millis": 1847,
               "missing_total": 878,
               "missing_time_in_millis": 56,
               "current": 0
            },
            "search": {
               "open_contexts": 0,
               "query_total": 3419,
               "query_time_in_millis": 262,
               "query_current": 0,
               "fetch_total": 319,
               "fetch_time_in_millis": 189,
               "fetch_current": 0
            },
            "merges": {
               "current": 1,
               "current_docs": 175,
               "current_size_in_bytes": 374,
               "total": 557,
               "total_time_in_millis": 77614,
               "total_docs": 3878,
               "total_size_in_bytes": 9545
            },
            "refresh": {
               "total": 4036,
               "total_time_in_millis": 3788
            },
            "flush": {
               "total": 07,
               "total_time_in_millis": 03
            },
            "warmer": {
               "current": 0,
               "total": 45,
               "total_time_in_millis": 233
            },
            "filter_cache": {
               "memory_size_in_bytes": 124,
               "evictions": 778
            },
            "id_cache": {
               "memory_size_in_bytes": 0
            },
            "fielddata": {
               "memory_size_in_bytes": 1348,
               "evictions": 0
            },
            "percolate": {
               "total": 0,
               "time_in_millis": 0,
               "current": 0,
               "memory_size_in_bytes": -1,
               "memory_size": "-1b",
               "queries": 0
            },
            "completion": {
               "size_in_bytes": 0
            },
            "segments": {
               "count": 350,
               "memory_in_bytes": 392,
               "index_writer_memory_in_bytes": 961,
               "version_map_memory_in_bytes": 8
            },
            "translog": {
               "operations": 12025,
               "size_in_bytes": 0
            },
            "suggest": {
               "total": 0,
               "time_in_millis": 0,
               "current": 0
            }
         }
```

* doc 部分显示node中的所有documents数量，以及已经被标记删除的。


* store显示物理存储信息。


* indexing 显示了被索引的docs数量，是一个累计递增值，只要内部进行index操作就会增加，所以，index、create、update都会增加。可以利用这个累计值，监控每分钟的变化，从而做出预警。
* get 显示的是调用 GET | HEAD 方法的次数，也是累计值。
* search 描述search 相关监控数据，```query_time_in_millis/query_total```可以用来粗略估计搜索引擎的查询效率。fetch 相关统计描述搜索过程（query-then-fetch）中fetch 操作的统计数据。
* merge 包含Lucene 段合并的一些信息，对于实时写数据量大的ES集群，merge相关指标十分重要，因为merge操作会消耗大量的CPU和IO。
* fielddata 描述fielddata占的内存以及被淘汰的情况，需要关注`fielddata.eviction`，这个值应该很小。另外，fielddata很耗内存，如果fielddata统计占内存很多，也要考虑下这样使用的业务场景是否合理。
* segment 统计Lucene 的segments相关信息，这个值应该不会太大。

Elasticsearch 自身维护它的线程池，一般来讲，我们不需要配置，默认的即可。在API返回的结果中，有很多种线程池，格式都是统一的，如下：

```json
            "index": {
               "threads": 24,
               "queue": 0,
               "active": 0,
               "rejected": 0,
               "largest": 24,
               "completed": 676231817
            }
```

上面的信息展示了线程池里现场的数量，有多少现成正在work，有多少在队列里。如果队列满了，新的请求将被rejected掉，显示在**rejected**，这时候，通常说明我们的系统到了瓶颈了，因为，我们的系统以最大的速度处理都处理不过来了，所以，这个指标要注意。

除了上面显示的index 线程池，还有很多线程池，不过很多都可以忽略，但有些重要的还是需要关注：

* index: 处理一般index请求的线程池。
* bulk：处理bulk indexing请求的线程池。
* get：处理get-by-ID请求的线程池。
* search：处理所有search和query请求的线程池。
* merge：管理 Lucene merge的线程池。

最后，需要关注[Circuit Breaker][3]的相关指标：

```json
         "fielddata_breaker": {
            "maximum_size_in_bytes": 11699,
            "maximum_size": "0.7gb",
            "estimated_size_in_bytes": 568,
            "estimated_size": "1.2gb",
            "overhead": 1.03,
            "tripped": 0
         }
```

简单来说，fielddata的大小只有在加载完了之后才会知道，如果fielddata的大小比分配内存还打，那就会导致OOM，于是，Elasticsearch 引入了断路器，用于预先估算内存够不够，如果不够，断路器就会被触发(tripped)并返回异常，而不至于导致OOM。

所以，这个指标里，最重要的就是 ```tripped```了，如果这个值很大或者说一直在增长，那么，就说明你的查询需要优化或者说需要更多内存了。

三、index 级别监控

index 级别的监控可以以index为单位进行统计，如某个index有多少search请求、search时耗费了多少时间等，可以用于发现热点索引或比较慢的索引。

然而，在实际应用中，index级别的监控并不是很有用，一般来说，瓶颈都出现在node级别，不会到index级别；而且，index级别的数据采集自不同机器，不同环境也会受影响。

查看index 级别的监控命令是 ```GET index_name/_stats```，如果index_name改成 ```_all```，则统计所有index的数据。

**同步程序监控**

同步程序监控主要包括三个方面：

1. 基础监控。同步数据量大，同步程序单独部署了一个集群，也需要相应的基础监控。
2. 同步程序进程数监控。
3. 同步程序日志监控，如版本冲突等ERROR都能采集到。

**业务监控**

我们还做了业务层面的监控，针对不同index进行基础埋点，同时，新上业务时，也会添加使用的Query语句进行埋点，系统定时跑这些埋点，当出现异常时报警。

# 总结

Elasticsearch 集群的监控是在运维同事的帮助下搭建的，到目前为止，基本能满足需求，但因为自身以前没有这方面经验，所以，这方面也许做得还不够，还望批评指正。

[1]: http://7sbmb0.com1.z0.glb.clouddn.com/es-architecture.png	"es architecture"
[2]: http://7sbmb0.com1.z0.glb.clouddn.com/es_node.jpg	"node stats"
[3]: https://www.elastic.co/guide/en/elasticsearch/guide/current/_limiting_memory_usage.html#circuit-breaker	"circuit breaker"

