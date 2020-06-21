---
title: 'InnoDB的MVCC（Multi Versioning Concurrency Control）'
date: 2020-05-05 20:25:29
tags: MySQL
declare: true
---
**MVCC主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突**，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读。（读指快照读）
###  定义
InnoDB is a multi-versioned storage engine: it keeps information about old versions of changed rows, to support transactional features such as concurrency and rollback.This information is stored in the tablespace in a data structure called a rollback segment（InndDB是多版本的存储引擎，它保留已更改行旧版本信息，以支持事务的并发和回滚，这些信息保存在被称为回滚段的表空间数据结构中）

###  实现细节
####  隐藏字段
+ DB_TRX_ID：6字节，记录每一行最近一次修改它的事务ID。
+ DB_ROLL_PTR：7字节，记录指向回滚段UNDO LOG的指针。
+ DB_ROW_ID：6字节，单调递增的行ID。

####  undo log
+ 分类
    * insert undo log：事务在insert操作时产生的undo log，只用于事务回滚，在事务提交时立即被丢弃。
    * update undo log：事务在update或delete操作时产生的undo log；在事务回归时需要，**在快照读时也需要**，不会随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被purge线程统一清除。

+ 执行流程
    * 假设person表有如下数据
    
    name | age | DB_ROW_ID | DB_TRX_ID | DB_ROLL_PTR
    :-: | :-: | :-: | :-: | :-: 
    jerry | 24 | 1 | null | null

    * 事务1将name修改为tom
        - 事务1获取并对该行加排他锁
        - 将行数据拷贝到undo log中
        - 将name改为tom，并经DB_TRX_ID改为事务1的ID,回滚指针指向之前拷贝到undo log的行数据
        - 提交事务，释放排他锁
 ![avatar](/images/InnoDB/update-undo-log-4-mvcc1.png)

    * 事务2将age改为30
        - 事务2获取并对该行加排他锁
        - 将行数据拷贝到undo log中，发现该行记录已有undo log，将本次的拷贝作为链表的表头。
        - 将age改为30，并经DB_TRX_ID改为事务2的ID,回滚指针指向之前拷贝到undo log的行数据
        - 提交事务，释放排他锁
 ![avatar](/images/InnoDB/update-undo-log-4-mvcc2.png)

MVCC使用的是 **update undo log ，undo log实际上就是存在rollback segment中旧记录链**

#### Read View
**Read View 是用来做可见性判断的**，当事务执行快照读时，会生成数据库当前的一个快照，记录并维护系统当前活跃事务的ID。

+ 算法（简化版）
```
id_list : 用来维护Read View生成时系统活跃的事务ID（分配的事务ID是递增的）
up_limit_id：id_list中最小的事务ID
low_limit_id：ReadView生成时刻系统尚未分配的下一个事务ID，**即已出现过最大的事务ID的+1**
```
    + 比较DB_TRX_ID与up_limit_id，如果小于，则当前事务能看到DB_TRX_ID所在的行数据，否则进入下一步。
    + 比较DB_TRX_ID与low_limit_id，如果大于，则当前事务不能看到DB_TRX_ID所在的行数据，否则进入下一步。
    + 判断DB_TRX_ID是否在id_list中，如果在，则当前事务不能看到DB_TRX_ID所在的行数据，否则能看到。

+ 流程
将当前数据的DB_TRX_ID取出，与Read View维护活跃事务ID进行比较（通过上面的算法），如果如果不符合可见性，就通过DB_ROLL_PTR取undo log中的记录，将它的DB_TRX_ID与Read View维护活跃事务ID进行比较，如果还不符合，就取链表的下一条记录，直至满足可见性，那么这个DB_TRX_ID所在的旧纪录就是当前事务能看见的记录。

+ RR与RC下Read View生成时机
**RC隔离级别下，每个快照读都会生成并获取最新的Read View；而在RR隔离级别下，同一个事务中的第一个快照读才会创建Read View, 之后的快照读获取的都是同一个Read View。**
这就是RR的快照读能防止幻读，而RC不能的原因。

参考
https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html

https://dev.mysql.com/doc/refman/5.6/en/innodb-undo-logs.html

https://blog.csdn.net/SnailMann/article/details/94724197