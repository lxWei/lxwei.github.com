---
layout: post
comments: true
date: 2016-03-19 
title: Saiku+Kylin多维分析平台探索
categories:  
- Big Data
- OLAP

---

# 背景
为了应对各种数据需求，通常，我们的做法是这样的：

1. 对于临时性的数据需求：写HQL到Hive里去查一遍，然后将结果转为excel发送给需求人员。
2. 对于周期性的、长期性的数据需求：编写脚本，结合Hive跑出结果，将结果写入对应DB库，然后开发前端页面对结果进行展现。

这样做简洁明了，但是，有很明显的问题：

1. 开发成本太高。每来一个需求，不管是临时需求还是长期需求，都需要进行定制开发，这种情况下，我们的人力深陷其中。
2. 使用不灵活。一个报表，只能进行展示，没有分析功能，如果要进行分析，需要将数据复制到excel里，利用excel进行处理分析，而我们的数据使用人员不一定具备这种能力。
3. 维护成本高。太多人开发报表，没有统一口径，没有详细文档，没有人能接手。
4. 资源浪费。不同人员开发的报表，很多情况下存在很多重复计算。

在这种情况下，开始考虑构建一个多维分析平台，提供平台和基础数据，数据需求方通过托拉拽的形式获取需要的数据。本文主要记录这方面的一些探索。



# Saiku
首先，选择使用 [Saiku][1] 作为我们的数据分析平台，下面是 Saiku 官网对 Saiku 的介绍：

>Saiku allows business users to explore complex data sources, using a familiar drag and drop interface and easy to understand business terminology, all within a browser. Select the data you are interested in, look at it from different perspectives, drill into the detail. Once you have your answer, save your results, share them, export them to Excel or PDF, all straight from the browser.


Saiku 是基于 [Mondrian][2] 开发的，Mondrian 是一款开源的 OLAP 引擎，能提供对大量数据的处理分析，但是，Mondrian 并没有提供友好的界面，而Saiku作为另一个开源产品，很好的充当了这个角色。

Saiku 提供JDBC 和 ODBC 接口，可以连接很多的数据源，既可以连接 MySQL、SQL Server、Oracle DB 等传统关系型数据库，也可以连接 Hive、Spark、Impala等 Big Data 平台。

## Saiku + MySQL
首先，我们用 Saiku+MySQL很快搭建了测试平台，可以实现基本的多维分析，但是，仅适用于小数据量的分析，否则会有很严重的性能问题。

## Saiku + Spark
由于 Saiku+MySQL 的性能问题，又尝试了 Saiku+Spark 的方案，数据存储于 Hive，分析引擎使用 Spark，在这种情况下，可以处理大数据量的查询分析，大部分查询都会在几十秒到几分钟内返回结果，如果数据量太大，也会存在执行失败的情况。所以，开始探索其他方案。
# Kylin 
>Apache Kylin™ is an open source Distributed Analytics Engine designed to provide SQL interface and multi-dimensional analysis (OLAP) on Hadoop supporting extremely large datasets, original contributed from eBay Inc.

[Kylin][3] 相对于其他OLAP分析引擎，一个重要特点是采用空间换时间，根据定义的cube进行预计算，并将计算结果存储到Hbase中，在进行查询时，直接查询Hbase，所以，Kylin 的查询可以到毫秒级，性能完全不是问题。

Kylin 提供 ANSI SQL 接口来对数据进行查询，没有可视化操作，所以，有一定的使用门槛。同时，由于Kylin 提供的 ANSI SQL接口，Kylin 也就可以和BI平台进行对接，比如，Kylin 已经支持 Tableau 等BI工具，奈何 Tableau 是商业软件，so...

# Saiku + Kylin 实现多维分析
Saiku 根据用户在页面的操作，生成 MDX，然后，Mondrian根据MDX生成查询语句SQL，而 Kylin 可以根据SQL 查询 cube，快速得到结果，所以，如果 Saiku 和 Kylin 中定义了相同的 cube，那么，就可以通过Saiku 来查询 Kylin了，从而将 Saiku 的操作页面和 Kylin 的高性能查询能力结合起来。

Google 一下，[Github 已经有人这么做了][4]，按照该项目说明就可以轻松搭建 Saiku+Kylin 多维分析平台。

Saiku+Kylin 的分析平台能实现的分析能力，很大程度取决于 Kylin 支持的 SQL 和 Mondrian 生成的 SQL的共通点，经过测试，Kylin 能对绝大多数 SQL 提供友好支持，可以和 Saiku 进行完美结合。

# 总结
Saiku 作为分析平台，提供可视化的操作，能方便的对数据进行查询、分析，并提供图形化显示，但是，Saiku 有一定的使用门槛，特别是对国内的使用者来说，所以，可能需要一些定制开发。

Kylin 作为分析引擎，根据空间换时间的思想，对数据进行预计算，从而提供极高的查询性能，并且提供 ANSI SQL 接口，可以极大程度满足日常查询需求。但是，Kylin 对 Hadoop 生态版本有较高的要求，所以，尽量按照官方推荐版本安装配置。

PS：本文采用 Saiku 3.7.4 和 Kylin 1.2。

[1]:http://www.meteorite.bi/products/saiku "saiku"
[2]:mondrian.pentaho.com "Mondrian"
[3]:http://kylin.apache.org/ "Kylin"
[4]:https://github.com/mustangore/kylin-mondrian-interaction "Saiku+Kylin"