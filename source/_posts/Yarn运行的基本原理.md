---
title: Yarn 运行的基本原理
date: 2020-12-02 12:22:00
tags:
  - 大数据
  - Yarn
  - Hadoop
categories: bigdata
---



Yarn 是分布式集群资源调度框架

YARN 是 hadoop 2.0 引入的集群资源管理系统。用户可以将各种服务框架部署在YARN上，由YARN进行统一的管理和资源分配。



Yarn 包括两个部分：一个是资源管理器（Resource Manger），一个是节点管理器（Node Manager）。

- ResourceManger：负责整个集群的资源调度管理，通常部署在独立的服务器上；ResourceManager 负责给用户提交的所有应用程序分配资源，它根据应用程序优先级、队列容量、ACLs、数据位置等信息，作出决策，然后以共享的、安全的、多租户的方式指定分配策略，调度集群资源。

- NodeManager：是YARN集群中每个具体节点的管理者。主要负责该节点内所有容器的生命周期的管理，监视资源和跟踪节点健康，在集群的每一台计算服务器上都会启动，基本上跟 HDFS 的 DataNode 进程一起出现。

- - 启动时向ResourceManager注册并定时发送心跳信息，等待ResourceManger的指令；
  - 维护Container的生命周期，监控Container的资源使用情况；
  - 管理任务运行时的相关依赖，根据ApplicationMaster 的需要，在启动 Container 之前将需要的程序及依赖拷贝到本地。

- ApplicationMaster：在用户提交一个应用程序时，YARN会启动一个轻量级的进程ApplicationMaster。ApplicationMaster负责协调来自ResourceManager的资源，并通过NodeManager监视容器内资源的使用情况，同时还负责任务的监控与容错。

- - 根据应用的运行状态来决定动态计算资源需求；
  - 向 ResourceManager申请资源，监控申请的资源的使用情况；
  - 跟踪任务状态和进度，报告资源的使用情况和应用的进度信息；
  - 负责任务的容错。

- Contain：是YARN中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。当 AM 向RM申请资源时，RM为AM返回的资源使用Container表示的。YARN会为每一个任务分配一个Container，该任务只能使用该Container中描述的资源。AM可在Container内运行任何类型的任务。例如，MapReduce ApplicationMaster 请求一个容器来启动 Map 或 Reduce 任务，而 Giraph ApplicationMaster 请求一个容器来运行 Giraph 任务。



## Yarn 工作流程

1. Client提交作业到YARN；
2. RM 选择一个 NM，启动一个Container 并运行AM 实例；
3. AM 根据实际需要向RM请求跟多的Container资源（如果作业很小，AM会选择在其自己的JVM中运行任务）；
4. AM通过获取到的Container资源执行分布式计算。



## **YARN**工作原理详述

**1.作业提交**

Step1：Client调用job.waitForCompletion 方法，向整个集群提交 MR 作业

Step2.：新的作业ID（应用ID）由RM分配

Step3：作业的Client核实作业的输出，计算输入的split，将作业的资源（包括jar包，配置文件，split信息）拷贝给HDFS

Step4：通过调用RM的submitApplication()来提交作业



**2.作业初始化**

Step5：当RM收到submitApplication()的请求时，就将该请求发给调度器（Scheduler），调度器分配Container，然后资源管理器在该Container内启动AM，由NM监控

Step6：MR作业的AM是一个主类为MRAppMaster的Java应用，其通过创造一些bookkeeping对象来监控作业的进度，得到任务的进度和完成报告

Step7：通过分布式文件系统得到由Client计算好的输入Split，然后为每一个输入split创建一个map任务，根据mapreduce.job.reduces创建reduce任务对象



**3.任务分配**

如果作业很小，AM会选择在其自己的JVM中运行任务

Step8：如果不是小作业，AM向RM请求Container来运行所有的map和reduce任务。这些请求时通过心跳来传输的，包括每个map任务的数据位置，比如存放输入split的主机名和机架（rack），调度器利用这些信息来调度任务，尽量将任务分配给存储数据的节点，或者分配给和存放输入spit的节点相同机架的节点。



**4.任务运行**

Step9：当一个任务由RM的调度器分配给一个container后，AM通过联系NM来启动container。

Step10：任务由一个主类为YarnChild的Java应用执行，在运行任务之前首先本地化任务需要的资源，比如作业配置，jar文件，以及分布式缓存的所有文件

Step11：最后运行map或者reduce任务



YarnChild运行在一个专用的JVM中，但是YARN不支持JVM重用。



**5.进度和状态更新**

YARN 中的任务将其进度和状态 (包括 counter) 返回给AM, 客户端每秒 (通 mapreduce.client.progressmonitor.pollinterval 设置) 向AM请求进度更新, 展示给用户。



**6.作业完成**

除了向AM请求作业进度外, 客户端每 5 分钟都会通过调用 waitForCompletion() 来检查作业是否完成，时间间隔可以通过 mapreduce.client.completion.pollinterval 来设置。作业完成之后, AM和 container 会清理工作状态， OutputCommiter 的作业清理方法也会被调用。作业的信息会被作业历史服务器存储以备之后用户核查。