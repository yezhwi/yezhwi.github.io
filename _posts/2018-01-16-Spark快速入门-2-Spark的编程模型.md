---
layout:     post
title:      Spark快速入门-2-Spark的编程模型
subtitle:   
date:       2018-01-16
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - Spark
---

### 准备

* 2017-12-19-Hadoop2.0架构及HA集群配置（1）
* 2017-12-24-Hadoop2.0架构及HA集群配置（2）
* 2017-12-25-Spark集群搭建
* 2017-12-29-Hadoop和Spark的异同
* 2017-12-28-Spark-HelloWorld(Spark开发环境搭建)

### 相关概念

* 2018-01-15-Spark快速入门-1-Spark-on-Yarn-Job的执行流程简介

### Spark的编程模型

![](https://tva2.sinaimg.cn/large/006tKfTcly1fndnl24uarj30ki0hkwp8.jpg)

* 示例

```
import org.apache.spark.{SparkConf, SparkContext}

/**
  * @author Yezhiwei
  * @date 17/12/27
  */

object WordCount {
  def main (args: Array[String]){
  
    val conf = new SparkConf().setAppName("WordCount")
    val sc = new SparkContext(conf)

    val inputRDD = sc.textFile("README.md")
    val pythonLinesRDD = inputRDD.filter(line => line.contains("Python"))
    val wordsRDD = pythonLinesRDD.flatMap(line => line.split(" "))
    val countsRDD = wordsRDD.map(word => (word, 1)).reduceByKey(_ + _)

    countsRDD.saveAsTextFile("outputFile")
    sc.stop()
  }
}
```

* 解释

> 1.创建应用程序 `SparkContext`
> 
> 2.创建RDD，有两种方式，方式一：输入算子，即读取外部存储创建RDD，Spark与Hadoop完全兼容，所以对Hadoop所支持的文件类型或者数据库类型，Spark同样支持。方式二：从集合创建RDD
> 
> 3.Transformation 算子，这种变换并不触发提交作业，完成作业中间过程处理。也就是说从一个RDD 转换生成另一个 RDD 的转换操作不是马上执行，需要等到有 Action 操作的时候才会真正触发运算。
> 
> 4.Action 算子，这类算子会触发 SparkContext 提交 Job 作业。并将数据输出 Spark系统。
> 
> 5.保存结果
> 
> 6.关闭应用程序


***






