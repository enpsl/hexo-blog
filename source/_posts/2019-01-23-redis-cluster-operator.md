---
layout:     post
title:      "Redis cluster 伸缩"
subtitle:   "Redis集群伸缩"
date:       2019-01-23 22:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - redis
---

## 集群伸缩原理

![](/img/in-post/2019-01-23/1.png)
![](/img/in-post/2019-01-23/2.png)

<center style="font-weight:bold">集群伸缩=槽和数据在节点之间的移动</center>

## 扩容集群

#### 准备新节点
新节点：
* 集群模式
* 配置和其它节点统一
* 启动后仍是孤儿节点

![](/img/in-post/2019-01-23/3.png)
#### 加入集群
```
cluster meet 127.0.0.1 6385
cluster meet 127.0.0.1 6386
```
![](/img/in-post/2019-01-23/4.png)
<center style="font-weight:bold">加入后的效果</center>

加入集群的作用：
* 为它迁移槽和数据实现扩容
* 作为从节点负责故障转移

```
redis-cli --cluster add-node new_host:new_port existing_host:existing_port --cluster-slave --cluster-master-id <arg>
```
> 建议使用redis-trib.rb能够避免新节点已经加入了其它集群，造成故障

#### 迁移槽和数据

###### 槽迁移计划:
![](/img/in-post/2019-01-23/6.png)
![](/img/in-post/2019-01-23/7.png)

###### 迁移数据：
1. 对目标节点发送: cluster setslot{slot} importing {sourceNodeId}命令，让目标节点准备导入槽的数据。
2. 对源节点发送: cluster setslot{slot} migrating {targetNodeId}命令，让源节点准备迁出槽的数据。
3. 源节点循环执行: cluster getkeysinslot{slot}{count}命令，每次获取count个属于槽的键。
4. 在源节点执行: migrate {targetIP}{targetPort} key 0 {timeout}命令把指定key迁移。
5. 重复执行3-4直到槽下所有数据节点均迁移到目标节点。
6. 向集群内所有主节点发送cluster setslot{slot} node {targetNodeId}命令，通知槽分配给目标节点。

###### 数据迁移伪python代码:
```python
def move_slot(source,target,slot):
    #目标节点准备导入槽slot
    target.cluster("setslot",slot,"importing",source.nodeID)
    #目标节点准备全出槽slot
    target.cluster("setslot",slot,"migrating",source.nodeID) 
    while True:
        #批量从源节点获取键
        keys = source.cluster("getkeysinslot",slot,pipeline_size)
        if keys.length ==0:
            #键列表为空时，退出循环
            break
    #批量迁移键到目标节点
    source.call("migrate",target.host,target.port,"",timeout,"keys")
    #向集群所有主节点通知槽slot被分配给目标节点
    for node in nodes:
        if node.flag =="slave":
            continue
        node.cluster("setslot",slot,"node",target.nodeID)
```

###### pipline迁移

![](/img/in-post/2019-01-23/8.png)

> 3.0.6 版本pipline数据迁移会有丢失数据bug，在3.2.8已解决

## 扩容演示

#### 环境准备
![](/img/in-post/2019-01-23/9.png)
当前集群是三主三从结构，此时我们加入两个新节点7006,7007。7007是7006的从节点，我们需要从7001,7002节点把一部分数据迁移给7006。
配置准备:
```bash
sed 's/7000/7006/g' redis-7000.conf > redis-7006.conf
sed 's/7000/7007/g' redis-7000.conf > redis-7007.conf
```
#### meet:
```bash
redis-cli -p 7000 cluster meet 127.0.0.1 7006
redis-cli -p 7000 cluster meet 127.0.0.1 7007
```
#### replicate:
```bash
redis-cli -p 7007 cluster replicate d57d27051ce9db7752f894394b621368f9e0a058
```
![](/img/in-post/2019-01-23/10.png)
> 此时7007已经属于7006的从节点

#### 迁移数据：
>由于槽数量比较多，所以这里使用redis-trib来迁移
```
redis-cli --cluster reshard 127.0.0.1 7000
```

![](/img/in-post/2019-01-23/11.png)
此时给我们提示出了当前集群的信息，由于我们现在是4个主节点，所以需要分成四等份来支持向master写入数据
![](/img/in-post/2019-01-23/12.png)
![](/img/in-post/2019-01-23/13.png)

槽迁移后的信息：
![](/img/in-post/2019-01-23/14.png)

## 收缩集群

#### 下线迁移槽

![](/img/in-post/2019-01-23/15.png)

#### 忘记节点
```bash
redis-cli>cluster forget {downNodeId}
```
![](/img/in-post/2019-01-23/16.png)
#### 关闭节点

## 收缩集群演示

例：下线7006，7007

#### 迁移槽：

迁移过程命令：

redis-cli --cluster reshard --cluster-from {7006nodeid} --cluster-to 7000{7000nodeid} --cluster-slots {slot num} 127.0.0.1:7006

redis-cli --cluster reshard --cluster-from {7006nodeid} --cluster-to 7001{7001nodeid} --cluster-slots {slot num} 127.0.0.1:7006

redis-cli --cluster reshard --cluster-from {7006nodeid} --cluster-to 7002{7002nodeid} --cluster-slots {slot num} 127.0.0.1:7006

迁移到7000示例：
```bash
redis-cli --cluster reshard --cluster-from d57d27051ce9db7752f894394b621368f9e0a058 --cluster-to 092fd7c3cf19693eddec5c0fae9894d681023ce5 --cluster-slots 1365 127.0.0.1:7006
```
之后选择yes即可

![](/img/in-post/2019-01-23/17.png)
可观察出0-1364的槽节点以迁移完毕，重复上述步骤，迁移剩余的槽

迁移后：
![](/img/in-post/2019-01-23/18.png)

#### 忘记节点：
```bash
redis-cli --cluster del-node 127.0.0.1:7000 d57d27051ce9db7752f894394b621368f9e0a058
```

>需要先下线从节点在下线主节点，否则会发生故障转移

#### 完成缩容
![](/img/in-post/2019-01-23/19.png)







