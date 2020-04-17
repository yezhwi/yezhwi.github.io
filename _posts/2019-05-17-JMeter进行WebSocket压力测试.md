---
layout:     post
title:      JMeter进行WebSocket压力测试
subtitle:   
date:       2019-05-17
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - WebSocket
    - JMeter
---

### 背景

之前两篇内容介绍了一下 [WebSocket](https://mp.weixin.qq.com/s/Fi5xAK7SshrycFO9Y2_v_w) 和 [SocketIO](https://mp.weixin.qq.com/s/2fTSrJTawdhh1D_9ExPVJw) 的基础内容。之后用 Netty-SocketIO 开发了一个简单的服务端，支持服务端主动向客户端发送消息，同时也支持客户端请求，服务端响应方式。本文主要想了解一下服务端的性能怎么样，选择使用 JMeter 对 WebSocket 应用进行性能测试。


### JMeter 扩展实现 WebSocket 支持

JMeter 是目前最为流行的开源性能测试工具，JMeter 本身提供的基于插件的机制允许第三方实现标准 JMeter 所不支持的协议，而 WebSocket 的一个比较好的实现是 [WebSocketSampler](https://github.com/XMeterSaaSService/JMeter-WebSocketSampler/releases) 。利用此插件，能完成基于 WebSocket 协议的基本性能测试。

#### 安装 WebSocketSampler 插件

* 通过插件地址 https://github.com/maciejzaleski/JMeter-WebSocketSampler/releases 下载最新版本（目前版本是1.0.2），本项目的源码是用 Maven 管理，直接通过 mvn package 就能生成 jar 包，然后将其拷贝到 JMeter 安装目录的 $JMETER_HOME/lib/ext 下。在生成 jar 包前要先对源码进行一点修改，因为在测试的时候报错，如下图：

![checkForComodification](https://ws4.sinaimg.cn/large/006tNc79ly1g33ixgp1gcj31jd0u0why.jpg)

很简单，用下面的代码替换

```
Queue<String> responeBacklog = new ConcurrentLinkedQueue<String>();
```

ServiceSocket.java 中的

```
protected Deque<String> responeBacklog = new LinkedList<String>();
```

一行即可。

java.util.LinkedList\$ListItr.checkForComodification(LinkedList.java:953)异常解决方案参考地址：

https://stackoverflow.com/questions/16152648/websocket-plugin-for-jmeter

https://github.com/maciejzaleski/JMeter-WebSocketSampler/issues/21


* 下载相关额外的依赖，并将他们也拷贝到 JMeter 安装目录的 $JMETER_HOME/lib/ext，如下：

```
jetty-http-9.1.1.v20140108.jar
jetty-io-9.1.1.v20140108.jar
jetty-util-9.1.1.v20140108.jar
websocket-api-9.1.1.v20140108.jar
websocket-client-9.1.1.v20140108.jar
websocket-common-9.1.1.v20140108.jar
```

> 注意版本号要写插件项目里的版本一致，我在最开始使用上面的 jar 包时用的最新版本，报错。
> 
> 如果没有上面的6个 jar 包，在进行测试的时候同样也会报错。


### 配置 JMeter 的测试脚本

#### 新建线程组

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33jr6kkbpj31cf0u041t.jpg)

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33jrkpr17j31c00u0wfz.jpg)

#### 创建循环控制器

![](https://ws2.sinaimg.cn/large/006tNc79ly1g33jtgoqvaj31c50u0jvz.jpg)

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33jtywfkoj31c00u0q3w.jpg)

#### 添加 WebSocket Sampler 

![](https://ws2.sinaimg.cn/large/006tNc79ly1g33jw09ijzj31cb0u0n1p.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79ly1g33jxrloerj31c00u0wgk.jpg)

#### 添加查看结果树

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33jz01iugj31c90u0796.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79ly1g33jzbna9tj31c00u0mym.jpg)

#### 添加聚合报告

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33k0f4uwlj31cd0u0wjc.jpg)

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33k0vbqjtj31c00u0dhb.jpg)

#### 配置 WebSocket Sampler 

* Server Name or IP：部署 WebSocket 应用所在的服务器地址；
* Port Number：端口号；
* Timeout：Connection，连接超时，超过此时间未建立连接则测试报错；Response，发送消息后的超时时间；
* Implementation：现在只支持 RFC6455；
* Protocol：ws 或者 wss。wss 指的是加密的 WebSocket，根据被测的配置而定；
* Path：所部署 WebSocket 服务的路径；
* Streaming connection：测试期间是否重用连接，如果处于非选中状态，每次得到服务器端的返回后就会关闭连接，下次执行时会新建连接；
* Request Data：发送出去的数据 —— 下面重点说怎么发送数据与接收数据；
* Response pattern：等待服务器返回的特定的字符集合；否则等待Response Timeout设定的超时时间；
* Close Connection Pattern：与8类似，但是符合条件的时候连接将被关闭；
* Message Backlog：定义最多留下的返回消息的数目。

#### 发送与接收数据

* 在 chrome 的调试模式下可以找到 WebSocket 的连接信息：

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33kcqke41j31270u0jug.jpg) 

* 查看发送的信息内容，右键可以进行 copy 

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33kd4qmwuj312n0u0jt9.jpg)

* 根据上面的 ws 连接信息配置 WebSocket Sampler 

![](https://ws2.sinaimg.cn/large/006tNc79ly1g33jxrloerj31c00u0wgk.jpg)

### 运行输出结果

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33kj3d58aj31c00u0gpd.jpg)

![](https://tva2.sinaimg.cn/large/006tNc79ly1g33kjivsklj31c00u0jt3.jpg)

### 通过聚合报告看性能

* Samples：样本总数量，等于线程总数 * 循环次数。
* Average：请求处理的平均时间(毫秒ms)，是压力测试的主要指标之一 。
* Median：请求处理的中值时间(毫秒ms)，样本数量中有一半的处理时间在这个值之上，有一半的处理时间在这个值之下。
* 90%Line，95%Line，99%Line：样本中百分之多少的处理时间都在这个值之下，是压力测试的主要指标之一。
* Min：耗时最少的请求时间。
* Max：耗时最多的请求时间。
* Error%：错误率。
* Throughput：吞吐量，服务器每秒处理的请求数。
* KB/sec：服务器每秒钟请求的字节数。


