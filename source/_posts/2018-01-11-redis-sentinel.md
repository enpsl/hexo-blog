---
layout:     post
title:      "Redis哨兵"
subtitle:   "Redis哨兵"
date:       2019-01-19 23:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - redis
---

## Redis Sentinel基本架构

1. 多个sentinel发现并确认master有问题
2. 选举出一个sentinel作为领导
3. 选出一个salve作为master
4. 通知其余slave成为新的master的slave
5. 通知客户端主从变化
6. 等待master复活成为新的master的slave

## 安装与配置

1. 配置开启主从节点
2. 配置开启sentinel监控master节点

Redis主节点
启动```redis-server redis-7000.conf```

#### redis配置:
主节点:
```
port 7000
daemonize yes
pidfile /var/run/redis/7000.pid
logfile 7000.log
dir "${redis-src}/data/"
```
从节点:
```
port 7001
daemonize yes
pidfile /var/run/redis/7001.pid
logfile 7001.log
dir "${redis-src}/data/"
slaveof 127.0.0.1 7000
```
```
port 7002
daemonize yes
pidfile /var/run/redis/7002.pid
logfile 7002.log
dir "${redis-src}/data/"
slaveof 127.0.0.1 7000
```
#### sentinel配置:

```
port ${port}
dir "${redis-src}/data/"
logfile ${port}.log
sentinel monitor mymaster 127.0.0.1 7000 2 #监控主节点的名字 2对应当两个sentinel发现主节点有问题就发生故障转移
sentinel down-after-millisenconds mymaster 30000 #ping30秒后不通认为有问题
sentinel parallel-sync mymaster 1
sentinel failover-timeout mymaster 180000
```

#### 安装演示
**配置redis**
> 单机演示实际为能相互ping通的多台机器

1：创建master节点配置

![](/img/in-post/2019-01-19/1.png)

2：创建7000,7001节点配置

![](/img/in-post/2019-01-19/2.png)
![](/img/in-post/2019-01-19/3.png)

3：查看主从状态

![](/img/in-post/2019-01-19/4.png)

> 到此为止主从的配置搭建完毕了

**配置sentinel**

通过官方提供的配置模板导入sentinel配置

![](/img/in-post/2019-01-19/5.png)
配置信息：

![](/img/in-post/2019-01-19/6.png)

查看Sentinel监控状态：

![](/img/in-post/2019-01-19/7.png)

到此我们可以发现Sentinel已经检测到了matser节点的主从信息，由于我们只启动一个Sentinel所以Sentinel发现的数目为1

再去看看redis-sentinel-26379.conf的变化:

![](/img/in-post/2019-01-19/8.png)

> 发现他已经把7000的两个slave节点的信息配置自动写入到了配置文件中

生成另外两个Sentinel配置

![](/img/in-post/2019-01-19/9.png)

查看sentinel状态

![](/img/in-post/2019-01-19/10.png)

> 发现26380的Sentinel并没有发现其他节点


查看原因：
```
cat /var/log/redis/26379.log
cat /var/log/redis/26380.log
cat /var/log/redis/26381.log
```

![](/img/in-post/2019-01-19/11.png)
![](/img/in-post/2019-01-19/12.png)

我们发现三个节点的pid都是一样的，所以需要配置pid
编辑26379,26380,26381.conf 去掉配置文件自动生成的myid,并加入
```
pidfile "/var/run/redis/redis-sentinel-26379.pid"
pidfile "/var/run/redis/redis-sentinel-26380.pid"
pidfile "/var/run/redis/redis-sentinel-26381.pid"
```
![](/img/in-post/2019-01-19/13.png)

状态已正常

## go客户端

```go
package main

import (
	"fmt"
	"github.com/gomodule/redigo/redis"
	"github.com/letsfire/redigo"
	"github.com/letsfire/redigo/mode"
	"github.com/letsfire/redigo/mode/sentinel"
	"strconv"
	"time"
	"log"
)

func main(){
	var (
		i int
		sentinelMode mode.IMode
	)
	sentinelMode = sentinel.New(
		sentinel.Addrs([]string{"127.0.0.1:26379", "127.0.0.1:26380", "127.0.0.1:26381"}),
		sentinel.PoolOpts(
			mode.MaxActive(20),       // 最大连接数，默认0无限制
			mode.MaxIdle(0),         // 最多保持空闲连接数，默认2*runtime.GOMAXPROCS(0)
			mode.Wait(false),        // 连接耗尽时是否等待，默认false
			mode.IdleTimeout(0),     // 空闲连接超时时间，默认0不超时
			mode.MaxConnLifetime(0), // 连接的生命周期，默认0不失效
			mode.TestOnBorrow(nil),  // 空间连接取出后检测是否健康，默认nil
		),
		sentinel.DialOpts(
			redis.DialReadTimeout(time.Second),    // 读取超时，默认time.Second
			redis.DialWriteTimeout(time.Second),   // 写入超时，默认time.Second
			redis.DialConnectTimeout(time.Second), // 连接超时，默认500*time.Millisecond
			redis.DialPassword(""),                // 鉴权密码，默认空
			redis.DialDatabase(0),                 // 数据库号，默认0
			redis.DialKeepAlive(time.Minute*5),    // 默认5*time.Minute
			redis.DialNetDial(nil),                // 自定义dial，默认nil
			redis.DialUseTLS(false),               // 是否用TLS，默认false
			redis.DialTLSSkipVerify(false),        // 服务器证书校验，默认false
			redis.DialTLSConfig(nil),              // 默认nil，详见tls.Config
		),
		// 连接哨兵配置，用法于sentinel.DialOpts()一致
		// 默认未配置的情况则直接使用sentinel.DialOpts()的配置
		// sentinel.SentinelDialOpts()
	)

	var instance = redigo.New(sentinelMode)

	for  {
		res, err := instance.String(func(c redis.Conn) (res interface{}, err error) {
			return c.Do("set", "test" + strconv.Itoa(i), i)
		})
		if err != nil {
			log.Println(err)
		} else {
			fmt.Println(res)
		}
		i++
		time.Sleep(1 * time.Second)
	}
}
```
执行以下命令，并模拟7000端口宕机
```
go run client.go
```
![](/img/in-post/2019-01-19/16.png)

执行结果：

![](/img/in-post/2019-01-19/14.png)
![](/img/in-post/2019-01-19/15.png)

## 日志分析

查看7001.log

![](/img/in-post/2019-01-19/17.png)

从日志中发现选举7002为master,执行```redis-cli -p 7002 info replication```验证下

![](/img/in-post/2019-01-19/18.png)

看看sentinel日志的变化

![](/img/in-post/2019-01-19/19.png)

## 故障转移

#### 三个定时任务

1. 每10秒每个sentinel对master和slave执行info
* 发现slave节点
* 确定主从关系
2. 每2秒每个sentinel通过master节点的channel节点交换信息(pub/sub)
* 通过__sentinel__和:hello频道交互
* 交互对节点的看法和自身的信息
3. 每1秒每个Sentinel对其它Sentinel和Redis执行ping
* 失败判定依据，心跳检测

#### 主观下线和客观下线

* 主观下线：每个sentinel节点对Redis节点失败的偏见
* 客观下线：所有sentinel节点对Redis节点失败达成共识（quorum:建议节点数/2+1）

#### 领导者选举

* 原因：只有一个sentinel节点完成故障转移
* 选举：通过sentinel is-master-down-by-addr命令都希望成为领导者
1. 每个主观下线的Sentinel节点向其他Sentinel节点发送命令，要求它设置为领导者
2. 收到命令的Sentinel节点如果没有同意通过其他Sentinel节点发送的命令，那么该同意将被拒绝
3. 如果该Sentinel节点发现通过的票数已经超过Sentinel集合半数且超过quorum，那么它将成为领导者
4. 如果此过程中有多个Sentinel节点成为领导者，那么将等待一段时间重新选举

#### 故障转移

1. 从slave节点选出一个"合适的"节点作为新的master节点
2. 对上面的slave节点执行```slaveof no one``` 命令让其成为master节点
3. 向剩余的slave节点发送命令，让他们成为新master节点的slave节点，复制规则和parallel-syncs参数有关
4. 更新原来的master节点并配置为slave,并保持对其"关注"，当其恢复后，命令他去复制新的master节点

#### 选择"合适的"slave节点

1. 选择slave-priority(slave节点优先级)最高的slave节点。如果存在则返回，不存在则继续
2. 选择复制偏移量最大的slave节点(复制的最完整)如果存在则返回，不存在则继续
3. 选择runid最小的slave节点



