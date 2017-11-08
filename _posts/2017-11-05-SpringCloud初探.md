---
layout:     post
title:      SpringCloud初探
subtitle:   SpringCloud全家桶
date:       2017-11-05
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
tags:
    - SpringCloud Eureka Hystrix Zuul Feign Ribbon Turbine
---


## 什么是SpringCloud？

Spring Cloud是一系列框架的有序集合，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架（服务发现注册、配置中心、消息总线、负载均衡、断路器、智能路由、数据监控、分布式会话和集群状态管理等）组合起来，通过Spring Boot进行再封装屏蔽掉了复杂的配置，巧妙地简化了分布式系统基础设施的开发，最终给我们一套简单易懂、易部署和易维护的分布式系统开发利器，做到一键启动和部署。

Spring Cloud包含了多个子项目（针对分布式系统中涉及的多个不同开源产品），比如：Spring Cloud Config、Spring Cloud Netflix、Spring Cloud CloudFoundry、Spring Cloud AWS、Spring Cloud Security、Spring Cloud Commons、Spring Cloud Zookeeper、Spring Cloud CLI等项目。

总之，Spring Cloud从技术架构上降低了对大型系统构建的要求，使我们以非常低的成本搭建一套高效、分布式、容错的平台。

## 微服务架构

简单的说，微服务架构就是将一个完整的应用从数据存储开始垂直拆分成多个不同的服务，每个服务都能独立部署、独立维护、独立扩展、独立访问（或者有独立的数据库）的服务单元，服务与服务间通过诸如RESTful API或Feign Service的方式互相调用。

关于微服务架构相关的产品社区也变得越来越活跃（比如：Netflix、Dubbo、Kubernetes），Spring Cloud Netflix与各种Netflix OSS组件集成，组成微服务的核心，主要有Eureka, Hystrix, Zuul, Archaius… ；

Dubbo是Alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合）。从服务模型的角度来看，Dubbo采用的是一种非常简单的模型，要么是提供方提供服务，要么是消费方消费服务，所以基于这一点可以抽象出服务提供方（Provider）和服务消费方（Consumer）两个角色，并且方便与Spring集成。它是国内公司用的比较多的高性能的分布式RPC框架。

> Dubbo |ˈdʌbəʊ| is a high-performance, java based RPC framework open-sourced by Alibaba. As in many RPC systems, dubbo is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a dubbo server to handle client calls. On the client side, the client has a stub that provides the same methods as the server.

Kubernetes是一个开源系统，用来自动部署、缩放和管理容器应用。它可以使用多语言并且提供原语服务开通、运行、缩放和分布式系统管理。它提供的服务，例如配置管理、服务发现、负载均衡、指标收集和日志聚集，都通过各种各样的语言来实现。


### Spring Cloud Netflix

#### Netflix Eureka

服务中心，服务注册与发现，一个基于 REST 的服务，用于定位服务，以实现服务发现和故障转移。它提供了完整的Service Registry和Service Discovery实现，也是Spring Cloud体系中最重要最核心的组件之一。

在服务调用之间增加一层，实现服务间的调用解耦，管理着服务提供者的生命周期，为消费者提供可用的服务，任何一个服务提供者的改动，不会牵连此服务的消费者跟着重启。服务之间通过服务中心来获取服务，我们不需要关注调用的服务IP地址、由几台服务器组成集群向外提供服务，每次直接去服务中心获取可以使用的服务去调用既可。因此，我们不再关心服务端是否为集群、服务端IP或端口变化，我们只需要将Server端的服务注册到Eureka中，Client端去Eureka中去获取配置中心Server端的服务既可。

由于各种服务都注册到了服务中心，那么我们就会想到：几台服务提供相同服务怎么来做均衡负载？怎么监控服务器调用成功率？什么是熔断？如何移除服务列表中的故障点？如何（监控服务调用时间）对不同的服务器设置不同的权重等等？ 

#### Netflix Feign

各个微服务都是以HTTP接口的形式暴露自身服务的，因此在调用远程服务时就必须使用HTTP客户端。但是，用起来最方便、最优雅的还是要属Feign了。Feign是一种声明式、模板化的HTTP客户端。在Spring Cloud中使用Feign, 我们可以做到使用HTTP请求远程服务时能与调用本地方法一样的编码体验，完全感知不到这是远程方法，更感知不到这是个HTTP请求，并且与Eureka结合使用，实现透明化的负载均衡。Feign 与 Hystrix结合使用，保护系统的可用性。

> Feign是一个声明式Web Service客户端。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

#### Netflix Hystrix

熔断器，容错管理工具，旨在通过熔断机制控制服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。如第三方服务超时，基础服务的故障可能会导致级联故障，进而造成整个系统不可用，这种现象被称为服务雪崩效应。

熔断器可以实现快速失败，当Hystrix Command请求后端服务失败数量超过一定比例(默认50%)， 断路器会切换到开路状态(Open)。这时所有请求会直接失败而不会发送到后端服务。断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN)。这时会判断下一次请求的返回情况，如果请求成功，断路器切回闭路状态(CLOSED)，否则重新切换到开路状态(OPEN)。Hystrix的断路器一旦发现后端服务不可用，断路器会直接切断请求链，避免发送大量无效请求影响系统吞吐量，并且断路器有自我检测并恢复的能力。

#### Netflix Zuul

Eureka用于服务的注册于发现，Feign支持服务的调用以及均衡负载，Hystrix处理服务的熔断防止故障扩散，Spring Cloud Config服务集群配置中心，似乎一个微服务框架已经完成了。

但是，外部的应用如何来访问内部各种各样的微服务呢？在微服务架构中，后端服务往往不直接开放给调用端，而是通过一个API网关向外统一提供服务，另外也简化客户端的调用。API网关会根据请求的url路由到真正的服务上。当添加API网关后，在外部第三方客户端和服务提供方之间就可以对调用方通信进行权限控制、访问频率限制等。满足条件后将请求均衡分发给后台服务端、对不同的客户端（PC Web、iOS、Android）提供不同的数据量（减少不必要的数据传输）。

Zuul 正是提供动态路由，监控，弹性，安全等边缘服务的框架。
> 在Spring Cloud体系中， Spring Cloud Zuul就是提供负载均衡、反向代理、权限认证的一个API gateway。

<!--#### Netflix Archaius

配置管理API，包含一系列配置管理API，提供动态类型化属性、线程安全配置操作、轮询框架、回调机制等功能。可以实现动态获取配置， 原理是每隔60s（默认，可配置）从配置源读取一次内容，这样修改了配置文件后不需要重启服务就可以使修改后的内容生效，前提使用archaius的API来读取。

### Spring Cloud Config

俗称的配置中心，配置管理工具包，让你可以把配置放到远程服务器，集中化管理集群配置，目前支持本地存储、Git以及Subversion。方便以后统一管理、升级装备。

### Spring Cloud Bus

事件、消息总线，用于在集群（例如，配置变化事件）中传播状态变化，可与Spring Cloud Config联合实现热部署。

### Spring Cloud Consul

Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源. Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对.

Spring Cloud Consul 封装了Consul操作，consul是一个服务发现与配置工具，与Docker容器可以无缝集成。

### Spring Cloud Sleuth

日志收集工具包，封装了Dapper和log-based追踪以及Zipkin和HTrace操作，为SpringCloud应用实现了一种分布式追踪解决方案。

### Spring Cloud Stream

Spring Cloud Stream是创建消息驱动微服务应用的框架。Spring Cloud Stream是基于Spring Boot创建，用来建立单独的／工业级spring应用，使用spring integration提供与消息代理之间的连接。数据流操作开发包，封装了与Redis,Rabbit、Kafka等发送接收消息。

一个业务会牵扯到多个任务，任务之间是通过事件触发的，这就是Spring Cloud stream要干的事了。-->

***还有更多的成员，但是了解上面这些，基本上就可以构建一套大型分布式系统了。***

![Spring Cloud](/img/spring-cloud.jpg)

## 和Spring Boot 是什么关系

Spring Boot 是 Spring 的一套快速配置脚手架，简化了很多配置（使用过Spring与其他框架集成的同学一定有体会），减少了依赖包冲突的机会哈，我们现在可以基于Spring Boot 快速开发单个微服务，Spring Cloud是一个基于Spring Boot实现的云应用开发、服务治理套件；Spring Boot专注于快速开发单个个体服务，而Spring Cloud是关注全局的服务治理框架；Spring Boot使用了默认大于配置的理念（提供各种starter），Spring Cloud很大的一部分是基于Spring Boot来实现。

Spring Boot可以离开Spring Cloud独立使用开发项目，但是Spring Cloud离不开Spring Boot，属于依赖的关系。

> Spring -> Spring Booot > Spring Cloud 这样的关系。

## Spring Cloud的优势

微服务的框架那么多比如：Dubbo、Kubernetes，为什么就要使用Spring Cloud的呢？

* 产出于Spring大家族，可以保证后续的更新、完善。
* 有Spring Boot 简化了很多配置，可快速搭建单个服务。
* 微服务治理考虑的很全面，方便开发开箱即用。
* Spring Cloud 活跃度很高，教程很丰富，遇到问题很容易找到解决方案。
* 轻轻松松几行代码就完成了熔断、均衡负责、服务中心的各种平台功能。

## 总结
使用Spring Cloud一站式解决方案能从容应对业务发展的同时大大减少开发成本，很轻松的搞出来套分布式系统基础设施。



