---
layout:     post
title:      1分钟系列-Elasticsearch 集群安装与常见问题
subtitle:   
date:       2020-05-10
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Java
    - SpringCloud
    - ELK
---


### 集群安装

> Elasticsearch 简介请参考上篇

#### 配置文件修改

> 配置文件位置 config/elasticsearch.yml

* 集群的名称

通过 `cluster.name` 可以配置集群的名称，集群是一个整体，因此名称都要一致，所有主机都配置成相同的名称，配置示例：

```
cluster.name: my-es-application
```

* 节点的名称

通过 `node.name` 可以配置每个节点的名称，每个节点都是集群的一部分，每个节点名称都不要相同，可以按照顺序编号，配置示例：

```
node.name: node-238
```

其他的主机可以配置为 `node-239`、`node-240` 等。

* 是否有资格成为主节点

通过 `node.master` 可以配置该节点是否有资格成为主节点，如果配置为 `true`，则主机有资格成为主节点，配置为 `false` 则主机就不会成为主节点，可以去当数据节点或负载均衡节点。注意这里是有资格成为主节点，不是一定会成为主节点，主节点需要集群经过选举产生。这里我配置所有主机都可以成为主节点，因此都配置为 `true`，配置示例：

```
node.master: true
```

* 是否是数据节点

通过 `node.data` 可以配置该节点是否为数据节点，如果配置为 `true`，则主机就会作为数据节点，注意主节点也可以作为数据节点，当 `node.master` 和 `node.data` 均为 `false`，则该主机会作为负载均衡节点。这里我配置所有主机都是数据节点，因此都配置为 `true`，配置示例：

```
node.data: true
```

* 数据和日志路径

通过 `path.data` 和 `path.logs` 可以配置 Elasticsearch 的数据存储路径和日志存储路径，可以指定任意位置，另外注意一下写入权限问题，配置示例：

```
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs
```

* 设置访问的地址和端口

我们需要设定 Elasticsearch 运行绑定的 Host，默认是无法公开访问的，如果设置为主机的公网 IP 或 0.0.0.0 就是可以公开访问的，这里我们可以都设置为公开访问或者部分主机公开访问，如果是公开访问就配置为：

```
network.host: 0.0.0.0
```

如果不想被公开访问就不用配置。另外还可以配置访问的端口，默认是 9200：

```
http.port: 9200
```

* 集群地址设置

通过 `discovery.zen.ping.unicast.hosts` 可以配置集群的主机地址，配置之后集群的主机之间可以自动发现，这里我配置的是内网地址，配置示例：

```
discovery.zen.ping.unicast.hosts: ["192.168.111.240:9300", "192.168.111.239:9300"]
```

这里请改成你的主机对应的 IP 地址。

* 节点数目相关配置

为了防止集群发生“脑裂”，即一个集群分裂成多个，通常需要配置集群最少主节点数目，通常为 (可成为主节点的主机数目 / 2) + 1，例如我这边可以成为主节点的主机数目为 3，那么结果就是 2，配置示例：

```
discovery.zen.minimum_master_nodes: 2
```

另外还可以配置当最少几个节点恢复之后，集群就正常工作，这里我设置为 4，可以酌情修改，配置示例：

```
gateway.recover_after_nodes: 4
```

> 注：在另外两个节点上也都需要进行上述配置 "192.168.111.240:9300", "192.168.111.239:9300"

#### 检查配置文件

```
# 修改完后的配置文件内容为（去掉注释和空行）
grep -Ev "^$|[#;]" elasticsearch.yml

cluster.name: my-es-application
node.name: node-238
network.host: 0.0.0.0
network.publish_host: 192.168.111.238
discovery.zen.ping.unicast.hosts: ["192.168.111.238:9300", "192.168.111.240:9300", "192.168.111.239:9300"]
http.cors.enabled: true
http.cors.allow-origin: "*"
discovery.zen.minimum_master_nodes: 2
```

#### 修改其他节点的配置

同上

#### 分别启动每个节点

```
[elasticsearch@localhost elasticsearch-6.6.1]$ ./bin/elasticsearch -d
```

#### 检查集群状态，如果结果和下图一致说明集群配置成功

```
curl 'localhost:9200/_cat/health?v'
```
![](https://tva1.sinaimg.cn/large/007S8ZIlly1gekeht1n57j31og03saa9.jpg)

```
curl -XGET 'http://192.168.111.239:9200/_cluster/state?pretty' | more
```

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gekejwofqfj30ut0u0q5a.jpg)

### 集群配置遇到的问题

#### 防火墙，导致连接不到主节点

##### discovery.zen.minimum_master_nodes: 2

* 无配置，通过查看每个节点的日志，发现都输出如下信息，说明个自为主

```
# 192.168.111.238 上
new_master {node-238} ...

# 192.168.111.239 上
new_master {node-239}...

# 192.168.111.240 上
new_master {node-240}...
```

* 配置，怀疑“脑裂”，所以增加上面的配置指定集群主节点的主机数目，再运行发现节点上不断输出

```
not enough master nodes discovered during pinging (found [[Candidate{node={node-240}......]], but needed [2]), pinging again...

.....

020-05-07T13:15:12,258][WARN ][o.e.d.z.ZenDiscovery     ] [node-239] failed to connect to master [{node-240}{4ttfd_QwRE65S0GLWcRsPw}{iKILM4_yS22E_utZG6lVAA}{192.168.111.240}{192.168.111.240:9300}{ml.machine_memory=1031090176, ml.max_open_jobs=20, xpack.installed=true, ml.enabled=true}], retrying...
org.elasticsearch.transport.ConnectTransportException: [node-240][192.168.111.240:9300] connect_exception
```

##### 通过观察日志，怀疑可能是因为防火墙的原因，通过下面的命令进行检查，关闭后再启动集群就测试通过了

```
# CentOS firewall 作为防火墙
# 查看防火墙状态
firewall-cmd --state
# 停止firewall
systemctl stop firewalld.service
# 禁止firewall开机启动
systemctl disable firewalld.service 
```

#### Linux 服务器系统配置

```
# 修改最大内存限制
vim /etc/sysctl.conf

# 在末尾增加如下配置
vm.max_map_count = 655360
vm.swappiness=1

# 保存退出，输入以下命令执行使其生效
sysctl -p

---

# 修改最大打开文件个数
vim /etc/security/limits.conf

# 在末尾添加如下内容
hard nofile 65536
soft nofile 65536

```

#### 注意观察日志，根据日志提示去解决问题


### 下一步计划

Nginx 安装，准备访问日志




