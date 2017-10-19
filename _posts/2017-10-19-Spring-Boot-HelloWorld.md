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

1. 为所有Spring开发者更快的入门
2. 开箱即用，提供各种默认配置来简化项目配置
3. 内嵌式容器简化Web项目
4. 没有冗余代码生成和XML配置的要求

## 快速入门

目标:完成Spring Boot基础项目的构建，并且实现一个简单的Http请求处理

1. 使用Maven构建项目或http://start.spring.io/向导构建项目
	
2. 项目结构
3. 引入Web模块

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

4. 编写HelloWorld服务

```
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```
> 启动主程序，打开浏览器访问http://localhost:8080/hello，可以看到页面输出Hello World

5. 编写单元测试用例

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
