---
title: Hive运行的基本原理
date: 2020-12-04 12:22:00
tags:
  - 大数据
  - Hive
categories: bigdata
---

## 简介

Hive 是一个构建在 Hadoop 之上的数据仓库，它可以将结构化的数据文件映射成表，并提供类 SQL 查询功能，用于查询的 SQL 语句会被转化为 MapReduce 作业，然后提交到 Hadoop 上运行。

特点：

1. 简单、容易上手 (提供了类似 sql 的查询语言 hql)，使得精通 sql 但是不了解 Java 编程的人也能很好地进行大数据分析；
2. 灵活性高，可以自定义用户函数 (UDF) 和存储格式；
3. 为超大的数据集设计的计算和存储能力，集群扩展容易;
4. 统一的元数据管理，可与 presto／impala／sparksql 等共享数据；
5. 执行延迟高，不适合做数据的实时处理，但适合做海量数据的离线处理。



## 基本数据类型

| 大类                                    | 类型                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| **Integers**（整型）                    | TINYINT—1 字节的有符号整数 SMALLINT—2 字节的有符号整数 INT—4 字节的有符号整数 BIGINT—8 字节的有符号整数 |
| **Boolean**（布尔型）                   | BOOLEAN—TRUE/FALSE                                           |
| **Floating point numbers**（浮点型）    | FLOAT— 单精度浮点型 DOUBLE—双精度浮点型                      |
| **Fixed point numbers**（定点数）       | DECIMAL—用户自定义精度定点数，比如 DECIMAL(7,2)              |
| **String types**（字符串）              | STRING—指定字符集的字符序列 VARCHAR—具有最大长度限制的字符序列 CHAR—固定长度的字符序列 |
| **Date and time types**（日期时间类型） | TIMESTAMP — 时间戳 TIMESTAMP WITH LOCAL TIME ZONE — 时间戳，纳秒精度 DATE—日期类型 |
| **Binary types**（二进制类型）          | BINARY—字节序列                                              |



## 隐式转换

type - primitive type - number - double - float - biting - int - smallint - tinyint

​                  \- boolean       - string

## 复杂类型

| 类型       | 描述                                                         | 示例                                   |
| ---------- | ------------------------------------------------------------ | -------------------------------------- |
| **STRUCT** | 类似于对象，是字段的集合，字段的类型可以不同，可以使用 名称.字段名 方式进行访问 | STRUCT ('xiaoming', 12 , '2018-12-12') |
| **MAP**    | 键值对的集合，可以使用 名称[key] 的方式访问对应的值          | map('a', 1, 'b', 2)                    |
| **ARRAY**  | 数组是一组具有相同类型和名称的变量的集合，可以使用 名称[index] 访问对应的值 | ARRAY('a', 'b', 'c', 'd')              |

内容格式

Hive 默认使用了几个平时很少出现的字符，这些字符一般不会作为内容出现在文件中

\n：对于文本文件来说，每行是一条记录，所以可以使用换行符来分割记录

^A：分割字段 (列)，在 CREATE TABLE 语句中也可以使用八进制编码 \001 来表示

^B：用于分割 ARRAY 或者 STRUCT 中的元素，或者用于 MAP 中键值对之间的分割，在 CREATE TABLE 语句中也可以使用八进制编码 \002 表示

^C：用于 MAP 中键和值之间的分割，在 CREATE TABLE 语句中也可以使用八进制编码 \003 表示



## 存储格式

**TextFile****：**存储为纯文本文件。这是Hive默认的文件存储格式。这种存储方式数据不做压缩，磁盘开销大，数据解析开销大。

**SequenceFile****：**SequenceFile 是 Hadoop API 提供的一种二进制文件，它将数据以<key,value>的形式序列化到文件中。这种二进制文件内部使用 Hadoop 的标准的 Writable 接口实现序列化和反序列化。它与 Hadoop API 中的 MapFile 是互相兼容的。Hive 中的 SequenceFile 继承自 Hadoop API 的 SequenceFile，不过它的 key 为空，使用 value 存放实际的值，这样是为了避免 MR 在运行 map 阶段进行额外的排序操作。

**RCFile****：**RCFile 文件格式是 FaceBook 开源的一种 Hive 的文件存储格式，首先将表分为几个行组，对每个行组内的数据按列存储，每一列的数据都是分开存储。

**ORC Files：**ORC 是在一定程度上扩展了 RCFile，是对 RCFile 的优化。

**Avro Files****：**Avro 是一个数据序列化系统，设计用于支持大批量数据交换的应用。它的主要特点有：支持二进制序列化方式，可以便捷，快速的处理大量数据；动态语言友好，Avro提供的机制使动态语言可以方便的处理Avro数据

**Parquet****：**Parquet 是基于 Dremel 的数据模型和算法实现的，面向分析型业务的列式存储格式。它通过按列进行高效压缩和特殊的编码技术，从而在降低存储空间的同时提高了IO效率。



**以上压缩格式中** **ORC** **和** **Parquet** **的综合性能突出，使用较为广泛，推荐使用这两种格式。**



通常在创建表的时候使用 STORED AS 参数指定存储格式

各个存储文件类型指定方式如下：

- STORED AS TEXTFILE
- STORED AS SEQUENCEFILE
- STORED AS ORC
- STORED AS PARQUET
- STORED AS AVRO
- STORED AS RCFILE



## 内部表和外部表

内部表又叫做管理表，创建表时不做任何指定，默认创建的就是内部表。创建外部表需要使用 External 进行修饰

- **存储位置**

​	内部表：由hive.metastore.warehouse.dir指定，默认在hdfs的/user/hive/warehouse/数据库名.db/表名/ 目录下

​	外部表：创建表时由 Location 参数指定

- **导入数据**

​	内部表：将数据移动到子弟的数据仓库目录下，数据的生命周期由 Hive 来进行管理

​	外部表：不回移动数据到数据仓库目录，只是在原数据中存储数据的位置

- **删除表**

​	内部表：删除元数据（metadata）和文件

​	外部表：只删除元数据（metadata）



## SQL转化为MapReduce的过程

1. 语法解析：Antlr 定义 SQL 的语法规则，完成 SQL 词法，语法解析，将 SQL 转化为抽象 语法树 AST Tree；
2. 语义解析：遍历 AST Tree，抽象出查询的基本组成单元 QueryBlock；
3. 生成逻辑执行计划：遍历 QueryBlock，翻译为执行操作树 OperatorTree；
4. 优化逻辑执行计划：逻辑层优化器进行 OperatorTree 变换，合并不必要的 ReduceSinkOperator，减少 shuffle 数据量；
5. 生成物理执行计划：遍历 OperatorTree，翻译为 MapReduce 任务；
6. 优化物理执行计划：物理层优化器进行 MapReduce 任务的变换，生成最终的执行计划。



源码分析

**Hive**是执行命令的流程

1. 启动时初始化SessionState，初始化Config，初始化Log

2. CliDriver main中接受命令行

3. 处理命令行，分为以下几种情况

4. 1. 处理quit和exit，直接退出
   2. 处理source，执行SQL文件
   3. 处理感叹号命令
   4. 其他命令，包括select等SQL，CommandProcessorFactory.get(tokens, (HiveConf) conf); 在该工厂中会通过用户输入的第一个单词判断命令类型，在HiveCommand这个枚举中定义了一些非SQL查询操作，匹配到命令会选择合适的CommandProcessor实现，比如dfs命令对应DFSProcessor，set命令对应的SetPRocessor等，如果是Select之类的SQL查询，则返回null，然后为这些SQL命令创建一个Driver。

5. 获得processor之后开始执行，int processLocalCmd(String cmd, CommandProcessor proc, CliSessionState ss)，调用processor的run方法开始执行命令

6. 执行完成之后，通过List<FieldSchema> fieldSchemas = qp.getSchema().getFieldSchemas()获取结果的列名，并随后打印，最后再打印结果集



## Hive执行SQL语句

根据上一节内容，在获取processor之后开始执行命令，这里我们来到了Driver，我们进入到 private CommandProcessorResponse runInternal(String command, boolean alreadyCompiled) 这样一个方法，在该方法中发现一个有意思的东西，Hook，意味着可以写一个外部class配置到hive中查看hive的执行情况，接着进入重点

1. ret = compileInternal(command); 在该方法中主要是将SQL解析为物理和逻辑执行计划，和Spark差不多

2. 1. 将SQL解析为AST树：ASTNode tree = pd.parse(command, ctx); 
   2. 初始化事务管理器，记录这次query的信息：SessionState.get().initTxnMgr(conf);
   3. 执行hook：tree = hook.preAnalyze(hookCtx, tree);  
   4. 创建逻辑和物理执行计划：sem.analyze(tree, ctx);
   5. 执行hook：hook.postAnalyze(hookCtx, sem.getRootTasks()); 

3. ret = execute();

4. 1. 循环执行hook
   2. 初始化运行容器：DriverContext driverCxt = new DriverContext(ctx); driverCxt.prepare(plan); 
   3.  添加running任务：driverCxt.addToRunnable(tsk); 任务会进入一个队列 Queue<Task<? extends Serializable>> runnable; runnable.add(tsk); 
   4. 在一个while中启动任务：TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
   5. poll已经完成的任务，并加到hookContext的完成任务列表中：TaskRunner tskRun = driverCxt.pollFinished(); hookContext.addCompleteTask(tskRun);
   6. 遍历子任务加到running：for (Task<? extends Serializable> child : tsk.getChildTasks()) … driverCxt.addToRunnable(child);
   7. 最后计算CPU使用情况，任务完成：plan.setDone()，将该planId加到一个Set集合中