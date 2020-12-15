---
title: Hive文件存储及优化
date: 2020-12-15 11:48:56
tags: 
  - hive
categories:
  - bigdata
---

最近在做大数据项目上线准备，需要知道Hive性能消耗情况，而我们数据仓库在建设使用的过程中，主要消耗的资源包含：CPU、MEMORY、DISK三部分。

数据仓库在计算过程中主要消耗CPU和Memory资源，当然也会消耗一些DISK资源用来存储计算过程中的临时结果。但是主要优化的方向，还是降低CPU和MEMORY的消耗，这方面主要依赖于模型设计的合理性，所以在模型设计阶段增加模型设计review的步骤，保证模型设计的合理性。

数据仓库在数据的存储阶段主要消耗MEMORY和DISK资源。memory资源主要是在NameNode存储文件信息的时候消耗掉；DISK在存储数据的时候消耗掉。在这个阶段需要严格控制HIVE表的的定义，来降低资源的消耗。

本次主要探讨是数据仓库在数据存储阶段对资源消耗的优化，下面将通过2个方面展开，分别是：数据仓库如何配置，可以实现数据压缩，降低数据的存储量，达到减少对DISK的消耗；数仓表如何设计，可以降低文件信息存储量，达到减少对MEMORY的消耗。

## 小文件问题

Hive的数据最终是存储在HDFS上，HDFS的NameNode会保存文件的元数据，为了提高响应速度，NameNode在启动的时候会将这些元数据加载到内存，而HDFS中的每一个文件、目录及文件块，在NameNode内存都会记录，每一条信息大约占用150字节的内存空间。如果HDFS 上存在大量的小文件（这里说的小文件是指文件大小要比一个 HDFS 块大小(在 Hadoop1.x 的时候默认块大小64M，可以通过 dfs.blocksize 来设置；但是到了 Hadoop 2.x 的时候默认块大小为128MB了，可以通过 dfs.block.size 设置) 小得多的文件）至少会产生以下几个负面影响：

- 大量的小文件会占用大量的NameNode内存，一个元数据对象大约占用150个字节，一千万文件及分块就大约 会占用3G的内存空间；
- 如果使用MapReduce任务来处理这些小文件，因为每一个Map会处理一个HDFS Block，这会导致程序需要启动大量的Map来处理这些小文件，虽然这些小文件总的大小不算很大，却占用了集群的大量资源。

## Hive小文件产生的原因

- MapReduce任务产生：我们使用 Hive 查询一张含有海量数据的表，然后存储在另外一张表中，而这个查询只有简单的过滤条件（比如 select * from iteblog where from = 'hadoop'），这种情况只会启动大量的 Map 来处理，这种情况可能会产生大量的小文件。也可能 Reduce 设置不合理，产生大量的小文件；
- Hive统计汇总表，越是上层的表汇总程度就越高，数据量也就越小，而且这些表通常会有日期分区，随着时间的推移，HDFS的文件数目就会逐步增加；

## 小文件问题解决

- 输入合并。即在map前合并小文件。

- 输出合并。即在输出结果的时候合并小文件。

### 1. 配置Map输入合并

```shell
-- 每个Map最大输入大小，决定合并后的文件数
set mapred.max.split.size=256000000;
-- 一个节点上split的至少的大小 ，决定了多个data node上的文件是否需要合并
set mapred.min.split.size.per.node=100000000;
-- 一个交换机下split的至少的大小，决定了多个交换机上的文件是否需要合并
set mapred.min.split.size.per.rack=100000000;
-- 执行Map前进行小文件合并
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

### 2. 配置Hive结果合并

```shell
set hive.merge.mapfiles = true #在Map-only的任务结束时合并小文件
set hive.merge.mapredfiles = true #在Map-Reduce的任务结束时合并小文件
set hive.merge.size.per.task = 256*1000*1000 #合并文件的大小
set hive.merge.smallfiles.avgsize=16000000 #当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge
```

hive在对结果文件进行合并时会执行一个额外的map-only脚本，mapper的数量是文件总大小除以size.per.task参数所得的值，触发合并的条件是：根据查询类型不同，相应的mapfiles/mapredfiles参数需要打开；结果文件的平均大小需要大于avgsize参数的值。

### 3. 压缩文件的处理

对于输出结果为压缩文件形式存储的情况，要解决小文件问题，如果在map输入前合并，对输出的文件存储格式并没有限制。但是如果使用输出合并，则必须配合SequenceFile来存储，否则无法进行合并，以下是实例：

```SQL
set mapred.output.compression.type=BLOCK;
set hive.exec.compress.output= true;
set mapred.output.compression.codec=org.apache.hadoop.io.compress.LzoCodec;
set hive.merge.smallfiles.avgsize=100000000;

drop table if exists dw_stage.zj_small;
create table dw_stage.zj_small
STORED AS SEQUENCEFILE
as select *
from dw_db.dw_soj_imp_dtl
where log_dt = '2014-04-14'
and paid like '%baidu%' ;
```

### 4. 使用HAR归档文件

Hadoop的归档文件格式也是解决小文件问题的方式之一。而且hive提供了原生支持：

```sql
set hive.archive.enabled= true;
set hive.archive.har.parentdir.settable= true;
set har.partfile.size=1099511627776;
ALTER TABLE srcpart ARCHIVE PARTITION(ds= '2008-04-08', hr= '12' );
ALTER TABLE srcpart UNARCHIVE PARTITION(ds= '2008-04-08', hr= '12' );
```

如果使用的不是分区表，则可以创建成外部表，并使用har://协议来指定路径。

## 文件类型

hive在存储数据时支持通过不同的文件类型来组织，并且为了节省相应的存储资源，也提供了多种类型的压缩算法，供用户选择。只要是配置正确的文件类型和压缩类型，hive都可以按预期读取并解析数据，不影响上层HQL语句的使用。例如：SequenceFile本身的结构已经设计了对内容进行压缩，所以对于SequenceFile文件的压缩，并不是先生成SequenceFile文件，再对文件进行压缩；而是生成SequenceFile文件时，就对其中的内容字段进行压缩。最终压缩后，对外仍然体现为一个SequenceFile。RCFile、ORCFile、Parquet、Avro对于压缩的处理方式与SequenceFile相同。

hive支持的文件类型有：TextFile、SequenceFile、RCFile、ORCFile、Parquet、Avro。

<table>
  <tr>
    <th>存储格式</td>
    <th>存储方式</td>
    <th>特点</td>
  </tr>
  <tr>
    <td rowspan="5">TextFile</td>
    <td rowspan="5">行存储</td>
    <td>存储空间消耗比较大</td>
  </tr>
  <tr>
    <td>压缩的text无法分割和合并</td>
  </tr>
  <tr>
    <td>查询效率低</td>
  </tr>
  <tr>
    <td>可以直接存储</td>
  </tr>
  <tr>
    <td>加载数据速度最高</td>
  </tr>
  <tr>
    <td rowspan="4">SequenceFile</td>
    <td rowspan="4">行存储</td>
    <td>存储空间消耗最大</td>
  </tr>
  <tr>
    <td>压缩的文件可以分割和合并</td>
  </tr>
  <tr>
    <td>查询效率高</td>
  </tr>
  <tr>
    <td>需要通过text文件转化来加载</td>
  </tr>
  <tr>
    <td rowspan="8">RCFile</td>
    <td rowspan="8">数据按行分块，每块按列存储</td>
    <td>存储空间最小</td>
  </tr>
  <tr>
    <td>查询效率最高</td>
  </tr>
  <tr>
    <td>需要通过text文件转化来加载</td>
  </tr>
  <tr>
    <td>加载的速度最低</td>
  </tr>
  <tr>
    <td>压缩快，快速列存储</td>
  </tr>
  <tr>
    <td>读记录尽量设计到的block最少</td>
  </tr>
  <tr>
    <td>读取需要的列只需要读取每个row group的头部定义</td>
  </tr>
  <tr>
    <td>读取全量数据的操作性能可能比sequencefile没有明显的优势</td>
  </tr>
  <tr>
    <td rowspan="2">ORCFile</td>
    <td rowspan="2">数据按行分块，每块按列存储</td>
    <td>压缩快，快速列存储</td>
  </tr>
  <tr>
    <td>效率比RCFile高，是RCFile的改良版</td>
  </tr>
  <tr>
    <td rowspan="4">Parquet</td>
    <td rowspan="4">列存储</td>
    <td>相对于PRC，Parquet压缩比较低</td>
  </tr>
  <tr>
    <td>查询效率比较低</td>
  </tr>
  <tr>
    <td>不支持update、insert和ACID</td>
  </tr>
  <tr>
    <td>支持Impala查询引擎</td>
  </tr>
  <tr>
    <td rowspan="5">Avro</td>
    <td rowspan="5">二进制存储</td>
    <td>基于列(在列中存储数据):用于数据存储是包含大量读取操作的优化分析工作负载</td>
  </tr>
  <tr>
    <td>与Snappy的压缩压缩率高(75%)</td>
  </tr>
  <tr>
    <td>只需要列将获取/读(减少磁盘I / O)</td>
  </tr>
  <tr>
    <td>可以使用Avro API和Avro读写模式</td>
  </tr>
  <tr>
    <td>支持谓词下推(减少磁盘I / O的成本)</td>
  </tr>
</table>

## 压缩算法

待补充

## 表分区优化

- 对于统计数据表、数据量不大的基础表、业务上无累计快照和周期性快照要求的数据表，尽可能的不创建分区，而采用数据合并回写的方式解决；
- 对于一些数据量大的表，如果需要创建分区，提高插叙过程中数据的加载速度，尽可能的只做天级分区。而对于原始数据，这种特大的数据量的，可以采用小时分区。对于月分区，坚决去掉。
- 对于一些周期快照和累计快照的表，我们尽可能只创建日分区。