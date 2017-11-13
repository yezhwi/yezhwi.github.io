---
layout:     post
title:      SpringCloud-Hystrix-让你的系统稳一点儿
subtitle:   服务容错保护（熔断器、服务降级、依赖隔离）
date:       2017-11-12
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud 
    - Hystrix
---


## 背景

在微服务架构中，我们将系统拆分成了一个个的服务单元，各单元应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成任务积压，线程资源无法释放，最终导致自身服务的瘫痪，进一步甚至出现故障的蔓延最终导致整个系统的瘫痪，导致服务雪崩效应。为了解决这样的问题，产生了断路器等一系列的服务保护机制。

## Hystrix——让你的系统稳一点儿

Spring Cloud Hystrix基于Netflix的开源框架 Hystrix实现的，该框架目标在于通过控制那些访问远程系统、服 务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大功能。

熔断器可以实现快速失败，当Hystrix Command请求后端服务失败数量超过一定比例(默认50%)， 断路器会切换到开路状态(OPEN)。这时所有请求会直接失败而不会发送到后端服务。断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN)。这时会判断下一次请求的返回情况，如果请求成功，断路器切回闭路状态(CLOSED)，否则重新切换到开路状态(OPEN)。Hystrix的断路器一旦发现后端服务不可用，断路器会直接切断请求链，避免发送大量无效请求影响系统吞吐量，并且断路器有自我检测并恢复的能力。

## 动手一试

* 复制一下之前实现的一个服务消费者：eureka-service-consumer，命名为eureka-service-consumer-hystrix，在pom.xml文件中增加 `Hystrix` 依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

* 在应用主类中使用@EnableCircuitBreaker或@EnableHystrix注解开启Hystrix的使用

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.feign.EnableFeignClients;

/**
 * Hello world!
 */

@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
@EnableCircuitBreaker
public class Server {
    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```

* 修改服务消费方式，新增ConsumerService类，然后逻辑调用。最后，在为具体执行逻辑的函数上增加@HystrixCommand注解来指定服务降级方法

```
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * Created by Yezhiwei on 17/11/13.
 */

@Slf4j
@Component
public class ConsumerService {

//    @Autowired
//    RestTemplate restTemplate;

    @Autowired
    private EntrustQueueClient entrustQueueClient;


    @HystrixCommand(fallbackMethod = "fallback")
    public ResultData addEntrust(EntrustQueue entrustQueue) {
//        return restTemplate.postForObject("http://simulate-trade-service/entrusQueue", entrustQueue, ResultData.class);
        return entrustQueueClient.addEntrust(entrustQueue).getBody();
    }


    public ResultData fallback(EntrustQueue entrustQueue) {
        ResultData resultData = new ResultData();
        Message message = Message.builder().code(10000).message("xxxx").build();
        resultData.setMessage(message);
        return resultData;
    }
}
```

***代码中，实现了`robbin` 和 `feign`方式调用***

* 注意事项， `fallback`方法中的参数和返回值，要与`@HystrixCommand(fallbackMethod = "fallback")`注解的方法一一对应。

注意：这里可以使用Spring Cloud应用中的`@SpringCloudApplication`注解来修饰应用主类，该注解的具体定义如下所示。可以看到该注解中包含了上我们所引用的三个注解，这也意味着一个Spring Cloud标准应用应包含服务发现以及断路器。

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```

* 最后，让我们运行起来看看效果，为了触发服务降级逻辑，我们将服务提供者eureka-service-provider的逻辑加一些延迟`Thread.sleep(5000L);` ，此时，将获得的返回结果为：fallback方法中定义的结果

```
{
  "message": {
    "code": 10000,
    "message": "xxxx"
  },
  "data": null
}
```

* 结论：从eureka-service-provider的日志中，可以看到服务提供方输出了原本要返回的结果，但是由于返回前延迟了5秒，而服务消费方触发了服务请求超时异常，服务消费者就通过`HystrixCommand`注解中指定的降级逻辑进行执行，因此该请求的结果返回了`fallback`方法对应的值。这样的机制，对自身服务起到了基础的保护，同时还为异常情况提供了自动的服务降级切换机制。


