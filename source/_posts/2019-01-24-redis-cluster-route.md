---
layout:     post
title:      "Redis cluster 客户端路由"
subtitle:   "Redis集群客户端"
date:       2019-01-24 21:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - redis
---

## MOVED重定向

![](/img/in-post/2019-01-24/1.png)

### 槽命中：直接返回
![](/img/in-post/2019-01-24/2.png)

算出key的slot值
```
>127.0.0.1> 6379 cluster keyslot hello
```
返回结果：
```
(integer) 866
```
### 槽不命中：moved异常
算出key的slot值
```
>127.0.0.1> 6379 cluster keyslot php
```
返回结果：
```
(integer) 9244
```
![](/img/in-post/2019-01-24/3.png)

看一个小例子:
```
redis-cli -c -p 7000    //集群模式
127.0.0.1:7000> cluster keyslot hello
(integer) 866
127.0.0.1:7000> set hello word
OK
127.0.0.1:7000> cluster keyslot php
(integer) 9244
127.0.0.1:7000> set php best
-> Redirected to slot [9244] located at 127.0.0.1:7001
OK
127.0.0.1:7001> get php
"best"
127.0.0.1:7001> 
redis-cli -p 7000
127.0.0.1:7000> cluster keyslot php
(integer) 9244
127.0.0.1:7000> set php best
(error) MOVED 9244 127.0.0.1:7001
127.0.0.1:7000>
```

![](/img/in-post/2019-01-24/4.png)

## ASK重定向

![](/img/in-post/2019-01-24/5.png)

在集群缩容扩容的时候，要对槽进行迁移，槽迁移过程中要遍历进行migrate,迁移时间比较长，
此时在此过程中访问一个key,但是key已经迁移到目标节点，就需要一个新的方案来解决这个问题，redis cluster 对这个问题已经有解决方案

我们来看它的一个实现演示：
![](/img/in-post/2019-01-24/6.png)

## moved & ask

* 两者都是客户端重定向
* moved:槽已确定迁移
* ask:槽还在迁移中

## smart客户端

### 基本原理
追求性能：
1. 从集群中选一个可运行的节点，使用cluster slots 初始化槽和节点映射
2. 将cluster slots结果映射到本地，为每个节点创建redisPool
3. 执行命令

基本流程：
![](/img/in-post/2019-01-24/7.png)

> 关于redis cluster 客户端使用可参考
[redis-go-cluster](https://github.com/enpsl/redis-go-cluster/tree/master/example)

### 批量操作优化
**批量操作怎么实现?meget meset必须在一个槽**
#### 串行mget 
![](/img/in-post/2019-01-24/8.png)
#### 串行IO
![](/img/in-post/2019-01-24/9.png)
#### 并行IO
![](/img/in-post/2019-01-24/10.png)
#### hash_tag
![](/img/in-post/2019-01-24/11.png)

### 四种方案优缺点对比
<table><tr><td>方案</td><td>优点</td><td>缺点</td><td>网络IO</td></tr><tr><td>串行mget</td><td>编程简单，少量keys满足需求</td><td>大量keys请求延迟严重</td><td>O(keys)</td></tr><tr><td>串行IO</td><td>编程简单，少量节点满足需求</td><td>大量node延迟严重</td><td>O(nodes)</td></tr><tr><td>并行IO</td><td>利用并行特性，延迟取决于最慢的节点</td><td>编程复杂，超市定位问题难</td><td>O(max(node))</td></tr><tr><td>hash_tag</td><td>性能最高</td><td>读写增加tag维护成本，tag分布易出现数据倾斜</td><td>O(1))</td></tr></table>