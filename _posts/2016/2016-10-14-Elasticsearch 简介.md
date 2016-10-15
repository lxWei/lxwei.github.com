---
layout: post
comments: true
date: 2016-10-14 
title: Elasticsearch 简介
categories:  
- Big Data
- Elasticsearch
---
这是一篇科普文。

# 1. 背景

Elasticsearch 在公司的使用越来越广，很多同事之前并没有接触过 Elasticsearch，所以，最近在公司准备了一次关于 Elasticsearch 的分享，整理成此文。此文面向 Elasticsearch 新手，老司机们可以撤了。

# 2. 搜索引擎原理



# 3. Elasticsearch 简介与基本概念

> Elasticsearch is a real-time **distributed search and analytics engine**. It allows you to explore your data at a speed and at a scale never before possible. It is used for **full-text search, structured search, analytics, and all three in combination**.

在 《Elasticsearch : The Definitive Guide》里，这样介绍Elasticsearch，总的来说，Elasticsearch 是一个分布式的搜索和分析引擎，可以用于全文检索、结构化检索和分析，并能将这三者结合起来。Elasticsearch 基于 Lucene 开发，现在是使用最广的开源搜索引擎之一，Wikipedia、Stack Overflow、GitHub 等都基于 Elasticsearch 来构建他们的搜索引擎。

先介绍下 Elasticsearch 里的基本概念，下图是 Elasticsearch 插件 head 的一个截图。

![Elasticsearch 插件head截图][1]

* node：即一个 Elasticsearch 的运行实例，使用多播或单播方式发现 cluster 并加入。
* cluster：包含一个或多个拥有相同集群名称的 node，其中包含至少一个master node。
* index：类比关系型数据库里的DB，是一个逻辑命名空间。
* alias：可以给 index 添加零个或多个alias，通过 alias 使用index 和根据index name 访问index一样，但是，alias给我们提供了一种切换index的能力，比如重建了index，取名customer_online_v2，这时，有了alias，我要访问新 index，只需要把 alias 添加到新 index 即可，并把alias从旧的 index 删除。不用修改代码。
* type：类比关系数据库里的Table。其中，一个index可以定义多个type，但一般使用习惯仅配一个type。
* mapping：类比关系型数据库中的 schema 概念，mapping 定义了 index 中的 type。mapping 可以显示的定义，也可以在 document 被索引时自动生成，如果有新的 field，Elasticsearch 会自动推测出 field 的type并加到mapping中。
* document：类比关系数据库里的一行记录(record)，document 是 Elasticsearch 里的一个 JSON 对象，包括零个或多个field。
* field：类比关系数据库里的field，每个field 都有自己的字段类型。
* shard：是一个Lucene 实例。Elasticsearch 基于 Lucene，shard 是一个 Lucene 实例，被 Elasticsearch 自动管理。之前提到，index 是一个逻辑命名空间，shard 是具体的物理概念，建索引、查询等都是具体的shard在工作。shard 包括primary shard 和 replica shard，写数据时，先写到primary shard，然后，同步到replica shard，查询时，primary 和 replica 充当相同的作用。replica shard 可以有多份，也可以没有，replica shard的存在有两个作用，一是容灾，如果primary shard 挂了，数据也不会丢失，集群仍然能正常工作；二是提高性能，因为replica 和 primary shard 都能处理查询。另外，如上图右侧红框所示，shard数和replica数都可以设置，但是，shard 数只能在建立index 时设置，后期不能更改，但是，replica 数可以随时更改。但是，由于 Elasticsearch 很友好的封装了这部分，在使用Elasticsearch 的过程中，我们一般仅需要关注 index 即可，不需关注shard。

综上所述，shard、node、cluster 在物理上构成了 Elasticsearch 集群，field、type、index 在逻辑上构成一个index的基本概念，在使用 Elasticsearch 过程中，我们一般关注到逻辑概念就好，就像我们在使用MySQL 时，我们一般就关注DB Name、Table和schema即可，而不会关注DBA维护了几个MySQL实例、master 和 slave 等怎么部署的一样。

下表用Elasticsearch 和 关系数据库做了类比：

| Elasticsearch | index    | type  | field | document | mapping |
| ------------- | -------- | ----- | ----- | -------- | ------- |
| DB            | database | table | field | record   | schema  |

最后，来从 Elasticsearch 中取出一条数据（document）看看：

![ES result][2]

由index、type和id三者唯一确定一个document，_source 字段中是具体的document 值，是一个JSON 对象，有5个field组成。

# 4. Elasticsearch 基本使用

下面介绍下 Elasticsearch 的基本使用，这里仅介绍 Elasticsearch 能做什么，而不详细介绍语法。

## 4.1 基础操作

* index：写 document 到 Elasticsearch 中，如果不存在，就创建，如果存在，就用新的取代旧的。
* create：写 document 到 Elasticsearch 中，与 index 不同的是，如果存在，就抛出异常```DocumentAlreadyExistException```。
* get：根据ID取出document。
* update：如果是更新整个 document，可用index 操作。如果是部分更新，用update操作。在Elasticsearch中，更新document时，是把旧数据取出来，然后改写要更新的部分，删除旧document，创建新document，而不是在原document上做修改。
* delete：删除document。Elasticsearch 会标记删除document，然后，在Lucene 底层进行merge时，会删除标记删除的document。

## 4.2 Filter 与 Query



## 4.3 一些重要的查询

Elasticsearch 使用 domain-specific language(DSL)进行查询，DSL 使用 JSON 进行表示，在Elasticsearch 中，有几类最重要的查询子句，掌握了就可以覆盖日常90%以上的需求。

### 4.2.1 match_all

```{"match_all": {}}```

表示取出所有documents，在与filter结合使用时，会经常使用match_all。

### 4.2.2 match

一般在全文检索时使用，首先利用analyzer 对具体查询字符串进行分析，然后进行查询；如果是在数值型字段、日期类型字段、布尔字段或not_analyzed 的字符串上进行查询时，不对查询字符串进行分析，表示精确匹配，一个简单的例子如：

```json
{ "match": { "tweet": "About Search" }}
```

```json
{ "match": { "age":    26           }}
```

### 4.2.3 term

term 用于精确查找，可用于数值、date、boolean值或not_analyzed string，当使用term时，不会对查询字符串进行分析，进行的是精确查找。

```json
{ "term": { "date":   "2014-09-01" }}
```

### 4.2.4 terms

terms 和 term 类似，但是，terms 里可以指定多个值，只要doc满足terms 里的任意值，就是满足查询条件的。与term 相同，terms 也是用于精确查找。

```json
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

注意，terms 表示的是contains 关系，而不是 equals关系。

### 4.2.5 range

类比数据库查找的范围查找，举个简单的例子：

```json
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

操作符可以是：

* gt：大于
* gte：大于等于
* lt：小于
* lte：小于等于

### 4.2.6 exists 和 missing

exists 用于查找字段含有一个或多个值的document，而missing用于查找某字段不存在值的document，可类比关系数据库里的 *is not null* (exists) 和 *is null* (missing).

```json
{
    "exists":   {
        "field":    "title"
    }
}
```



## 4.4 聚合功能



## 4.5 Geolocation 



# 5. Elasticsearch 使用时注意的几个问题



[1]: http://7sbmb0.com1.z0.glb.clouddn.com/es-head.jpeg	"head"
[2]: http://7sbmb0.com1.z0.glb.clouddn.com/es-result.png	"es result"