---
layout:     post
title:      "Redis cluster"
subtitle:   "Redis集群"
date:       2019-01-20 22:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - redis
---

## 呼唤集群

当系统应用需要有更大容量和QPS的支撑，此时就需要采用的集群的方式，也可以简单理解为加机器

数据分区:

![](/img/in-post/2019-01-20/1.jpg)

## 分布方式

#### 顺序分布

![](/img/in-post/2019-01-20/2.png)

#### 哈希分布

![](/img/in-post/2019-01-20/3.png)

* 节点取余
* 一致性hash
* 虚拟槽分区

##### 节点取余

![](/img/in-post/2019-01-20/3.png)

* 客户端分片: 哈希+取余 
* 节点伸缩: 数据节点关系变化，导致数据迁移
* 迁移数量和添加节点数量相关：建议翻倍扩容

##### 一致性hash

![](/img/in-post/2019-01-20/5.png)

一致性hash扩容

![](/img/in-post/2019-01-20/4.png)


* 客户端分片：哈希+顺时针（优化取余）
* 节点伸缩：只影响临近节点，但是还是有数据迁移
* 翻倍伸缩：保证最小迁移数据和负载均衡

##### 虚拟槽分布

![](/img/in-post/2019-01-20/2.jpg)

* 预设虚拟槽：每个槽映射一个数据子集，一般比节点数大
* 良好的hash函数：如crc16
* 服务端管理节点，槽，数据：例如redis cluster

#### 对比
<table><tr><th>分布方式</th><th>特点</th><th>典型产品</th></tr><tr><td>哈希分布</td><td>数据分散度高<br/>key,value分布业务无关<br>无法顺序访问<br>支持批量操作</td><td>一致性hash memcache<br/>redis cluster<br>其它缓存产品</td></tr><tr><td>顺序分布</td><td>数据分散度易倾斜<br>key,value业务相关<br>可顺序访问<br>支持批量操作</td><td>BigTable<br>HBase<br></td></tr></table>
## Redis Cluster

#### 架构
* 节点
* meet
* 指派槽
* 复制

#### 特性：
* 复制
* 分片
* 高可用

#### 安装

##### 原生安装

1: 配置开启节点:

```
port{$port}
daemonize yes
dir "${redis-src}/data/"
dbfilename "dump-{$port}.rdb"
logfile "{$port}.log"
cluster-enabled "yes
cluster-config-file nodes-{$port}.conf
```

![](/img/in-post/2019-01-20/6.png)

批量生成配置文件:

![](/img/in-post/2019-01-20/7.png)

执行：
```
redis-server redis-7000.conf
redis-server redis-7001.conf
redis-server redis-7002.conf
redis-server redis-7003.conf
redis-server redis-7004.conf
redis-server redis-7005.conf
```

![](/img/in-post/2019-01-20/8.png)

2: meet

**cluster meet ip port**

```
redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7001
redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7002
redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7003
redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7004
redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7005
```

先进行7000和7001的握手

![](/img/in-post/2019-01-20/9.png)

发现7000和7001已经完成握手，继续meet其他的节点

![](/img/in-post/2019-01-20/10.png)

此时执行```cluster nodes```和```cluster info```均发现6个节点相互关联，证明已经握手成功

3: 指派槽

**cluster addslots slot [slot...]**

由于一共要分配16384个槽，所以需要借助脚本去分配槽
```bash
start=$1
end=$2
port=$3
for slot in `seq ${start} ${end}`
do
    echo "slot:${slot}"
    redis-cli -p ${port} cluster addslots ${slot}
done
```
我们要配置的是三主三从，所以要把16384三等分
```
0-5461 7000 5462-10922 7001 10923-16383 7002
```
执行以下命令:
```
sh addslots.sh 0 5461 7000
sh addslots.sh 5462 10922 7001
sh addslots.sh 10923 16383 7002
```
查看槽分配状态

![](/img/in-post/2019-01-20/11.png)
![](/img/in-post/2019-01-20/12.png)

> 此时发现16384个槽确实已经分配完毕，槽分配完毕

4: 主从

cluster replicate node-id

给7003分配到master7000主节点上：
```
redis-cli -p 7003 cluster replicate 6d4942b15eb5e02bb6193453443ccb827c13c6df
redis-cli -p 7004 cluster replicate 916f84dee5fbef724ddf2f90fddf51fd654113f1
redis-cli -p 7005 cluster replicate fee14c1f872fc4c7d451f84a616cf735d220538b
```

主从分配结果：
![](/img/in-post/2019-01-20/13.png)
![](/img/in-post/2019-01-20/14.png)

##### 官方工具

由于原生安装过程比较麻烦，又容易出错，所以正常的生产环境使用官方工具安装，但是掌握原生安装的方式更容易让我们理解集群分配的原理

**ruby环境准备**

1. 下载编译安装ruby
2. 安装rubygem redis
3. 安装redis-trib.rb

![](/img/in-post/2019-01-20/15.png)

1: 配置开启节点:

```
port{$port}
daemonize yes
dir "${redis-src}/data/"
dbfilename "dump-{$port}.rdb"
logfile "{$port}.log"
cluster-enabled "yes
cluster-config-file nodes-{$port}.conf
```

执行：
```
redis-server redis-7000.conf
redis-server redis-7001.conf
redis-server redis-7002.conf
redis-server redis-7003.conf
redis-server redis-7004.conf
redis-server redis-7005.conf
```

2: 集群创建

![](/img/in-post/2019-01-20/16.png)

```
//1 表示1个主节点分配1个从节点
redis-cli --cluster create --cluster-replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```
![](/img/in-post/2019-01-20/17.png)

上图分别展示了槽分配，主从和节点信息，符合预期执行yes即可

分配成功信息：

![](/img/in-post/2019-01-20/18.png)

集群验证：

![](/img/in-post/2019-01-20/19.png)

>当然如果维护上百台集群显然也不是最好的方式，可以借助或构建云平台来管理集群






