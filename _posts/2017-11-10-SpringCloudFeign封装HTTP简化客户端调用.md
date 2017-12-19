---
layout:     post
title:      Feign封装RESTfulAPI简化客户端调用
subtitle:   Feign对服务的声明式定义和调用
date:       2017-11-10
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud
    - Feign
---


## Spring Cloud Feign

在此之前，各个微服务都是以HTTP接口的形式暴露自身服务的，因此在调用远程服务时就必须使用HTTP客户端，使用起来很不方便，需要了解URL，有时还需要拼装真正请求的URL。有没有一种用起来更方便、更优雅的方式吗？答案是肯定的，Spring Cloud想到了这些————Feign。Spring Cloud Feign是一套基于Netflix Feign实现的声明式服务调用客户端。它使得编写Web服务客户端变得更加简单。在Spring Cloud中使用Feign, 我们只需要通过创建接口并用注解来配置既可完成对Web服务接口的绑定，可以做到使用HTTP请求远程服务时能与调用本地方法一样的编码体验，感知不到这是个HTTP请求，同时还整合了Ribbon和Eureka来提供均衡负载的HTTP客户端实现，实现透明化的负载均衡。

> Feign是一个声明式Web Service客户端。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

## 举一个栗子

下面，通过一个例子来展现Feign如何方便的声明对`eureka-service-provider`服务的定义和调用。

* 为之前的服务提供者 `eureka-service-provider` 代码增加 `Feign` 支持，在pom.xml文件中添加依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

* 创建一个Feign的客户端接口定义。使用`@FeignClient`注解来指定这个接口所要调用的服务名称，接口中定义的各个函数使用Spring MVC的注解就可以来绑定服务提供方的REST接口，一般情况，我会把这些`Client` 放到一个core项目里，此项目就是提供给客户端调用，比如下面的代码放到`eureka-service-core`项目中：

```
import com.gemantic.commons.ResultData;
import com.gemantic.simulate.model.EntrustQueue;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;


@FeignClient("eureka-service-provider")
public interface EntrustQueueClient {

    @PostMapping("/entrusQueue")
    ResponseEntity<ResultData> addEntrust(@RequestBody EntrustQueue entrustQueue);
}
```

* 将上面定义的接口引入客户端，在客户端项目中的pom.xml文件中增加依赖

```
<!-- Fegin 自定义Client接口依赖 -->
<dependency>
    <groupId>com.xxx.demo</groupId>
    <artifactId>eureka-service-core</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>

<!-- Fegin 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

* 最后，在 `main`方法上增加注解`@EnableFeignClients`

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.feign.EnableFeignClients;

/**
 * Hello world!
 */

@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class Server {
    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```

* 客户端调用，调用本地方法一样的编码体验！通过对比发现是不是更方便、更优雅哈～

```
@Slf4j
@RestController
@Api(description = "模拟交易委托队列接口")
public class EntrustQueueController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private EntrustQueueClient entrustQueueClient;



    @ApiOperation(value = "新增加委托请求")
    @PostMapping("/entrusQueue")
    public ResponseEntity<ResultData> addEntrust(@RequestBody EntrustQueue entrustQueue) {

        log.info("entrusQueue" + entrustQueue.toString());

        ResultData resultData = restTemplate.postForObject("http://simulate-trade-service/entrusQueue", entrustQueue, ResultData.class);
        return  ResponseEntity.ok(resultData);
    }

    @ApiOperation(value = "新增加委托请求Feign")
    @PostMapping("/entrusQueueFeign")
    public ResponseEntity<ResultData> addEntrust2(@RequestBody EntrustQueue entrustQueue) {

        log.info("entrusQueueFeign:" + entrustQueue.toString());

        ResponseEntity<ResultData> resultData = entrustQueueClient.addEntrust(entrustQueue);
        return  resultData;
    }
}
```

在完成了上面你的代码编写之后，就可以把eureka服务注册中心、服务提供者（多启动几个，体验一下负载均衡）、服务消费者启动进行测试了。由于Feign是基于Ribbon实现的，所以它自带了客户端负载均衡功能。

<!--也可以通过Ribbon的IRule进行策略扩展。另外，Feign还整合的Hystrix来实现服务的容错保护，在Dalston版本中，Feign的Hystrix默认是关闭的。-->









