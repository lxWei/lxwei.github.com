---
layout: post
comments: true
date: 2018-02-23
title: 
categories:  
- Cassandra
- Big Data
---

Cassandra 设计用来处理多节点大型数据工作负载，系统中没有单点，Cassandra 采用peer-to-peer架构，数据在所有节点之间分发。

* cluster中所有node具有相同的角色。每个node互相独立，同时在内部又互相沟通。
* cluster中所有node都可以处理读写请求，而不用管数据具体在哪儿。
* 如果一个node挂了，其它node可以处理读写请求。

# 1. node之间的沟通

Cassandra 各node之间采用 [gossip][1] 协议进行沟通，gossip 进程每秒与集群中最多三个node交换信息，信息包括node自身的信息以及与该node交换过信息的node的信息，这样，所有node都可以很快获取集群中所有的node信息。

# 2. Data distribution and replication

Cassandra是一个**分区的按行存储的数据库**（partitioned row store database）。在Cassandra中，数据以table的形式组织起来，primary key唯一标记一行，同时，primary key也决定了数据行存储的node，replication是数据行的备份，Cassandra将数据复制到多个node上，从而实现高可用和容错。

下面分data distribution 和 replication两个方面进行阐述，其中distribution说明将数据分发到哪个node，replication说明如何备份。

## 2.1 Data distribution

partitioner 决定了数据是怎样在集群中分布的。简单的讲，partitioner是一个函数，根据partition key产生一个唯一标记一行的token，然后，这行数据根据token分发到集群中的节点，通常，partitioner是Hash函数。Cassandra提供了三种partitioner，包括Murmur3Partitioner（default）、RandomPartitioner、ByteOrderedPartitioner。(为更好理解这部分，可以参考[一致性哈希][2])。

## 2.2 Data replication

Cassandra将数据存储到多个node以实现高可用和容错，replication策略决定了将数据备份到哪些节点。

**replication factor** 指数据在整个cluster中的份数，如果replication factor等于1，则数据在整个集群中仅存在一份，此时，如果存储数据的node出现故障，那么数据就丢失了。在实践中，replication factor 应该不超过cluster中node的数量。

Cassandra提供了两种replication 策略：

* SimpleStrategy: 仅适用于单datacenter 单 rack。根据partitioner存储第一份replica，然后在顺时针方向的下一个node上存放下一份replica（不考虑网络拓扑信息）。
* NetworkTopologyStrategy: 可以方便的扩展到多datacenter，推荐使用，同时，NetworkTopologyStrategy尽量避免将数据存储到相同的rack上。

#3. Snitches

snitch 决定了node属于哪个datacenter的哪个rack。可以用于告知Cassandra集群网络拓扑信息，以实现高效的请求路由与分发、备份数据。

主要的snitch包括：

* dynamic snitching
* SimpleSnitch
* RackInferringSnitch
* PropertyFileSnitch
* GossipingPropertyFileSnitch
* ...

# 4. 总结

本文主要介绍Cassandra的架构、数据distribution 与 replication，下一章介绍Cassandra的内部信息，包括存储引擎、Cassandra读写、数据一致性等。

[1]: https://en.wikipedia.org/wiki/Gossip_protocol "gossip"
[2]: http://blog.codinglabs.org/articles/consistent-hashing.html " 一致性Hash"





