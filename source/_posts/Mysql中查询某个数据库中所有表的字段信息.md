---
title: Mysql中查询某个数据库中所有表的字段信息
date: 2017-08-09 17:55:55 
tags: [Mysql,数据库]
categories: 技术
---
## 前言 ##
有时候，需要在数据库中查询一些字段的具体信息，而这些字段又存在于不同的表中，那么我们如何来查询呢？

在每一个数据库链接的information_schema数据库中，存在这样一张表——COLUMNS，它记录了这个数据库中所有表的字段信息。
## 查询某个特定类型的字段信息##
如下：查询字段类型为decimal的字段信息
Sql语句：

```
SELECT
	TABLE_NAME,
	column_name,
	DATA_TYPE,
	column_comment
FROM
	information_schema. COLUMNS
WHERE
	TABLE_SCHEMA = 'evshare'
AND DATA_TYPE = 'decimal';
```

其中

 - TABLE_SCHEMA 为数据库的名称（所属的数据库）
 - TABLE_NAME 为表的名称
 - DATA_TYPE 为字段的数据类型
 - column_name  为字段名
 - column_comment 为字段注释
在Where的条件语句中，可以加入限制条件。
结果如下：

![1](http://osuskkx7k.bkt.clouddn.com/clipboard1.png) 

## 查询注释乱码的字段信息##

如果需要查询数据库中所有乱码的字段信息，那么可以对以上的Sql稍稍改进：

```
SELECT
	TABLE_NAME,
	column_name,
	DATA_TYPE,
	column_comment
FROM
	information_schema. COLUMNS
WHERE
	TABLE_SCHEMA = 'evshare'
AND column_comment LIKE '%?%';
```

结果如下：可以看到这个evshare数据库中，所有表的乱码字段都已显示

![1](http://osuskkx7k.bkt.clouddn.com/clipboard3.png) 

## 总结 ##

<font size="4">以上，就是在Mysql中如何查询某个数据库中所有表的字段信息的过程。