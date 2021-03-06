---
title: "MySQL 获取随机数据的优化"
description: MySQL 使用rand() 获取随机数效率非常低，特别是表数据量很大的情况下。如不进行优化，将导致代码执行速度下降。
date: 2021-05-29T10:55:45+08:00
image: 
math: 
license: 
hidden: false
comments: true
categories: MySQL
tags: [MySQL 优化]
image: "head.png"
---
## order by rand()存在的问题

想要从数据表中随机获取数据，MySQL给我们提供了rand()函数。但是，在使用的时候会发现，order by rand() 的查询效率非常低，我们可以验证一下。

测试的表中总共有238万行数据

```shell
mysql> select count(*) from 28hse_browse_all_back_up;
+----------+
| count(*) |
+----------+
|  2383872 |
+----------+
1 row in set (0.00 sec)
```

随机获取一行数据的花费

```shell
mysql> SELECT id,mobile,district  FROM 28hse_browse_all_back_up ORDER BY RAND() LIMIT 1;
+--------+---------------+----------+
| id     | mobile        | district |
+--------+---------------+----------+
| 915371 | +852-68964153 | 大埔     |
+--------+---------------+----------+
1 row in set (6.04 sec)

```

获取一条数据用了6.04秒，可谓非常之慢。

### 为什么会这么慢
为什么花费这么久，我们看一下执行计划，分析一下

```shell
mysql> desc SELECT id,mobile,district  FROM 28hse_browse_all_back_up ORDER BY RAND() LIMIT 1;
+----+-------------+--------------------------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
| id | select_type | table                    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra                           |
+----+-------------+--------------------------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
|  1 | SIMPLE      | 28hse_browse_all_back_up | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2273880 |   100.00 | Using temporary; Using filesort |
+----+-------------+--------------------------+------------+------+---------------+------+---------+------+---------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)

```

rows字段显示预计需要扫描220万行数据，差不多就是全表扫描了一次。
Extra字段显示Using temporary,表示的是需要使用临时表；Using filesort表示的是需要执行排序操作。因此这个Extra的意思就是，需要临时表，并且需要在临时表上排序。

### 执行流程
这条语句的执行流程是这样的：

1. 创建临时表，优先会使用内存临时表，临时表的大小超过了tmp_table_size的配置就会转成磁盘临时表。
1. 全表扫描，并用rand()为每一行生成对应的随机数，将数据放入到临时表中。
1. 初始化sort_buffer,从内存表中读取所有行，取出随机数和rowid,放入sort_buffer中
1. 在sort_buffer中根据随机数执行排序。
1. 排序完成后，取出排在第一个的rowid,根据rowid到临时表中获取数据返回结果。

由于需要全表扫描，并且涉及到临时表和文件排序，所以在数据量比较大的表中，直接使用order by rand()获取随机数据是不明智的，查询会非常慢。

## 优化思路

### 根据主键取随机
这个方案的思路就是，利用数据表中主键的最大值和最小值，生成一个随机数，然后查询表中主键大于等于这个数的第一行记录。

步骤如下：

1. 取这个表主键id最大的值M和最小值N；
1. 用随机函数生成M和N的随机数X;
1. 取id不小于X的第一行

```php
//测试表最小的id是1，最大的id是2633479
$min = 1;
$max = 2633479;
$number = mt_rand($min,$max);
$sql = "select id,mobile,district from 28hse_browse_all_back_up where id >= {$number} limit 1";
```

执行一下，随机数为1606750

```shell
mysql> select id,mobile,district from 28hse_browse_all_back_up where id >= 1606750 limit 1;
+---------+------------------------------+-----------------+
| id      | mobile                       | district        |
+---------+------------------------------+-----------------+
| 1606750 | +852-94380883##+852-60152968 | 日出康城區      |
+---------+------------------------------+-----------------+
1 row in set (0.00 sec)

```
在MySQL中的执行时间已经是毫秒级的了，几乎可以忽略不计。

查询非常快，但是会有一个问题，如果id中间有空洞的话，例如id分布是1,2,2000000,2000001...，那么大概率会随机到id值比较大的那一行。
### 根据行数取随机
这个方案的思路就是，根据数据总行数取随机，在数据表中获得随机行

步骤如下：

1. 取得整个表的行数C
1. 用随机函数生成1和C的随机数X;
1. 使用limit X,1取得目标行

```php
//测试表数据总行数为2633479
$count = 2633479;
$number = mt_rand(0,$count);
$sql = "select id,mobile,district from 28hse_browse_all_back_up  limit {$number},1";
```

执行一下，随机数为524982

```php
mysql> select id,mobile,district from 28hse_browse_all_back_up limit 524982,1;
+--------+---------------+----------+
| id     | mobile        | district |
+--------+---------------+----------+
| 685517 | +852-96510180 | 奧運     |
+--------+---------------+----------+
1 row in set (0.44 sec)

```

我们发现第二种方案的花费要比第一种方案的花费高了很多，这是因为MySQL处理limit X,1的做法就是按顺序一个一个的读出来，丢掉前Y个，然后将下一个的记录作为返回结果，因此需要扫描Y+1行，执行的代价比方案1高很多，而且随机出来行数越靠后，查询时间就越长。
但是相比方案1，主键存在空洞的情况也不会影响方案2的"随机性"。

## 总结

不管是根据主键还是根据行数取随机，核心思想都是将生成随机的步骤抽离出来，减少MySQL的计算量。根据行数随机得到的数据"随机性"更高，但根据主键随机查询的数据会更快。具体选择什么方案，还是要根据表大小，查询速度和业务需要进行选择。
## 参考

1. 极客时间《MySQL实战45讲》- 林晓斌
