---
layout:     post
title:      SpringCloud-Hystrix-让你的系统稳一点儿
subtitle:   服务容错保护（服务降级、依赖隔离、断路器）
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


## 服务降级（动手试一下）

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

扩展：这里可以使用Spring Cloud应用中的`@SpringCloudApplication`注解来修饰应用主类，该注解的具体定义如下所示。可以看到该注解中包含了上我们所引用的三个注解，这也意味着一个Spring Cloud包含服务发现以及断路器。

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


***

## 依赖隔离

Hystrix会为每一个Hystrix命令创建一个独立的线程池，实现线程池的隔离。这样就算某个在Hystrix命令包装下的依赖服务出现延迟过高的情况，也只是对该依赖服务的调用产生影响，而不会拖慢其他的服务。

如何使用Hystrix来实现依赖隔离呢？其实，在定义服务降级的时候，已经自动的实现了依赖隔离。`@HystrixCommand`将某个函数包装成了`Hystrix Command`，这里不仅定义服务降级，Hystrix框架还会自动的为这个函数实现调用的隔离。所以，依赖隔离、服务降级在使用时候都是一体化实现的，这样利用Hystrix来实现服务容错保护在编程模型上就非常方便。

对依赖服务的线程池隔离实现，可以带来如下优势：

* 系统自身得到完全的保护，即便给依赖服务分配的线程池被占满，也不会影响应用自身的其余部分。
* 可以有效的降低接入新服务的风险。如果新服务接入后运行不稳定或存在问题，完全不会影响到应用其他的请求。
* 当依赖的服务从失效恢复正常后，它的线程池会被清理并且能够马上恢复健康的服务。
* 当依赖的服务因实现机制调整等原因造成其性能出现很大变化的时候，此时线程池的监控指标信息会反映出这样的变化。同时，我们也可以通过实时动态刷新自身应用对依赖服务的阈值进行调整以适应依赖方的改变。
* 当依赖的服务出现配置错误的时候，线程池会快速的反应出此问题（通过失败次数、延迟、超时、拒绝等指标的增加情况）。我们可以在不影响应用功能的情况下通过实时的动态属性刷新来处理它。

总之，通过对依赖服务实现线程池隔离，让系统更加健壮，不会因为个别依赖服务出现问题而引起非相关服务的异常。

## 断路器——服务熔断

在服务消费端的服务降级逻辑因为`Hystrix Command`调用依赖服务超时，触发了降级逻辑，即使这样，由于受限于Hystrix超时的问题，调用依然很有可能产生堆积。这个时候断路器就会发挥作用，当满足什么条件，断路器才起作用呢？三个重要参数：快照时间窗、请求总数下限、错误百分比下限。这些参数的作用分别是：

* 快照时间窗：断路器确定是否打开需要统计一些请求和错误数据，而统计的时间范围就是快照时间窗，默认为最近的10秒。
* 请求总数下限：在快照时间窗内，必须满足请求总数下限才能熔断。默认为20，意味着在10秒内，如果该`Hystrix Command`的调用不足20次，即时所有的请求都超时或其他原因失败，断路器都不会打开。
* 错误百分比下限：当请求总数在快照时间窗内超过了下限，比如发生了30次调用，如果在这30次调用中，有16次发生了超时异常，也就是超过50%的错误百分比，在默认设定50%下限情况下，这时候就会将断路器打开。


打开之后，再有请求调用的时候，将不会调用主逻辑，而是直接调用降级逻辑，这个时候就不会等待5秒之后才返回fallback。通过断路器，实现了自动地发现错误并将降级逻辑切换为主逻辑，减少响应延迟的效果。

在断路器打开之后，处理逻辑并没有结束，降级逻辑已经被成了主逻辑，那么原来的主逻辑要如何恢复呢？对于这一问题，Hystrix也为我们实现了自动恢复功能。当断路器打开，对主逻辑进行熔断之后，Hystrix会启动一个休眠时间窗，在这个时间窗内，降级逻辑是临时的成为主逻辑，当休眠时间窗到期，断路器将进入半开状态，释放一次请求到原来的主逻辑上，如果此次请求正常返回，那么断路器将继续闭合，主逻辑恢复，如果这次请求依然有问题，断路器继续进入打开状态，休眠时间窗重新计时。

通过上面的一系列机制，Hystrix的断路器实现了对依赖资源故障、对降级策略的自动切换以及对主逻辑的自动恢复机制。这使得我们的微服务在依赖外部服务或资源的时候得到了非常好的保护，同时对于一些具备降级逻辑的业务需求可以实现自动化的切换与恢复，相比于设置开关由监控和运维来进行切换的传统实现方式显得更为智能和高效。

总结：断路器可以实现快速失败，当`Hystrix Command`请求后端服务失败数量超过一定比例(默认50%)， 断路器会切换到开路状态(OPEN)。这时所有请求会直接失败而不会发送到后端服务。断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN)。这时会判断下一次请求的返回情况，如果请求成功，断路器切回闭路状态(CLOSED)，否则重新切换到开路状态(OPEN)。Hystrix的断路器一旦发现后端服务不可用，断路器会直接切断请求链，避免发送大量无效请求影响系统吞吐量，并且断路器有自我检测并恢复的能力。
