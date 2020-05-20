---
layout:     post
title:      1分钟系列-在 Kibana 安装和使用 Nginx 的日志仪表盘
subtitle:   
date:       2020-05-20
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - SpringCloud
    - ELK
---


### 配置 Filebeat

```
# filebeat 安装在 192.168.111.238 服务器上

cd /usr/local/filebeat-6.6.1-linux-x86_64
vi filebeat-nginx-es-dashboard.yml
# 将以下内容拷贝到 filebeat-nginx-es-dashboard.yml 文件中

filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /usr/local/nginx/logs/*.log
setup.template.settings:
  index.number_of_shards: 3
output.elasticsearch:
  hosts: ["192.168.111.238:9200",  "192.168.111.239:9200",  "192.168.111.240:9200"]
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
# ----- 以上同容配置参考之前的内容 -----
# Kibana 安装在 239 服务器上
setup.kibana:
  host: "192.168.111.239:5601"
```

### 安装仪表盘到 Kibana 中

```
# 执行以下命令，前提是 Kibana 服务正在启动运行中
./filebeat -c filebeat-nginx-es-dashboard.yml setup
```

输出以下信息表示安装成功，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gewxokl0f3j314605edg3.jpg)

```
# 运行 filebeat
./filebeat -e -c filebeat-nginx-es-dashboard.yml
```

### 在 Kibana 查看 Nginx 仪表盘

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gewxra0acpj31hr0u041o.jpg)

选择上图中的 `Dashboards [Filebeat Nginx]`，然后
访问 Nginx（安装请参考 https://yezhwi.github.io/java/2020/05/11/1%E5%88%86%E9%92%9F%E7%B3%BB%E5%88%97-Nginx-%E5%AE%89%E8%A3%85-%E5%87%86%E5%A4%87%E8%AE%BF%E9%97%AE%E6%97%A5%E5%BF%97/）首页，观察数据变化，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gewy49hh21j31gg0u076y.jpg)

选择 `Nginx access and error logs`，点击进入

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gewy9jbjikj31h20u0gnu.jpg)

### 下一步计划

在 Kibana 自定义仪表盘


