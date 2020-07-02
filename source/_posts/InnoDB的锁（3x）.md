---
title: InnoDB的锁（3x）
date: 2020-05-16 16:15:16
tags: MySQL
declare: true
---
### 行锁策略
1. 加锁的基本单位是临键锁。（记录锁+间隙锁，左开右闭区间）
2. 查找到数据才会加记录锁。
3. 唯一索引上的等值查询（命中），间隙锁退化为记录锁。
4. 辅助索引上的等值查询（命中），会在右侧相邻间隙加上间隙锁。
5. 唯一索引上的范围查询（等值条件命中，例如<=的等于命中）,会在右侧相邻间隙加上临键锁。

### 例子
// todo


参考
https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html#innodb-auto-inc-locks

https://blog.guitar-coder.cn/MySql%E9%94%81-%E9%97%B4%E9%9A%99%E9%94%81%E5%92%8C%E4%B8%B4%E9%94%AE%E9%94%81.html

https://www.cnblogs.com/zhoujinyi/p/3435982.html

https://time.geekbang.org/column/article/75659

