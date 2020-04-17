---
layout:     post
title:      ContiPerf轻量级的测试工具
subtitle:   
date:       2018-08-24
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
---

## ContiPerf 介绍

> In order to assure software performance, software needs to be tested accordingly as early as possible - only weaknesses diagnosed early can be assessed quickly and cheaply. ContiPerf enables performance testing already in early development phases and in an easy-to-learn manner: 
A developer writes a performance test in form of a JUnit 4 test case and adds performance test execution settings as well as performance requirements in form of Java annotations. When JUnit is invoked by an IDE, build script or build server, ContiPerf activates, performs the tests and creates an HTML report. The report provides a detailed overview of execution, requirements and measurements, even providing a latency distribution chart.
A large feature set for execution settings and performance requirements is available, e.g. Ramp up, warm up, individual pause timing, concurrent exection of test groups and more.

> https://sourceforge.net/p/contiperf/wiki/Home/

* ContiPerf 是一个轻量级的测试工具，基于 JUnit4 开发，可用于效率测试等。
* 可以指定在线程数量和执行次数，通过限制最大时间和平均执行时间来进行效率测试。

## ContiPerf 使用

* 在 pom.xml 中引入 ContiPerf

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>

<dependency>
	<groupId>org.databene</groupId>
	<artifactId>contiperf</artifactId>
	<version>2.3.4</version>
	<scope>test</scope>
</dependency>
```

* 项目使用的 spring boot 搭建，依赖如下图：

![依赖关系](https://tva2.sinaimg.cn/large/006tNbRwly1fuktqf9uctj30wy0e2wfd.jpg)

* 编写单元测试

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class TokenFactoryTest {

    @Resource
    private TokenFactory tokenFactory;

    /**
     * @Rule 注解激活 ContiPerf
     */
    @Rule
    public ContiPerfRule contiPerfRule = new ContiPerfRule();

    private AppInfo appInfo = null;

    @Before
    public void init() {
        appInfo = new AppInfo();
        appInfo.setAppName("test");
        appInfo.setAppKey("cpkkkkxxxxx");
    }


    /**
     * 1个线程 执行100次
     * 要求95%的测试不超过10ms，99%的测试不超过20ms
     * @throws Exception
     */
    @Test
    @PerfTest(invocations = 100,threads = 1)
    @Required(percentiles = "95:10,99:20")
    public void createAccessToken() throws Exception {

        AccessToken accessToken = tokenFactory.createAccessToken(appInfo);
        System.out.println(accessToken);

    }
}
```

* 运行结果

```
...
samples: 100
max:     571
average: 10.85
median:  5
...
```
同时在工程的 target/contiperf-report 目录下生成一个 index.htm 文件，内容如下图：

![contiperf-report](https://tva4.sinaimg.cn/large/006tNbRwly1fuku1d8pe7j31kw10hgmp.jpg)



## ContiPerf 总结

* 使用 @Rule 注解激活 ContiPerf，通过 @Test 指定测试方法，@PerfTest 指定方法的调用次数和线程数量，@Required 指定方法的性能要求（每次执行的最长时间，平均时间，总时间等）。
* 也可以通过对类指定 @PerfTest 和 @Required，表示类中方法的默认设置。
* 同时在工程的 target/contiperf-report 目录下生成一个 index.htm 文件
* PerfTest参数

	1. @PerfTest(invocations = 300)：执行300次，和线程数量无关，默认值为1，表示执行1次；
	2. @PerfTest(threads=30)：并发执行30个线程，默认值为1个线程；
	3. @PerfTest(duration = 20000)：重复地执行测试至少执行20s。

* Required参数

	1. @Required(throughput = 20)：要求每秒至少执行20个测试；
	2. @Required(average = 50)：要求平均执行时间不超过50ms；
	3. @Required(median = 45)：要求所有执行的50%不超过45ms； 
	4. @Required(max = 2000)：要求没有测试超过2s；
	5. @Required(totalTime = 5000)：要求总的执行时间不超过5s；
	6. @Required(percentile90 = 3000)：要求90%的测试不超过3s；
	7. @Required(percentile95 = 5000)：要求95%的测试不超过5s； 
	8. @Required(percentile99 = 10000)：要求99%的测试不超过10s; 
	9. @Required(percentiles = "66:200,96:500")：要求66%的测试不超过200ms，96%的测试不超过500ms。







