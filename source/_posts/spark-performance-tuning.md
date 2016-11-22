---
title: Spark性能调优
date: 2016-11-20 10:44:15
tags: Spark
---

# 前言

Spark是一个优秀的弹性分布式计算系统，但是它不是一个“神奇”的分布式系统，你需要正确的使用它提供的API并且合理的分配资源。
本文介绍的技巧主要是针对原生的RDD API。Spark SQL和Dataset在引擎层面提供了不同程度的优化，以后的文章再专门介绍。

# 概念

在Spark体系里，有两个层面的概念需要掌握。

- 开发层面：RDD, Transformation, Action等
- 执行层面：Job、Stage、Task、Executor、Shuffle、Partition

那么在进行程序优化的时候，也要针对这两个层面进行优化。不仅要正确的编写Spark的程序，还要在执行程序的时候，正确的分配资源。

# Spark如何运行你的程序

一个典型的Spark应用，都有一个Driver和多个Executor。Driver是一个中心调度器，负责把Job分配给各个executor。executor分布
在集群的各个机器上，负责执行任务。

![Spark执行图](/images/spark-tuning-f1.png)

RDD的tranformation不会触发程序的执行，只有Action才会。一个Action API对应一个Job的执行。

一个Job分成很多个Stage，每个Stage被划分成很多个Task，每个Task运行的代码是一样的，只是他们操作在不同分区（partition）的数据上。这些Task会被分派
到各个Executor上去执行，一个Executor可以同时执行多个Task，取决于Executor能分配的资源(CPU,内存).

在一个Stage内部的所有Transformation操作都不会触发Shuffle。Stage之间会发生shuffle。（反过来想，如果两个stage之间没有shuffle，那么可以合成一个Stage ^-^)

在了解了Spark的运行机制后，那么就可以针对其中的每个环节进行优化。

# [优化1] 减少不必要的Shuffle

Shuffle的意思是洗牌。就像洗扑克牌一样，把数据重新排列分部。注意Shuffle很多时候是必可避免的，甚至有时候为了提高性能，需要先进行shuffle把数据重新分配到更多的分区以利用集群的计算资源。
但是我们一定要努力减少**不必要**的Shuffle操作，因为Shuffle会把数据写到硬盘，然后供下一个stage读取。这会大大增加网络和IO的消耗。记住，在大数据的世界里，“计算”是比“数据”更便宜的资源。
要尽量移动计算，而不是数据。

而不必要的Shuffle的产生，往往和错误的使用API有关。

## Shuffle是如何产生的

RDD是Spark的核心。RDD是一种分布式的数据结构，但RDD提供很多API来操作数据。

有一些API，例如filter(), map()， 对于一个分区的数据，可以直接进行操作。这类操作被称为Narrow Transformation (narrow我的理解就是没有夸partition之间的依赖)。
而又另外的一些API，例如groupByKey(), reduceByKey()，这些操作需要把相同key的数据都先放到一个partition，以便于能被同一个task执行。这类操作被称为Wide Transformation。
这时候就需要进行shuffle。

Here are all RDD Transformation (v2.0)

**Narrow Transformation:**
- map
- filter
- flatMap
- mapPartitions
- mapPartitionsWithIndex
- sample
- union
- intersection
- distinct ??


** Wide Transformation:**
- groupByKey
- reduceByKey
- aggregateByKey
- sortByKey
- join (Not always)
- cogroup
- cartesian
- repartition
- repartitionAndSortWithPartitions
- coalesce

*例子*

```scala
sc.textFile("someFile.txt").
  map(mapFunc).
  flatMap(flatMapFunc).
  filter(filterFunc).
  count()
```

这段代码里，有一个action：count()，因此会产生一个job。这个job只包含一个stage，所有的tranformation都没有操作涉及到shuffle。

```scala
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).
  reduceByKey(_ + _)
charCounts.collect()
```

这段代码有一个action：collect()。它有两个wide transformation: reduceByKey()， 这两个reduceByKey()操作把job划分为3个stage，第一个和第二个stage要做shuffle的操作。

1. 第一个stage：textFile --> flatMap --> map --> reduceByKey
2. 第二个stage: filter --> flatMap --> map --> reduceByKey
3. 第三个stage: count

## 正确的使用API

条条大路通罗马，但不是每条道路都省时省心。在使用Spark的API进行数据运算时，往往有很多种做法，但不是每种做法都是高效的。

总的原则：避免shuffle或者减少shuffle的数据量。

1. 在使用reduce能使数据减少的情况下，使用reduceBykey()而不是groupByKey()

rdd.groupByKey().mapValues(_.sum) 会将整个RDD先shuffle然后再对每个partition的数据进行相加
rdd.reduceByKey(_ + _) 会将每个partition的数据先加在一起，然后再shuffle

这样能显著减少shuffle的数据量。但是如果reduce的算子不是 _ + _， 而是某些让数据数量不变甚至增长的情况，则不适用。

2. 如果目标RDD的类型是一个集合类型，尽量使用mutable的集合类型


```
rdd.map(kv => (kv._1, new Set[String]() + kv._2))
    .reduceByKey(_ ++ _)
```
这种写法会为每一个map操作新建一个Set，非常损耗性能。

```
val zero = new collection.mutable.Set[String]()
rdd.aggregateByKey(zero)(
    (set, v) => set += v,
    (set1, set2) => set1 ++= set2)
```
这种写法没有每次new一个Set的开销，更加高效。

3. 当数据量很少时，使用Broadcast而不是建成一个RDD。

通常做法是把一个集合变成一个哈希表，然后广播到各个executor，然后在做Map操作时，直接通过Key来引用对应的值。

## Partition对shuffle的影响

Spark总是竭尽所能减少shuffle的发生。例如：

```
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
```

如果没有优化，那么产生rdd1需要一次shuffle，产生rdd2需要一次shuffle，产生rdd3需要两次shuffle （rdd1和rdd2各自shuffle一次）

rdd1和rdd2都使用reduceByKey()并且没有指定partitioner，那么他们使用的是默认的partitioner, 如果他们的partition数据也是一样的，
同样的key在两个RDD中都只能位于各自的某个partition中，rdd3的每个partition的数据都来源于rdd1和rdd2的一个partition。因此总共shuffle了两次。

但是如果rdd1和rdd2两个的partition数目不一样，或者数据一样，但是他们的partitioner不一样，那么就总共需要三次次shuffle。
第一次: someRdd进行shuffle然后得到rdd1
第二次: someOtherRDD进行shuffle然后得到rdd2
第三次：将partition数目较少的RDD进行shuffle，使得和partition数据较多的RDD partition数目一样，然后再进行join。


# 优化2：合理分配计算资源

除了尽量避免shuffle的产生或者减少shuffle的数据量，还需要为每个executor分配正确的资源。Spark管理的资源主要有CPU和内存。
对于网络和IO等资源，Spark并没有提供主动的管理。

这里主要讲基于Yarn的资源分配。

## CPU内核分配

Spark可以给每个executor指定其最大可以使用的CPU核数。

有三个地方可以指定：

- spark-submit,spark-shell: --executor-cores
- spark-defaults.conf: spark.executor.cores
- SparkConf

当Spark和Yarn结合时，Spark能申请的CPU资源还要受限于Yarn。

yarn.nodemanager.resource.cpu-vcores 参数控制每个Yarn容器能使用的CPU数目。

当Spark申请3个核时，实际上Spark是向Yarn申请了3个vcores.

## 内存分配

有三个地方可以指定：

-  spark-submit,spark-shell: --executor-memory
-  spark-defaults.conf: spark.executor.memory
-  SparkConf

Spark向Yarn申请内存时比申请CPU要复杂。

- --executor-memory 控制的是executor的堆大小，但是每个executor还需要使用额外的内存空间来做缓存。
- spark.yarn.executor.memoryOverhead 用于控制向Yarn申请的额外的内存。它的默认值等于：max(384, 0.07 * spark.executor.memory)
- yarn.scheduler.minimum-allocation-mb 控制一个yarn容器最小分配的内存

![Spark内存申请](/images/spark-tuning2-f1.png)

## 分配原则

1. Spark的application master也需要占用资源。在yarn-client模式下，默认占用1个CPU核和1G内存。在yarn-cluster模式下，application
master就是driver，由于application master同时也可能跑executor，因此要通过 --driver-memory 和 --driver-cores来为driver预留
足够的程序。
2. 不应该为一个executor分配太多的内存，这样反而会引起垃圾回收的延时。一般一个executor分配的内存最大不超过64G
3. 一个executor一般分配不超过5个CPU核心，太多的话可能会使得hadoop写入文件阻塞（希望后来没有这个问题 !)
4. 尽量不要分配一个executor只有一个CPU内核，然后在一台机器上创建很多个executor。主要有两个坏处：
    - broadcast是建立在executor上的，太多executor导致太多的广播变量
    - 每个executor都会占用一个额外的内存开销
5. 永远要预留一个CPU内核和一定的内存供操作系统使用

## 实例

假设一个集群有6台机器，每台机器有16核，64G内存，那么该如何分配资源呢？

首先给yarn分配资源：

yarn.nodemanager.resource.cpu-vcores 15
yarn.nodemanager.resource.memory-mb  63G

要为系统进程预留1个核和1G的内存。


然后给spark的executor分配CPU和内存。一种最直接的分配方案：

--num-executors=6  --executor-cores=15 --executor-memory=63G

也就是每天机器创建一个executor，每个executor占用15个核心，63G内存。但这是一个不可行的方案。首先每个executor分配15个核心，会导致HDFS
被阻塞，而且一个executor占用63G内存，加上额外的开销就超过63G了。

一种优化的方案为：

--num-executors=17  --executor-cores=5 --executor-memory=19

- 每个executor占用5个核心。--executor-cores=5
- 每个机器可以有 15 / 5 = 3 个executor, 6台机器一共可以创建18个executor，但是我们要除去application master， 因此共有18-1=17个executor
- 每天机器上的3个executor，每个executor可以分配到 63 / 3 = 21 G内存，但是 21G应该是包含了额外的开销的，假设额外开销为 0.07 * X
  0.07 * X + X = 21, X = 19.6, 向下舍去，为19G

这样的分配不仅可以充分利用资源，而且一般不会出现内存溢出的情况。


# 优化三：使用更高效的数据存储

例如使用parquet替代CSV，JSON，使用KryoSerializer替代默认的Java Serializer。这里不做重点介绍。

# 优化四：增加并行度

一般来说，在一个Stage里，task的数目和父亲RDD的partition数据是一样的 ，产生的子RDD的partition数目也是一样的。

但是有些操作，可以改变子RDD的partition数目：

- coalesce可以将父亲RDD的分区数目压缩
- union操作产生的RDD分区数目是两个父亲RDD分区的和。
- catesian产生的RDD分区数据是两个父亲RDD分区的乘积。

分区的数据，决定了Job并行执行的程度。如果有100机器，但是数据只有2个分区，那么一次就只有2个task在执行，其它机器都在空转。

分区太少，会使得单个task要执行的数据过多，占用的时间和空间也较大。那么到底分多少个分区合适呢？

第一种办法就是不断的尝试逼近：找到父亲RDD的分区数目，然后不断乘以1.5，知道发现性能无法获得提升为止。但是这种办法在显示中不太可行，
因为你不太可能一次次的去跑同样的job。

另外一种尝试就是，根据系统的CPU，内存结合程序的特点来大概计算，但是很难量化。

总的原则就是：多些分区要比少分区要好，因为在spark里创建task是很便宜的。



# 参考资料

- http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/
- http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-2/





