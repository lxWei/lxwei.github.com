---
layout: post
comments: true
date: 2017-01-06 
title: 
categories:  
- Elasticsearch
- Big Data
- Search Engine

---

Elasticsearch 说自己是一个准实时的搜索引擎，为什么能做到准实时呢？

往索引里写了一条数据，为什么有时需要等一段时间才能搜到？

往索引里写了一条数据，搜不出来，进行一下`refresh`或者`flush`操作就能搜出来了，是为什么呢，`flush` 和 `refresh`有什么区别？

Elasticsearch 是如何保证索引持久化且不丢数据的？

这篇文章将解释这些问题。

# 动态索引

众所周知，搜索引擎的基础是倒排索引，一般情况下，索引中的数据都是实时变化的，那么，索引系统如何实时反映这种变化呢？在搜索引擎中，普遍采用动态索引来做，如下图所示。

![老系统架构][1]

在这个动态索引中，有三个关键的索引结构：倒排列表、临时索引、已删除列表。倒排索引是已经建好的索引结果，倒排列表存在磁盘文件中，单词词典在内存中。临时索引是在内存中实时建立的倒排索引，结果与倒排列表一样，只是存在于内存中，当有新文档时，实时解析文档并加到这个临时索引中。已删除列表存储已被删除的文档的文档ID。另外，当一个文档被更改，搜索引擎中一个普遍的做法是删除旧文档，然后新建一个新文档，间接实现更新操作，这么做的原因主要是索引文件存储在磁盘文件，写磁盘不方便。

当用户搜索时，搜索引擎同时到倒排列表和临时索引进行查询，找到包含用户查询的文档集合，并对结果进行合并，之后利用删除文档进行过滤，形成最终结果，返回给用户。这样就实现了动态环境下的准实时搜索功能。

# Elasticsearch 实现
上面简单介绍了搜索引擎实现准实时搜的原理和普遍做法，下面看看Elasticsearch的具体实现。

## Elasticsearch 动态更新

Elasticsearch 基于Lucene开发，Lucene 提供了 `segment` 的概念，segment 代表 Lucene 的一个完成的索引段，通常一个索引包含多个 segment，每个segment 包含一个 `commit point`，这些segment对外提供搜索服务。

当往索引里新写数据时，新文档先写到内存中的一个buffer中，当buffer被commited时，就写到磁盘中，生成一个新的segment，并对外提供服务，同时，buffer被清空。

每个 commit point 维护了一个`.del`文件，存储已被删除的文档，即上一节介绍的已删除文档列表。

## Elasticsearch 准实时搜索

要把数据写到磁盘，需要调用 [fsync][2]，但是fsync十分耗资源，无法频繁的调用，在这种情况下，Elasticsearch 利用了`filesystem cache`，新文档先写到in-memory buffer，然后写入到 filesystem cache，过一段时间后，再将segment写到磁盘。在这个过程中，只要文档写到filesystem cache，就可以被搜索到了。

## Elasticsearch 持久化

必须调用fsync将segment刷到磁盘上，才能保证数据不丢失。

同时，Elasticsearch 使用`translog` 来记录Elasticsearch中的操作。

* 当新文档被添加到索引中时，新文档被加入到 in-memory buffer中，并且在translog中记录下来。
* 当进行refresh操作时，in-memory buffer里的数据被清空，translog保持不变。
* 经过一段时间，或者translog大到一定程度，整个index会被`flushed`，这时，进行一次commit，in-memory buffer的文档被写到新的segment，buffer被清空，一个commit point 被写到磁盘，filesystem cache 也被flushed到磁盘，老的translog被删除。

translog持久化存储了所有没有flush到磁盘的操作。当启动Elasticsearch时，Elasticsearch 首先根据最后的commit point 从磁盘恢复已知的segment，然后重放translog恢复没有commit的文档。这样，既实现了持久化，也能保证不丢数据。

## refresh VS flush

在Elasticsearch中，**refresh是轻量级的写和打开一个新segment的操作，默认情况下，每个分片每秒refresh一次，这就是我们说Elasticsearch是一个准实时搜索引擎的原因**，因为每个文档的修改，最多经过一秒钟就可以知道了。虽然refresh是一个轻量级的操作，但是，还是会带来一定的消耗，所以，还是要注意不要太频繁的操作，而且，我们很多应用并不需要这么实时，比如在`ELK`中，我们可以将这个时间设置到30s甚至更大。

在Elasticsearch中，执行commit操作并删除translog的操作叫`flush`，每个shard每30分钟或translog太大时自动flush一次，使用者很少需要手动进行flush操作。

## 段合并

如果不停的产生新的segment，Elasticsearch中很快就会段爆炸，每个段都要消耗文件描述符、内存、CPU 周期，且每个search请求都需要遍历所有的segment，会造成搜索操作很慢。

所以，Elasticsearch会在后台对segment进行合并，在段合并的过程中，被删除的文档被丢弃。

Elasticsearch 提供了 `optimize` 接口，可以看做是一个强制进行段合并的API，使shard进行段合并到指定段数目，从而可以提高查询性能。

需要注意的是，merge操作会消耗大量的CPU和I/O，默认情况下，Elasticsearch 会控制merge操作的资源使用，从而不至于影响正常的search操作。但是，如果是手动进行optimize操作，这时，Elasticsearch 不会对merge使用的资源进行控制，从而消耗大量I/O，影响正常的搜索和集群的稳定。

# 总结

本文首先介绍搜索引擎中普遍的做法，然后，介绍了Elasticsearch的具体做法，一是明白了Elasticsearch的原理和中这几个操作的不同，同时，也可以看出，具体形式多变，仍然摆脱不了发明多年的搜索引擎的基础。



***参考***

1. [Elasticsearch： The definitive guide][3]
2. [这就是搜索引擎][4]
3. [信息检索][5]

[1]: http://blog2017.qiniudn.com/dynamic_index.jpg	"dynamic index"

[2]: https://en.wikipedia.org/wiki/Fsync	"fsync"
[3]: https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html	"es definitive guide"
[4]: https://book.douban.com/subject/7006719/	"this is search engine"
[5]: https://book.douban.com/subject/7154449/	"information retrieval"