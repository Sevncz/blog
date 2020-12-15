---
title: Spark Streaming运行的基本原理
date: 2020-12-14 13:43:00
tags:
  - 大数据
  - Spark Streaming
categories: bigdata
---

## 概述

Spark Streaming运行时由Driver和Executor相互协调完成。

Driver端创建**StreamingContent**，其中包括了**DStreamGraph**和**JobSchedule**，它又包括了**ReceiverTracker**和**JobGenerator**），Driver主要负责生成调度job、与execuor进行交互、指导工作。

