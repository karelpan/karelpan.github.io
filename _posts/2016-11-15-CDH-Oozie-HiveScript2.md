---
layout: post
title: CDH-Oozie-HiveScript2 
uuid: e348fc7e-ab29-11e6-a030-005056359bbd
---

{{ page.title }}
================

# HiveScipt2 在CDH环境下的 Oozie WorkFlow 配置

## Script 配置

比如，我们使用如下的 Hive 语句作为一个任务， 做到事情为，把某个Hive表中的某天的数据查出来，并且存入临时表
临时表的格式以 \t 分割，并存为纯文本TEXTFILE

```sql
DROP TABLE IF EXISTS ${hivevar:tmptablename};
CREATE TABLE ${hivevar:tmptablename} ROW FORMAT DELIMITED FIELDS TERMINATED BY'\t'
STORED AS TEXTFILE
AS SELECT ${hivevar:fields} FROM ${hivevar:tablename}
WHERE DATE = ${hivevar:field_date}
```

其中： hivevar 是用来传递参数的对象, 在 sql 中的 ${hivevar:tmptablename} 
      将被 Oozie 中此HiveScript任务配置的对应变量 tmptablename=tmp_table 替代

假设我们此HiveScript的Oozie任务参数是

```
tmptablename=tmp_table_name
tablename=table_name
fields=*
field_date=from_unixtime(unix_timestamp()-24*3600,'yyyy-MM')
```

则，实际执行时，解析后的 Sql，会被替换成下面这样

```sql
DROP TABLE IF EXISTS tmp_table_name;
CREATE TABLE tmp_table_name ROW FORMAT DELIMITED FIELDS TERMINATED BY'\t'
STORED AS TEXTFILE
AS SELECT * FROM table_name
WHERE DATE = from_unixtime(unix_timestamp()-24*3600,'yyyy-MM')
```

## 单个 HiveScript2 任务的环境配置
需要注意：如果你使用的 Hive 的库不是默认的 default， 假设是 xxxxx， 那么你需要在点击任务配置Button，修改HiveServer的配置
比如修改为

```
jdbc:hive2://namenode1:10000/xxxxx
```
