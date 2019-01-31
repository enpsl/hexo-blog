---
layout:     post
title:      "Golang etcd api"
subtitle:   "Golang简单操作etcd"
date:       2019-01-05 9:30:00
author:     "Psl"
header-img: "img/post-bg-art.jpg"
catalog:    true
tags:
  - golang
  - etcd
---

## ETCD
ETCD是用于共享配置和服务发现的分布式，一致性的KV存储系统。ETCD是CoreOS公司发起的一个开源项目，授权协议为Apache。

核心特性：

* 将数据存储在集群中的高可用kv存储
* 允许应用实时监控kv变化
* 能够容忍单点故障，能够应对网络分区

复杂特性：

* 底层存储是按照key有序排列的，可以顺序遍历
* 因为key有序，所以etcd天然支持按目录结果高效遍历
* 支持复杂事物，提供if...then...else的事物能力
* 基于租约机制实现key的TTL过期

## 客户端连接实例

```golang
var (
        config clientv3.Config
        client *clientv3.Client
        err error
     )

config = clientv3.Config{
    Endpoints: []string{"127.0.0.1:2379"}, // 集群列表
    DialTimeout: 5 * time.Second,
}

if client, err = clientv3.New(config); err != nil {
    log.Printf("clientv3.New error:", err)
    return
}
```

## api介绍

**kv操作**

```golang
type KV interface {
    Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)

    Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)

    Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)

    Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)
    Do(ctx context.Context, op Op) (OpResponse, error)

    Txn(ctx context.Context) Txn
}
```
**关于租约**
```golang
type Lease interface {

    Grant(ctx context.Context, ttl int64) (*LeaseGrantResponse, error)

    Revoke(ctx context.Context, id LeaseID) (*LeaseRevokeResponse, error)


    TimeToLive(ctx context.Context, id LeaseID, opts ...LeaseOption) (*LeaseTimeToLiveResponse, error)

    KeepAlive(ctx context.Context, id LeaseID) (<-chan *LeaseKeepAliveResponse, error)

    KeepAliveOnce(ctx context.Context, id LeaseID) (*LeaseKeepAliveResponse, error)

    Close() error
}
```

### 创建、更新key
```golang
    var(
        kv clientv3.KV
        putResp *clientv3.PutResponse
    ) 
    kv = clientv3.NewKV(client)
	if putResp, err = kv.Put(context.TODO(), "/cron/jobs/job1",
		"bye", clientv3.WithPrevKV()); err != nil {
		log.Printf("kv.Put error:", err)
	} else {
		fmt.Println("Revision:", putResp.Header.Revision)
		if putResp.PrevKv != nil {
			fmt.Println("value:", string(putResp.PrevKv.Value))
		}
	}
```
<table><tr><th>SessionA</th></tr><tr><td>执行创建key脚本</td></tr><tr><td>再次执行创建key脚本</td></tr></table>

> clientv3.WithPrevKV()作用是可以获取put之前key的值；
操作示例第一次创建`hello`的key,第二次创建`bye`的key

输出结果：

![](https://enpsl.github.io/img/in-post/2019-01-05/2.png)

### 删除key
```golang
    var(
        kv clientv3.KV
        delResp *clientv3.DeleteResponse
        kvPair *mvccpb.KeyValue
    ) 
    if delResp, err = kv.Delete(context.TODO(), "/cron/jobs/job1", clientv3.WithPrevKV()); err != nil {
    		log.Printf("kv.delete error:", err)
    		return
    	}
    //被删除之前的value是什么
    if len(delResp.PrevKvs) != 0 {
        for _, kvPair = range delResp.PrevKvs {
            fmt.Println("删除了", string(kvPair.Key), string(kvPair.Value))
        }
    }
```
输出结果

![](https://enpsl.github.io/img/in-post/2019-01-05/7.png)

### 查询key
```golang
    var(
        kv clientv3.KV
        getResp *clientv3.GetResponse
    ) 
    if getResp, err = kv.Get(context.TODO(), "/cron/jobs/job1"); err != nil {
		log.Printf("kv.Put error:", err)
	} else {
		//fmt.Println(getResp.Kvs, getResp.Count)
	}

	if getResp, err = kv.Get(context.TODO(), "/cron/jobs/job1", clientv3.WithCountOnly()); err != nil {
		log.Printf("kv.Put error:", err)
	} else {
		fmt.Println(getResp.Kvs, getResp.Count)
	}
```
输出结果

![](https://enpsl.github.io/img/in-post/2019-01-05/5.png)

加入clientv3.WithCountOnly()输出结果

![](https://enpsl.github.io/img/in-post/2019-01-05/6.png)

> clientv3.WithCountOnly() 只输出count值

### watch key

简单demo：
>主要工作是开一个新的协程去模拟kv的删除更新操作,用main协程去监听key的put,delete操作，并在5秒后取消监听

```golang
    var (
		config clientv3.Config
		client *clientv3.Client
		err error
		kv clientv3.KV
		watcher clientv3.Watcher
		getResp *clientv3.GetResponse
		watchStartRevision int64
		watchRespChan <-chan clientv3.WatchResponse
		watchResp clientv3.WatchResponse
		event *clientv3.Event
	)

	// 客户端配置
	config = clientv3.Config{
		Endpoints: []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	}

	// 建立连接
	if client, err = clientv3.New(config); err != nil {
		fmt.Println(err)
		return
	}

	// KV
	kv = clientv3.NewKV(client)

	// 模拟etcd中KV的变化
	go func() {
		for {
			kv.Put(context.TODO(), "/cron/jobs/job3", "i am job3")

			kv.Delete(context.TODO(), "/cron/jobs/job3")

			time.Sleep(1 * time.Second)
		}
	}()

	// 先GET到当前的值，并监听后续变化
	if getResp, err = kv.Get(context.TODO(), "/cron/jobs/job3"); err != nil {
		fmt.Println(err)
		return
	}

	// 现在key是存在的
	if len(getResp.Kvs) != 0 {
		fmt.Println("当前值:", string(getResp.Kvs[0].Value))
	}

	// 当前etcd集群事务ID, 单调递增的
	watchStartRevision = getResp.Header.Revision + 1

	// 创建一个watcher
	watcher = clientv3.NewWatcher(client)

	// 启动监听
	fmt.Println("从该版本向后监听:", watchStartRevision)

	ctx, cancelFunc := context.WithCancel(context.TODO())
	time.AfterFunc(5 * time.Second, func() {
		cancelFunc()
	})

	watchRespChan = watcher.Watch(ctx, "/cron/jobs/job3", clientv3.WithRev(watchStartRevision))

	// 处理kv变化事件
	for watchResp = range watchRespChan {
		for _, event = range watchResp.Events {
			switch event.Type {
			case mvccpb.PUT:
				fmt.Println("修改为:", string(event.Kv.Value), "Revision:", event.Kv.CreateRevision, event.Kv.ModRevision)
			case mvccpb.DELETE:
				fmt.Println("删除了", "Revision:", event.Kv.ModRevision)
			}
		}
	}
```

打印结果:

![](https://enpsl.github.io/img/in-post/2019-01-05/10.png)

### 申请租约

简单demo：

```golang
    var (
		config clientv3.Config
		client *clientv3.Client
		err error
		lease clientv3.Lease
		leaseResp *clientv3.LeaseGrantResponse
		leaseId clientv3.LeaseID
		kv clientv3.KV
		putResp *clientv3.PutResponse
		getResp *clientv3.GetResponse
	)

	config = clientv3.Config{
		Endpoints: []string{"127.0.0.1:2379"}, // 集群列表
		DialTimeout: 5 * time.Second,
	}

	if client, err = clientv3.New(config); err != nil {
		log.Printf("clientv3.New error:", err)
		return
	}

	//申请一个4秒生周期的租约
	lease = clientv3.NewLease(client)
	if leaseResp, err = lease.Grant(context.TODO(), 4); err != nil {
		log.Printf("lease.Grant error:", err)
	}
	//拿到租约id
	leaseId = leaseResp.ID

	//put一个key
	kv = clientv3.NewKV(client)
	if putResp, err = kv.Put(context.TODO(), "/cron/jobs/job2",
		"bye", clientv3.WithLease(leaseId)); err != nil {
		log.Printf("kv.Put error:", err)
		return
	}
	fmt.Println("写入成功：", putResp.Header.Revision)
	//循环检测租约是否过期
	for  {
		if getResp, err = kv.Get(context.TODO(), "/cron/jobs/job2"); err != nil {
			log.Printf("kv.Put error:", err)
			return
		}

		if getResp.Count == 0 {
			fmt.Println("kv过期了")
			break
		}

		fmt.Println("还未过期：", getResp.Kvs)
		time.Sleep(5 * time.Second)
	}
```

打印结果:

![](https://enpsl.github.io/img/in-post/2019-01-05/8.png)

### 租约续约

```golang
    var (
        keepRespChan <-chan *clientv3.LeaseKeepAliveResponse
        keepResp *clientv3.LeaseKeepAliveResponse
    )
    //租约续租
	if keepRespChan, err = lease.KeepAlive(context.TODO(), leaseId); err != nil {
		fmt.Println(err)
		return
	}
	//消费keepRespChan
	go func() {
		for  {
			select {
			case keepResp = <-keepRespChan:
				if keepRespChan == nil {
					fmt.Println("租约已经失效了")
					goto END
				} else {	// 每秒会续租一次, 所以就会受到一次应答
					fmt.Println("收到自动续租应答:", keepResp.ID)
				}
			}
		}
		END:
	}()
```

打印结果:

![](https://enpsl.github.io/img/in-post/2019-01-05/9.png)

### op操作

```golang
    var (
		config clientv3.Config
		client *clientv3.Client
		err error
		kv clientv3.KV
		op clientv3.Op
		opResp clientv3.OpResponse
	)

	// 客户端配置
	config = clientv3.Config{
		Endpoints: []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	}

	// 建立连接
	if client, err = clientv3.New(config); err != nil {
		fmt.Println(err)
		return
	}

	// KV
	kv = clientv3.NewKV(client)

	op = clientv3.OpPut("/cron/jobs/job4", "opPut")
	if opResp, err = kv.Do(context.TODO(), op); err != nil {
		log.Printf("clientv3.OpPut error:", err)
	}

	fmt.Println("写入Revision：", opResp.Put().Header.Revision)

	op = clientv3.OpGet("/cron/jobs/job4")
	if opResp, err = kv.Do(context.TODO(), op); err != nil {
		log.Printf("clientv3.OpGet error:", err)
	}

	fmt.Println("读取Revision：", opResp.Get().Kvs[0].ModRevision)
	fmt.Println("读取Value：", string(opResp.Get().Kvs[0].Value))
```

打印结果:

![](https://enpsl.github.io/img/in-post/2019-01-05/11.png)

### 分布式锁

**上锁**

```golang
    lease = clientv3.NewLease(client)
	//申请一个5秒生周期的租约
	if leaseResp, err = lease.Grant(context.TODO(), 5); err != nil {
		log.Printf("lease.Grant error:", err)
	}
	//拿到租约id
	leaseId = leaseResp.ID

	// 准备一个用于取消自动续租的context
	ctx, cancelFunc = context.WithCancel(context.TODO())

	// 5秒后会取消自动续租
	if keepRespChan, err = lease.KeepAlive(ctx, leaseId); err != nil {
		fmt.Println(err)
		return
	}
```

**抢锁**

```golang
    // 创建事务
	txn = kv.Txn(context.TODO())

	// 定义事务

	// 如果key不存在
	txn.If(clientv3.Compare(clientv3.CreateRevision("/cron/lock/job5"), "=", 0)).
		Then(clientv3.OpPut("/cron/lock/job5", "xxx", clientv3.WithLease(leaseId))).
		Else(clientv3.OpGet("/cron/lock/job5")) // 否则抢锁失败

	// 提交事务
	if txnResp, err = txn.Commit(); err != nil {
		fmt.Println(err)
		return // 没有问题
	}

	// 判断是否抢到了锁
	if !txnResp.Succeeded {
		fmt.Println("锁被占用:", string(txnResp.Responses[0].GetResponseRange().Kvs[0].Value))
		return
	}
```

**处理业务**

```golang
	fmt.Println("处理任务")
	time.Sleep(5 * time.Second)
```

**释放锁**

```golang
    // defer 会把租约释放掉, 关联的KV就被删除了
	// 确保函数退出后, 自动续租会停止
	defer cancelFunc()
	defer lease.Revoke(context.TODO(), leaseId)
```

**完整Demo**
```golang
    var (
		config clientv3.Config
		client *clientv3.Client
		err error
		lease clientv3.Lease
		leaseResp *clientv3.LeaseGrantResponse
		leaseId clientv3.LeaseID
		kv clientv3.KV
		keepRespChan <-chan *clientv3.LeaseKeepAliveResponse
		keepResp *clientv3.LeaseKeepAliveResponse
		ctx context.Context
		cancelFunc context.CancelFunc
		txn clientv3.Txn
		txnResp *clientv3.TxnResponse
	)

	// 客户端配置
	config = clientv3.Config{
		Endpoints: []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	}

	// 建立连接
	if client, err = clientv3.New(config); err != nil {
		fmt.Println(err)
		return
	}

	// KV
	kv = clientv3.NewKV(client)

	// 1, 上锁 (创建租约, 自动续租, 拿着租约去抢占一个key)
	lease = clientv3.NewLease(client)
	//申请一个5秒生周期的租约
	if leaseResp, err = lease.Grant(context.TODO(), 5); err != nil {
		log.Printf("lease.Grant error:", err)
	}
	//拿到租约id
	leaseId = leaseResp.ID

	// 准备一个用于取消自动续租的context
	ctx, cancelFunc = context.WithCancel(context.TODO())

	// defer 会把租约释放掉, 关联的KV就被删除了
	// 确保函数退出后, 自动续租会停止
	defer cancelFunc()
	defer lease.Revoke(context.TODO(), leaseId)

	// 5秒后会取消自动续租
	if keepRespChan, err = lease.KeepAlive(ctx, leaseId); err != nil {
		fmt.Println(err)
		return
	}

	//处理续约应答的协程
	go func() {
		for  {
			select {
			case keepResp = <-keepRespChan:
				if keepRespChan == nil {
					fmt.Println("租约已经失效了")
					goto END
				} else {	// 每秒会续租一次, 所以就会受到一次应答
					fmt.Println("收到自动续租应答:", keepResp.ID)
				}
			}
		}
	END:
	}()

	//  if 不存在key， then 设置它, else 抢锁失败
	kv = clientv3.NewKV(client)

	// 创建事务
	txn = kv.Txn(context.TODO())

	// 定义事务

	// 如果key不存在
	txn.If(clientv3.Compare(clientv3.CreateRevision("/cron/lock/job5"), "=", 0)).
		Then(clientv3.OpPut("/cron/lock/job5", "xxx", clientv3.WithLease(leaseId))).
		Else(clientv3.OpGet("/cron/lock/job5")) // 否则抢锁失败

	// 提交事务
	if txnResp, err = txn.Commit(); err != nil {
		fmt.Println(err)
		return // 没有问题
	}

	// 判断是否抢到了锁
	if !txnResp.Succeeded {
		fmt.Println("锁被占用:", string(txnResp.Responses[0].GetResponseRange().Kvs[0].Value))
		return
	}

	// 2, 处理业务

	fmt.Println("处理任务")
	time.Sleep(5 * time.Second)
```
<table><tr><th>SessionA</th><th>SessionB</th></tr><tr><td>go run main.go</td><td>go run main.go</td></tr></table>
![](https://enpsl.github.io/img/in-post/2019-01-05/13.png)
![](https://enpsl.github.io/img/in-post/2019-01-05/12.png)

>SesionA先执行main.go先获得一个锁，随后执行SessionB main.go,会提示锁被占用

> 参考
> 
> -  [《Go Etcd Docs》](https://godoc.org/github.com/coreos/etcd/clientv3)