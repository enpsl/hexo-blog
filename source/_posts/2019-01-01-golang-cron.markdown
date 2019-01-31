---
layout:     post
title:      "golang 调度多个cron"
subtitle:   "golang 调度多个cron的简易demo"
date:       2019-01-01 19:00:00
author:     "Psl"
header-img: "img/post-bg-alitrip.jpg"
catalog:    true
tags:
  - Linux
  - golang
  - cron
---

## 类库引用
>下载cron包到$Gopath下
```bash
pengshiliang@pengshiliang-virtual-machine:go get github.com/gorhill/cronexpr
```
>导入
```golang
import "github.com/gorhill/cronexpr"
```

## 一个简单的cron实例

```golang
    var (
        expr *cronexpr.Expression
        err error
        now time.Time
        nextTime time.Time
        )
    // golang可支持到秒粒度, 年配置(2018-2099)
    // 哪一分钟（0-59），哪小时（0-23），哪天（1-31），哪月（1-12），星期几（0-6）
    
    // 每隔5秒钟执行1次
    if expr, err = cronexpr.Parse("*/5 * * * * * *"); err != nil {
        fmt.Println(err)
        return
    }
    
    // 当前时间
    now = time.Now()
    // 下次调度时间
    nextTime = expr.Next(now)
    
    // 等待这个定时器超时
    time.AfterFunc(nextTime.Sub(now), func() {
        fmt.Printf("当前时间：%+v 调度时间:%+v\n", now, nextTime)
    })
    
    time.Sleep(5 * time.Second)
```
输出结果：
```bash
pengshiliang@pengshiliang-virtual-machine:~/code/go/src/cron$ go run sim.go 
当前时间：2019-01-01 19:57:30.179548305 +0800 CST m=+0.000772719 调度时间:2019-01-01 19:57:35 +0800 CST
```

## 多任务版

```golang
type CronJob struct {
	expr *cronexpr.Expression
	nextTime time.Time
}

var (
	expr *cronexpr.Expression
	now time.Time
	cronJob *CronJob
	cronSchedule map[string]*CronJob
)

func main() {
	cronSchedule = make(map[string]*CronJob)
	//当前时间
	now = time.Now()
	//添加任务job1到cronSchedule中
	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		expr: expr,
		nextTime:expr.Next(now),
	}
	cronSchedule["job1"] = cronJob
	//添加任务job2到cronSchedule中
	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		expr: expr,
		nextTime:expr.Next(now),
	}
	cronSchedule["job2"] = cronJob
	// 启动一个调度协程
	go func() {
		var (
			jobName string
			cronJob *CronJob
			now time.Time
		)
		// 定时检查一下任务调度表
		for {
			now = time.Now()

			for jobName, cronJob = range cronSchedule {
				// 判断是否过期
				if cronJob.nextTime.Before(now) || cronJob.nextTime.Equal(now) {
					// 启动一个协程, 执行这个任务
					go func(jobName string) {
						fmt.Printf("当前执行时间:%+v执行任务%s\n", now, jobName)
					}(jobName)
					// 计算下一次调度时间
					cronJob.nextTime = cronJob.expr.Next(now)
					fmt.Println(jobName, "下次执行时间:", cronJob.nextTime)
				}
			}
		}
	}()
	//防止main协程退出
	time.Sleep(100 * time.Second)
}
```
输出结果:
```bash
pengshiliang@pengshiliang-virtual-machine:~/code/go/src/cron/demo$ go run test1.go
job2 下次执行时间: 2019-01-01 20:04:45 +0800 CST
job1 下次执行时间: 2019-01-01 20:04:45 +0800 CST
当前执行时间:2019-01-01 20:04:40.000827793 +0800 CST m=+0.107461981执行任务job2
当前执行时间:2019-01-01 20:04:40.00104295 +0800 CST m=+0.107677140执行任务job1
job1 下次执行时间: 2019-01-01 20:04:50 +0800 CST
job2 下次执行时间: 2019-01-01 20:04:50 +0800 CST
当前执行时间:2019-01-01 20:04:45.000103213 +0800 CST m=+5.106737398执行任务job1
当前执行时间:2019-01-01 20:04:45.000204198 +0800 CST m=+5.106838391执行任务job2
当前执行时间:2019-01-01 20:04:50.000000023 +0800 CST m=+10.106634228执行任务job1
job1 下次执行时间: 2019-01-01 20:04:55 +0800 CST
job2 下次执行时间: 2019-01-01 20:04:55 +0800 CST
当前执行时间:2019-01-01 20:04:50.001381032 +0800 CST m=+10.108015219执行任务job2
job1 下次执行时间: 2019-01-01 20:05:00 +0800 CST
job2 下次执行时间: 2019-01-01 20:05:00 +0800 CST
当前执行时间:2019-01-01 20:04:55.000000033 +0800 CST m=+15.106634234执行任务job1
当前执行时间:2019-01-01 20:04:55.001663389 +0800 CST m=+15.108297584执行任务job2
...
```
> cronexpr.MustParse()和cronexpr.Parse()区别,cronexpr.Parse()会多返回一个error变量，
如果确定接收到的cron表达式正确的话可以选择用MustParse

