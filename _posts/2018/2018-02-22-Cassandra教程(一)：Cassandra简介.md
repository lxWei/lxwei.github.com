---
layout: post
comments: true
date: 2018-02-22 
title: 
categories:  
- Cassandra
- Big Data
---

# Overview

Apache Cassandra 是一个大规模可扩展的分布式开源NoSQL数据库，完美适用于跨数据中心／云端的结构化数据、半结构化数据和非结构化数据，同时，Cassandra 高可用、线性可扩展、高性能、无单点。

## 特点

* scalable，线性可扩展
* fault-tolerant，且没有单点（peer-to-peer）
* column-oriented database & partitioned row store database
* distribution design 基于 Amazon 的 Dynamo 
* data model 基于 Google 的 Bigtable
* 灵活的数据存储，支持结构化、半结构化、非结构化数据
* 支持事务
* 写性能好
* 由 Facebook开源

# 数据模型

## 内部数据结构

Cassandra是一个column-oriented database，也就是说，不用像关系型数据库一样事先定义好列，在Cassandra中，不同行的列可以不一样。

在Cassandra中，数据模型由keyspaces、column families、primary key 和 columns组成，对比关系型数据库，如下表：

| 关系型数据库       | Cassandra         |
| ------------ | ----------------- |
| Database     | Keyspace          |
| Table        | CF(column family) |
| Primary Key  | Primary Key       |
| Column Name  | Key / Column Name |
| Column Value | Column Value      |

在Cassandra中，Primary Key包括partition key 和 cluster key两部分，其中cluster key可选，partition key确定数据行分发到哪个node，cluster key用于node内部数据排序。

对于每一个column family，不要想象成关系型数据库的表，而要想像成一个多层嵌套的排序散列表（Nested sorted map）。这样能更好地理解和设计Cassandra的数据模型。

散列表可用提供高效的键值查找，排序的散列表可提供高效的范围查找，在Cassandra里，我们可以使用primary key和column key做高效的键值查询和范围查询，而且，在Cassandra中，列的名称可以直接包含数据，也就是说，有的列可以只有列名没有列值。

> ```
> Map<RowKey, SortedMap<ColumnKey, ColumnValue>>
> ```

## CQL

CQL (Cassandra Query Language)是用于 Cassandra 的查询语言，可类比用于关系型数据库的SQL，注意，虽然CQL 和 SQL 看起来比较相似，但二者内部原理完全不同。

# 举个例子

## 搭建Cassandra

学习Cassandra时搭建环境最简单的方式是使用docker，可以参考[镜像][1] 。

## 例子

如下图所示，首先创建keyspaces，然后创建table，往table中插入数据，再查询该table。

![例子](http://blog2018.qiniudn.com/introduce-cassandra.png)

# 总结

本文简单介绍了Cassandra，并举例说明了基本的使用。下一篇将介绍Cassandra的数据模型。

[1]: https://hub.docker.com/_/cassandra/ "cassandra docker image"





