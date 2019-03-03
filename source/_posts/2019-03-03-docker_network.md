---
layout:     post
title:      "Docker network"
subtitle:   "Docker network"
date:       2019-03-03 13:30:00
author:     "Psl"
catalog:    true
tags:
  - Docker
  - Docker Network
---

## 环境说明

两台vagrant 创建的 docker 虚拟机，虚拟机配置信息

```bash
boxes = [
    {
        :name => "docker-node1",
        :eth1 => "192.168.205.10",
        :mem => "1024",
        :cpu => "1"
    },
    {
        :name => "docker-node2",
        :eth1 => "192.168.205.11",
        :mem => "1024",
        :cpu => "1"
    }
]
```

启动两个 docker 容器

```bash
sudo docker run -d --name test busybox /bin/sh -c "while true; do sleep 3600; done"
sudo docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```
![avatar](/img/in-post/2019-03-03/8.png)

此时发现da678d729936的ip地址为172.17.0.3，cf393747710e的ip地址为172.17.0.2

执行```docker exec da678d729936 ping 172.17.0.2```试试可不可以ping通cf393747710e的ip

![avatar](/img/in-post/2019-03-03/9.png)

发现可以ping通，证明docker两个container之间的ip namespace是隔离开的，但是两个ip之间是可以ping通的，那么具体原理是什么呢？
我们可以通过下面的实验模拟来理解一下

## network namespace

### 创建net namespace

```bash
sudo ip netns add <name>        //  添加一个namespace
sudo ip netns list
sudo ip netns exec test ip a    //  查看test ip info
```

创建两个namespace

```bash
sudo ip netns add test
sudo ip netns add test1
```

执行```sudo ip netns exec test ip link``` 查看ip信息
![avatar](/img/in-post/2019-03-03/2.png)

接口启动命令
```bash
sudo ip netns exec test ip link set dev lo up
```

此时lo网卡是down状态

执行```sudo ip netns exec test ip link set dev lo up``` 将lo 开启
![avatar](/img/in-post/2019-03-03/3.png)

出现unknown原因是因为lo是单个接口，只有做link后的成对接口可以up

### link namespace

添加一对veth接口，执行```sudo ip link add veth-test type veth peer name veth-test1```

![avatar](/img/in-post/2019-03-03/4.png)

添加 veth-test 到 test namespace 中

```bash
sudo ip link set veth-test netns test
```
分别查看本地和test ip link 信息

![avatar](/img/in-post/2019-03-03/5.png)

发现本地的veth-test接口被添加到了test中，同理，添加test1

```bash
sudo ip link set veth-test1 netns test1
```

### ip link

执行
```bash
sudo ip netns exec test ip addr add 192.168.1.1/24 dev veth-test
sudo ip netns exec test1 ip addr add 192.168.1.2/24 dev veth-test1
```

然后启动接口，参照上面启动lo接口的命令

```bash
sudo ip netns exec test ip link set dev veth-test up
sudo ip netns exec test1 ip link set dev veth-test1 up
```

查看test和test1 ip 状态
![avatar](/img/in-post/2019-03-03/6.png)

发现两个接口状态都是state up，并且test和test1均已分配192.168.1.1和192.168.1.2表示接口已经启动成功,测试相互ping一下

![avatar](/img/in-post/2019-03-03/7.png)

## docker network

### bridge

把之前docker container 中的两个容器stop和rm掉，重新启动一个test容器

```bash
sudo docker run -d --name test busybox /bin/sh -c "while true; do sleep 3600; done"
```

```bash
// 列举出docker的网络
sudo docker network ls
// 查看某个docker network id 的具体信息
sudo docker network inspect <networkid>
```

eg: 查看bridge网络的详细信息

```bash
[vagrant@docker-node1 labs]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
8378c2ff8198        busybox             "/bin/sh -c 'while t…"   3 seconds ago       Up 3 seconds                            test
[vagrant@docker-node1 labs]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
1fb7b167deb2        bridge              bridge              local
2d13173ae2bf        host                host                local
6d2c375cd6cd        none                null                local
[vagrant@docker-node1 labs]$ docker network inspect 1fb7b167deb2
[
    {
        "Name": "bridge",
        "Id": "1fb7b167deb233148d61b85c03bb68d9b8e3cbb124b60deb7be8acf88abdac21",
        "Created": "2019-03-03T02:04:36.334606846Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "8378c2ff8198bbac6647463286408de11a7814c7882e99232404ecab0e1517ad": {
                "Name": "test",
                "EndpointID": "f976820b8133d11cd37602ece5f762106aba2f3735420272ad0ed3bd8b1dc58a",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
[vagrant@docker-node1 labs]$
```

![avatar](/img/in-post/2019-03-03/10.png)

通过上图可以看出本地的 vethd347bdc@if11 和 容器内的 eth0@if12 是一对接口
这一对veth peer 连接到了docker0的网络上面，可以通过下面的演示验证

```bash
sudo yum install bridge-utils
```

brctl命令说明

```bash
[vagrant@docker-node1 labs]$ brctl
Usage: brctl [commands]
commands:
	addbr     	<bridge>		add bridge
	delbr     	<bridge>		delete bridge
	addif     	<bridge> <device>	add interface to bridge
	delif     	<bridge> <device>	delete interface from bridge
	hairpin   	<bridge> <port> {on|off}	turn hairpin on/off
	setageing 	<bridge> <time>		set ageing time
	setbridgeprio	<bridge> <prio>		set bridge priority
	setfd     	<bridge> <time>		set bridge forward delay
	sethello  	<bridge> <time>		set hello time
	setmaxage 	<bridge> <time>		set max message age
	setpathcost	<bridge> <port> <cost>	set path cost
	setportprio	<bridge> <port> <prio>	set port priority
	show      	[ <bridge> ]		show a list of bridges
	showmacs  	<bridge>		show a list of mac addrs
	showstp   	<bridge>		show bridge stp info
	stp       	<bridge> {on|off}	turn stp on/off
[vagrant@docker-node1 labs]$
```

启动test2容器

```bash
sudo docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```

通过```brctl show```命令可以看到如下结果：

```bash
[vagrant@docker-node1 labs]$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02425db75bb1	no		veth4de57f6
							            vethd347bdc
```

可以看到docker0和两个veth peer接口的关系

### host 和 none

#### none
```bash
sudo docker run -d --name test4 --network none busybox /bin/sh -c "while true; do sleep 3600; done"
```

![avatar](/img/in-post/2019-03-03/25.png)

这种方式创建的容器没有ip地址，只能通过exec的方式进入

![avatar](/img/in-post/2019-03-03/26.png)

#### host

```bash
sudo docker run -d --name test5 --network host busybox /bin/sh -c "while true; do sleep 3600; done"
```

![avatar](/img/in-post/2019-03-03/27.png)

进入test5容器查看ip

```bash
[vagrant@docker-node1 ~]$ docker exec test5 ip a
...
37: veth9cb9ecd@if36: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0
    link/ether ce:61:3e:72:57:f9 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::cc61:3eff:fe72:57f9/64 scope link
       valid_lft forever preferred_lft forever
...
[vagrant@docker-node1 ~]$  ip a
...
37: veth9cb9ecd@if36: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether ce:61:3e:72:57:f9 brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::cc61:3eff:fe72:57f9/64 scope link
       valid_lft forever preferred_lft forever
...
```
我们发现这种方式的docker container 共享了主机里面的ip namespace

## 多机器通信

![avatar](/img/in-post/2019-03-03/28.png)

### Mutil-host networking with etcd

#### setup etcd cluster

在docker-node1上

```
vagrant@docker-node1:~$ wget https://github.com/coreos/etcd/releases/download/v3.0.12/etcd-v3.0.12-linux-amd64.tar.gz
vagrant@docker-node1:~$ tar zxvf etcd-v3.0.12-linux-amd64.tar.gz
vagrant@docker-node1:~$ cd etcd-v3.0.12-linux-amd64
vagrant@docker-node1:~$ nohup ./etcd --name docker-node1 --initial-advertise-peer-urls http://192.168.205.10:2380 \
--listen-peer-urls http://192.168.205.10:2380 \
--listen-client-urls http://192.168.205.10:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.205.10:2379 \
--initial-cluster-token etcd-cluster \
--initial-cluster docker-node1=http://192.168.205.10:2380,docker-node2=http://192.168.205.11:2380 \
--initial-cluster-state new&
```


在docker-node2上

```
vagrant@docker-node2:~$ wget https://github.com/coreos/etcd/releases/download/v3.0.12/etcd-v3.0.12-linux-amd64.tar.gz
vagrant@docker-node2:~$ tar zxvf etcd-v3.0.12-linux-amd64.tar.gz
vagrant@docker-node2:~$ cd etcd-v3.0.12-linux-amd64/
vagrant@docker-node2:~$ nohup ./etcd --name docker-node2 --initial-advertise-peer-urls http://192.168.205.11:2380 \
--listen-peer-urls http://192.168.205.11:2380 \
--listen-client-urls http://192.168.205.11:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.205.11:2379 \
--initial-cluster-token etcd-cluster \
--initial-cluster docker-node1=http://192.168.205.10:2380,docker-node2=http://192.168.205.11:2380 \
--initial-cluster-state new&
```

检查cluster状态

```
vagrant@docker-node2:~/etcd-v3.0.12-linux-amd64$ ./etcdctl cluster-health
member 21eca106efe4caee is healthy: got healthy result from http://192.168.205.10:2379
member 8614974c83d1cc6d is healthy: got healthy result from http://192.168.205.11:2379
cluster is healthy
```

#### 重启docker服务


在docker-node1上

```
$ sudo service docker stop
$ sudo /usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.205.10:2379 --cluster-advertise=192.168.205.10:2375&
```

在docker-node2上

```
$ sudo service docker stop
$ sudo /usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.205.11:2379 --cluster-advertise=192.168.205.11:2375&
```

#### 创建overlay network

在docker-node1上创建一个demo的overlay network

```
[vagrant@docker-node1 etcd-v3.0.12-linux-amd64]$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
22c070951371        bridge              bridge              local
2d13173ae2bf        host                host                local
c144a2167891        my-bridge           bridge              local
6d2c375cd6cd        none                null                local
[vagrant@docker-node1 etcd-v3.0.12-linux-amd64]$ sudo docker network create -d overlay demo
6f58ba8913c3e0df5ac9086f79e87cc62e57ac723d26dcb05edb7635f19103c8
[vagrant@docker-node1 etcd-v3.0.12-linux-amd64]$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
22c070951371        bridge              bridge              local
6f58ba8913c3        demo                overlay             global
2d13173ae2bf        host                host                local
c144a2167891        my-bridge           bridge              local
6d2c375cd6cd        none                null                local
[vagrant@docker-node1 etcd-v3.0.12-linux-amd64]$ sudo docker network inspect demo
[
    {
        "Name": "demo",
        "Id": "6f58ba8913c3e0df5ac9086f79e87cc62e57ac723d26dcb05edb7635f19103c8",
        "Created": "2019-03-03T13:03:20.535696601Z",
        "Scope": "global",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.0.0.0/24",
                    "Gateway": "10.0.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

我们会看到在node2上，这个demo的overlay network会被同步创建

```
vagrant@docker-node2:~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
c9947d4c3669        bridge              bridge              local
3d430f3338a2        demo                overlay             global
fa5168034de1        host                host                local
c2ca34abec2a        none                null                local
```

通过查看etcd的key-value, 我们获取到，这个demo的network是通过etcd从node1同步到node2的

```
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ ./etcdctl ls /docker
/docker/network
/docker/nodes
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ ./etcdctl ls /docker/nodes
/docker/nodes/192.168.205.10:2375
/docker/nodes/192.168.205.11:2375
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ ./etcdctl ls /docker/network/v1.0/network
/docker/network/v1.0/network/6f58ba8913c3e0df5ac9086f79e87cc62e57ac723d26dcb05edb7635f19103c8
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ ./etcdctl get /docker/network/v1.0/network/6f58ba8913c3e0df5ac9086f79e87cc62e57ac723d26dcb05edb7635f19103c8
{"addrSpace":"GlobalDefault","attachable":false,"configFrom":"","configOnly":false,"created":"2019-03-03T13:03:20.535696601Z","enableIPv6":false,"generic":{"com.docker.network.enable_ipv6":false,"com.docker.network.generic":{}},"id":"6f58ba8913c3e0df5ac9086f79e87cc62e57ac723d26dcb05edb7635f19103c8","inDelete":false,"ingress":false,"internal":false,"ipamOptions":{},"ipamType":"default","ipamV4Config":"[{\"PreferredPool\":\"\",\"SubPool\":\"\",\"Gateway\":\"\",\"AuxAddresses\":null}]","ipamV4Info":"[{\"IPAMData\":\"{\\\"AddressSpace\\\":\\\"GlobalDefault\\\",\\\"Gateway\\\":\\\"10.0.0.1/24\\\",\\\"Pool\\\":\\\"10.0.0.0/24\\\"}\",\"PoolID\":\"GlobalDefault/10.0.0.0/24\"}]","labels":{},"loadBalancerIP":"","loadBalancerMode":"NAT","name":"demo","networkType":"overlay","persist":true,"postIPv6":false,"scope":"global"}
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$
```

在node1中执行

```bash
sudo docker run -d --name test1 --net demo busybox /bin/sh -c "while true; do sleep 3600; done"
```

在node2中执行

```bash
sudo docker run -d --name test2 --net demo busybox /bin/sh -c "while true; do sleep 3600; done"
```

分别查看test1,test2 ip

```bash
[vagrant@docker-node1 etcd-v3.0.12-linux-amd64]$ docker exec test1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
19: eth0@if20: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
21: eth1@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth1
       valid_lft forever preferred_lft forever
```

```bash
docker exec test2 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:00:03 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.3/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
```

查看是否能ping通

```bash
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ docker exec test2 ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=9.354 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.789 ms
64 bytes from 10.0.0.2: seq=2 ttl=64 time=1.062 ms
^C
[vagrant@docker-node2 etcd-v3.0.12-linux-amd64]$ docker exec test2 ping test1
PING test1 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.724 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.590 ms
ç64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.789 ms
64 bytes from 10.0.0.2: seq=3 ttl=64 time=0.651 ms
```


