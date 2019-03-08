---
layout:     post
title:      "Docker Swarm"
subtitle:   "Docker Swarm"
date:       2019-03-06 13:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
---

## 环境准备

![avatar](/img/in-post/2019-03-06/5.png)

## 环境搭建

主节点：192.168.205.10
从节点1：192.168.205.11
从节点2：192.168.205.12

![avatar](/img/in-post/2019-03-06/1.png)

```bash
docker swarm init --advertise-addr 192.168.205.10
```

复制提示的命令到其它两个容器中执行

![avatar](/img/in-post/2019-03-06/2.png)
![avatar](/img/in-post/2019-03-06/3.png)

查看node信息

![avatar](/img/in-post/2019-03-06/4.png)

## service 创建和水平扩展

```bash
docker service COMMAND

Manage services

Commands:
  create      Create a new service
  inspect     Display detailed information on one or more services
  logs        Fetch the logs of a service or task
  ls          List services
  ps          List the tasks of one or more services
  rm          Remove one or more services
  rollback    Revert changes to a service's configuration
  scale       Scale one or multiple replicated services
  update      Update a service

Run 'docker service COMMAND --help' for more information on a command
```

```bash
docker service create --name demo busybox /bin/sh -c "while true; do sleep 3600; done"
```

查看容器详细信息

```bash
docker service ls       
docker service ps demo  //查看service存放在哪台机器
docker ps   //查看当前机器下的service
```

![avatar](/img/in-post/2019-03-06/Snip20190306_6.png)

replicated表明可以横向扩展

```bash
docker service scale demo=5
```

![avatar](/img/in-post/2019-03-06/Snip20190306_7.png)

5/5表示5个都已经ready/共5个scale

swarm 分布情况
![avatar](/img/in-post/2019-03-06/Snip20190306_1.png)
![avatar](/img/in-post/2019-03-06/Snip20190306_2.png)

删除swarm

```bash
docker service rm demo
```

## 在docker swarm下部署wordpress

```bash
//创建overlay网络
[vagrant@swarm-manager ~]$ docker network create -d overlay demo
lqem7ybsxvsqupsamhs7gwuym
[vagrant@swarm-manager ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
81117831f35e        bridge              bridge              local
lqem7ybsxvsq        demo                overlay             swarm
f5cf7cd988bf        docker_gwbridge     bridge              local
6fca47e6833e        host                host                local
qn8r5p813ae9        ingress             overlay             swarm
5d8be2887f8b        none                null                local
```

创建mysql service

```bash
docker service create --name mysql --env MYSQL_ROOT_PASSWORD=root --env MYSQL_DATABASE=wordpress --network demo --mount type=volume,source=mysql-data,destination=/var/lib/mysql mysql:5.7
```

创建wordpress service

```bash
docker service create --name wordpress -p 80:80 --env WORDPRESS_DB_PASSWORD=root --env WORDPRESS_DB_HOST=mysql --network demo wordpress
```

部署成功
![avatar](/img/in-post/2019-03-06/Snip20190307_2.png)

此时访问集群中任意一台机器的ip都可以访问到wordpress，实际上,docker在swarm中建立了一个虚拟ip来实现通信，关于通信方式可参考[routingmesh](https://www.jianshu.com/p/bfb2c125d2bf)


## Routing Mesh的两种体现

1. internal:Container与Container之间访问通过overlay网络(虚拟ip)
2. ingress: 如果服务有绑定接口，此服务可通过swarm中任意节点访问

## ingress 负载均衡

作用：
1. 外部访问的负载均衡
2. 服务端口被暴漏给各个swarm节点
3. 内部通过ipvs进行负载均衡


## docker stack

### docker stack 语法梳理

#### ENDPOINT_MODE

ENDPOINT_MODE: vip | dnsrr

vip 通过lvs负载均衡虚拟ip的方式（默认推荐使用）
dnsrr：dns负载均衡轮询策略

#### LABELS 

lables是起到帮助信息的作用

```bash
version: "3"
services:
  web:
    image: web
    deploy:
      labels:
        com.example.description: "This label will appear on the web service"
```

#### mode

mode: global | replicated(默认)

replicated可以做横向扩展，global不可以

#### PLACEMENT

node.role == manager    指定部署在swarm manager节点上
constraints     一些限制条件

```bash
version: '3.3'
services:
  db:
    image: postgres
    deploy:
      placement:
        constraints:
          - node.role == manager            
          - engine.labels.operatingsystem == ubuntu 14.04
        preferences:
          - spread: node.labels.zone
```

#### REPLICAS

在mode：replicated下有效
replicas: 6     部署6个service

```bash
version: '3'
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 6
```
#### RESOURCES

(cpu_shares, cpu_quota, cpuset, mem_limit, memswap_limit, mem_swappiness)的一些资源设置


```bash
version: '3'
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```

表示redis服务被限制使用不超过50M的内存和0.50(50%)的可用处理时间(CPU)，并保留20M的内存和0.25 CPU时间(始终可用)

#### RESTART_POLICY

```bash
version: "3"
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```
表示服务停止时最大尝试次数为3,重启时间间隔是5秒，重启是否成功之前需要等待120s时间

#### ROLLBACK_CONFIG

parallelism：最多可以同时update2个replicas，每次只能更新一个
delay：每次更新之间的间隔时间，
order: 更新期间的操作顺序。stop-first(旧任务在启动新任务之前停止)或start-first(新任务首先启动，正在运行的任务短暂搁置)(默认stop-first)注意:只支持v3.4或更高版本。

```bash
version: '3.4'
services:
  vote:
    image: dockersamples/examplevotingapp_vote:before
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
        order: stop-first
```

详情参考[compose-file](https://docs.docker.com/compose/compose-file/)


### 通过docker stack 部署wordpress

sudo mkdir -p /usr/docker-vol/mysql/data

vim docker-compose.yml 

```bash
version: '3'

services:

  web:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: root
    networks:
      - my-network
    depends_on:
      - mysql
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
    volumes:
      - /usr/docker-vol/mysql/data:/var/lib/mysql
    networks:
      - my-network
    deploy:
      mode: global
      resources:
        limits:
          cpus: "0.2"
          memory: 512M
      placement:
        constraints:
          - node.role == manager

volumes:
  mysql-data:

networks:
  my-network:
    driver: overlay

```


```bash
docker stack deploy wordpress --compose-file=docker-compose.yml
```
docker stack services wordpress 查看service准备情况

docker stack ls 列举stack

docker stack ps wordpress 查看wordpress运行情况

![avatar](/img/in-post/2019-03-06/Snip20190308_5.png)

看到如下界面配置成功

![avatar](/img/in-post/2019-03-06/Snip20190308_6.png)


清空环境
```bash
docker stack rm wordpress
```
