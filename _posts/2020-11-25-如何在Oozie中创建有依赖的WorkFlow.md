---
layout:     post
title:      如何在 Oozie 中创建有依赖的 WorkFlow
subtitle:   
date:       2020-11-25
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - 数据仓库
---

> 原文地址：https://cloud.tencent.com/developer/article/1158324
> 
> 转载自微信公众号：Hadoop实操


### 1.文档编写目的

* * *

在使用 Hue 创建 WorkFlow 时，单个 WorkFlow 中可以添加多个模块的依赖，使各个模块之间在 WorkFlow 内产生依赖关系，如果对于一个 WorkFlow 被其它多个 WorkFlow 依赖（如：AWorkFlow 执行成功后，BWorkFlow 和 CWorkFlow 依赖 AWorkFlow 的执行结果），这时不可能将 AWorkFLow 作为 BWorkFlow 和 CWorkFlow 中的一个处理模块来，这样会重复执行 AWorkFlow，可能会导致输入 BWorkFlow 和 CWorkFlow 的输入不一致等问题，那本篇文章 Fayson 主要介绍如何使用 Oozie 的 Coordinator 功能来实现 WorkFlow 之间的依赖。

* 内容概述：

	1. 环境准备
	
	2. 创建测试 WorkFlow 与 Coordinator
	
	3. WorkFlow 依赖测试
	
	4. 总结

* 测试环境：

	1. CM5.14.3/CDH5.14.2
	
	2. 操作系统版本为 Redhat7.3
	
	3. 采用 root 用户进行操作
	
	4. 集群已启用 Kerberos

### 2.环境准备

* * *

1. 由于是 Kerberos 环境，在 shell 脚本中需要一个 keytab，生成一个 hiveadmin.keytab 文件

```
[root@cdh01 ~]# kadmin.local
Authenticating as principal hbase/admin@FAYSON.COM with password.
kadmin.local:  xst -norandkey -k hiveadmin.keytab hive/admin@FAYSON.COM
```

使用 klist 命令查看导出的 keytab 文件是否正常

```
[root@cdh02 wordcount]# klist -ek hiveadmin.keytab
```

![](https://ask.qcloudimg.com/http-save/yehe-1522219/hjpf4sfyls.jpeg?imageView2/2/w/1620)

2. 准备两个 shell 脚本用于创建两个 WorkFlow

generator_wordcount.sh 脚本内容如下：

```
#!/bin/bash

kinit -kt hiveadmin.keytab hive/admin@FAYSON.COM
INPUT_HDFS=/benchmarks/wordcount/input
DATASIZE=1073741824
hadoop fs -rmr ${INPUT_HDFS} || true
hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar randomtextwriter -Dmapreduce.randomtextwriter.totalbytes=${DATASIZE} ${INPUT_HDFS}
```

![](https://ask.qcloudimg.com/http-save/yehe-1522219/bfwev41y4a.jpeg?imageView2/2/w/1620)

wordcount.sh 脚本内容如下：

```
#!/bin/bash

kinit -kt hiveadmin.keytab hive/admin@FAYSON.COM
INPUT_HDFS=/benchmarks/wordcount/input
OUTPUT_HDFS=/benchmarks/wordcount/output
hadoop fs -rmr $OUTPUT_HDFS
NUM_REDS=160
hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount \
    -Dmapreduce.input.fileinputformat.split.minsize=1073741824 \
    -Dmapreduce.reduce.tasks=${NUM_REDS} \
    ${INPUT_HDFS} ${OUTPUT_HDFS}
hadoop fs -rmr ${INPUT_HDFS}
```

![](https://ask.qcloudimg.com/http-save/yehe-1522219/1te4543pxq.jpeg?imageView2/2/w/1620)

### 3.创建测试WorkFlow

* * *

这里创建 Shell 类型的 Oozie 工作流就不再详细的说明，可以参考 Fayson 前面的文章《[Hue中使用Oozie创建Shell工作流在脚本中切换不同用户](http://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==&mid=2247486600&idx=1&sn=b4140e2bea60e7d20707716d6c4fac24&chksm=ec2adc81db5d5597c4a9d5c0c429a7bf2cd502a1abeacf402aa2151a6a7155dc6fb598697fa1&scene=21#wechat_redirect)》中有介绍如何创建一个 shell 类型的  Oozie工作流，这里需要注意的是 Kerberos 环境下，我们需要将 keytab 文件也上传至对应 WorkFlow 的 WorkSpace/lib 目录下，如下图所示：

![](https://ask.qcloudimg.com/http-save/yehe-1522219/ovao05r7cv.jpeg?imageView2/2/w/1620)

1. 创建一个 GeneratorWorkFlow

![](https://ask.qcloudimg.com/http-save/yehe-1522219/blyt4f1w70.jpeg?imageView2/2/w/1620)

2. 创建一个 WordCountWorkFlow

![](https://ask.qcloudimg.com/http-save/yehe-1522219/2cepkwnghi.jpeg?imageView2/2/w/1620)

这两个 WorkFlow 的依赖关系，只有 GeneratorWorkFlow 执行成功，生成了 WordCount 的输入数据后，WordCountWorkFlow 才可以执行，**否则 WordCountWorkFlow 一直处于等待状态**。

### 4.创建Coordinator

* * *

在 Hue 中创建 Oozie 的 Coordinator 即对应 Hue 中的功能为 Scheduler

![](https://ask.qcloudimg.com/http-save/yehe-1522219/2wrg9b1rxd.jpeg?imageView2/2/w/1620)

1. 先创建一个生成数据的 Coordinator，用于定时生成 WordCount 测试数据

![](https://ask.qcloudimg.com/http-save/yehe-1522219/xmacw52hl4.jpeg?imageView2/2/w/1620)

2. 创建一个 WordCountSchedule，用于定时的去执行 WordCount 作业

![](https://ask.qcloudimg.com/http-save/yehe-1522219/6d0eflat7w.jpeg?imageView2/2/w/1620)

注意：下面的配置比较关键，通过对 GeneratorWorkflow 工作流输出的 `/benchmarks/wordcount/input` 目录进行判断，如果满足条件则执行 WordCountWorkFlow 工作流。

![](https://ask.qcloudimg.com/http-save/yehe-1522219/o1041my3pl.jpeg?imageView2/2/w/1620)

完成上述两个 Schedule 的创建后，保存配置并启动该 Schedule。

![](https://ask.qcloudimg.com/http-save/yehe-1522219/hrg2ecuaw3.jpeg?imageView2/2/w/1620)

### 5.WorkFlow 依赖测试

* * *

1. 点击 Jobs 可以看到如下两个正在运行的 WorkFlow

![](https://ask.qcloudimg.com/http-save/yehe-1522219/ixpc2jtwgz.jpeg?imageView2/2/w/1620)

2. 通过 Yarn 查看作业的执行情况，这里的作业已经执行成功了，我们通过时间来分析

![](https://ask.qcloudimg.com/http-save/yehe-1522219/tz2dcjk38p.jpeg?imageView2/2/w/1620)

3. 通过 GeneratorWorkflow 工作流的作业执行情况可以看到

![](https://ask.qcloudimg.com/http-save/yehe-1522219/k46tvlu4c8.jpeg?imageView2/2/w/1620)

在 2018-06-10 23:10:00 看到 GeneratorWorkflow 向集群提交了作业，与我们定义的启动时间一致，到 2018-06-10 23:10:14 可以看到开始执行生成数据的 MR 作业，并成功执行，作业执行结束时间为 2018-06-10 23:10:34。

4. 通过 WordCountWorkFlow 工作流的作业执行情况可以看到

![](https://ask.qcloudimg.com/http-save/yehe-1522219/delnajwk4j.jpeg?imageView2/2/w/1620)

在 2018-06-10 23:11:00 才启动 WordCountWorkFlow 工作流，本应该在 2018-06-10 23:03:00 执行的工作流一致处于等待状态，直到 2018-06-10 23:11:00 GeneratorWorkflow 工作流执行成功，生成了 `/benchmarks/wordcount/input` 目录的数据后， WordCountWorkFlow 工作流才开始执行，可以看到 WordCount 作业的开始执行时间为 2018-06-10 23:11:14 ，在生成了 WordCount 测试数据后才执行。GeneratorWorkflow 工作流执行成功后与 WordCountWorkFlow 的执行时间间隔为1分钟，即为我们在 WordCountSchedule 中配置的每个一分钟检查一次。

5. 通过如上作业执行情况分析，可以得出 WordCountWorkFlow 工作流的执行是依赖 GeneratorWorkflow 工作流

### 6.总结

* * *

1. 在创建有依赖关系的 WorkFlow 时，我们可以通过 Coordinator 的方式来是实现工作流之间的依赖关系，可以避免被依赖的 WorkFlow 工作流被重复执行。

2. Coordinator 是一个定时执行 WorkFlow 的调度工具，可以基于时间与数据生成为条件的方式触发。

3. Coordinator 指定 HDFS 的数据目录，可以使用 `${YEAR}、${MONTH}` 等EL表达式的方式进行设置。

4. done_flag 即为数据目录生成的文件标识，若未指定则默认为 `_SUCCESS` 文件，若指定为空，则表示文件夹本身。


> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:



