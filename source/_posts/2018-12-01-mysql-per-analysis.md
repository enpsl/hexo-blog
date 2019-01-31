---
layout:     post
title:      "mysql语句查询性能分析"
subtitle:   "为什么我只查一行的语句，也执行这么慢"
date:       2018-12-01 19:30:00
author:     "Psl"
header-img: "img/post-kejidanao.jpg"
catalog:    true
copyright: true
tags:
  - mysql
---

## 前言
一般情况下，如果谈起查询性能优化，多数人的第一感知都是想到一些复杂的语句，想到查询需要返回大量的数据。
但有些情况下，“查一行”，也会执行得特别慢。今天，就来探讨这个问题，看看什么情况下，会出现这个现象。

当然，如果 MySQL 数据库本身就有很大的压力，导致数据库服务器 CPU 占用率很高或 ioutil（IO 利用率）很高，
这种情况下所有语句的执行都有可能变慢，不属于今天的探讨范围。

## 正题
首先，先做一个测试表在插入1万条数据
```mysql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=10000)do
    insert into t1 values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

### 几种场景类型分析

#### 第一类：查询长时间不返回
执行下面的sql
```
mysql> select * from t1 where id=1;
```
查询长时间不返回

![](https://enpsl.github.io/img/in-post/2018-12-01/1.png)

一般碰到这种情况的话，大概率是表 t1 被锁住了。接下来分析原因的时候，一般都是首先执行一下 show processlist 命令，看看当前语句处于什么状态。

然后再根据状态去分析产生的原因，以及能否复现

**等MDL锁**

如下图所示，就是使用 show processlist 命令查看 Waiting for table metadata lock 的示意图。
![](https://enpsl.github.io/img/in-post/2018-12-01/2.png)
这个状态表示的是，现在**有一个线程正在表 t1 上请求或者持有 MDL 写锁，把 select 语句堵住了。**

不过，在 MySQL 5.7 版本下复现这个场景，也很容易。
<table><tr><th>SessionA</th><th>SessionB</th></tr><tr><td>lock table t1 write</td><td></td></tr><tr><td></td><td>select * from t1 where id=1;</td></tr></table>

session A 通过 lock table 命令持有表 t1 的 MDL 写锁，而 session B 的查询需要获取 MDL 读锁。所以，session B 进入等待状态。

这类问题的处理方式，就是找到谁持有 MDL 写锁，然后把它 kill 掉。

但是，由于在 show processlist 的结果里面，session A 的 Command 列是“Sleep”，导致查找起来很不方便。不过有了 performance_schema 和 sys 系统库以后，就方便多了。（MySQL 启动时需要设置 performance_schema=on，相比于设置为 off 会有 10% 左右的性能损失)

查看是否支持performance_schema
```sql
select * from information_schema.engines where engine ='performance_schema';
```
![](https://enpsl.github.io/img/in-post/2018-12-01/3.png)
是否开启performance_schema
```sql
show variables like 'performance_schema';
```
![](https://enpsl.github.io/img/in-post/2018-12-01/4.png)

通过查询 sys.schema_table_lock_waits 这张表，我们就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。

**等FLUSH**

另一种场景
我在表t1上执行这样一条sql
```
mysql> select * from information_schema.processlist where id=1;
```
查出来这个线程的状态是 Waiting for table flush

这个状态表示的是，现在有一个线程正要对表 t1 做 flush 操作。MySQL 里面对表做 flush 操作的用法，一般有以下两个：

```
flush tables t1 with read lock;
flush tables with read lock;
```

这两个 flush 语句，如果指定表 t1 的话，代表的是只关闭表 t1；如果没有指定具体的表名，则表示关闭 MySQL 里所有打开的表。

但是正常这两个语句执行起来都很快，除非它们也被别的线程堵住了。

所以，出现 Waiting for table flush 状态的可能情况是：有一个 flush tables 命令被别的语句堵住了，然后它又堵住了我们的 select 语句。

复现一下这种情况:
<table><tr><th>SessionA</th><th>SessionB</th><th>SessionC</th></tr><tr><td>select sleep(1) from t1</td><td></td><td></td></tr><tr><td></td><td>flush table t1</td><td></td></tr><tr><td></td><td></td><td>select * from t1 where id=1;</td></tr></table> 
在 session A 中，我故意每行都调用一次 sleep(1)，这样这个语句默认要执行 1 万秒，在这期间表 t 一直是被 session A“打开”着。然后，session B 的 flush tables t 命令再要去关闭表 t，就需要等 session A 的查询结束。这样，session C 要再次查询的话，就会被 flush 命令堵住了。
 
下面是show processlist 结果
 ![](https://enpsl.github.io/img/in-post/2018-12-01/5.png)
 
 **等行锁**
 
现在，经过了表级锁的考验，我们的 select 语句终于来到引擎里了。

```
mysql> select * from t1 where id=1 lock in share mode; 
```
由于访问 id=1 这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的 select 语句就会被堵住。

行锁操作复现:
<table><tr><th>SessionA</th><th>SessionB</th></tr><tr><td>begin;<br>update t1 set c=c+1 where id=1;</td><td></td></tr><tr><td></td><td>select * from t1 where id=1 lock in share mode;</td></tr></table>
下面是show processlist 结果

 ![](https://enpsl.github.io/img/in-post/2018-12-01/6.png)

显然，session A 启动了事务，占有写锁，还不提交，是导致 session B 被堵住的原因。

这个问题并不难分析，但问题是怎么查出是谁占着这个写锁。如果你用的是 MySQL 5.7 版本，可以通过 sys.innodb_lock_waits 表查到。

查询方法：
```
mysql> select * from t1 sys.innodb_lock_waits where locked_table='`test`.`t1`'\G
```
 ![](https://enpsl.github.io/img/in-post/2018-12-01/8.png)
 
可以看到，这个信息很全，9957 号线程是造成堵塞的罪魁祸首。而干掉这个罪魁祸首的方式，就是 KILL QUERY 9957 或 KILL 9957。

不过，这里不应该显示“KILL QUERY 9957”。这个命令表示停止 9957 号线程当前正在执行的语句，而这个方法其实是没有用的。因为占有行锁的是 update 语句，这个语句已经是之前执行完成了的，现在执行 KILL QUERY，无法让这个事务去掉 id=1 上的行锁。

实际上，KILL 9957 才有效，也就是说直接断开这个连接。这里隐含的一个逻辑就是，连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了 id=1 上的行锁。

#### 第二类：查询慢

经过了重重封“锁”，再来看看一些查询慢的例子

```sql
mysql> select * from t1 where c=9000 limit 1;
```

由于字段 c 上没有索引，这个语句只能走 id 主键顺序扫描，因此需要扫描 5 千行。

通过slow_log 看一下
```sql
set global slow_query_log=ON;
set long_query_time=0;  #我们让所有查询都加入到slow_log中
```

```sql
show variables like '%slow%';
```
 ![](https://enpsl.github.io/img/in-post/2018-12-01/10.png)
 
 slow_log结果
 
  ![](https://enpsl.github.io/img/in-post/2018-12-01/11.png)

Rows_examined 显示扫描了 9000 行。你可能会说，不是很慢呀，6 毫秒就返回了，我们线上一般都配置超过 1 秒才算慢查询。但你要记住：
**坏查询不一定是慢查询**。我们这个例子里面只有 1 万行记录，数据量大起来的话，执行时间就线性涨上去了。

扫描行数多，所以执行慢，这个很好理解。

但是接下来，我们再看一个只扫描一行，但是执行很慢的语句。

看下面的例子
<table><tr><th>SessionA</th><th>SessionB</th></tr><tr><td>start transaction with consistent snapshot;</td><td></td></tr><tr><td></td><td>update t1 set c=c+1 where id=1;//执行100万次</td></tr><tr><td>select * from t1 where id = 1;</td><td></td></tr><tr><td>select * from t1 where id = 1 lock in share mode;</td><td></td></tr></table>
先看看执行一次的结果

![](https://enpsl.github.io/img/in-post/2018-12-01/12.png)

由此可推出SessionB执行100万次后结果
select * from t1 where id = 1 lock in share mode;
<table style="width:15%"><tr><th>id</th><th>c</th></tr><tr><td>1</td><td>1000001</td></tr></table>
1 row in set (0.00 sec)

此时，session A 先用 start transaction with consistent snapshot 命令启动了一个事务，之后 session B 才开始执行 update 语句。

session B 更新完 100 万次，生成了 100 万个回滚日志 (undo log)。

带 lock in share mode 的 SQL 语句，是当前读，因此会直接读到 1000001 这个结果，所以速度很快；而 select * from t where id=1 这个语句，是一致性读，因此需要从 1000001 开始，依次执行 undo log，执行了 100 万次以后，才将 1 这个结果返回。


#### 小结
今天列举了在一个简单的表上，执行“查一行”，可能会出现的被锁住和执行慢的例子。这其中涉及到了表锁、行锁和一致性读的概念。
在实际使用中，碰到的场景会更复杂。但大同小异。

> 参考
> 
> -  《极客时间林晓斌Mysql专场》