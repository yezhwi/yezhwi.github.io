---
layout:     post
title:      从0到1编写一个RPC框架（基于Zookeeper）
subtitle:   
date:       2021-03-05
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - 架构
    - Java
    - RPC
---

> 原文地址：http://www.iloveqyc.com/2019/04/06/air-rpc/

### 零、前言

这是我很久之前造的一个RPC轮子，名叫AirRPC，它基于zookeeper，和阿里dubbo、美团pigeon等框架比较类似（毕竟RPC框架原理都一样）。源码在github上，有兴趣的同学可以看看：[https://github.com/qiuyongchen/AirRPC](https://github.com/qiuyongchen/AirRPC)。

下面将详细描述出整个项目的设计思路与实现，包括相关的理论与模型、框架建模与框架模块设计、部署步骤与测试结果等。

### 一、引言

#### 1.1 研究背景和意义

团队在发展初期，由于规模和业务量小，网站开发人员只需将网站以Tomcat + Linux + MySQL + Java的形式部署在一台机器上便足以应对业务流量。

随着时间发展，团队内的业务日益复杂，仅依靠一台机器已不能应付流量压力，此时为了及时跟进业务的发展，网站开发人员通过使用分而治之的手段，把整个网站业务进行垂直拆分。比如，可以根据网站的业务的不同，把一个项目分拆成多个，每个项目专人专职，由专门的开发人员负责，各个业务的流量压力由不同的项目承担，这在一定程度上可以缓解业务发展带来的压力。

久而久之，各个项目的开发人员发现，各一个项目都需要执行许多相同的业务操作，比如用户管理，产品管理和供应商的管理等，而且，每个项目都要和数据库保持连接，给数据库带来了极大压力，数据库有拒绝服务的可能性。为了解决该问题，开发人员提出了水平切分，也就是分布式服务的解决方案，将多个项目共同拥有的业务操作提取出来，根据业务类型放到多个Service项目中，各个非Service项目均调用Service项目提供的服务。

本项目正是一个致力于解决水平切分问题的分布式服务框架，让小型开发团队可以透明地从单机服务架构扩展到分布式服务架构。

#### 1.2 研究现状

目前在业界开源出来的RPC框架有以下一些。

Dubbo 是阿里巴巴公司开源的一个Java服务框架，但该框架已无人维护\[1\] ，相关的依赖类比如Spring，Netty还是很老的版本。  
Motan是新浪微博开源的一个Java 服务框架，支持通过spring配置方式集成\[2\]，但在配置Zookeeper时需要显示调用开关，配置服务时每个服务均需配置注册中心 ，略显繁琐。  
Pigeon是点评开源的一个 Java 服务框架，很好地支持了服务降级、服务限流等服务治理功能\[3\] ，是一款强大的高性能RPC框架。但Pigeon过于庞大，尤其是在兼容thrift的过程中，引入了大量的代码。

现有的RPC框架要么是没人维护，要么是框架太大太重，使用复杂维护成本高，不能很好地满足发展初期的小团队的需求，鉴于该事实，有必要打造一个超轻量级的分布式服务框架，使得发展团队透明地过渡到分布式服务时代，而在团队继续壮大后，拥有专门负责中间件开发的基础架构部门，还能继续透明地过渡到超大型RPC服务框架。

#### 1.3 项目的目标和范围

项目的目标是团队的开发人员，具体的说，是提供服务的开发人员和调用服务的开发人员。提供服务的开发人员编写好自己的服务后，将自己的服务暴露出去，供上层的业务调用。调用服务的开发人员在需要时，寻找到自己所需服务，直接调用即可，无须关注目标服务部署在另外一台机器上。

在基本的分布式服务调用实现后，开发人员还需要监控服务的运维状况，查看服务调用的成功率、调用时长、调用总数、吞吐率等数据。

可见，本项目可提供以下功能：

（1）服务发布，服务提供者将自己所能提供的服务通过服务注册中心发布出去。  
（2）服务订阅，服务调用者在启动时向服务注册中心订阅自己想要的服务，服务注册中心将拥有该服务的所有机器地址返回给服务调用者，服务调用者根据负载均衡策略自动选择服务提供者。  
（3）服务更新，当有新的服务提供者通过服务注册中心发布某项已有服务时，注册中心有能力将新机器的地址推送给已订阅该服务的服务调用者；当服务提供者下线时，会通过服务注册中心将某项服务的下线通告给所有订阅该服务的服务调用者。

### 二、相关理论与技术

本章主要介绍分布式服务框架的实现过程中依赖的主要技术和理论，包括Java远程调用、面向服务的架构、订阅发布模式、ZooKeeper等，在主要列出项目依赖的技术的同时，还适当列举出类似的技术，用作比较。

#### 2.1 Java远程调用

本项目中，实现远程调用使用的是RPC，而不是HTTP。HTTP是七层网络协议\[4\]， RPC可采用TCP协议，速度较HTTP略有优势。

RPC（Remote Procedure Call Protocol），远程过程调用协议，它是一种通过网络从远程服务器上请求服务，而不需要关心和了解底层网络技术的协议。

RPC是一种C/S编程模型，客户端与服务端建立连接后，客户端的调用参数通过底层服务通道，根据传输前所提供的目的地址，传给服务端，此时客户端处于等待状态，直到收到应答或 TimeOut 超时信号。当服务器收到请求信息时，会根据注册RPC时告诉RPC系统的例程入口地址，执行相应的操作，并将结果返回至客户端\[5\]。

在面向过程的编程世界里，RPC服务架构把服务器看做由一些过程组成，客户端调用这些过程来执行特定的任务\[6\]；而在面向对象的编程世界中，RPC使得分布在不同机器上的对象的属性和行为都像是本地对象一样\[7\]，也就是说，借助RPC，我们可以将本地服务部署在远程机器上，并像调用本地服务一个调用部署在远程机器上的服务。

在Java中，有RMI和WebService两种技术可以实现远程过程调用，但它们没有足够的能力处理大型对象。

HTTP作为在客户和服务端传输超文本数据的协议，它只规定了少量的用于沟通信息的请求报文和应答报文，使得它的使用变得简单。但是，另一方面，由于它是应用层协议，建立在传输层协议TCP的基础上，缺点是传输过程中步骤较多，协议报文头较长，需要更多次的编码和解码 。因此，本项目的RCP框架建立在TCP长连的基础上，而不是建立在HTTP的基础上。

#### 2.2 面向服务的架构

##### 2.2.1 SOA的介绍

SOA(Service Oriented Architecture， SOA)，面向服务的体系结构，来源于早期的基于构件的分布式计算方式\[8\]。根据服务之间先前定义好的接口，将服务调用者和服务提供者以松耦合的方式联结起来。在SOA中，组件是若干个Web服务\[9\]，服务间使用接口沟通，以契约的形式组成一个个的集群，集群内的服务形成一个整体，在其它集群看来，该集群内部是一个黑盒子，集群内部有多少台机器，各台机器的运行健康状况是不透明的。

一个集群可以调用另一个集群的服务，一个集群无需理会其它集群的状况。同时，一个集群既可以是服务调用者，也可以是服务提供者，图2-1是一个SOA的参考模型。

![图2-1 SOA的参考模型](https://gitee.com/yzhw/img/raw/master/img/air-rpc-2-1-20210305010623431-20210305010628560.jpg)

SOA代表了面向服务的架构，也就是说，在SOA的世界里，存活的是一个个独立的松耦合的黑盒子服务，这些服务可以编排在一起以实现特定的功能，举个例子，风控拦截服务、商品参数校验服务、库存服务、价格服务、标签服务等多种服务编排在一起，实现一个“用户下单”的功能，这些服务内部都是黑盒子，服务之间互相不透明，服务之间有先后依赖关系。

##### 2.2.2 SOA的特征

SOA一般会有松耦合高内聚、黑匣子、自定义、可管理、即插即用等特性。

（1）松耦合高内聚：这意味着每一个服务是自包含单独存在的逻辑，每个服务内部的逻辑是完整的。举例来说，我们采取了“支付服务”，该服务内部有一套完整的逻辑来判别用户是否已经支付，只关心用户什么时候支付，而无需关注库存服务的结果。

（2）黑匣子：在SOA中，服务隐藏有内在的复杂性。他们只使用交互消息，服务接受和发送消息。举例来说，支付服务不需关注库存服务的实现方式，同理，库存服务也不需关注支付服务的实现。

（3）自定义： SOA服务应该能够自己定义。

（4）可管理： SOA服务保持在一个中央存储库。即可以删除服务，也可以新添服务，服务就像一个个箱子一样，可以存放在仓库里，也可以拿出仓库。应用程序可以在中央存储库中搜索服务，并调用相应服务。

（5）即插即用：SOA服务可以编排和链接实现一个特定功能。例如，“业务流程”中有两个服务“风控服务”和“支付服务”，先风控再支付，或者先支付再风控都可以，只要符合业务目标，编排顺序只要适合就可以工作。使用SOA可以松散耦合的方式管理服务之间的工作流。

在本项目中，基于ZooKeeper打造了一个服务注册中心，服务注册中心实际上也是一个服务管理平台，如果不是发生单机一个服务的生命周期。提供服务的项目是服务提供者，通过注册中心发布服务。调用服务的项目是服务调用者，通过注册中心查找和订阅服务，找到具体的服务后再向拥有的该服务的机器发起调用请求。

#### 2.3 订阅模式

发布订阅模式，即是观察者模式，指一个实体观察着另一个实体里的事件，当特定的事件发生时，观察的实体做出相应的动作，调用相应的业务逻辑流程进行处理。发布/订阅系统是一种使分布式系统中的各参与者能以“发布/订阅”的方式进行交互的中间件系统\[10\]。在发布订阅模式中，发布者和订阅者之间通过“事件消息”松耦合，发布者发布某种事件，发布订阅系统将事件的产生告知订阅者，发布者完全不知晓订阅者的存在，发布者和订阅者之间单向依赖，即只有订阅者依赖于发布者。

图2-2即是一个发布订阅系统的概念模型。在本项目中，服务注册中心是发布订阅系统，服务提供者是事件发布者，服务调用者是事件订阅者。

![图2-2 发布订阅系统的概念模型](https://gitee.com/yzhw/img/raw/master/img/air-rpc-2-2.jpg)

在发布订阅模式中，各个组件的工作模式如下：  
（1）发布订阅系统维护着发布者的状态，维护着订阅者的状态。一个发布者可能会对应着多个订阅者，实际上，发布订阅者模式的优点也在于一个发布者能引起多个订阅者注意和行动。在本项目中，服务注册中心担任发布订阅系统的责任，维护服务提供者和服务调用者的状态。  
（2）发布者发布一条状态变更的事件消息给发布订阅系统。比如在本项目里，服务提供者将自己的服务上线或下线变更告知服务注册中心。  
（3）发布订阅系统根据Push模型或 Pull 模型将发布者的状态变更的事件消息告知所有的订阅者。比如在本项目中，获悉到服务提供者要下线，服务注册中心会广播该消息给所有的服务调用者。

##### 2.3.1 Push模型

在Push模型中，发布订阅系统主要向订阅者推送消息。每当发布者有新的消息到达时，发布订阅系统便会将消息的全部内容全部推向订阅者。在该模型下，消息的实时性有很好的保证，系统一旦收到消息，便尽可能让订阅者知道。同时，Push 模型里，负载均衡的工作由系统统一处理和控制。

##### 2.3.2 Pull模型

在该模型下，新事件到来时，发布订阅系统仅会将少量关键信息告知订阅者，订阅者若要知道详细的情况，需要主动向发布订阅系统请求详情。可见，在该模型下，发布订阅系统的压力比较小，负载均衡由订阅者控制。而且，事件消息的实时性取决于pull的时间间隔。

#### 2.4 ZooKeeper

在分布式应用中，锁机制是个极为复杂的和难解决的问题，开发人员在锁机制上需要耗费的人力物力，往往是力倍功半，取不到好的效果，即使是采用基于消息的协调机制，有时仍不能解决该问题，因此需要一种开源的工具帮助开发人员解决此问题，ZooKeeper就是为此而生的。ZooKeeper是一个开源项目，实现了一种可靠的、可扩展的、分布式的、可配置的协调机制，为分布式系统提供简单易用而且可靠的分布式配置服务、分布式同步服务和分布式命名注册。Zookeeper是Hadoop的正式子项目,用于提供高效和稳定的一致性服务接口,基于它可以实现分布式锁、配置维护等服务\[11\]。

ZooKeeper集群最大的一个特点是同步，在外界看来，集群内各个节点的数据都是一样的，如果某个节点挂了，整个集群对外提供的内容不会受到影响。在ZooKeeper内有Leader和Follower的概念，多个的Follower和唯一一个Leader保持同步，一旦Leader挂掉，ZooKeeper集群在几毫秒内足以选出新Leader，而Follower挂了是无关紧要的。

ZooKeeper为了保证数据的唯一性，其命名空间结构和Linux文件系统很像，是一棵树，所有的数据均存放在树节点上。图2-3是ZooKeeper的命名空间结构。

![图2-3 ZooKeeper的命名空间结构](https://gitee.com/yzhw/img/raw/master/img/air-rpc-2-3.jpg)

##### 2.4.1 Push模型

（1）最终一致性：对客户端来说，不管连接到集群内的任何一台服务器，得到的结果都是一样的，这是zookeeper最重要的性能。

（2）可靠性：具有简单、健壮、良好的性能，如果消息被一台服务器接受，那么它将被所有的服务器接受。

（3）实时性：Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。必要时，应该在读数据之前调用sync()接口。

（4）等待无关：慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。

（5）原子性：更新只能成功或者失败。

（6）顺序性：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。

本项目中，ZooKeeper将被用作服务注册中心，专门用于管理服务的上线和下线。

### 三、基于ZooKeeper的分布式服务框架建模

本项目中，面对的用户主要有两类，一是提供服务的开发人员，二是调用服务的开发人员。

提供服务的开发人员，在他的项目里定义服务的接口，并且编写完成服务的实现，引入本分布式框架到项目里，在框架的配置文件中记录该服务，配置接口名称、接口实现类，在框架随着项目启动后，该服务自动发布在服务注册中心。

调用服务的开发人员，可以在他负责的项目中，加入本分布式框架的部分client代码，在框架的配置文件中引入该服务，即可在代码里引用该服务。项目启动的过程中，会启动本框架，框架自动从服务注册中心引用订阅该服务。

为方便描述，本分布式服务框架取名为AirRPC，后续将以AirRPC代表本框架项目。下面我将画出系统用例图，让读者对系统的总功能有所了解，对几个关键的用例进行详细的描述，接着使用包图来描述应用领域概念的关系。

#### 3.1 系统用例图

用例表示系统用户的目标，以及他们执行操作以达到目标的过程。本分布式服务框架的用例图如图3-1。

![图3-1 系统用例图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-3-1.jpg)

#### 3.2 关键用例分析

在本项目中，比较重要的用例有注册服务、依赖服务。

##### 3.2.1 注册服务

注册服务是指开发人员开发完成自己负责的服务后，将服务记录在AirRPC的Server配置文件上，AirRPC将其发布在服务注册中心，使得其它开发人员在需要时依赖，具体的流程如下：

（1）开发人员定义Service服务接口，并编写接口的实现。

（2）开发人员在AirRPC的特有配置文件中记录服务的名称及服务对应的具体的类。

（3）开发人员引入AirRPC的Pom依赖，并使得AirRPC随同自己的项目一起启动。

（4）如果在项目启动过程中，AirRPC察觉到之前曾有其它机器已经在提供同名服务，则将项目所在机器的ip和端口连同其它机器的ip和端口放在一个集合中，形成一个服务集群。

（5）项目启动成功，可以看到日志中AirRPC打出的日志，表明提供服务成功。

##### 3.2.2 依赖服务

依赖服务是指开发人员在需要某一项服务时，将需要的服务记录在AirRPC的Client配置文件上，AirRPC在服务注册中心订阅该服务，开发人员在本地使用该服务时，AirRPC以动态代理的方式调用远程的服务，具体的流程如下：

（1）开发人员引入AirRPC的Pom依赖，并使得AirRPC随同自己的项目一起启动。

（2）开发人员在AirRPC的特有配置文件中配置自己引用的服务的名称及接口。

（3）开发人员在项目代码里如同使用本地服务一般使用依赖的服务。

（4）项目启动后，每次调用该服务，AirRPC都能透明地帮助开发者调用远程服务。

### 第4章 架构设计

AirRPC 是一个分布式框架，内部含有Zookeeper模块、Netty模块、Client模块和Server模块等，本章将采用架构图对AirRPC的整体架构设计做出说明，并使用顺序图描述用例的实现。

#### 4.1 系统架构及原理

AirRPC的总体架构类似于发布订阅模型，架构的中心是服务注册中心，服务调用端和服务提供端围绕者服务注册中心而工作。图4-1是系统的总架构。

![图4-1 系统总架构图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-4-1.jpg)

服务注册中心：用zookeeper实现，运行在一台独立的服务器上，提供服务订阅和服务注册功能。

服务提供端：对外提供远程服务的一个服务端，由服务实例对象、服务对应的接口名称、接口版本、服务所在IP地址及服务所在端口组成。

服务调用端：调用远程服务的客户端，由调用远程服务的接口名称、接口版本、接口对应的Java类名、调用方法名称、调用方法参数值以及客户端所在IP等信息组成。

服务调用端和服务提供端都依赖于服务注册中心，若后者没有响应，则项目启动失败；在初始化成功，服务调用端和服务提供端都正常运行时，服务注册中心的作用弱化，只需通告服务的变更即可，重点流程在于服务提供端和服务调用端的交互通信。

在具体的项目分层设计中，AirRPC在自身内部封装了Proxy、Filter、Group、Hession、Netty、Reflect等多个层。图4-2是系统的分层架构图。

![图4-2 系统的分层架构图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-4-2.jpg)

（1）Proxy: 该层内封装了SpringBean管理、Java动态代理等技术，在调用者调用某个接口服务时，调用了代理对象的invoke方法，在invoke方法里生成了Filter，将调用信息封装好，传给Filter。

（2）Filter: 该层的主要模型是责任链模式，责任链上有各个Handler，每个Handler都拥有着自己的filter功能，比如日志记录、调用统计和实际的远程调功能。

（3）Group: 该层只用于服务调用端。如果某服务有多个提供端，调用端就需要在多个提供端之间选择一个，这个过程叫负载均衡，选定了提供者之后，调用端就可以把一个Java对象传给Hession层。

（4）Hession: 该层会将请求的参数、结果等数据进行序列化和反序列化。比如，调用端的Hession层会将装载请求的Java对象序列化二进制流，将接收到的装载response响应的二进制流反序列化成Java对象。项目中采用了Hession序列化方式，序列化成二进制数据后再利用Netty层把二进制流发往服务提供端。

（5）Netty: 该层是负责网络通信的层，属于最底层，专注于数据传输，在本项目中，Netty层基于netty的TCP长连实现，Netty可以实现稳定的连接\[12\]。在服务调用端初始化时，它向服务注册中心查询提供端，并和所有的提供端建立TCP长连接，往后的每一次通信，比如调用端的Netty层给提供端的Netty层发请求，或反过来，提供端的Netty层给调用端的Netty层发响应，均会复用已建立好的长连接。

（6）zk: 该层独立于系统其它层，但又和其它层有联系，用于和服务注册中心交互，比如服务注册、订阅和变更推送等。服务调用端在初始化时，会利用zk层向服务注册中心订阅它需要的服务，获取到所有服务提供端的信息，如果服务提供端的数据发生了变化，服务注册中心向服务调用端推送变更信息。服务提供端在初始化时，首先要向服务注册中心发布自己的服务，服务提供端下线后，服务注册中心通过心跳机制知晓服务提供者已下线，随即将其从服务提供端列表里删除。

#### 4.2 业务用例的实现

在本项目中，用例相对比较多，在这里，只列举开发人员注册服务的用例实现。

##### 4.2.1 注册服务用例的实现

在一个开发人员编写好接口实现，同时将接口记录在AirRPC的配置文件中，项目启动后，AirRPC内部运作的逻辑如下：

（1）AirRPC读取配置文件，读取相应的信息用于初始化Server端，这些信息包括对外连接的端口、服务的名称、服务对应的类、总线程数等。

（2）AirRPC借用Spring的框架能力，拿到AirRPC对外暴露的接口，具体表现是从Spring容器里获取一个接口实例，如果没有该实例，Spring会自动初始化一个\[13\]。

（3）AirRPC初始化自己的Reflect模块、Filter模块、Hession模块。

（4）AirRPC利用zk模块在服务注册中心发布服务，包括接口的名称、接口版本、机器的IP和端口。

（5）AirRPC的Netty层在对外连接的端口上监听所有的连接请求，若是服务调用端请求建立连接，则新建TCP长连，用于和调用端建立连接。  
经过这些步骤，服务提供端准备妥当，足以担起提供者的责任。图4-3是注册服务用例的实现顺序图。

![图4-3 注册服务用例的实现顺序图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-4-3.jpg)

### 第5章 模块设计

本章主要是对AirRPC中的各个模块做了相对详细的介绍，包括用Zookeeper实现的服务注册中心模块、服务代理及责任链调用模块、HeartBeat检测模块、Netty传输模块等。

服务注册中心模块，用于和Zookeeper服务器进行交互，包括把服务添加到Zookeeper的目录下，也就是服务注册，还包括监听Zookeeper的数据变化，并将Zookeeper上的数据变化通知服务订阅者，让后者及时更新服务列表。

服务代理模块，其实现原理是Java Proxy，当用户调用某个服务接口时，服务代理模块会建立Java Proxy对象，调用AirRPC内部的逻辑，封装调用为请求，向服务提供端发起网络通信请求，最后把得到的结果还给用户。

责任链调用模块，用于封装调用为请求，在链路上设置多个Filter，分别执行不同的操作\[14\]，如接口调用情况监控、接口日志记录、发起远程网络通信传输等。

Netty传输模块，利用Netty这一拥有异步非阻塞特性的开源网络通信框架，打造最底层的传输模块，为AirRPC的高性能通信提供了必要的保障。

HeartBeat检测模块，用于监测服务调用端和服务提供端之间的连接是否可用，若多次心跳均未收到回复，则服务调用端自动将服务提供端从服务列表中删除。

#### 5.1 服务注册中心模块

服务注册中心模块，和Zookeeper服务器交互，把Zookeeper当做一个高一致性的数据库，在服务提供端，服务注册中心将服务接口的信息存放于Zookeeper上，在服务调用端，服务注册中心查询Zookeeper上的服务接口的信息，并监听服务接口信息的变化。

AirRPC使用CuratorFramework框架，用于实现和Zookeeper的通信与监听。

服务注册中心是分布式服务框架的目录服务器，它的内容路径类似于Linux目录树，比如/zk/services/serviceA/，services是/zk的子znode，表示服务注册中心所有的服务，serviceA znode表示服务serviceA的信息，该znode下面还可以有子节点，代表一台台提供该服务的机器\[15\]。

##### 5.1.1 Zookeeper数据结构设计

Zookeeper是一个高可用、高性能、强一致性的分布式框架，其内部的数据形式为目录树，可以将其当做一个提供数据存储功能的数据库。为了把服务接口的信息存放在Zookeeper上，我设计了图5-1那种的结构。

![图5-1 AirRPC的Zookeeper目录结构](https://gitee.com/yzhw/img/raw/master/img/air-rpc-5-1.jpg)

##### 5.1.2 监听Zookeeper目录

在初始化阶段，服务注册模块建立了一个CuratorFramework对象，和Zookeeper建立起连接，维持和Zookeeper的心跳，同时还建立一个listener对象，用于监听Zookeeper上的变化。  
监听到Zookeeper上有变化时，监听器判断其Event类型，如果是NodeChildrenChanged类型的变化，说明服务提供端发生了变动，再调用其余方法进行处理，比如删除缓存起来的服务示例。其时序图如图5-2。

![图5-2 监听Zookeeper目录的时序图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-5-2.jpg)

##### 5.1.3 注册服务

服务提供端在启动的时候，会把服务注册到服务注册中心，服务注册中心再把服务的具体详情刷写到Zookeeper上去，其算法流程图如图5-3所示。

![图5-3 注册服务的算法流程图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-5-3.jpg)

##### 5.1.4 服务自动上下线

服务调用端在启动阶段，首先向服务注册中心获取服务列表。具体地说，服务注册中心向Zookeeper查询某个接口对应的目录的子节点，有多少个子节点就意味着有多少个服务提供端已经上线，服务注册中心将这些子节点信息翻译为服务提供端列表，返回给服务调用端。服务调用端获得信息后，和各个服务提供者建立初始连接。

在服务调用端启动完毕后，监听服务接口目录子节点变化。如果有新的服务提供端上线，则服务注册中心会在Zookeeper的服务接口目录下新建新的临时节点，并保存服务提供端的IP端口等信息。服务调用端监听到这一点，则会更新自己的本地服务提供端缓存列表，同时新建一个和刚刚上线的服务提供端的长连接，以此实现服务自动上线。

而服务自动下线功能则是利用了Zookeeper和服务提供端之间的心跳信息。在服务提供端上线后，它会和Zookeeper维持一个心跳连接，在seesion\_timeout时间内若是没有给Zookeeper发出一个存活示意，则Zookeeper会将服务提供端建立的临时子节点删除。该子节点被删除的事件被服务调用端获知，后者再次更新自己的本地服务提供端缓存列表，并尝试断开和子节点被删除的服务提供端的长连，因而实现了服务的自动下线。

#### 5.2 HeartBeat检测模块

依赖服务注册中心模块，在服务提供端下线时，服务调用端能根据监听到的事件，判断服务提供端是否可用。但是，如果服务注册中心模块不可用，服务调用端还必须要拥有其它手段来判断服务提供端的存活情况。在服务注册中心模块不可靠的情况下，比如Zookeeper服务器崩溃的情况下，我们需要依靠额外的HeartBeat检测模块，来剔除不可用的服务提供端。

##### 5.2.1 心跳检测流程

在服务调用端启动之后，会额外开启一个新的线程，每隔5秒给所有的已建立连接的服务提供端发一个心跳请求，如果超过5次服务提供端都能返回正常响应，则重置心跳计数，如果超过5次服务提供端都没能返回响应，则认为该服务提供端不再存活，将其从服务提供端列表中删除。心跳监测流程的算法流程图如图5-4所示。

![图5-4 心跳监测流程的算法流程图](https://gitee.com/yzhw/img/raw/master/img/air-rpc-5-4.jpg)

#### 5.3 Netty传输模块

Netty传输模块分为两小模块，一个是服务调用端的client模块，另一个是服务提供端的server模块，它们都基于Netty框架。

##### 5.3.1 服务提供端server模块

在服务提供端启动时会初始化server模块。具体表现为，实例化ServerBootstrap对象，创建ByteToMessageDecoder对象用于把接收到的二进制流转换为Java对象，创建MessageToByteEncoder对象把Java对象转换为二进制流，创建业务handler用于处理接收到的请求，随即启动ServerBootstrap，监听4080端口。

若有新的服务调用端请求建立新连接，则新建线程和调用端维持长连接；若在已有的长连接里收到请求，则调用ByteToMessageDecoder、MessageToByteEncoder和业务handler等来处理请求。其时序图如图5-5所示。

![图5-5 服务提供端server模块](https://gitee.com/yzhw/img/raw/master/img/air-rpc-5-5.jpg)

当server模块收到请求，会为请求创建Callable对象，并将Callable对象放入线程池，Callable对象内部的run方法执行完毕后，会给发来请求的client机器发回响应。

### 第6章 部署与应用

#### 6.1 安装环境

AirRPC是一个应用型框架，需要安装基本的服务注册中心，服务注册中心使用了ZooKeeper，所以我们需要先安装ZooKeeper。

##### 6.1.1 安装ZooKeeper

在Linux系统中，使用以下命令安装并启动ZooKeeper。

（1）下载并解压ZooKeeper  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">wget http://mirrors.cnnic.cn/apache/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz</span><br><span class="line">tar zxvf zookeeper-3.4.8.tar.gz</span><br></pre></td></tr></tbody></table>

（2）配置ZooKeeper，这里使用官方自带配置  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">cd zookeeper-3.4.8/conf/</span><br><span class="line">cp zoo_sample.cfg zoo.cfg</span><br></pre></td></tr></tbody></table>

（3）启动ZooKeeper  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">cd ../</span><br><span class="line">sh bin/zkServer.sh start</span><br></pre></td></tr></tbody></table>

（4）经过以上步骤，ZooKeeper就安装好了  
使用以下命令即可连接ZooKeeper。  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sh zkCli.sh -server 127.0.0.1:2181</span><br></pre></td></tr></tbody></table>

##### 6.1.2 引入jar包

AirRPC主要有AirRPC-registry.jar、AirRPC-core.jar、AirRPC-transport.jar、AirPRC-spring.jar等几个包，将它们引入到你的项目中，即可开始使用AirRPC的功能。

#### 6.2 应用Demo

下面的Demo将给示范如何使用AirRPC。

（1）编写服务接口并实现  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">public</span> <span class="class"><span class="keyword">class</span> <span class="title">ILoveYouServiceImpl</span> <span class="keyword">implements</span> <span class="title">ILoveYouService</span> </span>{</span><br><span class="line">   <span class="function"><span class="keyword">public</span> String <span class="title">iLoveYou</span><span class="params">(String yourName, String yourLoverName)</span> </span>{</span><br><span class="line">        <span class="keyword">return</span> <span class="string">"this is "</span> + yourName + <span class="string">"'s love letter for his girlfriend "</span> + yourLoverName + <span class="string">" hahaha"</span>;</span><br><span class="line">   }</span><br><span class="line">}</span><br></pre></td></tr></tbody></table>

（2）在server端注册服务  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line"><span class="tag">&lt;<span class="name">bean</span> <span class="attr">id</span>=<span class="string">"iLoveYouService"</span> <span class="attr">class</span>=<span class="string">"com.iloveqyc.test.service.Impl.ILoveYouServiceImpl"</span>/&gt;</span></span><br><span class="line"><span class="tag">&lt;<span class="name">bean</span> <span class="attr">id</span>=<span class="string">"myServer"</span> <span class="attr">class</span>=<span class="string">"com.iloveqyc.spring.ServiceRegister"</span> <span class="attr">init-method</span>=<span class="string">"init"</span>&gt;</span></span><br><span class="line">    <span class="tag">&lt;<span class="name">property</span> <span class="attr">name</span>=<span class="string">"services"</span>&gt;</span></span><br><span class="line">        <span class="tag">&lt;<span class="name">map</span>&gt;</span></span><br><span class="line">            <span class="tag">&lt;<span class="name">entry</span> <span class="attr">key</span>=<span class="string">"qiuyongcheniLoveYouService1"</span> <span class="attr">value-ref</span>=<span class="string">"iLoveYouService"</span>/&gt;</span></span><br><span class="line">        <span class="tag">&lt;/<span class="name">map</span>&gt;</span></span><br><span class="line">    <span class="tag">&lt;/<span class="name">property</span>&gt;</span></span><br><span class="line"><span class="tag">&lt;/<span class="name">bean</span>&gt;</span></span><br></pre></td></tr></tbody></table>

（3）在clinet端配置所依赖服务，并在项目中以bean形式调用服务  

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line">   <span class="tag">&lt;<span class="name">bean</span> <span class="attr">id</span>=<span class="string">"iLoveYouService"</span> <span class="attr">class</span>=<span class="string">"com.iloveqyc.spring.ServiceProxy"</span> <span class="attr">init-method</span>=<span class="string">"init"</span>&gt;</span></span><br><span class="line">       <span class="tag">&lt;<span class="name">property</span> <span class="attr">name</span>=<span class="string">"iface"</span> <span class="attr">value</span>=<span class="string">"com.iloveqyc.sample.api.ILoveYouService"</span>/&gt;</span></span><br><span class="line">       <span class="tag">&lt;<span class="name">property</span> <span class="attr">name</span>=<span class="string">"serviceName"</span> <span class="attr">value</span>=<span class="string">"qiuyongcheniLoveYouService1"</span>/&gt;</span></span><br><span class="line"><span class="tag">&lt;/<span class="name">bean</span>&gt;</span></span><br></pre></td></tr></tbody></table>

### 第7章 测试与分析

#### 7.1 测试准备

本次测试中，客户端部署在笔者的Mac Pro上，服务端则分别部署在笔者个人的阿里云上和Mac Pro上。

##### 7.1.1 测试环境

本次测试使用了Mac Pro和阿里云虚拟机作为测试机器。  
Map Pro的配置如下：  
处理器：2.7 GHz Intel Core i5 4核  
内存：8 GB 1867 MHz DDR3  
阿里云虚拟机的配置如下：  
处理器：Intel Xeon E5-2682 v4 1核  
内存：1GB DDR4

##### 7.1.2 测试用例

（1）服务端和客户端部署在同一台Mac Pro上，客户端向服务端发送10万次大小为10字节大小的请求，服务端传回大小为10字节大小的响应。

（2）服务端和客户端部署在同一台Mac Pro上，客户端向服务端发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

（3）服务端和客户端部署在同一台Mac Pro上，客户端向服务端发送10万次大小为1000字节大小的请求，服务端传回大小为100字节大小的响应。

（4）服务端和客户端部署在同一台Mac Pro上，客户端并发度为100，向服务端发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

（5）服务端和客户端部署在同一台Mac Pro上，客户端并发度为10000，向服务端发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

（6）服务端部署在阿里云虚拟机上，客户端部署在Mac Pro上，客户端向服务端发送10万次大小为10字节大小的请求，服务端传回大小为10字节大小的响应。

（7）服务端部署在阿里云虚拟机上，客户端部署在Mac Pro上，客户端向服务端发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

（8）服务端分别部署在阿里云虚拟机上和Mac Pro上，客户端部署在Mac Pro上，客户端向服务端发送10万次大小为10字节大小的请求，服务端传回大小为10字节大小的响应。

（9）服务端分别部署在阿里云虚拟机上和Mac Pro上，客户端部署在Mac Pro上，客户端向服务端发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

（10）服务端分别部署在阿里云虚拟机上和Mac Pro上，客户端部署在Mac Pro上，客户端向服务端发起10个并发，各自发送10万次大小为100字节大小的请求，服务端传回大小为100字节大小的响应。

7.2 测试结果

下表是根据测试用例得到的测试结果。

| 测试用例 | 请求大小 | 并发度 | 服务端      | 平均响应时间 |
| -------- | -------- | ------ | ----------- | ------------ |
| 1        | 10字节   | 1      | 本地        | 0.322ms      |
| 2        | 100字节  | 1      | 本地        | 0.348ms      |
| 3        | 1000字节 | 1      | 本地        | 0.369ms      |
| 4        | 100字节  | 100    | 本地        | 0.350ms      |
| 5        | 100字节  | 10000  | 本地        | 1000+ms      |
| 6        | 10字节   | 1      | 阿里云      | 52.142ms     |
| 7        | 100字节  | 1      | 阿里云      | 54.400ms     |
| 8        | 10字节   | 1      | 本地+阿里云 | 27.318ms     |
| 9        | 100字节  | 1      | 本地+阿里云 | 29.708ms     |
| 10       | 100字节  | 100    | 本地+阿里云 | 30.102ms     |

#### 7.3 结果分析

在本次测试中，测试的变量有请求大小、并发度、服务提供端的位置3个。

从测试用例1、2和3的对比中，**我们可以发现请求大小越大，则平均响应时间越高，这是合理的，请求越大，需求更多时间去序列化和反序列化请求**。同时，我们能看出AirRPC的响应耗时在0.3ms左右，如果不考虑网络延时，AirRPC自身耗时很少，可以投入使用。

从测试用例2、3和4的对比中，我们可以看到，在一定范围内，并发度变大，并不会使得响应时间严重变长，但如果并发度过大，则响应时间基本上为正无穷。这也是合理的，**如果并发度过大，线程数量超过了机器所能承载的最大值，服务器大量资源用于管理线程**，则服务器将进入假死状态，基本上不会给请求者返回响应，平均响应时间渐渐趋向正无穷。

从测试用例1和5、2和6的对比中，我们发现，当服务端部署在阿里云时，请求必须跨越公网，**网络延迟成了平均响应时间的最大占比**。

从测试用例1、5和7的对比中，我们可以看到，当有一台服务端部署在阿里云，一台服务端部署在本地，大量的请求随机发往本地和阿里云的服务端，平均响应时间为本地平均响应时间和阿里云平均响应时间的平均值。

#### 7.4 测试结论

本次测试中，AirRPC表现出了优异的响应时间性能，同时也测出了AirRPC的缺点，即并发度不能无限大。另外，我们还看到了网络延时对分布式服务调用框架的影响，网络延时越大，平均响应时间也越大。

### 第8章 总结与展望

#### 8.1 总结

在AirRPC这个项目中，我从无到有，设计并实现了一个轻量级的Java 远程调用框架。在这个过程中，我自学了许多以前未曾掌握的新技术，包括netty、zookeeper等，既开阔了视野，也强化了自己的动手能力，把在实习过程中学习到的新知识运用到项目的编码和论文的编写中来，自身能力获得较大提升。

在AirRPC中，它的优点是足够简单和非常的易用。开发人员只需配置部署ZooKeeper服务器，并在项目中加入两个xml配置文件即可使用AirRPC的功能，甚至都不需要帮助文档。AirRPC的代码量很少，这有利于开发者在遇到问题时及时查找问题、发现问题和解决问题。

另一方面，AirRPC的目标不是一个高大全式的RPC框架，它仅提供了最基本的远程调用和服务更新功能，不够健壮、不支持异步调用、缺少可配置项等。AirRPC仅仅用一台ZooKeeper来保证服务的更新功能，若是ZooKeeper机器崩溃，则服务发布订阅更新等功能都得不到正常运行的保证。

但是，正是因为很多功能都被削减，AirRPC才能保持得如此简洁，堪比超轻量级，剔除了小团队不需要的功能，让AirRPC成为一个纯粹的RPC框架。

#### 8.2 展望

AirRPC虽然是个轻量级的框架，但它内部的代码仍旧称不上clean，有许多可以改善的地方。它对代码仍旧有些许的侵入性，这点也是可以优化的。  
另外，本项目没有可视化管理后台，没有支持服务的可视化，可以考虑打造服务监控中心。

#### 参考文献

\[1\] Dubbo官方技术网站\[EB/OL\]. [http://dubbo.io/,\[2017-04-10\]](http://dubbo.io/,[2017-04-10]).

\[2\] Motan官方技术网站\[EB/OL\].[https://github.com/weibocom/motan/wiki/zh\_overview](https://github.com/weibocom/motan/wiki/zh_overview), \[2017-04-10\].

\[3\] Pigeon 官方技术网站\[EB/OL\]. [https://github.com/wu-xiang/pigeon/blob/master/USER\_GUIDE.md](https://github.com/wu-xiang/pigeon/blob/master/USER_GUIDE.md), \[2017-04-10\].

\[4\] David Gourley … \[等. HTTP权威指南\[M\]. 人民邮电出版社, 2012.

\[5\] 姚吉, 谢荣传. SOAP中的远程过程调用\[J\]. 计算机技术与发展, 2001, 11(5):48-50.

\[6\] 冯新扬, 沈建京. REST和RPC:两种Web服务架构风格比较分析\[J\]. 小型微型计算机系统, 2010, 31(7):1393-1395.

\[7\] 陈更力, 胡燕, 张青,等. 基于Java RMI的RPC进一步研究\[J\]. 长江大学学报(自科版), 2005, 2(7):251-252.

\[8\] 丁兆青, 董传良. 基于SOA的分布式应用集成研究\[J\]. 计算机工程, 2007, 33(10):246-248.

\[9\] 裴树军. 面向服务的信息系统关键技术研究\[D\]. 哈尔滨理工大学, 2012.

\[10\] 汪锦岭. 面向Internet的发布/订阅系统的关键技术研究\[D\]. 中国科学院软件研究所, 2005.

\[11\] 刘芬, 王芳, 田昊. 基于Zookeeper的分布式锁服务及性能优化\[J\]. 计算机研究与发展, 2014(S1):229-234.

\[12\] 李林锋. Netty权威指南\[M\]. 电子工业出版社, 2015.

\[13\] Spring官方技术网站\[EB/OL\]. [https://spring.io/](https://spring.io/), \[2017-04-10\].

\[14\] 齐鑫. 责任链设计模式的改进\[J\]. 计算机工程, 2010, 36(10):56-57.

\[15\] ZooKeeper官方技术网站\[EB/OL\]. [https://zookeeper.apache.org/,\[2017-04-10\]](https://zookeeper.apache.org/,[2017-04-10]).

> 如果觉得还有帮助的话，你的关注和转发是对我最大的支持，O(∩_∩)O:
