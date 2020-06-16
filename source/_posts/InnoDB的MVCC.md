---
title: 'InnoDB的MVCC（Multi Versioning Concurrency Control）'
date: 2020-05-05 14:25:29
tags: MySQL
declare: true
---
###  定义
InnoDB is a multi-versioned storage engine: it keeps information about old versions of changed rows, to support transactional features such as concurrency and rollback.This information is stored in the tablespace in a data structure called a rollback segment（InndDB是多版本的存储引擎，它保留已更改行旧版本信息，以支持事务的并发和回滚，这些信息保存在被称为回滚段的表空间数据结构中）

###  实现原理
+ InnoDB会在所有行上加上三个属性：
DB_TRX_ID：6字节，记录每一行最近一次修改它的事务ID。
DB_ROLL_PTR：7字节，记录指向回滚段UNDO LOG的指针。
DB_ROW_ID：6字节，单调递增的行ID。

+ undo log  

参考
https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html

https://dev.mysql.com/doc/refman/5.6/en/innodb-undo-logs.html

https://blog.csdn.net/SnailMann/article/details/94724197