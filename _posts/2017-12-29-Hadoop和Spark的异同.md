---
layout:     post
title:      Hadoop和Spark的异同
subtitle:   
date:       2017-12-29
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - Yarn
    - Hadoop
    - Spark
---

### 解决问题的层面不一样

Hadoop和Spark两者都是大数据框架，但是各自存在的目的不尽相同。

* Hadoop实质上是解决大数据大到无法在一台计算机上进行存储、无法在要求的时间内进行处理的问题，是一个分布式数据基础设施。
* HDFS，它将巨大的数据集分派到一个由普通计算机组成的集群中的多个节点进行存储，通过将块保存到多个副本上，提供高可靠的文件存储。
* MapReduce，通过简单的Mapper和Reducer的抽象提供一个编程模型，可以在一个由几十台上百台的机器上并发地分布式处理大量数据集，而把并发、分布式和故障恢复等细节隐藏。
* Hadoop复杂的数据处理需要分解为多个Job（包含一个Mapper和一个Reducer）组成的有向无环图。
* Spark则允许程序开发者使用有向无环图（DAG）开发复杂的多步数据管道。而且还支持跨有向无环图的内存数据共享，以便不同的作业可以共同处理同一个数据。是一个专门用来对那些分布式存储的大数据进行处理的工具，它并不会进行分布式数据的存储。
* 可将Spark看作是Hadoop MapReduce的一个替代品而不是Hadoop的替代品。

### Hadoop的局限和不足

* 一个Job只有Map和Reduce两个阶段，复杂的计算需要大量的Job完成，Job间的依赖关系由开发人员进行管理。
* 中间结果也放到HDFS文件系统中。对于迭代式数据处理性能比较差。
* Reduce Task需要等待所有的Map Task都完成后才开始计算。
* 时延高，只适用批量数据处理，对于交互式数据处理，实时数据处理的支持不够。


### 两者可合可分

* Hadoop除了提供HDFS分布式数据存储功能之外，还提供了MapReduce的数据处理功能。所以我们完全可以抛开Spark，仅使用Hadoop自身的MapReduce来完成数据的处理。
* 相反，Spark也不是非要依附在Hadoop身上才能生存。但它没有提供文件管理系统，所以，它必须和其他的分布式文件系统进行集成才能运作。我们可以选择Hadoop的HDFS，也可以选择其他的基于云的数据系统平台。但Spark默认来说还是被用在Hadoop上面的，被认为它们的结合是最好的选择。

![Spark计算引擎](https://tva4.sinaimg.cn/large/006tKfTcly1fmxtrmh67gj30fg0b4wgg.jpg)

### Spark数据处理速度秒杀MapReduce

* Spark因为处理数据的方式不一样，会比MapReduce快上很多。MapReduce是分步对数据进行处理的: “从集群中读取数据，进行一次处理，将结果写到集群，从集群中读取更新后的数据，进行下一次的处理，将结果写到集群，等等…”
* Spark会在内存中以接近“实时”的时间完成所有的数据分析：“从集群中读取数据，完成所有必须的分析处理（依赖多个算子），将结果写回集群，完成，” Spark的批处理速度比MapReduce快近10倍，内存中的数据分析速度则快近100倍。
* 如果需要处理的数据和结果需求大部分情况下是静态的，且有充足的时间等待批处理的完成，MapReduce的处理方式也是完全可以接受的。
* 但如果你需要对时实流数据进行分析，比如来自工厂的传感器收集回来的数据，又或者用户访问网站的日志信息，那么更应该使用Spark进行处理。

### 灾难恢复机制

* 两者的灾难恢复方式不同，因为Hadoop将每次处理后的数据都写入到磁盘上，所以其天生就能很有弹性的对系统错误进行处理。
* Spark的数据对象存储在分布于数据集群中的叫做弹性分布式数据集(RDD: Resilient Distributed Dataset)中。这些数据对象既可以放在内存，也可以放在磁盘，所以RDD同样也可以提供完成的灾难恢复功能。

### Spark优势

* Spark的优势不仅体现在性能提升上，Spark框架为批处理（Spark Core），交互式（Spark SQL），流式（Spark Streaming），机器学习（MLlib），图计算（GraphX）提供了一个统一的数据处理平台。
* Spark通过在数据处理过程中成本更低的Shuffle方式，将MapReduce提升到一个更高的层次。利用内存数据存储和接近实时的处理能力，Spark比其他的大数据处理技术的性能要快很多倍。
* Spark将中间结果保存在内存中而不是写入磁盘，当需要多次处理同一数据集时，这一点特别实用。
* 支持比Map和Reduce更多的函数。
* Spark的RDD是分布式大数据处理的高层次抽象的数据集合，对这个集合的任何操作都可以像函数式编程中操作内存中的集合一样直观、简便，但集合操作的实现确是在后台分解成一系列Task发送到集群上完成。

![核心架构](https://tva4.sinaimg.cn/large/006tKfTcly1fmxtrnn85qj30go07l749.jpg) 






