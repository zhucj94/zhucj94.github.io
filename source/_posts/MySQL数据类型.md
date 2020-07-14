---
title: MySQL数据类型
date: 2020-07-14 21:56:29
tags: [MySQL , 读书笔记 , 高性能MySQL第3版]
declare: true
mathjax: true
---

### 整数类型
TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT，分别使用8，16，24，32，64位存储空间，范围从$-2^{N-1}$ 到 $2^{N-1} -1$ ，可选UNSIGNED属性，即不允许负，这样正数上线提高一倍。

**指定的宽度通常是没有意义的，例如INT(11)，它不会限制值的合法范围，只规定MySQL交互工具来显示字符个数。**

### 实数类型
DECIMAL，FLOAT，DOUBLE
DECIMAL用于存储精确的小数，5.0以后支持精确计算。通常DECIMAL比FLOAT，DOUBLE占用更多的空间。
应为需要额外的空间和计算开销，**尽量只在对小数进行精确计算时使用DECIMAL（例如钱），其余情况使用浮点型即可，在数据量比较大时，可以考虑使用BIGINT代替DECIMAL，将存储的数据乘相应位数存储。**

### 字符串类型
+ VARCHAR 与 CHAR

VARCHAR是变长的，它仅需要使用必要的空间，它需要使用1或2个字节表示字符串的长度，它节约了空间，但是UPDATE可能使行比原来更长，占用的空间更大，如果页内没有空余的空间存储，那么InnoDB需要分裂页使行放进页中。

CHAR是定长的，MySQL总是根据定义的字符串长度分配足够的空间，MySQL会删除所有末尾的空格，它适合存储所有值都接近同一长度，例如密码的MD5值。

+ BLOG 与 TEXT
