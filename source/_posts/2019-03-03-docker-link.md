---
layout:     post
title:      "Docker network link"
subtitle:   "Docker network link"
date:       2019-03-03 10:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
  - Docker Network
---

# docker link

## docker link

由于docker container 之中的 ip在为创建之前是未知的，不利于服务与服务之间的配置连接，所以docker 提供了一种办法来解决这个问题，
可以通过 docker name 之间的link来解决

![avatar](/img/in-post/2019-03-03/11.png)

创建test2并link到test1

```bash
docker run -d --name test2 --link test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```

进入到test2容器

```bash
docker exec -it test2 /bin/sh
ping test1
```
![avatar](/img/in-post/2019-03-03/12.png)

这种方式的优点是：
假如test1有一个数据库，我们可以在test2容器中通过```mysql -u <name> -P <port> -h test1```来访问了

> 由于是test2 去link test1 所以，在test1容器中，ping test2是不可用的

## network 创建

删除掉test2容器并重新创建test2

```bash
docker run -d --name test2 busybox /bin/sh -c "while true; do sleep 3600; done"
```

### 创建bridge

```bash
docker network create -d bridge my-bridge
docker network ls
```

![avatar](/img/in-post/2019-03-03/13.png)

创建test3并指定network到my-bridge
> 如果不指定network默认连接是docker0

```bash
docker run -d --name test3 --network my-bridge busybox /bin/sh -c "while true; do sleep 3600; done"
```

![avatar](/img/in-post/2019-03-03/14.png)


查看test3 container network信息

```bash
docker network inspect <my-bridge id>
```

![avatar](/img/in-post/2019-03-03/15.png)

### bridge连接

```bash
docker network connect my-bridge test2
```

```bash
docker network inspect bridge
docker network inspect my-bridge
```

我们可以看到bridge和my-bridge的container中都包含了test2

进入到test2容器中
```bash
docker exec -it test2 /bin/sh
```
![avatar](/img/in-post/2019-03-03/16.png)

可以发现在test2容器中可以ping通test3但是不能ping test1, 实际上docker在用户自己创建的bridge中做了一层link，所以test2和test3容器可以相互ping 通对方

把test1也加入到my-bridge中
```bash
docker network connect my-bridge test1
```
![avatar](/img/in-post/2019-03-03/17.png)

此时test1也可以ping通了
