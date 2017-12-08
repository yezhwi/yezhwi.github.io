---
layout:     post
title:      实践bug总结-SpringBoot项目中有两个main函数
subtitle:   SpringBoot项目只能存在一个main函数
date:       2017-12-08
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
---

问题一：异常信息如下

```
Failed to execute goal org.springframework.boot:spring-boot-maven-plugin:1.5.4.RELEASE:repackage (default) on project springcloud-hystrix-dashboard: Execution default of goal org.springframework.boot:spring-boot-maven-plugin:1.5.4.RELEASE:repackage failed: Unable to find a single main class from the following candidates [com.gemantic.Server, com.gemantic.commons.ResultData] -> [Help 1]
```

解决方案：

通过分析日志，一个是服务的main类 `com.gemantic.Server` 另一个为 `com.gemantic.commons.ResultData` 通过查看源代码，发现此类中也有一个main方法，SpringBoot项目规范只能有一个main方法，去掉之后可以正常打包。

***

问题二：项目能正常打jar包，但是在执行jar时，提示无法找到含有main方法的主类

解决方案：

在pom.xml文件中增加如下配置，再重新打包就可以正常执行了

```
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










