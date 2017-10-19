---
layout:     post
title:      Spring Boot 快速入门（HelloWorld）
subtitle:   
date:       2017-10-19
author:     Yezhiwei
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring Boot
---


## Spring Boot的主要优点

1. 开箱即用，提供各种默认配置
2. 内嵌式容器简化Web项目
3. 没有冗余代码和XML配置的要求

## 快速入门

目标：完成Spring Boot基础项目的构建，并且实现一个简单的Http请求处理

### 使用Maven构建项目或http://start.spring.io/向导构建项目

> 略
	
### 项目结构

```
src
	main
		java
			com.xxx.controller
				HelloController
		resources
	test
		java
		resources
```

### 引入Web模块

> 当前的pom.xml内容如下，仅引入了两个模块：
> 
*  spring-boot-starter：核心模块，包括自动配置支持、日志和YAML
> 
*  spring-boot-starter-test：测试模块，包括JUnit、Hamcrest、Mockito

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```
> 引入Web模块，需添加spring-boot-starter-web模块：

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

> 完整的POM文件配置

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.gemantic</groupId>
    <artifactId>springboot-hello</artifactId>
    <packaging>war</packaging>
    <version>1.0-SNAPSHOT</version>
    <name>springboot-hello Maven Webapp</name>
    <url>http://maven.apache.org</url>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.2.RELEASE</version>
        <relativePath/>
        <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <!-- 
        <jetty.version>8.1.15.v20140411</jetty.version>
        <jetty-jsp.version>2.2.0.v201112011158</jetty-jsp.version>
        <java.version>1.7</java.version>
        -->
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        
        <dependency>
	        <groupId>org.springframework.boot</groupId>
	        <artifactId>spring-boot-starter-test</artifactId>
	        <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
    </dependencies>


    <build>
        <finalName>springboot-hello</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

### 编写HelloWorld服务

```
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```

### 编写主程序

```
@SpringBootApplication
public class Server {

    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```
> 启动主程序，打开浏览器访问http://localhost:8080/hello，可以看到页面输出Hello World

### 编写单元测试用例

> 下面编写一个简单的单元测试来模拟http请求，具体如下：

```
import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = MockServletContext.class)
@WebAppConfiguration
public class ApplicationTests {
	private MockMvc mvc;
	
	@Before
	public void setUp() throws Exception {
		mvc = MockMvcBuilders.standaloneSetup(new HelloController()).build();
	}
	
	@Test
	public void getHello() throws Exception {
		mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
				.andExpect(status().isOk())
				.andExpect(content().string(equalTo("Hello World")));
	}
}
```
