---
title: Spark源码
date: 2020-12-11 12:22:00
tags:
  - 大数据
  - Spark
categories: bigdata
---

# Spark SQL源码分析

**调用SQL时**

```scala
SparkSession ->

def sql(sqlText: String): DataFrame = {

 Dataset.ofRows(self, sessionState.sqlParser.parsePlan(sqlText))

}

```

## Parser

parsePlan方法会返回一个LogicalPlan对象；

第一步，利用 antlr4 生成的 SqlBaseLexer【val lexer = new SqlBaseLexer(new UpperCaseCharStream(CharStreams.fromString(command)))】 对SQL进行词法分析，生成一个CommonTokenStream 对象【val tokenStream = new CommonTokenStream(lexer)】

第二步，利用 antlr4 生成的 SqlBaseParser 【val parser = new SqlBaseParser(tokenStream)】对SQL进行语法分析，得到 Unresolved LogicalPlan



以下均在 QueryExecution 中执行

## Analyzer

Analyzer 持有一个 SessionCatalog 对象的引用

Analyzer 继承自 RuleExecutor[LogicalPlan]，因此可以对 LogicalPlan 进行转换

```scala
lazy val analyzed: LogicalPlan = {

 SparkSession.setActiveSession(sparkSession)

 sparkSession.sessionState.analyzer.executeAndCheck(logical)

}

```

通过 Catalog 确定每张表对应的字段集、字段类型、数据存储位置，生成Resolved Logical Plan

```scala
def checkAnalysis(plan: LogicalPlan): Unit = {

	case u: UnresolvedRelation =>

	 u.failAnalysis(s"Table or view not found: ${u.tableIdentifier}")

}
```

ResolveRelations

```scala
// 关联表
private def lookupTableFromCatalog(

  u: UnresolvedRelation,

  defaultDatabase: Option[String] = None): LogicalPlan ={

}

```

## Optimizer

```scala
lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)
```

逻辑优化器，会进行谓词下推，列值裁剪，常量折叠，谓词合并等等一系列逻辑优化

根据预先定义好的规则对 Resolved Logical Plan 进行优化并生成 Optimized Logical Plan

## SparkPlanner

```scala
lazy val sparkPlan: SparkPlan = {

 SparkSession.setActiveSession(sparkSession)

 // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,

 //    but we will implement to choose the best plan.

 planner.plan(ReturnAnswer(optimizedPlan)).next()

}
```

把 Logical Plan 转变为 Physical Plan

## 执行 Physical Plan

lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

**转成RDD**

```scala
/** Internal version of the RDD. Avoids copies and has no schema */

lazy val toRdd: RDD[InternalRow] = {

 if (sparkSession.sessionState.conf.getConf(SQLConf.USE_CONF_ON_RDD_OPERATION)) {

  new SQLExecutionRDD(executedPlan.execute(), sparkSession.sessionState.conf)

 } else {

  executedPlan.execute()

 }

}
```

