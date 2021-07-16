---
layout:     post
title:      Excel数据模型自动生成Hive建表语句
subtitle:   
date:       2021-07-15
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - 数据仓库
---

最近在「空白女侠」公号上看到她回答了大家会困扰的精力问题，比如为什么我（空白女侠）能同时做那么多事情，精力那么充沛？工作中遵循一个真理：

**复杂的事情简单化，简单的事情标准化，标准的事情流程化，流程化的事情工具化，工具化的事情自动化，无法自动化的事情外包化。**

数据开发过程中，有些过程是可以工具化来提高工作效率，腾出更多的时间和精力去提高自己（摸鱼～）

### 工具化

在日常数据开发过程中，会经常需要根据数据模型编写建表语句，每次写建表语句都会用几分钟的时间，而且还容易出一些低级的错误，于是打算做个 Excel 模板，把表字段、表分区、表名写在里面，通过程序自动生成建表语句。效果如下图：

![数据模型生成建表语句](https://gitee.com/yzhw/img/raw/master/img/数据模型生成建表语句.gif)

### 制作模板

主要包括表名、表中文描述、字段名、数据类型、字段说明、是不可以为空、唯一主键、表分区等写在里面（可以根据实际情况进行调整）

模板组成分为三个部分：

- 1-3行是表名、表中文描述和模板列说明
- 4-19行为创建表字段的基本信息
- 20行为分区字段

![数据模型模板](https://gitee.com/yzhw/img/raw/master/img/image-20210715002857840.png)

### 实现思路

获取指定目录下的数据模型文件，约定为以「数据模型.xls」或「数据模型.xlsx」文件名结尾的 Excel 文件

循环遍历每个模板文件，根据模板组成规范，解析文件，拼接 SQL 语句，生成建表语句文件

### 生成结果

```sql
CREATE EXTERNAL TABLE ods_cbonddescription (
    object_id string  COMMENT '对象ID',
    b_info_fullname string  COMMENT '债券名称',
    s_info_name string  COMMENT '债券简称',
    b_info_issuer string  COMMENT '发行人',
    b_issue_announcement string  COMMENT '发行公告日',
    b_issue_firstissue string  COMMENT '发行起始日',
    b_issue_lastissue string  COMMENT '发行截至日',
    b_issue_amountplan bigint  COMMENT '计划发行总量（亿元）',
    b_issue_amountact bigint  COMMENT '实际发行总量（亿元）',
    b_info_issueprice bigint  COMMENT '发行价格',
    b_info_par bigint  COMMENT '面值',
    b_info_term_year int  COMMENT '债券期限（年）',
    b_info_term_day int  COMMENT '债券期限（天）',
    b_info_paymentdate int  COMMENT '兑付日',
    b_info_paymenttype int  COMMENT '计息方式',
    s_info_exchmarket string  COMMENT '交易所'
)
COMMENT '债券基本信息' 
PARTITIONED BY(dt string) 
ROW FORMAT DELIMITED '\t' 
STORED AS ORC 
LOCATION 'hdfs://host:8020/dw/ods/ods_cbonddescription';

```

### 结语

如果大家在工作中使用了什么样的模板，可以一起交流如何形成一定的工具化的脚本来提高我们工作的效率。

最后，关注公众号，回复「608」获取模板和代码。
