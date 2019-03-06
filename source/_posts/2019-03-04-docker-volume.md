---
layout:     post
title:      "Docker Volume"
subtitle:   "Docker Volume"
date:       2019-03-04 13:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
---

## 基本命令

```bash
docker volume ls                // 列出所有volume
docker volume rm <VOLUME...>    // 删除一个或多个volume
docker volume rm $(docker volume ls -qf dangling=true) //删除失效的volume:
```

## Data Volume

volume是docker数据持久化的一种方式，那么怎样使用volume呢？

### Dockerfile使用

![](/img/in-post/2019-03-04/2.png)

可以通过mysql官方的Dockerfile看到volume的使用方式

### 命令模式使用

启动一台mysql;

```bash
docker run -d --name mysql1 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql
```

查看volume:

```bash
pengshiliang@pengshiliang-virtual-machine:~$ docker volume ls
DRIVER              VOLUME NAME
local               195d16514c70f7990f190b1557fb2131a2b8942c48ef50025b7f08fc7b082dcd
pengshiliang@pengshiliang-virtual-machine:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
20752e526d9e        mysql               "docker-entrypoint.s…"   14 seconds ago      Up 13 seconds       3306/tcp, 33060/tcp   mysql1
pengshiliang@pengshiliang-virtual-machine:~$ docker volume inspect 195d16514c70f7990f190b1557fb2131a2b8942c48ef50025b7f08fc7b082dcd
[
    {
        "CreatedAt": "2019-03-04T21:41:29+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/195d16514c70f7990f190b1557fb2131a2b8942c48ef50025b7f08fc7b082dcd/_data",
        "Name": "195d16514c70f7990f190b1557fb2131a2b8942c48ef50025b7f08fc7b082dcd",
        "Options": {},
        "Scope": "local"
    }
]
pengshiliang@pengshiliang-virtual-machine:~$
```

可以通过inspect命令查看volume的存储路径

删掉mysql container 

![](/img/in-post/2019-03-04/1.png)

发现volume仍然存在，也确认了docker可以通过volume持久化存储数据

还可以通过下面的实例来证实

为了避免环境影响,删掉刚才产生的volume，重新启动一台mysql

![](/img/in-post/2019-03-04/3.png)

valume的名称太长了,加入一个-v参数，来给volume起个别名,然后启动mysql,并指定volume存放位置

```bash
docker run -d -v mysql:/var/lib/mysql --name mysql1 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql
```

![](/img/in-post/2019-03-04/4.png)

进入当前container，并创建一个数据库

![](/img/in-post/2019-03-04/5.png)

然后退出，把当前的mysql1容器删掉

![](/img/in-post/2019-03-04/6.png)

检查volume

![](/img/in-post/2019-03-04/7.png)

再去启动一台mysql2

```bash
docker run -d -v mysql:/var/lib/mysql --name mysql2 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql
```

进入到当前容器的mysql数据库查看详情

![](/img/in-post/2019-03-04/8.png)

发现刚才创建的数据库还在,也证明了volume是持久化存储的方式

## bind mouting

命令:
```bash
docker -run -v /home/aaa:/root/aaa
```

通过docker bind mouting将本地和服务器(容器)上的资源绑定，改变一方都对数据同步，从而达到直接修改本地资源，服务器上的资源自动更新

准备一个目录，创建index.html文件

![](/img/in-post/2019-03-04/9.png)

![](/img/in-post/2019-03-04/10.png)

可以发现，当前目录下的文件是和container内部的/usr/share/nginx/html文件是同步的


