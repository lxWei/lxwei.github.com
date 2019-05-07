---
layout: post
comments: true
date: 2019-05-06
title: Hive On Spark 的一些经验
categories:  
- Hive
- Spark
- Yarn
- Big Data
---

Hive 在现在数仓中被广泛使用，然而，Hive On MR 速度太慢，随着Spark SQL的兴起，似乎Spark SQL 才是未来。但对很多公司来说，由于历史包袱，无法将HQL 编写的任务轻松迁移到Spark SQL，在这种情况下，Cloudera 发起了Hive On Spark，将 Spark 作为Hive的计算引擎，提高Hive的查询性能，同时，兼容了老的HQL 任务。

在我们的生产环境中，采用了Hive On Spark 作为 Hive的计算引擎，下面是一些经验值，主要是Yarn 和 Spark 相关。

**假设集群节点16核64G**。

# Yarn

Yarn 集群主要需要设置分配和计算的 cores 数 和内存大小。一般会预留一部分资源给操作系统、DataNode、NodeManager，在这里我们设置：

* yarn.nodemanager.resource.cpu-vcores=12
* yarn.nodemanager.resource.memory-mb=48G

# Spark

## Driver

Driver 需要配置内存相关的两个参数：

* spark.driver.memory：spark driver 能申请的最大JVM 堆内存
* spark.driver.memoryOverhead：spark driver 从yarn 申请的堆外内存

这俩参数共同决定了 driver 的内存大小，一般来说，driver 内存大小并不直接影响性能，根据网上经验来看（非亲测），假设yarn.nodemanager.resource.memory-mb=M，driver 内存位N

* M > 50G，则 N = 12G
* 12G < M <= 50G， 则 N=4G
* 1G < M <=12G，则N=1G
* M <= 1G，则 N=256M

在我们的场景中，设置N为4G，设置spark.driver.memoryOverhead=500Mb，spark.driver.memory=3.5G

## Executor

### Spark Executor 内存与CPU

Executor 设置包括内存和CPU。

HDFS 在某些情况下无法很好的处理并发写，所以，过多的 core 数，可能导致竞争。同时，executor的内存，内存大的时候可以减少OOM，但同时，也会增加 GC 压力。

一般推荐将 core 设置为4、5、6，由于48可以被4整除，此时，每个node 可以分配到3个executor，每个execuror 可以分配到16G内存。设置参数：

* spark.executor.cores=4

内存分堆内内存和堆外内存，一般建议堆外内存占15%到20%。我们设置

* spark.executor.memory=12G
* spark.executor.memoryOverhead=4G

这两个参数只是我们设置的平台级别的参数，每个Spark 任务可以自行调整相应参数。

### Spark Executor 动态资源申请

对于开发人员来说，并不清楚需要申请多少资源，所以，并不推荐在提交任务时指定executor-number 数量，推荐的做法是动态申请。

现在来看，动态资源申请并不是一个可选项，而是必选项，一是有效利用资源，二是，Spark 与 MR 不同，MR 用完资源就释放了，而Spark 在会话终止前都不会释放（比如在命令行里跑Spark SQL，不从终端退出，是不释放资源的，即使任务已经跑完了），静态资源分配会导致后面的任务无资源可用，即使在集群空闲的情况下。

> spark.dynamicAllocation.enabled = true
>
> spark.dynamicAllocation.minExecutors = 1
>
> spark.dynamicAllocation.maxExecutors = 1000
>
> spark.dynamicAllocation.executorIdleTimeout = 100s
>
> spark.dynamicAllocation.cachedExecutorIdleTimeout = 600s

注意，在开启动态资源申请时，一定要开启 Spark Shuffle Service。



# 其他

另外，推荐另一篇[博文](<https://mp.weixin.qq.com/s/ITwwTDkWVwToshjHQEp5Dg>) 。发现我要写的他都几乎写了，我没用到的，他也写了。



博客评论要FQ，可以公众号交流。

![follow me](../../wxqr.jpg)