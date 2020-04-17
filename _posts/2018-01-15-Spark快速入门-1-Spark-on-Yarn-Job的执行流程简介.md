---
layout:     post
title:      Spark快速入门-1-Spark on Yarn Job的执行流程简介
subtitle:   
date:       2018-01-15
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
* 2017-12-24-Hadoop2架构及HA集群配置（2）
* 2017-12-25-Spark集群搭建
* 2017-12-29-Hadoop和Spark的异同
* 2017-12-28-Spark-HelloWorld(Spark开发环境搭建)

### 相关概念

> 在介绍一个典型的 Spark Job 是如何被调度执行前，先了解以下几个重要的概念

![](https://ws2.sinaimg.cn/large/006tKfTcly1fncvn069uej30gk07ygln.jpg)

* DAG：即 Directed Acyclic Graph，有向无环图。
* Application：Application 是用户编写的 Spark 应用程序，其中包含了一个 Driver 功能的代码和分布在集群中多个节点上运行的 Executor 代码。
* Driver：使用 Driver 这一概念的分布式框架很多，比如 Hive 等。 Spark 中的 Driver 即运行 Application 的 main() 函数并创建SparkContext，创建 SparkContext 的目的是为了准备 Spark 应用程序的运行环境。在 Spark 中由 SparkContext 负责与ClusterManager 通信，进行资源的申请、任务的分配和监控等。当 Executor 部分运行完毕后，Driver 同时负责将 SparkContext 关闭。通常用 SparkContext 代表 Driver。
* Executor：某个 Application 运行在 Worker 节点上的一个进程，该进程负责运行某些 Task，并且负责将数据存在内存或者磁盘上，每个 Application 都有各自独立的一批 Executor。 在 Spark on Yarn 模式下它负责将 Task 包装成 taskRunner ，并从线程池抽取出一个空闲线程运行 Task。
* Cluster Manager：指的是在集群上获取资源的外部服务，目前有三种类型：

> Standalone：Spark 原生的资源管理，由 Master 负责资源的分配。
> 
> Apache Mesos：与 Hadoop MapReduce 兼容性良好的一种资源调度框架。
> 
> Hadoop Yarn：主要是指的 Yarn 中的 ResourceManager。

* Worker：集群中任何可以运行 Application 代码的节点。在 Standalone 模式中指的就是通过 slave 文件配置的 Worker 节点，在 Spark on Yarn 模式中指的就是 NodeManager 节点。
* Task：被送到某个 Executor 上的工作单元，和 Hadoop MapReduce 中的 MapTask 和 ReduceTask 概念一样，是运行Application 的基本单元，代表单个数据分区上的最小处理单元。Task 分为 ShuffleMapTask 和 ResultTask 两类。ShuffleMapTask 执行任务并把任务的输出划分到 (基于 task 的对应的数据分区) 多个 bucket(ArrayBuffer) 中，ResultTask 执行任务并把任务的输出发送给驱动程序。多个 Task 组成一个 Stage，而 Task 的调度和管理等由下面的 TaskScheduler 负责。
* TaskSet：代表一组相关联的没有 shuffle 依赖关系的任务组成任务集。一组任务会被一起提交到更加底层的 TaskScheduler 进行管理。
* Stage：Job 被确定后，Spark 的调度器 (DAGScheduler) 会根据该计算作业的计算步骤把作业划分成一个或者多个 Stage。Stage 又分为 ShuffleMapStage 和 ResultStage，每一个 Stage 将包含一个 TaskSet。
* Job：Spark 的计算操作是 lazy 执行的，只有当碰到一个动作 (Action) 算子时才会触发真正的计算。一个 Job 就是由动作算子而产生包含一个或多个 Stage 的计算作业。
* RDD ：Spark的基本计算单元，可以通过一系列算子进行操作（主要有 Transformation 和 Action 操作）的弹性分布式集合（Resilient Distributed Datasets）简称，是分布式只读且已分区集合对象。***RDD 是 Spark 最核心的东西，它表示已被分区、被序列化的、不可变的、有容错机制的，并且能够被并行操作的数据集合。***其存储级别可以是内存，也可以是磁盘，可通过spark.storage.StorageLevel属性配置。
* Spark 算子：大致可以分为以下两类：

> Transformation 变换/转换算子：这种变换并不触发提交作业，只是完成作业中间过程处理。Transformation 是延迟计算的，也就是说从一个 RDD 转换生成另一个 RDD 的转换操作不是马上执行，需要等到有 Action 操作的时候才会真正触发运算。
> 
> Action 行动算子：这类算子会触发 SparkContext 提交 Job 作业，并将数据输出 Spark系统。
> 

![DAGScheduler](https://tva2.sinaimg.cn/large/006tKfTcly1fndgd4p883j30hz0810sy.jpg)

* DAGScheduler：根据 Job 构建基于 Stage 的 DAG，并提交 Stage 给 TaskScheduler。其划分 Stage 的依据是 RDD 之间的依赖关系，根据 RDD 和 Stage 之间的关系找出开销最小的调度方法，然后把 Stage 以 TaskSet 的形式提交给 TaskScheduler。此外，DAGScheduler 还处理由于 Shuffle 数据丢失导致的失败，这有可能需要重新提交运行之前的 Stage（非 Shuffle 数据丢失导致的 Task 失败由 TaskScheduler 处理）。 

![TaskScheduler](https://ws2.sinaimg.cn/large/006tKfTcly1fndge3c4p6j30ev07974c.jpg)

* TaskScheduler：将 Taskset 提交给 Worker（集群）运行，每个 Executor 运行什么 Task 就是在此处分配的。TaskScheduler 还维护着所有 Task 的运行状态，重试失败的 Task。

![](https://tva2.sinaimg.cn/large/006tKfTcly1fnd2c22o3oj30l309c449.jpg)

* 宽依赖：与 Hadoop MapReduce 中 Shuffle 的数据依赖相同，宽依赖需要计算好所有父 RDD 对应分区的数据，然后在节点之间进行 Shuffle。
* 窄依赖：指某个具体的 RDD，其分区 partitoin a 最多被子 RDD 中的一个分区 partitoin b 依赖。此种情况只有 Map 任务，是不需要发生 Shuffle 过程的。
* Stage 的划分依据：是以 ShuffleDependency 为依据的，也就是说当某个 RDD 的运算需要将数据进行 Shuffle 时，这个包含了 Shuffle 依赖关系的 RDD 将被用来作为输入信息，进而构建一个新的 Stage。我们可以看到用这样的方式划分 Stage，能够保证有依赖关系的数据可以以正确的顺序执行。根据每个 Stage 所依赖的 RDD 数据的 partition 的分布，会产生出与 partition 数量相等的 Task，这些 Task 根据 partition 的位置进行分布。
 
### Spark on Yarn 的Job执行流程


![](https://ws4.sinaimg.cn/large/006tKfTcly1fnd36ro2b1j311m0nazmn.jpg)

Spark 应用程序被提交后，当某个动作算子触发了计算操作时，SparkContext 会向 DAGScheduler 提交一个作业，接着 DAGScheduler 会根据 RDD 生成的依赖关系划分 Stage，并决定各个 Stage 之间的依赖关系，Stage 之间的依赖关系就形成了 DAG。

![](https://ws1.sinaimg.cn/large/006tKfTcly1fnd3l4sqwyj30if0csjs8.jpg)

在 Yarn-Cluster 模式中，当用户向 Yarn 中提交一个应用程序后， Yarn 将分两个阶段运行该应用程序：第一个阶段是把 Spark 的 Driver   作为一个 ApplicationMaster 在 Yarn 集群中先启动；第二个阶段是由 ApplicationMaster 创建应用程序，然后为它向 ResourceManager 申请资源，并启动 Executor 来运行 Task，同时监控它的整个运行过程，直到运行完成。 

> 1.Spark Yarn Client 向 Yarn 中提交应用程序。
> 
> 2.ResourceManager 收到请求后，在集群中选择一个 NodeManager，并为该应用程序分配一个 Container，在这个 Container 中启动应用程序的 ApplicationMaster， ApplicationMaster 进行 SparkContext 等的初始化。
> 
> 3.ApplicationMaster 向 ResourceManager 注册，这样用户可以直接通过 ResourceManager 查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束。
> 
> 4.ApplicationMaster 申请到资源（也就是Container）后，便与对应的 NodeManager 通信，并在获得的 Container 中启动 CoarseGrainedExecutorBackend，启动后会向 ApplicationMaster 中的 SparkContext 注册并申请 Task。
> 
> 5.ApplicationMaster 中的 SparkContext 分配 Task 给 CoarseGrainedExecutorBackend 执行，CoarseGrainedExecutorBackend 运行 Task 并向ApplicationMaster 汇报运行的状态和进度，以让 ApplicationMaster 随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。
> 
> 6.应用程序运行完成后，ApplicationMaster 向 ResourceManager申请注销并关闭自己。

***






