---
title: mysql分区备份及转移操作
date: 2017-09-13 15:01:17
tags: mysql
---

昨天写的脚本不小心把电量和功率数据弄反了，原始csv文件也被我给删了，悲剧，于是只能通过分区数据备份来转移数据了，意料之外的速度很快，把操作过程记录下。

- 快速创建被与原表结构一致的备份表并删除所有分区

  ```sql
  CREATE TABLE dianliangbak LIKE ckr_dau_dianliang;
  ALTER TABLE dianliangbak REMOVE PARTITIONING;
  ```

- 将分区表数据转移至被分表```ALTER TABLE ckr_dau_dianliang EXCHANGE PARTITION p20170913 WITH TABLE dianliangbak;```  这一步操作之后，该分区数据被清空。

- 检查分区数据 ```select count(1) from ckr_dau_dianliang partition(p20170913);```

- 针对功率表重复以上操作

  ```sql
  CREATE TABLE gonglvbak LIKE ckr_dau_gonglv;

  ALTER TABLE gonglvbak REMOVE PARTITIONING;

  ALTER TABLE ckr_dau_gonglv EXCHANGE PARTITION p20170913 WITH TABLE gonglvbak;

  select count(1) from ckr_dau_gonglv partition(p20170913);
  ```

- 恢复数据

  ```sql
  insert into ckr_dau_dianliang (devId, time, value) select devId, time, value from gonglvbak;

  insert into ckr_dau_gonglv (devId, time, value) select devId, time, value from dianliangbak;

  truncate table dianliangbak;
  truncate table gonglvbak;
  ```

  ​

  可以用来作为未来分区数据清理并备份的方法之一。