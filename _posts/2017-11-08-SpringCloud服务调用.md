---
layout:     post
title:      SpringCloud服务调用
subtitle:   服务调用Ribbon
date:       2017-11-08
author:     Yezhiwei
category:   springcloud
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud Eureka Hystrix Zuul Feign Ribbon Turbine
---


## Spring Cloud Ribbon

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套客户端负载均衡的工具。它是一个基于HTTP和TCP的客户端负载均衡器。它可以通过在客户端中配置ribbonServerList来设置服务端列表去轮询访问以达到均衡负载的作用。

当Ribbon与Eureka联合使用时，ribbonServerList会被DiscoveryEnabledNIWSServerList重写，扩展成从Eureka注册中心中获取服务实例列表。同时它也会用NIWSDiscoveryPing来取代IPing，它将职责委托给Eureka来确定服务端是否已经启动。

而当Ribbon与Consul（类似于Eureka的另一种服务注册中心）联合使用时，ribbonServerList会被ConsulServerList来扩展成从Consul获取服务实例列表。同时由ConsulPing来作为IPing接口的实现。

我们在使用Spring Cloud Ribbon的时候，不论是与Eureka还是Consul结合，都会在引入Spring Cloud Eureka或Spring Cloud Consul依赖的时候通过自动化配置来加载上述所说的配置内容，所以我们可以快速在Spring Cloud中实现服务间调用的负载均衡。

### 做一个客户端的例子

基于Spring Cloud Ribbon实现的消费者，我们可以根据eureka-service-provider实现的内容进行简单改在就能完成，具体步骤如下，拷贝eureka-service-provider，然后重命名为eureka-service-consumer

* 在pom.xml中增加下面的依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

* 定义`RestTemplate`增加`@LoadBalanced`注解

```
@Configuration
public class BeansConfig {

    @Autowired
    private Environment environment;

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        requestFactory.setReadTimeout(environment.getProperty("client.http.request.readTimeout", Integer.class, 15000));
        requestFactory.setConnectTimeout(environment.getProperty("client.http.request.connectTimeout", Integer.class, 3000));
        RestTemplate restTemplate = new RestTemplate(requestFactory);
        return restTemplate;
    }
}
```

* 调用服务端，直接通过RestTemplate发起请求

```
@Slf4j
@RestController
@Api(description = "模拟交易委托队列接口")
public class EntrustQueueController {

    @Autowired
    private RestTemplate restTemplate;



    @ApiOperation(value = "新增加委托请求")
    @PostMapping("/entrusQueue")
    public ResponseEntity<ResultData> addEntrust(@RequestBody EntrustQueue entrustQueue) {

        log.info(entrustQueue.toString());

        ResultData resultData = restTemplate.postForObject("http://eureka-service-provider/entrusQueue", entrustQueue, ResultData.class);
        return  ResponseEntity.ok(resultData);
    }
}
```

注意：

这里的url参数有一些特别，请求地址host的位置并没有使用一个具体的IP地址和端口的形式，而是服务名组成的。那么这样的请求为什么可以调用成功呢？因为Spring Cloud Ribbon有一个拦截器，在这里自动的去选取可用的服务实例，并将实际要请求的IP地址和端口替换这里的服务名，从而完成服务接口的调用。（通过测试，当服务注册中心Eureka挂掉后，客户端不能调通服务端）。





