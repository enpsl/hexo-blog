---
layout:     post
title:      "Docker network port"
subtitle:   "Docker network port"
date:       2019-03-03 11:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
  - Docker Network
---

## 端口映射

```bash
docker run --name web -d nginx
```

![avatar](/img/in-post/2019-03-03/19.png)

由上图可以看到我们在虚拟机内部访问container的ip的方式可以访问到nginx欢迎页，但是访问本地地址映射不到,
可以通过端口映射来解决这个问题

删除刚才的nginx container,重新启动

```bash
docker stop web
docker rm web
docker run --name web -d -p 80:80 nginx
docker ps
```
![avatar](/img/in-post/2019-03-03/20.png)

映射成功

我的vagrant ip 映射配置
![avatar](/img/in-post/2019-03-03/21.png)
![avatar](/img/in-post/2019-03-03/22.png)

映射流程

![avatar](/img/in-post/2019-03-03/24.png)

图中的192.168.205.10:80为我本机的ip私有地址，外网不能访问，如果我们是在一个云主机上创建的web服务，云主机就可以分配一个public的ip就可以作为外网的出口ip来提供服务









