---
layout:     post
title:      "php swoole进程"
subtitle:   "php swoole进程demo"
date:       2018-01-02 22:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - php
  - swoole
  - linux
---

## 前言

最近看了下swoole的相关文档，发现了它弥补了php以前再做异步通信方面的不足，也正如官方文档所说：
是一个"面向生产环境的 PHP 异步网络通信引擎"。

## 那为什么要使用 Swoole？
主要有一下几点：

* 常驻内存，避免重复加载带来的性能损耗，提升海量性能
* 协程异步，提高对 I/O 密集型场景并发处理能力（如：微信开发、支付、登录等）
* 方便地开发 Http、WebSocket、TCP、UDP 等应用，可以与硬件通信
* PHP 高性能微服务架构成为现实

当然今天主要介绍的还是swoole得Process模块得基础学习

## 进入正题

### Process 出现的意义
在1.7.2版本时,swoole增加了一个进程管理模块，用来替代PHP的pcntl。

PHP自带的pcntl，存在很多不足，如：

* pcntl没有提供进程间通信的功能
* pcntl不支持重定向标准输入和输出
* pcntl只提供了fork这样原始的接口，容易使用错误
* swoole_process提供了比pcntl更强大的功能，更易用的API，使PHP在多进程编程方面更加轻松。

Swoole\Process提供了如下特性：

* 基于Unix Socket和sysvmsg消息队列的进程间通信，只需调用write/read或者push/pop即可
* 支持重定向标准输入和输出，在子进程内echo不会打印屏幕，而是写入管道，读键盘输入可以重定向为管道读取数据
* 配合Event模块，创建的PHP子进程可以异步的事件驱动模式
* 提供了exec接口，创建的进程可以执行其他程序，与原PHP父进程之间可以方便的通信

### 一个简单的Process实例
创建一个http.php脚本:

```php
use Swoole\Http\Server;
$http = new Server("0.0.0.0", 9000);
$http->set(array(
    'worker_num' => 8,    //worker process num
));
$http->on('request', function ($request, $response) {
    $response->end("<h1>Hello Swoole. #".rand(1000, 9999)."</h1>");
});
$http->start();
```
创建一个process.php脚本:

```php
$process = new swoole_process('callback_function', true);   //true 表示输出到管道中而不输出到屏幕中

$pid = $process->start();
function callback_function(swoole_process $worker)
{
    $worker->exec('/usr/local/php/bin/php', [__DIR__.'/http.php']);
}
echo $pid . PHP_EOL;
swoole_process::wait();
```

在终端执行`php process` 后我们会得到一个 php process 的子进程id
![](https://enpsl.github.io/img/in-post/2018-01-02/1.png)
 
 通过`ps -aux|grep process.php`我们知道了process脚本的进程号
![](https://enpsl.github.io/img/in-post/2018-01-02/2.png)
  
查看process 进程的详细信息：
```bash
pstree -p 3700
```
![](https://enpsl.github.io/img/in-post/2018-01-02/3.png)

可以很容易发现出执行`php process` 输出的进程号3701是当前脚本的一个管理子进程的进程，
它的下面有4个子进程和一个管理http_server(3702)的子进程``` $worker->exec('/usr/local/php/bin/php', [__DIR__.'/http.php']);```
而http_server下面刚好有8个进程（我们在http_server中配置了8个进程,不配置默认4个进程）,这就很好解释了这颗进程树产生的原因。

那么如何实现多进程呢？

### 多进程实现

创建一个worker.php脚本:
```php
function add($i) {
    $i++;
    return $i;
}

//创建6个子进程
for ($i = 0; $i < 6; $i++) {
    $process = new swoole_process(function (swoole_process $worker) use($i, $arr) {
        $n = add($i);
        //将函数返回结果写入到管道中
        $worker->write($n);
    }, true);
    $pid = $process->start();
    $workers[$pid] = $process;
}

foreach ($workers as $worker) {
    //遍历worker读取管道中的结果
    var_dump($worker->read());
}
```

执行结果：
![](https://enpsl.github.io/img/in-post/2018-01-02/4.png)

很好如预期所示

那么如果管道里传入的是一个数组呢？是否可以像golang的channel可以写入任意类型的数据呢？

```php
function array_add_str($arr, $i){
    $arr[] = $i;
    return $arr;
}

$arr = [
    "hello",
    "world"
];

for ($i = 0; $i < 6; $i++) {
    $process = new swoole_process(function (swoole_process $worker) use($i, $arr) {
        //$n = add($i);
        $arr = array_add_str($arr, $i);
        $worker->write($arr);
    }, true);
    $pid = $process->start();
    $workers[$pid] = $process;
}

foreach ($workers as $worker) {
    var_dump($worker->read());
}
```
![](https://enpsl.github.io/img/in-post/2018-01-02/4.png)

由图可见：数组是不可以传递的，由此可推测出object等等其他复杂类型也是不可以的
我们只能通过序列化反序列化的方式去实现通信传输

## 小结 
通过一些代码的演练了解了process的一些使用方式，但是在使用过程中还需要注意：
**Process进程在系统是非常昂贵的资源，创建进程消耗很大。另外创建的进程过多会导致进程切换开销大幅上升。**

> 参考
> 
> -  [《Swoole官方文档》](https://wiki.swoole.com/wiki/page/p-process.html)





