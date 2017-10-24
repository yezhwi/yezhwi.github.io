---
layout:     post
title:      Spring Boot 工程结构
subtitle:   项目定义及包结构
date:       2017-10-24
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
tags:
    - Spring Boot
---


## Spring Boot项目结构

Spring Boot框架并没有对工程结构有特别的要求，但是按照最佳实践的工程结构可以帮助我们少踩很多坑，尤其是Spring包扫描机制，可以免去不少特殊的配置工作。良好的工程结构划分可以使项目更清晰、明确，减少不必要的冲突，提高代码的统一性。

SpringBoot提供了很多基础设施，在创建生产中的独立程序上非常简便、只需要一些简便的配置就能运行起来。

* 创建独立的Spring Application
* 能够使用内嵌的Tomcat, Jetty or Undertow，不需要部署war
* 提供starter pom来简化maven配置
* 自动配置Spring
* 提供一些生产环境的特性，比如metrics, health checks and externalized configuration
* 绝对没有代码生成和XML配置要求
* 支持多环境配置

## 提供HTTP协议的服务

如果仅提供http协议，并且不考虑调用方的感受（调用方自己将json转成对象），可以使用下面的示例

* root package结构：com.example.myproject
* 应用主类Application.java置于root package下，可以帮助程序减少手工配置来加载Spring的内容
* 实体（Entity）置于com.example.myproject.domain(或model)包下
* 数据访问层（Repository）置于com.example.myproject.repository包下
* 逻辑层（Service）置于com.example.myproject.service包下
* Web层（web）置于com.example.myproject.web包下

```
com
  +- example
    +- myproject
      +- Application.java
      |
      +- domain
      |  +- Customer.java
      |
      +- repository
      |  +- CustomerRepository.java
      |
      +- service
      |  +- CustomerService.java
      |
      +- web
      |  +- CustomerController.java
      |
```

上面的方式还需要约定HTTP协议对资源操作的RESTful API

***


## 其他的RPC协议，多个微服务之间

* 整个项目可以拆分成多个微服务，每个微服务工程又可以分为core和service两个工程，如：

```
user-core
user-service
```

为什么要分为core、service呢？

从两者的作用上来看，core主要是model、接口、常量，被service依赖，被使用方依赖，同时考虑到了使用方的感觉哈；service主要是对接口的实现，以及对外提供多种RPC协议的服务。

### core推荐的工程结构

代码层的结构

根目录：com.gemantic.user

实体类(domain)置于com.gemantic.user.domain，主要是与数据库的对应关系

1. 实体类(domain)置于com.gemantic.user.domain
3. 数据访问层(Dao)置于com.gemantic.user.repository
4. 数据服务层(Service)置于com.gemantic.user.service
5. 常量接口类(consist)置于com.gemantic.user.consist
6. 数据传输类(vo)置于com.gemantic.user.vo

### service推荐的工程结构

代码层的结构

根目录：com.gemantic.user

1. 工程启动类(ApplicationServer.java)置于com.gemantic.user包下
2. 数据服务的实现接口(serviceImpl)至于com.gemantic.user.service.impl
3. 前端控制器(Controller)置于com.gemantic.user.controller
4. 工具类(utils)置于com.gemantic.user.utils
5. 配置信息类(config)置于com.gemantic.user.config

***

之后再总结一下，RESTful API、微服务带来的问题

