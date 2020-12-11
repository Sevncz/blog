---
title: HDFS原理浅析
date: 2020-12-11 12:22:00
tags:
  - 大数据
  - HDFS
categories: bigdata
---

# HDFS原理浅析

## 概述

- HDFS集群分为两大角色：NameNode、DataNode
- NameNode负责管理整个文件系统和每一个文件的元数据（元数据是block的ID，存储在哪一个DataNode）
- DataNode负责管理用户的文件数据块
- 文件会按照固定的大小（BlockSize）切成若干块后分布式存储在若干台DataNode上；默认大小：bock块在2.x版本中是128M，老版本中是64M
- 每一个文件块可以有多个副本，并存放在不同的DataNode上
- DataNode会定期向NameNode汇报自身所保存的文件Block信息，而NameNode则会负责保持文件的副本数量
- HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过NameNode申请来进行

## HDFS写文件步骤

客户端要向HDFS写数据，首先要跟NameNode通信以确认可以写文件并获得接收文件的Block和DataNode。然后，客户端按顺序将文件逐个Block传递给相应的DataNode，并由接收到Block的DataNode负责向其他DataNode复制Block的副本。

**详细步骤：**

1. Client向NameNode通信请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在；
2. NameNode返回是否可以上传；
3. Client请求第一个Block该传输到哪些DataNode服务器上；
4. NameNode返回3个DataNode服务器；
5. Client请求3台DataNode中的一台上传数据（本质是一个RPC调用，建立Pipeline），服务器接收到一个Packet就会传给第二台服务器，第二台服务器传给第三台服务器；服务器每传一个Packet会放入一个应答队列等待应答；
6. 当一个Block传输完成之后，Client再次请求Namenode上传第二个Block的服务器。

## HDFS读文件步骤

客户端将要读取的文件路径发送给NameNode，NameNode获取文件的元信息返回给客户端，客户端根据返回的信息找到相应的DataNode逐个获取文件的Block并在客户端本地进行数据追加合并从而获得整个文件。

**详细步骤：**

1. Client向NameNode通信请求读取文件；NameNode查询元数据，找到文件块所在的DataNode服务器；
2. 挑选一台DataNode（就近原则，然后随机）服务器，请求建立Socket连接；
3. DataNode开发发送数据（从磁盘里面读取数据放入流，以Packet为单位来做校验）；
4. Client以Packet为单位接收，现在本地缓存，然后写入目标文件

## NameNode工作机制

### NameNode工作职责

- 负责客户端请求的相应；

- 元数据的管理（查询，修改）；

  namenode对数据的管理采用了三种存储形式：

  - 内存元数据(NameSystem)
  - 磁盘元数据镜像文件
  - 数据操作日志文件（可通过日志运算出元数据）

### 元数据存储机制

A. 内存中有一份完整的元数据(内存meta data)

B. 磁盘有一个“准完整”的元数据镜像（fsimage）文件(在NameNode的工作目录中)

C. 用于衔接内存metadata和持久化元数据镜像fsimage之间的操作日志（edits文件）

> 注：当客户端对hdfs中的文件进行新增或者修改操作，操作记录首先被记入edits日志文件中，当客户端操作成功后，相应的元数据会更新到内存meta.data中

### 元数据手动查看

可以通过hdfs的一个工具来查看edits中的信息

```
bin/hdfs oev -i edits -o edits.xml` 
bin/hdfs oiv -i fsimage_0000000000000000087 -p XML -o fsimage.xml
```

### 元数据的checkpoin

每隔一段时间，会由secondary namenode将namenode上积累的所有edits和一个最新的fsimage下载到本地，并加载到内存进行merge（这个过程称为checkpoint）

**checkpoint的详细过程：**



**checkpoint触发条件配置参数：**

```properties
dfs.namenode.checkpoint.checkpoint.period=60 #检查触发条件是否满足的频率，60秒
dfs.namenode.checkpoint.checkpoint.dir=file://${hadoop.tmp.dir}/dfs/namesecondary
dfs.namenode.checkpoint.edits.dir=${dfs.namenode.checkpoint.dir} #以上两个参数做checkpoint操作时，secondary namenode的本地工作目录
dfs.namenode.checkpoint.max-retries=3 #最大重试次数
dfs.namenode.checkpoint.period=3600 #两次checkpoint之间的时间间隔3600秒
dfs.namenode.checkpoint.txns=1000000 #两次checkpoint之间最大的操作记录
```

**checkpoint的附带作用**

namenode和secondary namenode的工作目录存储结构完全相同，所以，当namenode故障退出需要重新恢复时，可以从secondary namenode的工作目录中将fsimage拷贝到namenode的工作目录，以恢复namenode的元数据

## DataNode的工作机制

### DataNode工作职责：

- 存储管理用户的文件块数据
- 定期向NameNode汇报自身所持有的Block信息（通过心跳信息上报）（这点很重要，因为，当集群中发生某些block副本失效时，集群如何恢复block初始副本数量的问题）

### Datanode掉线判断时限参数

datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为:
`timeout = 2 * heartbeat.recheck.interval + 10 * dfs.heartbeat.interval`

而默认的heartbeat.recheck.interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。
需要注意的是hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒。所以，举个例子，如果heartbeat.recheck.interval设置为5000（毫秒），dfs.heartbeat.interval设置为3（秒，默认），则总的超时时间为40秒。

## fsimage 和 edits

#### fsimage

NameNode 的元数据检查点，里面记录了自最后一次检查点之前HDFS文件系统中所有目录和文件的序列化信息

#### edits

保存了自最后一次检查点之后所有针对HDFS文件系统的操作，比如：增加文件、重命名文件、删除目录等等

#### 合并

1. 配置好HA后，客户端所有的更新操作将会写到JournalNode节点的共享目录中
2. Active NameNode和Standby NameNode从JournalNpde的edits共享目录同步到自己edits目录中
3. Standby NameNode中的StandbyCheckpointer类会定期的检查合并的条件是否成立，如果成立会合并fsimage和edits文件
4. Standby NameNode中的StandbyCheckpointer类合并完之后，将合并之后的fsimage上传到Active NameNode相应目录中
5. Active NameNode接到最新的fsimage文件之后，将旧的fsimage和edits文件清理掉
6. 通过上面的几步，fsimage和edits文件就完成了合并，由于HA机制，会使得Standby NameNode和Active NameNode都拥有最新的fsimage和edits文件（之前Hadoop 1.x的SecondaryNameNode中的fsimage和edits不是最新的）