---
layout:     post
title:      SpringCloud基础服务注册中心Eureka
subtitle:   基础服务注册中心Eureka
date:       2017-11-06
author:     Yezhiwei
category:   springcloud
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud
    - Eureka 
    - Hystrix 
    - Zuul 
    - Feign 
    - Ribbon 
    - Turbine
---


## 如何使用Spring Cloud来实现服务治理?

由于Spring Cloud为服务治理做了一层抽象接口，所以在Spring Cloud应用中可以支持多种不同的服务治理框架，比如：Netflix Eureka、Consul、Zookeeper。在Spring Cloud服务治理抽象层的作用下，我们可以无缝地切换服务治理实现，并且不影响任何其他的服务注册、服务发现、服务调用等逻辑。(有过面向接口编程的同学应该能体会到这一层抽象接口带来的好处。)

## Eureka

Spring Cloud Eureka是Spring Cloud Netflix项目下的服务治理模块。而Spring Cloud Netflix项目是Spring Cloud的子项目之一，主要内容是对Netflix公司一系列开源产品的包装，它为Spring Boot应用提供了自配置的Netflix OSS整合。通过一些简单的注解，我们就可以快速的在应用中配置一下常用模块并构建庞大的分布式系统。它主要提供的模块包括：服务发现（Eureka），断路器（Hystrix），智能路由（Zuul），客户端负载均衡（Ribbon）等。

服务中心，服务注册与发现，一个基于 REST 的服务，用于定位服务，以实现服务发现和故障转移。它提供了完整的Service Registry和Service Discovery实现，也是Spring Cloud体系中最重要最核心的组件之一。

在服务调用之间增加一层，实现服务间的调用解耦，管理着服务提供者的生命周期，为消费者提供可用的服务，任何一个服务提供者的改动，不会牵连此服务的消费者跟着重启。服务之间通过服务中心来获取服务，我们不需要关注调用的服务IP地址、由几台服务器组成集群向外提供服务，每次直接去服务中心获取可以使用的服务去调用既可。因此，我们不再关心服务端是否为集群、服务端IP或端口变化，我们只需要将Server端的服务注册到Eureka中，Client端去Eureka中去获取配置中心Server端的服务既可。

## 创建“服务注册中心”

* 创建一个maven项目，项目名为itsm-eureka，然后再pom.xml中引入依赖，注意Spring Cloud项目的版本（Dalston.SR1）

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.4.RELEASE</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>
<dependencyManagement>
    <dependencies>
        <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-dependencies</artifactId>
           <version>Dalston.SR1</version>
           <type>pom</type>
           <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

* 通过`@EnableEurekaServer`注解启动一个服务注册中心提供给其他应用进行对话。这一步非常的简单，只需要在一个普通的Spring Boot应用中添加这个注解就能开启此功能。

```
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * 服务注册与发现
 */
@EnableEurekaServer
@SpringBootApplication
public class Server {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Server.class).web(true).run(args);
    }
}
```

* 在默认设置下，该服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为，只需要在`application.properties`配置文件中增加如下信息

```
# 服务的名字
spring.application.name=itsm-enreka

#spring.profiles.active=dev

logging.path=./logs

# 服务中心端口号
server.port=9900

spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true

spring.mvc.throw-exception-if-no-handler-found=true


eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

* 部署到服务器，然后运行此服务，访问 http://ip:9900/，可以看到下面的页面，其中没有发现任何服务。现在就已经搭建好单机版的服务注册中心了。

![Eureka](/img/spring-cloud-starter-dalston-1-1.png)

---

## 创建“服务提供者”，并注册到服务注册中心

我们创建提供服务的提供者，并向服务注册中心注册自己。

* 首先，创建一个基本的Spring Boot的 Maven应用。命名为eureka-service-provider，在pom.xml中，加入如下配置：

```
<parent> 
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.4.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
<dependencyManagement>
    <dependencies>
        <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-dependencies</artifactId>
           <version>Dalston.SR1</version>
           <type>pom</type>
           <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

* 其次，实现`/entrusQueue`请求处理接口，通过`DiscoveryClient`对象，在日志中打印出服务实例的相关内容。

```
@Slf4j
@RestController
@Api(description = "模拟交易委托队列接口")
public class EntrustQueueController {

    @Autowired
    private DiscoveryClient client;

    @Autowired
    private EntrustQueueService entrustQueueService;

    @ApiOperation(value = "新增加委托请求")
    @PostMapping("/entrusQueue")
    public ResponseEntity<ResultData> addEntrust(@RequestBody EntrustQueue entrustQueue) {

        log.info(entrustQueue.toString());

        ServiceInstance instance = client.getLocalServiceInstance();

        log.info("host:" + instance.getHost() + ", service_id:" + instance.getServiceId());

        ResultData resultData = ResultData.builder().message(Message.builder().code(0).message(Message.SUCCESS_MESSAGE).build()).build();
        return  ResponseEntity.ok(resultData);
    }
}
```

* 然后在应用主类中通过加上`@EnableDiscoveryClient`注解，该注解能激活Eureka中的`DiscoveryClient`实现，这样才能实现Controller中对服务信息的输出。

```
@SpringBootApplication
@EnableDiscoveryClient
public class Server {
    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```

* 最后指定服务注册中心的位置

```
# serviceId，后续在调用的时候只需要使用该名称就可以进行服务的访问
spring.application.name=eureka-service-provider

# 本服务的端口号
server.port=9901

# 服务注册中心的配置内容，指定服务注册中心的位置
eureka.client.serviceUrl.defaultZone=http://master:9900/eureka/
```

* 配置Hosts

```
192.168.1.11 master
```

---

## 测试服务

1. 启动`eureka-service-provider`工程后，再次访问 http://ip:9900/，可以看到我们定义的服务被注册成功了。
2. http://ip:9901/entrusQueue，可以看到获取服务注册中心的日志信息。




