---
layout:     post
title:      利用 Log4j2 异步保存日志到 MongoDB 中
subtitle:   
date:       2018-10-26
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
tags:
    - java
    - Log4j2
    - MongoDB
---

### 需求

> 将 Log4j2 日志文件写到 MongoDB 中，并且希望能按自定义字段进行保存。

### 添加依赖

> 由于此工程没有使用 Spring / SpringBoot 框架，主要演示怎么配置 Log4j2 配置将日志保存到 MongoDB，如果使用了 SpringBoot 框架，请按 spring-boot-starter-xxxx 的方式配置。
> 
> 注意版本问题，如果使用 log4j2 2.11.0 以上的版本配置有区别，可以参考 github 上 log4j2 对应 tag 的项目结构


```
<properties>
	<log4j2.version>2.9.0</log4j2.version>
	<log4j2-nosql.version>2.9.0</log4j2-nosql.version>
	<log4j2-mongodb3.version>2.11.1</log4j2-mongodb3.version>
	<mongodb-driver.version>3.8.2</mongodb-driver.version>
</properties>


<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongo-java-driver</artifactId>
    <version>${mongodb-driver.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-nosql</artifactId>
    <version>${log4j2.version}</version>
</dependency>
```

### Log4j2 配置中增加

```
<appenders>
    ...
    <NoSql name="mongoAppender" bufferSize="10">
        <MongoDb databaseName="phone-schedule" collectionName="asr_log" server="10.0.0.xx" port="xxx" username="xxxx" password="xxxx"/>
    </NoSql>
   
   	<!-- 异步保存数据 -->   
    <Async name="mongoAppenderAsync">
        <AppenderRef ref="mongoAppender" />
    </Async>
   
	...
    
</appenders>


<loggers>

	...
    <logger name="mongolog" level="info" additivity="false">
        <AppenderRef ref = "mongoAppenderAsync" />
        <!--<AppenderRef ref="Console" />--> <!-- 生产环境要注释掉 -->
    </logger>
    ...
<loggers>

```

### 单元测试

```
@Log4j2(topic = "mongolog")
public class LogTest {

    @Test
    public void testLog() {
        log.info("xxxx");
        log.error("error message");
    }
}
```

结果：

![保存结果](https://ws3.sinaimg.cn/large/006tNbRwly1fwk9nmnai0j31kw0q940j.jpg)

结论：

从上图中，发现日志已经保存到 MongoDB 中且结构一致（自动创建 Field），同时，无论什么级别的日志都是将消息保存到 message 字段中，如果想保存结构化的信息怎么办？

### 通过 MDC 增加自定义字段

```
@Log4j2(topic = "mongolog")
public class LogTest {

    @Test
    public void testLog() {
        MDC.put("sessionId", UUID.randomUUID());
        MDC.put("ask", "Hello");
        MDC.put("asr", "Jack");
        MDC.put("phone", "15011227340");
        MDC.put("audioPath", "/data/file");
        log.info("xxxx");
        MDC.clear();
//        log.error("error message");
    }
}
```

结果：

![自定义字段](https://ws3.sinaimg.cn/large/006tNbRwly1fwk9wdt1g0j31kw0hsq46.jpg)

结论：

从上图中，可以发现在 contextMap 中已经有5个 Fields 了，然后就方便提供 Restful API 在 Web UI 中 渲染了。

### 遇到的问题

> 在本地测试通过之后，打包发布测试环境
> 错误日志如下：

```
ERROR StatusLogger appenders contains an invalid element or attribute "NoSql"
ERROR StatusLogger No appender named mongoAppender was configured
Exception in thread "main" java.lang.ExceptionInInitializerError
Caused by: org.apache.logging.log4j.core.config.ConfigurationException: No appenders are available for AsyncAppender mongoAppenderAsync
        at org.apache.logging.log4j.core.appender.AsyncAppender.start(AsyncAppender.java:120)
        at org.apache.logging.log4j.core.config.AbstractConfiguration.start(AbstractConfiguration.java:265)
        at org.apache.logging.log4j.core.LoggerContext.setConfiguration(LoggerContext.java:545)
        at org.apache.logging.log4j.core.LoggerContext.reconfigure(LoggerContext.java:617)
        at org.apache.logging.log4j.core.LoggerContext.reconfigure(LoggerContext.java:634)
        at org.apache.logging.log4j.core.LoggerContext.start(LoggerContext.java:229)
        at org.apache.logging.log4j.core.impl.Log4jContextFactory.getContext(Log4jContextFactory.java:242)
        at org.apache.logging.log4j.core.impl.Log4jContextFactory.getContext(Log4jContextFactory.java:45)
        at org.apache.logging.log4j.LogManager.getContext(LogManager.java:174)
        at org.apache.logging.log4j.LogManager.getLogger(LogManager.java:618)
```

> 解决方案
> 
> [LOG4J2-691](https://issues.apache.org/jira/browse/LOG4J2-691)
> 

默认的 assembly 配置，这样打出一个 jar 包里没有依赖的 lib 目录，可以通过自定义 assembly 文件，指定把什么内容打到包里。

```
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <appendAssemblyId>false</appendAssemblyId>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
            <manifest>
                <mainClass>com.gemantic.phone.PhoneServer</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>assembly</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

