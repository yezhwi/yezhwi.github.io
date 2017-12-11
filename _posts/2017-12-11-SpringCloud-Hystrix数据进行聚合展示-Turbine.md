---
layout:     post
title:      SpringCloud-Hystrix数据进行聚合展示-Turbine
subtitle:   Turbine-通过HTTP方式收集聚合
date:       2017-12-11
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud 
    - Hystrix
    - Turbine
---


## 背景

[上一篇](https://yezhwi.github.io/springcloud/2017/11/14/SpringCloud-Hystrix-Dashboard%E7%9B%91%E6%8E%A7%E9%9D%A2%E6%9D%BF/)使用Hystrix Dashboard来展示Hystrix用于熔断的各项指标。通过Hystrix Dashboard，可以方便的查看服务实例的综合情况，但是，在实际的生产环境中，我们的服务是肯定需要做高可用的，而仅通过Hystrix Dashboard只能实现对服务单个实例的数据展现。那么对于多实例的情况，我们就需要将这些指标数据进行聚合。接下来将使用另外一个工具：Turbine来聚合指标数据。

## 依赖项目

* 服务注册中心——`springcloud-eureka`
* 服务提供者——`springcloud-addition-service`(加法计算服务模拟微服务)
* 使用ribbon和hystrix实现的服务消费者——`springcloud-consumer-hystrix` 端口9604
* 用于展示服务消费者Hystrix数据的dashboard——`springcloud-hystrix-dashboard` 端口9605

## 通过HTTP方式收集聚合

* 创建一个标准的Spring Boot工程，命名为：`springcloud-turbine`，端口9700。
* 编辑pom.xml，具体依赖内容如下：

```
  <parent>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-parent</artifactId>
    <version>Dalston.SR1</version>
    <relativePath /> <!-- lookup parent from repository -->
  </parent>

  <dependencies>

    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-turbine</artifactId>
    </dependency>
   
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
  
  </dependencies>


  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <fork>true</fork>
          <executable>true</executable>
        </configuration>
      </plugin>
    </plugins>
  </build>
```

* 创建应用主类Server，并使用@EnableTurbine注解开启Turbine。

```
@SpringBootApplication
@EnableTurbine
public class Server {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Server.class).web(true).run(args);
    }
}
```

* 在application.properties加入eureka和turbine的相关配置，具体如下：

```
spring.application.name=springcloud-turbine
logging.path=./logs
server.port=9700

eureka.client.serviceUrl.defaultZone=http://peer1:9600/eureka/,http://peer2:9600/eureka/,http://peer3:9600/eureka/

turbine.app-config=springcloud-consumer-hystrix
turbine.cluster-name-expression="default"
turbine.combine-host-port=true
```

*参数说明*

1. turbine.app-config参数指定了需要收集监控信息的服务名，如果有多个用英文的逗号分隔；
2. turbine.cluster-name-expression 参数指定了集群名称为default，该参数可以用来区分这些不同的聚合集群，同时该参数值可以在Hystrix仪表盘中用来定位不同的聚合集群，只需要在Hystrix Stream的URL中通过cluster参数来指定；
3. turbine.combine-host-port参数设置为true，可以让同一主机上的服务通过主机名与端口号的组合来进行区分，默认情况下会以host来区分不同的服务，这会使得在本地调试的时候，本机上的不同服务聚合成一个服务来统计。

* 最后，分别启动`springcloud-eureka`、`springcloud-addition-service`、`springcloud-consumer-hystrix`、`springcloud-turbine`以及`springcloud-hystrix-dashboard`。

查看`springcloud-turbine`服务的日志中看到
```
c.n.t.monitor.instance.InstanceMonitor   : Url for host: http://192.168.1.112:9604/hystrix.stream default
```
说明已经开始收集到服务的信息了

访问`springcloud-hystrix-dashboard` http://192.168.1.112:9605/hystrix，然后开启对http://192.168.1.112:9700/turbine.stream 的监控，如图：

![Monitor Stream](/img/turbine.png)

点击`Monitor Stream`按钮

![loading...](/img/turbine-0.png)

最初显示loading...，当我们调用`springcloud-consumer-hystrix`服务的hystrix接口后，可以正常显示了

![turbine](/img/turbine-1.png)

这时候，将看到针对服务`springcloud-consumer-hystrix`的聚合监控数据。






