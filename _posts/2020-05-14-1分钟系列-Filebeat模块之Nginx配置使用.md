---
layout:     post
title:      1分钟系列-Filebeat 模块之 Nginx 配置使用
subtitle:   
date:       2020-05-14
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - SpringCloud
    - ELK
---


### 背景

在上一篇分享中，我们发现数据采集到的日志数据都在 `message` 属性里，本次分享如何利用模块格式化日志信息

### Filebeat Module

Filebeat 内置提供了许多开箱即用的 modules ，对日志文件做简单的收集和解析处理，可以简化我们的配置，直接使用就可以

```
cd /usr/local/filebeat-7.6.2-linux-x86_64

# 查看支持哪些模块
./filebeat modules list
Enabled:

Disabled:
activemq
apache
auditd
aws
azure
cef
cisco
coredns
elasticsearch
envoyproxy
googlecloud
haproxy
ibmmq
icinga
iis
iptables
kafka
kibana
logstash
misp
mongodb
mssql
mysql
nats
netflow
nginx
osquery
panw
postgresql
rabbitmq
redis
santa
suricata
system
traefik
zeek

```

#### 配置 nginx 模块

可以看到，内置了很多的 module，但是都没有启用，如果需要启用需要进行 enable 操作

```
# 启动
./filebeat modules enable nginx

# 输出，说明 nginx 模块已可用
Enabled nginx

# 禁用
./filebeat modules disable nginx
```

模块开启之后，进入到 `modules.d` 目录下，发现只有 `nginx.yml` 后面没有 `.disabled`

```
cd modules.d/
ls -al

-rw-r--r-- 1 root root  483 Mar 26 01:23 activemq.yml.disabled
-rw-r--r-- 1 root root  475 Mar 26 01:23 apache.yml.disabled
-rw-r--r-- 1 root root  280 Mar 26 01:23 auditd.yml.disabled
-rw-r--r-- 1 root root 3064 Mar 26 01:23 aws.yml.disabled
-rw-r--r-- 1 root root 1382 Mar 26 01:23 azure.yml.disabled
-rw-r--r-- 1 root root  200 Mar 26 01:23 cef.yml.disabled
-rw-r--r-- 1 root root 1978 Mar 26 01:23 cisco.yml.disabled
-rw-r--r-- 1 root root  318 Mar 26 01:23 coredns.yml.disabled
-rw-r--r-- 1 root root  964 Mar 26 01:23 elasticsearch.yml.disabled
-rw-r--r-- 1 root root  327 Mar 26 01:23 envoyproxy.yml.disabled
-rw-r--r-- 1 root root 2019 Mar 26 01:23 googlecloud.yml.disabled
-rw-r--r-- 1 root root  376 Mar 26 01:23 haproxy.yml.disabled
-rw-r--r-- 1 root root  295 Mar 26 01:23 ibmmq.yml.disabled
-rw-r--r-- 1 root root  651 Mar 26 01:23 icinga.yml.disabled
-rw-r--r-- 1 root root  470 Mar 26 01:23 iis.yml.disabled
-rw-r--r-- 1 root root  366 Mar 26 01:23 iptables.yml.disabled
-rw-r--r-- 1 root root  398 Mar 26 01:23 kafka.yml.disabled
-rw-r--r-- 1 root root  293 Mar 26 01:23 kibana.yml.disabled
-rw-r--r-- 1 root root  471 Mar 26 01:23 logstash.yml.disabled
-rw-r--r-- 1 root root  300 Mar 26 01:23 misp.yml.disabled
-rw-r--r-- 1 root root  296 Mar 26 01:23 mongodb.yml.disabled
-rw-r--r-- 1 root root  311 Mar 26 01:23 mssql.yml.disabled
-rw-r--r-- 1 root root  471 Mar 26 01:23 mysql.yml.disabled
-rw-r--r-- 1 root root  287 Mar 26 01:23 nats.yml.disabled
-rw-r--r-- 1 root root  214 Mar 26 01:23 netflow.yml.disabled
-rw-r--r-- 1 root root  472 Mar 26 01:23 nginx.yml
-rw-r--r-- 1 root root  495 Mar 26 01:23 osquery.yml.disabled
-rw-r--r-- 1 root root  356 Mar 26 01:23 panw.yml.disabled
-rw-r--r-- 1 root root  305 Mar 26 01:23 postgresql.yml.disabled
-rw-r--r-- 1 root root  343 Mar 26 01:23 rabbitmq.yml.disabled
-rw-r--r-- 1 root root  566 Mar 26 01:23 redis.yml.disabled
-rw-r--r-- 1 root root  266 Mar 26 01:23 santa.yml.disabled
-rw-r--r-- 1 root root  299 Mar 26 01:23 suricata.yml.disabled
-rw-r--r-- 1 root root  477 Mar 26 01:23 system.yml.disabled
-rw-r--r-- 1 root root  302 Mar 26 01:23 traefik.yml.disabled
-rw-r--r-- 1 root root 1294 Mar 26 01:23 zeek.yml.disabled
```
修改 `nginx.yml` 配置文件，分别增加 `access` 和 `error` 的日志文件路径，注意路径最后增加 `*`，因为 `Nginx` 以日期归档日志文件

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ger5mh5fjxj310u0jc0ti.jpg)

```
[root@localhost filebeat-7.6.2-linux-x86_64]# vi filebeat-nginx-es.yml
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

# 加载 module
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
```

启动 `filebeat`，发现日志报错

```
./filebeat -e -c filebeat-nginx-es.yml
```

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ger5t8ecvlj32ay04qaai.jpg)

根据上面的日志提示，执行如下命令进行安装（在线安装有点慢...）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ger6f9493bj30zq078aaa.jpg)

离线安装方式，注意切换到 Elasticsearch 启动权限的用户

```
wget https://artifacts.elastic.co/downloads/elasticsearch-plugins/ingest-user-agent/ingest-user-agent-6.6.1.zip
wget https://artifacts.elastic.co/downloads/elasticsearch-plugins/ingest-geoip/ingest-geoip-6.6.1.zip

bin/elasticsearch-plugin install file:///usr/local/src/ingest-user-agent-6.6.1.zip 
bin/elasticsearch-plugin install file:///usr/local/src/ingest-geoip-6.6.1.zip

# 查看已经安装的插件
bin/elasticsearch-plugin list

```

在集群中安装成功后，重启 Elasticsearch

#### 测试观察输出

访问 Nginx（安装请参考 https://yezhwi.github.io/java/2020/05/11/1%E5%88%86%E9%92%9F%E7%B3%BB%E5%88%97-Nginx-%E5%AE%89%E8%A3%85-%E5%87%86%E5%A4%87%E8%AE%BF%E9%97%AE%E6%97%A5%E5%BF%97/）首页

在数据浏览查看索引数据，如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1germqvzvtfj31hw0u0n2g.jpg)

> 可对比上篇文章中的测试结果

### 下一步计划

在 Kibana 中查看数据




