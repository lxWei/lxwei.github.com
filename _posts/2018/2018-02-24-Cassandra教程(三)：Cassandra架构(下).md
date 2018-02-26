---
layout: post
comments: true
date: 2018-02-24
title: 
categories:  
- Cassandra
- Big Data
---

上篇介绍了Cassandra的架构、数据distribution 与 replication，本文主要介绍Cassandra的内部工作机制，包括存储引擎、Cassandra读写、数据一致性等。

# 1. 存储引擎

在分布式系统中，有些系统写数据采用**read-and-write** 的方式（如Elasticsearch），Cassandra为了避免read-and-write 带来的性能问题，没有采用read-and-write的方式，存储引擎将写操作保存于内存，每过一段时间，将内存中的数据以追加的方式，写入磁盘，磁盘中的数据都是不可更改、不可重写的。当读数据时，需要将读取的数据组合起来以得到正确的数据。

在内部实现上，Cassandra 采用了类似 [Log-Structured merge tree][1] 的存储结构存储数据，采用顺序IO，这样的话，即使采用HDD也能有不错的性能。

# 2. 数据读写

**write**

如下图所示，node接收write请求，将数据写入**memtable**，同时记录到**commit log**。commit log 记录node接收到的每一次write请求，这样，即使发生断电等故障，也不会丢失数据。

memtable是一个cache，按顺序存储write的数据，当memtable 的内容大小达到配置的阈值或者commit log的存储空间大于阈值，memtable里的数据被flush到磁盘，保存为**SSTables**。当memtable中的数据flush到磁盘后，commit log被删除。

在内部实现上，memtable 和 SSTable按table进行划分，不同的table可以共享一个commit log。SSTable本质上是磁盘文件，不可更改，因此，一个partition 包含了多个SSTables。

*best practice*: 重启node前先使用nodetool flush memtable，这样可以减少commit log重放。

![cassandra写入流程](http://blog2018.qiniudn.com/cassandra-write-process.png)

**compaction**

Cassandra不会采用类似insert／update的方式更新已有数据，而是创建带有时间戳版本信息的新的数据，同时，Cassandra也不删除数据，而是将数据标记为tombstones。这样，随着时间过去，每行数据可能包括不同时间戳版本的多个列集合，读取数据时，可能需要读取越来越多的列才能组成完整的一行数据。为了避免这种情况，Cassandra周期性的合并SSTables并删除旧数据，这个过程称作**compaction**。compaction 读取每行数据所有版本的数据然后用最新的数据组成完整的一行，新数据写入新的SSTable，旧版本数据随后被删除。compaction 提高了Cassandra的read 性能。

另外，在compaction过程中，新旧数据可能同时存在，所以，磁盘使用率上会存在突增；同时，由于数据按照partition key 按序存储，所以，compaction过程中，不使用随机IO。

**update**

Cassandra 将每个新行视为upsert，如果已经存在该primary key，则视作是对原有数据的update，

**delete**

Cassandra 删除数据时使用tombstone，tombstone是一个标记，标记column被删除了，在compaction阶段，标记删除的columns被物理删除。在读取阶段，标记为tombstone的数据被忽略。

**read**

读取数据时，Cassandra可能需要联合memtable和多个SSTables才能拼装出完整的数据。

# 3. 数据一致性

根据 [CAP][3] 理论，Cassandra 是一个AP系统，提供最终一致性。同时，Cassandra可以灵活配置，使系统更趋向一个CP系统。

## 3.1 Two consistency features

### 3.1.1 Tunable consistency

高一致性意味着高延迟，低一致性意味着低延迟，需要根据自己的需求，自己调节。而且，Cassandra 不仅支持集群级别的一致性设置，还支持请求级别的一致性设置，用户可以针对请求设置一致性。

一致性等级决定了处理读／写请求返回成功的数据副本数，Cassandra赋予用户充分的自主选择权，通常情况下，设置读／写的的一致性等级为"**QUORUM**"，其中，quorum = (sum_of_replication_factors / 2) + 1，sum_of_replication_factors表示所有datacenter中replication factor求和。

### 3.1.2 Linearizable consistency

存在一些场景，一些操作需要顺序执行且不能被中断，Cassandra通过**lightweight transactions** 来支持这种场景。

## 3.2 一致性计算

**强一致性**: R + W > N

**最终一致性**：R + W <= N

其中，R代表read操作的一致性，W表示write操作的一致性，N表示副本数。

# 总结

本文介绍了Cassandra的内部实现，下一篇开始介绍CQL。

[1]: https://en.wikipedia.org/wiki/Log-structured_merge-tree "Log-structured merge tree"
[2]: https://docs.datastax.com/en/glossary/doc/glossary/gloss_zombie.html "zombie"
[3]: https://en.wikipedia.org/wiki/CAP_theorem "CAP"



