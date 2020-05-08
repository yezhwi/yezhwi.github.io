---
layout:     post
title:      1分钟系列-Elasticsearch 简介与单机版安装
subtitle:   
date:       2020-05-09
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - SpringCloud
    - ELK
---

### Elasticsearch 简介

Elasticsearch 是一个分布式的开源搜索和分析引擎，适用于所有类型的数据，包括文本、数字、地理空间、结构化和非结构化数据。Elasticsearch 在 Apache Lucene 的基础上开发而成，以其简单的 RESTful API、分布式特性、速度和可扩展性而闻名，是 Elastic Stack 的核心组件。

* Elasticsearch 使用的是一种名为倒排索引的数据结构，这一结构的设计可以允许十分快速地进行全文本搜索。倒排索引会列出在所有文档中出现的每个特有词汇，并且可以找到包含每个词汇的全部文档。

* Elasticsearch 索引指相互关联的文档集合。以 JSON 文档的形式存储数据。每个文档都会在一组键（字段或属性的名称）和它们对应的值（字符串、数字、布尔值、日期、数值组、地理位置或其他类型的数据）之间建立联系。

* 在索引过程中，Elasticsearch 会存储文档并构建倒排索引，这样用户便可以近实时地对文档数据进行搜索。

### Elasticsearch 在 Elastic Stack 中的作用

通过 Beats 将原始数据会从多个来源（包括日志、系统指标和网络应用程序）输入到 Elasticsearch 中。这些数据在 Elasticsearch 中索引完成之后，用户便可针对他们的数据运行复杂的查询，并使用聚合来检索自身数据的复杂汇总。在 Kibana 中，用户可以基于自己的数据创建强大的可视化，分享仪表板，并对 Elastic Stack 进行管理。

> 数据采集指在 Elasticsearch 中进行索引之前解析、标准化并充实这些原始数据的过程，一般通过 Logstash 完成。

### Elasticsearch 优点

**Elasticsearch 很快**。 由于 Elasticsearch 是在 Lucene 基础上构建而成的，所以在全文本搜索方面表现十分出色。Elasticsearch 同时还是一个近实时的搜索平台，这意味着从文档索引操作到文档变为可搜索状态之间的延时很短，一般只有一秒。因此，Elasticsearch 非常适用于对时间有严苛要求的用例，例如安全分析和基础设施监测。

**Elasticsearch 具有分布式的本质特征**。 Elasticsearch 中存储的文档分布在不同的容器中，这些容器称为分片，可以进行复制以提供数据冗余副本，以防发生硬件故障。Elasticsearch 的分布式特性使得它可以扩展至数百台（甚至数千台）服务器，并处理 PB 量级的数据。

**Elasticsearch 支持多种编程语言**，官方客户端包括：Java、Go、PHP、Python、Node.js 等

**Elasticsearch 提供强大且全面的 RESTful API 集合**，这些 API 可用来执行各种任务，例如检查集群的运行状况、针对索引执行 CRUD（创建、读取、更新、删除）和搜索操作，以及执行诸如筛选和聚合等高级搜索操作。

**Elasticsearch 支持 34 种文本语言**，从阿拉伯语到泰语，十分全面，同时还针对每种语言提供了分词工具。完整列表详见 [Elasticsearch Language Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html) 文档。通过定制插件还可以新增对其他语言的支持。

### 下载并安装

#### 安装依赖

Elasticsearch 是基于 Lucene 的，而 Lucene 又是基于 Java 的。所以第一步我们就需要在每台主机上安装 Java。安装好了之后可以检查下 Java 的版本

```
java -version
```

#### 根据自己的操作系统选择相应版本进行安装

最新版本下载地址：https://www.elastic.co/cn/downloads/elasticsearch，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gek9kv61dvj31im0u0q4x.jpg)

如果想使用其他版本，请使用这个地址进行下载：https://www.elastic.co/cn/downloads/past-releases#elasticsearch，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gek9ijv66oj31iz0u0q4o.jpg)

> 目前我电脑上已经下载 elasticsearch-6.6.1 版本，所以下面就用这个版本进行演示 

#### 创建用户

由于 ElasticSearch 默认是不支持 root 账号权限启动，所以第一步要先创建启动账号。

```
# 创建一个 ElasticSearch 的运行组 es
groupadd elasticsearch
# 创建用户并设置密码
useradd elasticsearch -g elasticsearch -p 123456
# 给解压出的 ElasticSearch 包授权
chown -R elasticsearch:elasticsearch elasticsearch-6.6.1/
```

#### 运行

```
# 执行命令 
bin/elasticsearch (or bin\elasticsearch.bat on Windows)
```

#### 测试

```
# 执行命令 
curl http://localhost:9200/ 
```
出现下图说明单机版本已经完装成功

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gekag8hdyoj30oy0g2q3d.jpg)


### 下一步计划

Elasticsearch 集群安装与常见问题


> 参考资料：https://www.elastic.co/cn

