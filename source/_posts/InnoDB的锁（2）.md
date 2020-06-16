---
title: InnoDB的锁（2）
date: 2020-05-12 21:18:18
tags: MySQL
declare: true
---

### 共享/排他锁（Shared and Exclusice Locks）
**行级锁**
+ 定义
    + **事务**拿到某一行记录的共享S锁，才可以读取（当前读）这一行；
    + **事务**拿到某一行记录的排他X锁，才可以修改或者删除这一行；

+ 加锁方式
    + 共享锁（S）：SELECT ... LOCK IN SHARE MODE
    + 排他锁（X）：SELECT ... FOR UPDATE （**update/delete/insert自动加排他锁**）

+ 互斥关系

 - | S | X 
:-: | :-: | :-: 
S | 兼容 | 互斥
X | 互斥 | 互斥

### 意向锁（Intention Locks）
**不与行级锁冲突表级锁**

+ 分类
    + 意向共享锁(intention shared lock, IS)，它预示着，事务有意向对表中的某些行加共享S锁。
    + 意向排它锁(intention exclusive lock, IX)，它预示着，事务有意向对表中的某些行加排它X锁。

+ intention locking protocol
    + 事务要获得某些**行**的S锁，必须先获得表的IS锁。
    + 事务要获得某些**行**的X锁，必须先获得表的IX锁。

+ 互斥关系（**下表的S/X锁是表级的!!!**）

- | X | IX | S | IS 
:-: | :-: | :-: | :-: | :-: 
X | 互斥 | 互斥 | 互斥 | 互斥
IX | 互斥 | 兼容 | 互斥 | 兼容
S | 互斥 | 互斥 | 兼容 | 兼容
IS | 互斥 | 兼容 | 兼容 | 兼容

+ 判断冲突时的作用
无意向锁 ： 
    + 判断表是否被其他事务加表锁。
    + 判断表中的每一行是否被其他事务加行锁。
有意向锁 ：
    + 判断表是否被其他事务加表锁。
    + 判断表是否被其他事务加意向锁。



参考
https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html#innodb-auto-inc-locks