---
layout: post
comments: true
date: 2018-02-26
title: 
categories:  
- Cassandra
- Big Data
---

本文不是详细的[CQL][1]教程，仅记录下CQL的一些要点。

# Keyspace

keyspace类似关系型数据库中的database概念，Cassandra 的 keyspace 是一个命名空间，定义了数据备份的方式。举例如下，keyspace cycling 中所有的table 在数据中心 datacenter1中存在3个replicas。

```sql
CREATE KEYSPACE IF NOT EXISTS cycling WITH REPLICATION = { 'class' : 'NetworkTopologyStrategy', 'datacenter1' : 3 };
```

# Table

创建table的语句类似下面的例子，跟SQL很类似。

```sql
CREATE TABLE cycling.cyclist_alt_stats ( id UUID PRIMARY KEY, lastname text, birthday timestamp, nationality text, weight text, height text );
```

几个核心概念：

## Primary Key

primary key 用来唯一标记table中的一行数据，定义了数据在table中的位置和顺序，且primary key一旦定义，后续无法修改。primary key 包含两部分，可以表示为(part1, part2)，其中，part1是partition key，用于确定数据的partition，part2为可选部分，为clustering key，用于确定数据在partition内部的顺序，其中，clustering key是可选的，而partition key是必须部分。

**Simple Primary Key**

指primary key只包含一个column，该column为partition key，无clustering key。这种情况下，数据的读写都很快，推荐使用。

**Composite Partition Key**

partition key包含多columns时，称为Composite Partition Key。这种Partition Key 可以将数据划分为多份，可以解决Cassandra中的热点问题或者是大量数据写入单节点的问题。不过，这种情况下，如果要读取数据，需要指定所有的partition key的columns。

**Compound Primary Key**

同时包含partition key 和 cluster key。cluster key用于数据在节点内的排序，指定cluster key后，可以快速的读取数据。

## Counter Table

counter 是一个特殊的column，存储了一个只会增加或减少的整数。需要注意的是，counter table里只能包含primary key 和 counter column两部分。举例如下：

```sql
CREATE TABLE popular_count (  id UUID PRIMARY KEY,  popularity counter  );
```

另外，counter table里的counter column值的写入跟普通column不一样，counter column需要用update方法，而不是insert方法。举例来说，下面的语句第一次之后，popularity的值将等于1。

```sql
UPDATE popular_count
 SET popularity = popularity + 1
 WHERE id = 6ab09bec-e68e-48d9-a5f8-97e6fb4c9b47;
```

# 物化视图

Cassandra 也提供了 materialized view。Cassandra 的 materialized view 其实是基于另外的表的数据创建的一张新表， 新表与原来的表相比具有不同的primay key和属性。在 Cassandra, 数据模型是由query驱动的（相对的，关系型数据库由实体关系驱动），标准的做法是为query创建表，如果有不同的query，则需要建立不同的表，但是，这种情况会给数据维护增加困难，需要应用去维护多张表的更新。materialized view 解决了这个问题，源表更新后，会自动更新materialized view中的数据。

物化视图有些要求：

* 源表的primary key必须是物化视图的primary key的一部分。
* 物化视图的primary key只能添加一列新列，且不能为static column。

同时，物化视图也会有些其他影响：

* 写性能不如普通的表。
* 源表的写操作性能变差。
* 物化视图的数据有延迟。原因是用户只能向源表写数据，然后异步的写到物化视图。

## 二级索引

primary key是一级索引，所以，索引也叫二级索引，用于辅助找到一级索引。二级索引提供了一种利用普通的columns访问数据而不使用partition key，可以快速高效的查询数据。二级索引将索引列存储到一个单独的、隐藏的表中，索引列作为primary key，源表的primary key作为新表的值。不过，二级索引有适用场景，也有些场景不适合使用：

**适用场景**：表中很多行都具有相同的indexed value的情况比较适合，索引列的值越多，维护和查询的成本就越多。

**不适用场景**：

* high-cardinality 列，这一列有很多不同的值， 此时，查询了很多值，却只会取其中很小一部分作为结果。***此时，可以用物化视图***。另一个极端，如布尔值，区分度太小，也不适合。

- counter column。
- 频繁更新／删除的列。
- 从多个partition 取数据时，此时，涉及到多个partition，随着集群越来越大，可能越来越慢。

## Insert/Update

[INSERT](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlInsert.html) 和 [UPDATE](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlUpdate.html) 操作在使用 `IF` 从句时，支持lightweight transactions(Compare and Set (CAS))。  [Lightweight transactions](https://docs.datastax.com/en/cassandra/3.0/cassandra/dml/dmlLtwtTransactions.html) 应该谨慎使用，因为会带来更大的延迟。

## Time-To-Live（TTL）

columns ／ table 支持TTL。数据过期后就会被标记为 [tombstone](https://docs.datastax.com/en/glossary/doc/glossary/gloss_tombstone.html). 

##Batch Insert/Update

批量insert/update是一个原子操作。单 partition的批量操作自动保证原子性，涉及到多 partition的批量操作，需要用到 batchlog来保证原子性。同时，批量操作可以减少网络传输，可以提高性能。

使用批量操作的一个重要考虑就是一组操作需要保证原子性。单分区的批量操作在服务器端一次性完成。涉及到多分区的批量操作往往会带来性能问题，需慎用。在批量操作时，coordinator 节点负责管理所有的写操作，coordinator 节点很容易成为瓶颈，

***注意，不能为了性能而盲目的使用批量操作***。

## Query

Where 条件中 partition key 和 cluster key 产生的结果必须是一个连续的结果集，比如，表的primary为(colA, colB, colC)，则where 条件 colA = XXX and colC=XXX 是会报错的。

可以使用ORDER BY子句指定结果集的顺序，partition key 必须在where子句指明， order by子句必须指明cluster key以便排序。

# 总结

Cassandra相对于Hbase等更容易上手的一个重要原因是提供了类似SQL的CQL，但是，我们在使用CQL时应该明白，CQL并不是SQL，Cassandra的原理与关系型数据库不一样，CQL与SQL也不一样，如果仅仅是将SQL的经验带过来，会带来很多意想不到的问题。本文只是一些CQL的要点纪录，并不是完整的CQL教程，可以参考[官网][1]系统学习。

下一篇准备讲如何在Spark中使用Cassandra。

[1]: https://cassandra.apache.org/doc/latest/cql/index.html "CQL"







