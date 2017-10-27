---
layout: post
comments: true
date: 2017-10-24 
title: 
categories:  
- Spark
- Big Data
---

本文主要介绍如何开发Spark项目，其中，采用 **Scala**进行开发，工具选用**IDEA** 和 **SBT**。

# 创建项目

打开IDEA ，点击 *Create New Project* > *Scala*  > *SBT*，点击 *Next*，填写Name和Location，选择项目存放目录，**注意，Spark 2.2 与 Scala 2.12 不兼容，所以，Scala版本需要选择2.11版本**，点击 *Finish*，如下图所示。

![创建页面](http://blog2017.qiniudn.com/intellij-info.png)

项目创建完毕，如下图所示。由于此前选择了"SBT"选项，此时，IDEA已经创建了一些目录。其中，*src* 目录是具体的代码目录， build.sbt。

![new class](http://blog2017.qiniudn.com/intellij-new-class.png)

# 配置项目依赖

配置 *build.sbt*，如下所示：

```scala
name := "spark"

version := "1.0"

scalaVersion := "2.11.8"

val sparkVersion = "2.2.0"

libraryDependencies ++= Seq(
  "org.apache.spark" % "spark-core_2.11" % "2.2.0" % "provided",
  "org.apache.spark" % "spark-sql_2.11" % "2.2.0" % "provided",
  "org.apache.spark" % "spark-streaming_2.11" % "2.2.0" % "provided",
  "org.apache.spark" % "spark-mllib_2.11" % "2.2.0" % "provided",
  "org.apache.spark" % "spark-graphx_2.11" % "2.2.0" % "provided"
)
```

具体的可以去[https://mvnrepository.com/](https://mvnrepository.com/ 仓库) 搜索。

# 编码

右击 *src* > *main* > *scala* 目录，选择 *New* > *package*，在弹框中输入package名，点击 "OK"。创建package 完毕。

右击刚创建的package，选择 *New* > *Scala Class*，输入 Class 名(举例："Example")，将 Kind 改为 Object，点击OK完成按钮，在新建的源文件中编写源代码（示例参考Spark官网，package名字、logFile地址需要根据自身情况修改）

```scala
package io.github.lxwei
import org.apache.spark.sql.SparkSession

/**
  * Created by lxwei on 17/10/24.
  */
object Example {

  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
```

#打包

打包采用sbt，运行`sbt package`.

> ***sbt package***
> [info] Loading global plugins from /Users/ldtech/.sbt/0.13/plugins
> [info] Loading project definition from /Users/ldtech/learn/spark/spark-template-lxwei/project
> [info] Set current project to spark (in build file:/Users/ldtech/learn/spark/spark-template-lxwei/)
> [info] Compiling 1 Scala source to /Users/ldtech/learn/spark/spark-template-lxwei/target/scala-2.11/classes...
> [info] Packaging $SomePlace/spark-template-lxwei/target/scala-2.11/spark_2.11-1.0.jar ...
> [info] Done packaging.
> [success] Total time: 5 s, completed 2017-10-24 22:24:44

可以看到，打包生成的jar包为`$SomePlace/spark-template-lxwei/target/scala-2.11/spark_2.11-1.0.jar` 。

# 运行

采用 `spark-submit` (位于spark安装目录的bin目录下) 提交应用程序。举例来说：

`spark-submit --class io.github.lxwei.Example --master local[4] $SomePlace/spark-template-lxwei/target/scala-2.11/spark_2.11-1.0.jar`

其中，spark-submit有多个参数：

* —class: 应用程序入口。
* —master: 集群的master地址，如spark://XXXXXX:7077；或者使用local模式在本地单线程运行，或使用 local[N] 在本地以N个线程运行。其中，local模式主要用于测试。
* —deploy-mode: 集群的部署模式，分为 Client 模式（默认）和 Cluster 模式。
* —application-jar: 包含应用程序和所有依赖的jar包的路径。
* —application-arguments: 传给主类main函数的参数。

举例来说，假设我们采用Yarn运行Spark，提交命令可以为

> spark-submit \
>
> —class "io.github.lxwei.Example" \
>
> —master yarn-cluster \
>
> —executor-memory 2G \
>
> —number-executors 10 \
>
> $SomePlace/spark-template-lxwei/target/scala-2.11/spark_2.11-1.0.jar

spark-submit 还有很多参数，详细的可以参阅相关文档。