---
layout:     post
title:      SpringCloud智能网关入门介绍
subtitle:   Zuul入门介绍
date:       2017-12-13
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud 
    - Zuul 
---

### 问题

通过之前几篇Spring Cloud中几个核心组件的介绍：Eureka用于服务的注册与发现，Ribbon或Feign支持服务的调用以及均衡负载，Hystrix处理服务的熔断防止故障扩散，似乎一个微服务框架已经完成了。

但是，为了保证对外服务的安全性，我们需要实现对服务访问的权限控制，如果这些功能实现在微服务中，导致在工作中除了要考虑实际的业务逻辑之外，还需要额外为每个微服务增加对外接口访问的控制处理，或增加一个代理调用来实现权限控制。

服务网关就是解决这类问题的方案。

### 服务网关起到的作用

为了解决上面这些问题，我们需要将权限控制从我们的微服务中抽离出去，而最适合这些逻辑的地方就是处于对外访问的最前端，并且需要一个更强大一些的均衡负载器，它就是服务网关。

服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制、调用频率限制、统一异常处理、合并请求等功能。Spring Cloud Netflix中的Zuul就是这样的一个角色，为微服务架构提供了前置保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

### 构建服务网关

* 新建Spring Boot项目`springcloud-api-gateway`，添加依赖

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
    <artifactId>spring-cloud-starter-zuul</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
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

* 创建应用主类，并使用`@EnableZuulProxy`注解开启Zuul的功能。

```
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.cloud.client.SpringCloudApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

/**
 * 网关服务启动主类
 * @author Yezhiwei
 * @date 17/12/4
 */
@SpringCloudApplication
@EnableZuulProxy
public class Server {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Server.class).web(true).run(args);
    }
}
```

* 在`application.properties`文件中增加服务名、端口号、eureka注册中心的地址

```
spring.application.name=springcloud-api-gateway

#spring.profiles.active=dev

logging.path=./logs
server.port=9710

spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true

spring.mvc.throw-exception-if-no-handler-found=true


eureka.client.serviceUrl.defaultZone=http://peer1:9600/eureka/,http://peer2:9600/eureka/,http://peer3:9600/eureka/
```

* 到这，一个默认的服务网关就构建完毕了。由于Spring Cloud Zuul在整合了Eureka之后，具备默认的服务路由功能，即：当我们这里构建的springcloud-api-gateway应用启动并注册到eureka之后，服务网关会发现上面我们启动的两个服务springcloud-addition-service和springcloud-consumer-hystrix，这时候Zuul就会创建两个路由规则。每个路由规则都包含两部分，一部分是外部请求的匹配规则，另一部分是路由的服务ID。

>转发到`springcloud-addition-service`服务的请求规则为：`/springcloud-addition-service/**`

>转发到`springcloud-consumer-hystrix`服务的请求规则为：`/springcloud-consumer-hystrix/**`


* 访问试一下

比如访问`http://localhost:9710/springcloud-addition-service/add?a=1&b=1` 该请求将最终被路由到`springcloud-addition-service`的`/add`接口上，输出结果为：

![zuul](/img/zuul-0.png)

`springcloud-addition-service`服务器端日志如下：

```
2017-12-12 13:54:39.066  INFO 12850 --- [http-nio-9601-exec-1] c.g.s.controller.AdditionController      : add 1 + 1
```




