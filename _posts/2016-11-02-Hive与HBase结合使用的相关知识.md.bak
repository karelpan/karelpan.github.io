---
layout: post
title: Hive与HBase结合使用的相关知识
uuid: 30d758a0-a0da-11e6-bfc2-005056359bbd
---

{{ page.title }}
================

<p class="meta">02 Nov 2016 - Su Zhou</p>

# Hive 和 HBase 在 hadoop 生态圈内， 一般总是结合着使用
相关的文章有
 * [让 Hive Scripts 可以使用 HBase](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_hive_hbase.html#topic_18_10)
 * [使用 Hive 在安全受限的 HBase Server 上运行 query 操作](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_hive_query_secure_hbase.html#topic_9_3)
 * [Hive 使用引导](http://www.cloudera.com/documentation/enterprise/latest/topics/hive.html#hbase)
 * [HBase 使用引导](http://www.cloudera.com/documentation/enterprise/latest/topics/hbase.html)

## Q1. 如何在在Hive和HBase间建起桥梁
A: 我们可以使用如下的语句，在Hue管理控制界面，进行Hive表创建; 

```sql
CREATE TABLE APP_Per_bak_statistics_Hbase (
rowkey string,
date String,
flow_type string,
NowPrice string,
LastPrice string
) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,info:date,info:flow_type,info:NowPrice,info:LastPrice")
TBLPROPERTIES ("hbase.table.name" = "APP_Per_bak_statistics");
```

这样的创建方式， 会在使用Hive 对Hive表 APP_Per_bak_statistics_Hbase 进行 insert 操作的时候， HBase 的表 APP_Per_bak_statistics 中也会同步的插入同样的数据 

## Q2. TODO