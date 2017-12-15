---
layout:     post
title:      实践bug总结-Feign使用Hystrix配置
subtitle:   
date:       2017-12-15
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud
    - Feign
    - Hystrix
---

* 在之前的文章中使用注解`@HystrixCommand` 的 `fallbackMethod`属性实现回调的。

```
@Slf4j
@Component
public class AdditionCommand {

    @Autowired
    private AdditionClient additionClient;

    @HystrixCommand(fallbackMethod = "fallback")
    public ResponseEntity<ResultData> add(Integer a, Integer b) {
        // 模拟接口异常
        Random random = new Random();
        int r = random.nextInt(10);
        if (r % 2 == 0) {
            throw new RuntimeException();
        }
        return additionClient.add(a, b);
    }

    public ResponseEntity<ResultData> fallback(Integer a, Integer b) {
        ResultData resultData = new ResultData();
        Message message = Message.builder().code(10000).message("xxxx").build();
        resultData.setMessage(message);
        return ResponseEntity.ok(resultData);
    }
}
```

* 然后，Feign是以接口的形式进行工作的，本身没有方法体。那么Feign怎么与Hystrix整合呢？

* 事实上，SpringCloud默认已经为Feign整合了Hystrix，只要添加Hystrix依赖，***Feign默认就会用断路器包裹所有方法***。

 Feign接口定义：

```
import com.gemantic.commons.ResultData;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author Yezhiwei
 * @date 17/12/14
 */
@FeignClient(name = "springcloud-subtraction-service", fallback = SubtractionClientFallBack.class)
public interface SubtractionClient {

    /**
     * 减法接口
     * @param a
     * @param b
     * @return
     */
    @GetMapping(value = "/sub")
    ResponseEntity<ResultData> sub(@RequestParam("a") Integer a, @RequestParam("b") Integer b);
}
```

回调实现`SubtractionClient `接口：

```
import com.gemantic.commons.ResultData;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author Yezhiwei
 * @date 17/12/14
 */
@Component
public class SubtractionClientFallBack implements SubtractionClient {
    @Override
    public ResponseEntity<ResultData> sub(@RequestParam("a") Integer a, @RequestParam("b") Integer b) {
        // 这里是实现的回调逻辑！
        return null;
    }
}
```

* 好了，开始测试。but  but  but 问题来了，访问了几次接口，断路器Hystrix一直不起作用，`Hystrix Dashboard`也没有收到任何统计信息。感觉Feign中并没有启用Hystrix，通过查看文档发现了Hystrix在Feign中的开关配置， 从[Disable HystrixCommands For FeignClients By Default](https://github.com/spring-cloud/spring-cloud-netflix/issues/1277) 中知道了产生错误的原因，以及为什么要默认关闭Hystrix。

* 解决方案，在properties文件中增加`feign.hystrix.enabled=true`

![Feing Hystrix](/img/hystrix-dashboard-4.png)















