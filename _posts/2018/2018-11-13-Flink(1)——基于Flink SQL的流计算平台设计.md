---
layout: post
comments: true
date: 2018-11-13
title: 
categories:  
- Flink
- Big Data
---



本文转自个人微信公众号，[原文链接](https://mp.weixin.qq.com/s/8ICLIEzuGvDuzgOddXwTGg)。

接[上篇](http://lxwei.github.io/posts/Flink(0)-%E5%9F%BA%E4%BA%8EFlink%E7%9A%84%E6%B5%81%E8%AE%A1%E7%AE%97.html)。

**使用场景**

先说流计算平台应用场景。在我们的业务中，实时平台核心包括几个部分：一是大促看板，比如刚过去的双11，供领导层和运营查看决策使用；二是实时风控的技术支持；三是实时数据接入、清洗、入库功能，为下游提供实时、准确的数据。

为了支持这些业务需求，并最小化技术人员的介入，设计并实现了实时计算平台。

**设计**

首先，是数据源部分。数据接入包括埋点日志、数据库数据、API上报数据等，埋点数据、API上报的数据等都接入Kafka，平台支持的数据源包括Kafka、MySQL、Redis、Elasticsearch，根据使用经验，Kafka和MySQL 已经基本覆盖我们的业务需求。我们将数据源统一在平台进行管理，使用者不需要关注数据源的具体来源信息。

其次，是Job。Job由数据源和具体的task组成。数据接入后，需要进行运算，要定义算子和工作流。算子就是我们要对数据流进行的操作，同时，对数据可能需要经过中间很多层处理，所以，还需要定义工作流。算子我们采用Flink SQL，且目前仅支持Flink SQL。Flink 使用 [Apache calcite](https://calcite.apache.org/docs/reference.html) 解析SQL，它支持 ANSI SQL，这对于BI和分析师，都是比较**容易使用**的。在当前情况下，Flink SQL 对有些语法还不支持，对我们来说，这不算大问题，一是先有语法已经覆盖我们的绝大多数需求，如果我们要等它完美支持后再来使用，反而是得不偿失，正所谓***Done is better than perfect.***；其次是对于刚需语法，我们可以根据Flink 提供的[UDF](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/udfs.html) 自行开发，比如函数  LAST_VALUE()。

**部署**

Flink 集群支持Standalone、Yarn、Mesos、K8S等多种模式，我们目前的版本采用Standalone cluster模式，现在流行的在生产环境使用较多的是Yarn模式，下表是Standalone 模式和 Yarn 模式的优缺点对比。我们之前采用Standalone 模式的两个原因，一是为了快速实现；二是尽量减少外部依赖特别是对 Yarn 集群的依赖（Yarn 集群主要是离线计算和BI、分析师日常取数使用，尽量减少对他们的影响。如果要采用Yarn 集群模式，我也推荐单独搭建Yarn 集群）。但我还是更推荐Yarn 模式，Job 级别的资源隔离以及失败自动重启会更加重要点。

|      | Standalone cluster                                           | Yarn cluster                                          |
| ---- | ------------------------------------------------------------ | ----------------------------------------------------- |
| 优点 | 1. 外部依赖少 <br />2. 添加、减少task manager方便，可以快速实现<br />3. 方便调试和查看日志 | 1. Job 级别的资源隔离<br />2. Node 失败自动恢复<br /> |
| 缺点 | 1. 需要额外部署ZK<br />2. 不支持Job 级别的资源隔离           | 需要依赖外部系统 Yarn                                 |

**资源**

不同的任务数据量不同，计算量不同，需要的资源也不同，我们支持对不同的Job 配置不同的 parallelism，从而满足不同的资源需求，该值还只是一个经验值，暂时无法做到自适应配置。

**使用**

Flink 将 savepoint 保存到HDFS，在使用过程中，我们发现HDFS上的savepoint 数量巨大，但一段时间前的savepoint是没有用处的，所以，我们对savepoint 进行了生命周期管理，自动删除过期的savepoint。

另外，在业务方使用过程中，也要做Job的生命周期管理，比如大促看板，否则，实时计算平台的资源就是一个黑洞。

**其它**

系统还涉及用户管理、权限管理、监控告警等部分，暂不做详细介绍。



扫描下方二维码关注我。

![wx](../../wxqr.jpg)