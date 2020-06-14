---
title: InnoDB之缓冲池(buffer pool)
date: 2020-06-01 22:20:10
tags: MySQL
declare: true
---
MySQL作为一个存储系统，具有 ***buffer pool*** 机制，以避免每次查询都进行磁盘IO。

### 缓存什么
缓存表数据与索引数据

### 预读（read-ahead）
+ page是InnoDB存储引擎的最小磁盘单位，一次至少读取一个page，extent是由连续的page组成的空间。
+ 如果一个extent中被顺序读取的page超过或者等于innodb_read_ahead_threshold时，InnoDB将会异步的将下一个extent读取到buffer pool中（以linear read-ahead概述，random read-ahead在5.5后被废弃）。
+ 预读的依据是**局部性原理** : CPU访问存储器时，无论是存取指令还是存取数据，所访问的存储单元都趋于聚集在一个较小的连续区域中。

### 淘汰算法
采用改进版的LRU算法，传统的LRU算法会遇到两个问题
+ 预读失败 
由于预读，提前将page放入缓冲池，但最终没有从页中读取数据。
改进：
    + 将LRU分为新生代（new sublist），老生代（old sublist）。
    + 新老生代首尾相连。 
    + 新页加入缓冲池时，只加入老生代头部，如果数据被真正读取到，才会加入新生代头部，如果没有被读取，则会比新生代里的数据更早被淘汰

+ 缓冲池污染
当执行某个SQL时，要批量扫描大量数据（例如未命中索引，而全表扫描），导致大量热数据被淘汰。
改进：
    * 插入老生代头部的页，即使立刻被访问，也不会放入新生代头部。
    * 只有满足 ***被访问*** 且 ***在老生代停留时间大于T***，才会被放入新生代头部。

![avatar](/images/InnoDB/newlru.png)

### 相关参数
+ innodb_buffer_pool_size
缓冲池的大小
+ innodb_old_blocks_pct
老生代占整个LRU链长度的比例，默认是37

