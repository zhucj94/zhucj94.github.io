---
title: InnoDB之锁（1）自增锁（AUTO-INC Locks）
date: 2020-05-05 21:27:21
tags: MySQL
declare: true
---
自增锁是一种特殊的表级别锁（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。

### insert分类
+ Simple inserts
Statements for which the number of rows to be inserted can be determined in advance (when the statement is initially processed). This includes single-row and multiple-row INSERT and REPLACE statements that do not have a nested subquery, but not INSERT ... ON DUPLICATE KEY UPDATE.

+ Bulk inserts
Statements for which the number of rows to be inserted (and the number of required auto-increment values) is not known in advance.This includes INSERT ... SELECT, REPLACE ... SELECT, and LOAD DATA statements, but not plain INSERT. InnoDB will assign new values for the AUTO_INCREMENT column one at a time as each row is processed.

+ Mixed-mode inserts
These are “simple insert” statements that specify the auto-increment value for some (but not all) of the new rows. An example follows, where c1 is an AUTO_INCREMENT column of table t1:

INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');

Another type of “mixed-mode insert” is INSERT ... ON DUPLICATE KEY UPDATE, which in the worst case is in effect an INSERT followed by a UPDATE, where the allocated value for the AUTO_INCREMENT column may or may not be used during the update phase.

### 锁模式（参数innodb_autoinc_lock_mode）
+ innodb_autoinc_lock_mode = 0 （traditional）
    + 此模式与innodb_autoinc_lock_mode存在前的行为相同，对于所有的‘INSERT-like’语句，都会使用特殊的AUTO-INC table-level lock，保持到语句结束。这确保了任何给定语句分配的自动增量值是连续的。

+ innodb_autoinc_lock_mode = 1 （consecutive）
    + 默认的模式
    + 发生Simple inserts时，在分配AUTO_INCREMENT期间，会使用轻量级的锁，获取到AUTO_INCREMENT后就释放锁，如果其他事务持有AUTO-INC table-level lock，获取轻量级锁也会等待。
    + 发生Bulk inserts时，会使用特殊的AUTO-INC table-level lock，保持到语句结束。
    + 发生Mixed-mode inserts时，一些行指定了AUTO_INCREMENT，将分配比要插入的行数更多的AUTO_INCREMENT，所有AUTO_INCREMENT是连续生成的，多余的丢失。

+ innodb_autoinc_lock_mode = 2 （interleaved）
    + 对于所有的‘INSERT-like’语句不加锁
    + 对于Bulk inserts，AUTO_INCREMENT可能不连续
    + binlog为SBR模式时，数据复制或恢复不安全





