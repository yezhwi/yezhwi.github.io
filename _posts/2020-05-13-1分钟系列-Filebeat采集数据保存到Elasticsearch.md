---
layout:     post
title:      1分钟系列-Filebeat 采集数据保存到 Elasticsearch
subtitle:   
date:       2020-05-13
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - SpringCloud
    - ELK
---


### Filebeat 采集日志到架构

![](https://tva1.sinaimg.cn/large/007S8ZIlly1geoxwc1zejj30kc09s0tg.jpg)

从上面的架构图中，我们可以看到 Beats 与 Nginx 部署在同一台机器上，目的是收集 Nginx 上的访问日志进行分析

### 读取 Nginx 日志文件

#### 读取文件配置

```
# 配置读取日志文件 filebeat-log.yml
filebeat.inputs: 
- type: log
  enabled: true
  paths:
    - /usr/local/nginx/logs/*.log 
setup.template.settings:
  index.number_of_shards: 3 
output.console:
  pretty: true
  enable: true
```

#### 启动执行

```
cd /usr/local/filebeat-7.6.2-linux-x86_64

./filebeat -e -c filebeat-log.yml

# 参数说明
-e: 输出到标准输出，默认输出到syslog和logs下 
-c: 指定配置文件
```

如下图，监听到 `/usr/local/nginx/logs` 目录下的 `access.log` 和 `error.log`

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gepzvqhqnjj31em02k3yl.jpg)

并且立刻读取到之前日志的内容，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gepzz9aoclj30u00uadhm.jpg)

> 后面会分享如何将 `access.log` 和 `error.log` 分开采集

#### 测试观察输出

访问 Nginx（安装请参考 https://yezhwi.github.io/java/2020/05/11/1%E5%88%86%E9%92%9F%E7%B3%BB%E5%88%97-Nginx-%E5%AE%89%E8%A3%85-%E5%87%86%E5%A4%87%E8%AE%BF%E9%97%AE%E6%97%A5%E5%BF%97/）首页， 等待一会儿会在控制台上输出如下图所示信息，`message` 字段就是 `Nginx` 的访问日志

![](https://tva1.sinaimg.cn/large/007S8ZIlly1geq01pznuqj321p0u0mzc.jpg)

### 输出到 Elasticsearch 日志文件

#### 写入 Elasticsearch 配置

```
# 配置读取日志文件 filebeat-es.yml
filebeat.inputs: 
- type: log
  enabled: true
  paths:
    - /usr/local/nginx/logs/*.log 

# 指定索引的分区数
setup.template.settings:
  index.number_of_shards: 3 
# 指定 ES 的配置
output.elasticsearch: 
  hosts: ["192.168.111.238:9200", "192.168.111.239:9200", "192.168.111.240:9200"]
```

#### 启动执行

```
cd /usr/local/filebeat-7.6.2-linux-x86_64

./filebeat -e -c filebeat-es.yml
```

#### 测试观察输出

访问 Nginx 首页，在 es head 的概览中看到新索引，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1geq2rnmj5nj309m08kq2v.jpg)

在数据浏览查看索引数据，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1geq2o547qzj31lt0u0q8q.jpg)

### 下一步计划

我们发现数据采集到的日志数据都在 `message` 属性里，下一步分享如何利用模块格式化日志信息，以及在 Kibana 中查看数据




