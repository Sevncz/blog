---
title: Spark 运行的基本原理
date: 2020-12-05 12:22:00
tags:
  - 大数据
  - Spark
categories: bigdata
---

## **RDD**基本概念

A Resilient Distributed Dataset (RDD), the basic abstraction in Spark. Represents an immutable, partitioned collection of elements that can be operated on in parallel.

- RDD是Resilient Distributed Dataset(弹性分布式数据集)的简称。RDD的弹性体现在计算方面，当Spark进行计算时，某一阶段出现数据丢失或者故障，可以通过RDD的血缘关系就行修复。     

- - \1. 内存的弹性：内存与磁盘的自动切换	
  - \2. 容错的弹性：数据丢失可以自动恢复
  - \3. 计算的弹性：计算出错重试机制
  - \4. 分片的弹性：根据需要重新分片 

- RDD是不可变(immutable)的，一旦创建就不可改变。RDDA-->RDDB，RDDA 经过转换操作变成RDDB，这两个RDD具有血缘关系，但是是两个不同的RDD，体现了RDD一旦创建就不可变的性质。

## RDD特点

- A list of partitions
- A function for computing each split
- A list of dependencies on other RDDs
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)



源码解析

- A list of partitions

/**

 \* Implemented by subclasses to return the set of partitions in this RDD. This method will only

 \* be called once, so it is safe to implement a time-consuming computation in it.

 *

 \* The partitions in this array must satisfy the following property:

 \*  `rdd.partitions.zipWithIndex.forall { case (partition, index) => partition.index == index }`

 */

protected def getPartitions: Array[Partition]



- compute函数的入参必然是partition，因为对RDD做计算相当于对每个partition做计算

/**

 \* :: DeveloperApi ::

 \* Implemented by subclasses to compute a given partition.

 */

@DeveloperApi

def compute(split: Partition, context: TaskContext): Iterator[T]



- RDD之间有依赖关系

/**

 \* Implemented by subclasses to return how this RDD depends on parent RDDs. This method will only

 \* be called once, so it is safe to implement a time-consuming computation in it.

 */

protected def getDependencies: Seq[Dependency[_]] = deps



- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)

/**

 \* Optionally overridden by subclasses to specify placement preferences.

 */

protected def getPreferredLocations(split: Partition): Seq[String] = Nil



- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

/** Optionally overridden by subclasses to specify how they are partitioned. */

@transient val partitioner: Option[Partitioner] = None

## RDD操作

- Transformations：接受RDD并返回RDD

- - Transformation采用惰性调用机制，每个RDD记录父RDD转换的方法，这种调用链表称之为血缘（lineage）

- Actions：接受RDD但是返回非RDD

- - Action调用会直接计算

## DAG、Stage、Shuffle

有向无环图，DAGScheduler负责生成DAG，然后将程序分发到分布式计算集群，按计算阶段的先后关系调度执行

并不是RDD上的每一个转换函数都会生成一个计算计算，通过观察DAG图，RDD之间的转换连接线呈现多对多交叉连接的时候，就会产生新的阶段，一个RDD代表一个数据集，每一个RDD都包含多个分片。

一个数据集中的多个数据分片需要进行分区传输，写入到另一个数据集的不同分片中，这种数据分区交叉传输的操作，和MapReduce类似，是shuffle过程，Spark也需要通过shuffle将数据进行重新组合，相同的Key的数据放在一起，进行聚合、关联等操作，因而每次shuffle都产生新的计算阶段。这也是为什么计算阶段会有依赖关系，它需要的数据来源于前面一个或多个计算阶段产生的数据，必须等待前面的阶段执行完毕才能进行shuffle，并得到数据

所以，计算阶段的划分以及是shuffle，而不是转换函数的类型，有的函数有时候有shuffle，有时候没有。

RDD 已经进行过分区，分区数目和分区Key不变，就不需要再进行shuffle，这种不需要进行shuffle的依赖，被称作窄依赖；相反的，需要进行shuffle的依赖，被称作宽依赖。



为什么**Spark**比**MapReduce**的效率更高？

从本质上看，Spark也算是一种MapReduce计算模型的不同实现。Hadoop MapReduce简单粗暴的根据shuffle将大数据计算分成了Map和Reduce阶段，然后就算完事了。而Spark更细腻一些，将前一个的Reduce和后一个的Map连接起来，当作一个阶段持续计算，形成一个更加优雅、高效的计算模型，虽然本质上仍然是Map和Reduce，但是这种多个计算阶段依赖执行的方案可以有效减少对HDFS的访问，减少作业的调度执行次数，因此执行速度也更快。并且Hadoop MapReduce主要使用磁盘存储Shuffle过程中的数据，Spark优先使用内存进行数据存储，包括RDD数据，除非是内存不够用了，否则是尽可能使用内存，这也是Spark性能比Hadoop MapReduce高的原因。