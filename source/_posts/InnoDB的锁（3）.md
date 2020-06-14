---
title: InnoDB的锁（3）
date: 2020-05-11 21:56:50
tags: MySQL
declare: true
---
InnoDB performs row-level locking in such a way that when it searches or scans a table index, it sets shared or exclusive locks on the index records it encounters. Thus, the row-level locks are actually index-record locks. (InnoDB执行行级锁方式是当它搜索或扫描表索引时，会在遇到的索引上加共享或排他锁，故行级锁实际上时索引记录锁)

### 记录锁（Record Locks）
+ 定义
A record lock is a lock on an **index record** （记录锁锁定的是**索引**记录）

+ 例子
```
SELECT * FROM WHERE id = 1 FOR UPDATE;
```
它会在id=1的索引记录上加锁，以阻止其他事务插入，更新，删除id=1的这一行

### 间隙锁（Gap Locks）
+ 定义
A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record.（间隙锁锁定的是**索引**记录间的间隔，或是第一条索引之前的范围，亦或是最后一条索引之后的范围）
+ 例子
```
SELECT * FROM t WHERE id BETWEEN 10 and 20 FOR UPDATE;
```
会锁定（10，20），防止其他事务插入，更新，删除id在（10，20）间的记录

**对于使用唯一索引来锁定唯一行的语句，不会产生间隙锁，而是使用记录锁。** (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur. 这句话暂时没明白)
```
SELECT * FROM WHERE id = 1 FOR UPDATE;
```
如果id未建立索引或具有非唯一索引，则该语句会产生间隙锁，锁定前面的间隙。

### 临键锁（Next-Key Locks）
+ 定义
A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.（临键锁是锁定索引的记录锁和锁定该索引之前区间的间隙锁的组合）

**默认情况下（RR），InnoDB使用临键锁进行搜索和索引扫描。**

假设索引记录为 10, 11, 13, 20
```
(-∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +∞)
```


### 插入意向锁（Insert Intention Locks）
+ 定义
An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion. （插入意向锁时插入行之前的插入操作设置的一种间隙锁）

多个事务，在同一个索引，同一个范围区间插入记录时，**如果插入的位置不冲突，不会阻塞彼此。**


MySQL 5.6 锁文档
https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html#innodb-auto-inc-locks