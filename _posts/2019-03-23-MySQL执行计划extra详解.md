---
layout:     post
title:      MySQL执行计划 Extrav中的vusing index 和 using where using index 的区别
subtitle:   
date:       2019-03-23
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - MySQL
---

> 原文地址：http://www.cnblogs.com/wy123/p/7366486.html 

### 背景

MySQL 执行计划中的 Extra 列中表明了执行计划的每一步中的实现细节，其中跟索引有关的 Using index 在不同的情况下会出现 Using index， Using where Using index，Using index condition 等，那么 Using index 和 Using where;Using index 有什么区别？网上搜了一大把文章，说实在话也没怎么弄懂，于是就自己动手试试。


本文仅从最简单的单表去测试 Using index 和 Using where Using index 以及简单测试 Using index condition 的情况的出现时机。
**执行计划的生成与表结构，表数据量，索引结构，统计信息等等上下文等多种环境有关，无法一概而论，复杂情况另论**。
 

### 测试环境搭建

测试表以及测试数据搭建，类似于订单表和订单明细表，暂时先用订单表做测试

#### 测试表结构

```
create table test_order
(
    id int auto_increment primary key,
    user_id int,
    order_id int,
    order_status tinyint,
    create_date datetime
);

create table test_orderdetail
(
    id int auto_increment primary key,
    order_id int,
    product_name varchar(100),
    cnt int,
    create_date datetime
);

create index idx_userid_order_id_createdate on test_order(user_id,order_id,create_date);

create index idx_orderid_productname on test_orderdetail(order_id,product_name);

```

#### 测试数据（50W）

```
CREATE DEFINER=`root`@`%` PROCEDURE `test_insertdata`(IN `loopcount` INT)
    LANGUAGE SQL
    NOT DETERMINISTIC
    CONTAINS SQL
    SQL SECURITY DEFINER
    COMMENT ''
BEGIN
    declare v_uuid  varchar(50);
    while loopcount>0 do
        set v_uuid = uuid();
        insert into test_order (user_id,order_id,order_status,create_date) values (rand()*1000,id,rand()*10,DATE_ADD(NOW(), INTERVAL - RAND()*20000 HOUR));
        insert into test_orderdetail(order_id,product_name,cnt,create_date) values (rand()*100000,v_uuid,rand()*10,DATE_ADD(NOW(), INTERVAL - RAND()*20000 HOUR));
        set loopcount = loopcount -1;
    end while;
END
```

### 实验步骤

#### Using index VS Using where Using index

首先，在"订单表"上，这里是一个多列复合索引

```
create index idx_userid_order_id_createdate on test_order(user_id,order_id,create_date);
```


#### Using index 

* **查询的列被索引覆盖**，并且 where 筛选条件是**索引的是前导列**，Extra 中为 Using index

![Using index](https://ws3.sinaimg.cn/large/006tKfTcly1g1bfgvzxqxj30uv04sglk.jpg)

#### Using where Using index

* **查询的列被索引覆盖**，并且 where **筛选条件是索引列之一但是不是索引的不是前导列**，Extra 中为 Using where;Using index，意味着无法直接通过索引查找来查询到符合条件的数据

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bhprbnd6j30u604zq2w.jpg)

* **查询的列被索引覆盖**，并且 where 筛选条件是**索引列前导列的一个范围**，同样意味着无法直接通过索引查找查询到符合条件的数据

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1bhq5zv8fj30vt04pt8o.jpg) 　　

#### NULL（既没有 Using index，也没有 Using where Using index，也没有 Using where）

* **查询的列未被索引覆盖**，并且 where 筛选条件是索引的前导列，意味着用到了索引，但是部分字段未被索引覆盖，必须通过“**回表**”来实现，不是纯粹地用到了索引，也不是完全没用到索引，Extra 中为NULL(没有信息)

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1bht44949j30to04q3yg.jpg)

#### Using where

* 查询的列未被索引覆盖，where 筛选条件非索引的前导列，Extra 中为 Using where

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1bhtm0alij30qf04u747.jpg)

* 查询的列未被索引覆盖，where 筛选条件非索引列，Extra 中为 Using where

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1bhtvc4prj30vt04pt8o.jpg)

以上说明：Using where 意味着通过索引或者表扫描的方式进程 where 条件的过滤，反过来说，也就是没有可用的索引查找，当然这里也要考虑**索引扫描+回表与表扫描的代价**。这里的 type 都是 all，说明 MySQL 认为全表扫描是一种比较低的代价。

#### Using index condition

* 查询的列不全在索引中，where 条件中是一个前导列的范围

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1bhu9sbb6j30zv05jgll.jpg)

* 查询列不完全被索引覆盖，查询条件完全可以使用到索引（进行索引查找）

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1bhuid8f3j30vt04pt8o.jpg)　　

参考：MySQL · 特性分析 · Index Condition Pushdown (ICP)
using index conditoin 意味着查询列的某一部分无法直接使用索引

上述 case1 中：
如果禁用ICP（set optimizer_switch='index_condition_pushdown=off'），执行计划是 Using where，意味着全表扫描，如果启用ICP，执行计划为 Using index Condition，意味着在筛选的过程中实现过滤

上述 case2 中
第二个查询条件无法直接使用索引，隐含了一个查找+筛选的过程。
两个case的共同点就是无法直接使用索引。

### 结论：

* Extra 中的为 Using index 的情况

where 筛选列是索引的前导列 &&查询列被索引覆盖 && where 筛选条件是一个基于索引前导列的查询，意味着通过索引超找就能直接找到符合条件的数据，并且无须回表

* Extra 中的为空的情况

查询列存在未被索引覆盖 && where 筛选列是索引的前导列，意味着通过索引超找并且通过“回表”来找到未被索引覆盖的字段，

* Extra 中的为 Using where Using index： 

出现 Using where Using index 意味着是通过索引扫描（或者表扫描）来实现 SQL 语句执行的，即便是索引前导列的索引范围查找也有一点范围扫描的动作，不管是前非索引前导列引起的，还是非索引列查询引起的。

尚未解决的问题：

查询1

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bhw58d38j30tn03v0sm.jpg)

查询2

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bhwdgfcwj30tk030a9y.jpg)

查询3（逻辑上等价于查询1+查询2），执行计划发生了很大的变化。

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1bhwwytkgj30vn04jwef.jpg)

### 总结：

MySQL 执行计划中的 Extra 中信息非常多，不仅仅包括 Using index，Using where Using index，Using index condition，Using where，尤其是在多表连接的时候，这一点在相对 MSSQL 来说，不够直观或者结构化。
MSSQL 中是通过区分索引查找（index seek），索引扫描（index scan），表扫描（table scan）来实现具体的查询的，这图形化的执行计划在不同的场景下是非常直观的，要想完全弄懂 MySQL 的这个执行计划，可能要更多地在实践中摸索。










