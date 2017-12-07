---
layout:     post
title:      SpringCloud @RestController返回值问题
subtitle:   2017-12-05遇到的问题，百思不得其解，又是一天终于解决...
date:       2017-12-06
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springcloud
tags:
    - SpringCloud
    - Eureka
---

* 实现一下加法的Controller进行测试，代码如下：

```
import com.gemantic.commons.Message;
import com.gemantic.commons.ResultData;
import com.gemantic.springcloud.service.Addition;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.constraints.NotNull;

/**
 * @author Yezhiwei
 * @date 17/12/4
 */

@Slf4j
@RestController
@Api(description = "加法接口")
public class AdditionController {

    @Autowired
    private Addition addition;

//    @GetMapping(value = "/add", produces = {"application/json"})
    @GetMapping(value = "/add")
//    @RequestMapping("/add")
    @ApiOperation(value = "a + b = ?")
    public ResponseEntity<ResultData> add(@RequestParam @NotNull Integer a, @RequestParam @NotNull Integer b) {

        if (log.isInfoEnabled()) {
            log.info("add {} + {}", a, b);
        }
        int result = addition.add(a, b);
        ResultData resultData = ResultData.builder().message(Message.builder().code(0).message(Message.SUCCESS_MESSAGE).build()).build();
        resultData.setData(result);
        return ResponseEntity.ok(resultData);
    }
}
```

* pom.xml配置文件

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.4.RELEASE</version>
        <relativePath/>
        <!-- lookup parent from repository -->
    </parent>

    <groupId>com.gemantic.springcloud</groupId>
    <artifactId>springcloud-addition-service</artifactId>
    <version>1.0-SNAPSHOT</version>


    <packaging>jar</packaging>
    <name>springcloud-addition-service</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.7</java.version>
        <springfox.version>2.6.1</springfox.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.gemantic.springcloud</groupId>
            <artifactId>springcloud-interface</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jetty</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>

        <!-- swagger -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>${springfox.version}</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>${springfox.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.14</version>
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
</project>
```

* 访问测试

出现了很奇怪的再现：

返回的结果不是json

![](/img/add1.jpeg)

在接口的后缀增加.json，接口正常返回了json数据

![](/img/add2.jpeg)

在接口的后缀增加.xml，接口返回了xml数据

![](/img/add3.jpeg)

* 思考，为什么会出现这样的情况，怎么与之前不一样了，难道是`spring-boot-starter-parent`版本的问题？带着问题去找答案

1. 修改 `spring-boot-starter-parent` 版本，测试没有效果
2. `@GetMapping(value = "/add", produces = {"application/json"})` 增加produces类型说明，可以解决，但是要对每个接口都这样做，太累了。。。
3. 对比之前的项目（因为之前都是返回的json），最后发现是配置原因

把下面的依赖
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

修改为：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
修改之后测试

图一的访问方式能正常返回与图二的结果一致。

再按图三的访问方式访问后，提示404。终于找到根本原因了。

更详细的原因或区别就得看源码了

==马虎==

***

问题已经发布到 [http://www.spring4all.com/](http://www.spring4all.com/)上，

[http://www.spring4all.com/question/93](http://www.spring4all.com/question/93)











