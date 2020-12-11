---
title: Spark SQL原理
date: 2020-12-11 12:22:00
tags:
  - 大数据
  - Spark SQL
categories: bigdata
---

# Spark SQL原理

## 简介

Spark SQL 是 Spark 中的一个子模块，主要用于操作结构化数据。它具有以下特点：

- 能够将 SQL 查询与 Spark 程序无缝混合，允许您使用 SQL 或 DataFrame API 对结构化数据进行查询；
- 支持多种开发语言；
- 支持多达上百种的外部数据源，包括 Hive，Avro，Parquet，ORC，JSON 和 JDBC 等；
- 支持 HiveQL 语法以及 Hive SerDes 和 UDF，允许你访问现有的 Hive 仓库；
- 支持标准的 JDBC 和 ODBC 连接；
- 支持优化器，列式存储和代码生成等特性；
- 支持扩展并能保证容错。

## DataFrame和RDD

主要区别在于RDD面向的是非结构化数据，DataFreame面向的是结构化数据。

DataFrame 内部的有明确 Scheme 结构，即列名、列字段类型都是已知的，这带来的好处是可以减少数据读取以及更好地优化执行计划，从而保证查询效率。

## DataFrame 和 RDDs 应该如何选择？

- 如果你想使用函数式编程而不是 DataFrame API，则使用 RDDs；
- 如果你的数据是非结构化的 (比如流媒体或者字符流)，则使用 RDDs，
- 如果你的数据是结构化的 (如 RDBMS 中的数据) 或者半结构化的 (如日志)，出于性能上的考虑，应优先使用 DataFrame。

## DataSet

DataSet 也是分布式的数据集合，在Spark1.6版本引入，它继承了RDD和DataFrame的优点，具备强类型的特点，同时支持Lambda函数，但只能在 Scala 和 Java 语言中使用。在 Spark 2.0 后，为了方便开发者，Spark 将 DataFrame 和 Dataset 的 API 融合到一起，提供了结构化的 API(Structured API)，即用户可以通过一套标准的 API 就能完成对两者的操作。



静态类型与运行时类型安全

静态类型 (Static-typing) 与运行时类型安全 (runtime type-safety) 主要表现如下:



在实际使用中，如果你用的是 Spark SQL 的查询语句，则直到运行时你才会发现有语法错误，而如果你用的是 DataFrame 和 Dataset，则在编译时就可以发现错误 (这节省了开发时间和整体代价)。DataFrame 和 Dataset 主要区别在于：



在 DataFrame 中，当你调用了 API 之外的函数，编译器就会报错，但如果你使用了一个不存在的字段名字，编译器依然无法发现。而 Dataset 的 API 都是用 Lambda 函数和 JVM 类型对象表示的，所有不匹配的类型参数在编译时就会被发现。



以上这些最终都被解释成关于类型安全图谱，对应开发中的语法和分析错误。在图谱中，Dataset 最严格，但对于开发者来说效率最高。

## Untypeed 和 Typed

DataFrame API 被标记为 Untyped API，而 DataSet API 被标记为 Typed API。DataFrame 的 Untyped 是相对于语言或 API 层面而言，它确实有明确的 Scheme 结构，即列名，列类型都是确定的，但这些信息完全由 Spark 来维护，Spark 只会在运行时检查这些类型和指定类型是否一致。这也就是为什么在 Spark 2.0 之后，官方推荐把 DataFrame 看做是 DatSet[Row]，Row 是 Spark 中定义的一个 trait，其子类中封装了列字段的信息。



相对而言，DataSet 是 Typed 的，即强类型。如下面代码，DataSet 的类型由 Case Class(Scala) 或者 Java Bean(Java) 来明确指定的，在这里即每一行数据代表一个 Person，这些信息由 JVM 来保证正确性，所以字段名错误和类型错误在编译的时候就会被 IDE 所发现。



val dataSet: Dataset[Person] = spark.read.json("people.json").as[Person]

## DataFrame & DataSet & RDDs 总结

- RDDs 适合非结构化数据的处理，而 DataFrame & DataSet 更适合结构化数据和半结构化的处理；
- DataFrame & DataSet 可以通过统一的 Structured API 进行访问，而 RDDs 则更适合函数式编程的场景；
- 相比于 DataFrame 而言，DataSet 是强类型的 (Typed)，有着更为严格的静态类型检查；
- DataSets、DataFrames、SQL 的底层都依赖了 RDDs API，并对外提供结构化的访问接口。

## 运行原理

DataFrame、DataSet 和 Spark SQL 的实际执行流程都是相同的：

1. 进行 DataFrame/Dataset/SQL 编程；
2. 如果是有效的代码，即代码没有编译错误，Spark 会将其转换为一个逻辑计划；
3. Spark 将此逻辑计划转换为物理计划，同时进行代码优化；
4. Spark 然后在集群上执行这个物理计划 (基于 RDD 操作) 。