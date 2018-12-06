---
layout:     post
title:      Spark快速入门-6-Spark算子的选择
subtitle:   
date:       2018-01-20
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - Spark
---

### 知识点

https://www.cnblogs.com/arachis/p/Spark_API.html

* 使用reduceByKey/aggregateByKey替代groupByKey
* 使用mapPartitions替代普通map
* 使用foreachPartitions替代foreach
* 使用filter之后进行coalesce操作
* 使用repartitionAndSortWithinPartitions替代repartition与sort类操作
* 使用broadcast使各task共享同一Executor的集合替代算子函数中各task传送一份集合
* 使用相同分区方式的join可以避免Shuffle
* map和flatMap选择
* cache和persist选择
* zipWithIndex和zipWithUniqueId选择


***






