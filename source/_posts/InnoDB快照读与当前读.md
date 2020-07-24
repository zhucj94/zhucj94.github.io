---
title: InnoDB快照读与当前读
tags: MySQL
declare: true
abbrlink: cdee20fb
date: 2020-05-04 21:51:31
---
### 快照读（Consistent Nonlocking Reads）
A consistent read means that InnoDB uses multi-versioning to present to a query a snapshot of the database at a point in time.（快照读是InnoDB使用多版本来查询某个时间点的快照）

Consistent read is the default mode in which InnoDB processes SELECT statements in READ COMMITTED and REPEATABLE READ isolation levels. A consistent read does not set any locks on the tables it accesses, and therefore other sessions are free to modify those tables at the same time a consistent read is being performed on the table.（快照读是RR和RC下处理SELECT的默认模式，快照读不在访问的表上设置任何锁，因此其他会话可以自由的修改这些表。）

### 当前读（Locking Reads）
If you query data and then insert or update related data within the same transaction, the regular SELECT statement does not give enough protection. Other transactions can update or delete the same rows you just queried. （在同一事务中查询数据，然后更新或修改相关数据，快照读不能提供足够保护，其他事务可以修改或删除刚查询的行。）

InnoDB提供了 **SELECT ... LOCK IN SHARE MODE（共享锁）**和 **SELECT ... FOR UPDATE（排他锁）**提供额外的安全。

参考：
https://dev.mysql.com/doc/refman/5.6/en/innodb-consistent-read.html

https://dev.mysql.com/doc/refman/5.6/en/innodb-locking-reads.html