---
title: PHP在Web服务上的探索
date: 2018-03-20
categories:
  - 技术
tags: 
  - PHP 
  - 网络编程
---
## 传统模式

### cgi

早期的web程序其实都是一种cgi程序，cgi是什么？cgi程序又是什么？通用网关接口（Common Gateway Interface/CGI）是一种互联网技术，可以让一个客户端向服务器上的程序请求数据。CGI描述了服务器（nginx\apache）和请求处理程序(php)之间传输数据的一种标准。这是一种标准、协议，不是程序。cgi程序是实现了cgi标准的程序。

### fastcgi

cgi协议是这样的处理机制：请求来一个，便建起一个进程，这样的程序很容易实现，但是不好扩展，当并发量大了后，可能会有成千上万的进程，操作系统面对如此大的进程数，增加了进程上下文切换、新建以及销毁进程的开销，cpu资源占据居高不下，拖慢系统。内存上，由于进程处理完请求便销毁，地址空间无法共享与重用。

现在的php-fpm采用的是fastcgi协议，与cgi不同，fastcgi使用持续的进程（1 master,n workers）来处理一连串的请求。这些进程由fastcgi服务器(php-fpm)管理，而不是web服务器。 当进来一个请求时，web服务器把环境变量和这个页面请求通过一个socket比如fastcgi进程与web服务器(都位于本地）或者一个TCP connection（fastcgi进程在远端的server farm）传递给fastcgi进程。

### 黄金搭档

#### LAMP

php在apache中是以mod_php扩展的形式存在，必须依靠apache才能启动，当请求到达，apache判断是否是php程序，提交给php解释器进行处理，处理完并返回。

#### LNMP

nginx可以配置资源类型，当请求到达时，比如/index.html，这是静态资源，服务器会到配置的web root文件系统中寻找，然后返回这个文件。如果请求的是/index.php，nginx知道必须把这个交给php解释器，这时候，根据cgi规范，把该有的数据（查询字符串、post数据，http header等等）发到相应的fastcgi进程进行处理。

#### LANMP

LANMP使用nginx作为前端和代理服务器，利用nginx应对高并发的能力，真正的服务器可能有多个，比如apache。nginx把请求反向代理给后端的服务器。

## Swoole常驻内存模式

php在这之前都是属于cgi程序，依赖于php-fpm。php-fpm是同步阻塞模式，虽然进程复用，但是PHP是短生命周期的，一次请求结束，php的变量和对象就会销毁。相较之下，swoole应运而生了。swoole的http server是常驻内存的，而且worker为异步非阻塞，可以处理更大的请求数。

### swoole机制

swoole的reacter线程就好像nginx，异步非阻塞（epoll）的处理网络请求、协议，缓冲数据frame并拆分。然后把数据包通过unixsocket发送到worker（好像fpm的worker）进行处理。有时候我们需要处理一些IO耗时任务，可以通过投递task进程进行处理，比如消息队列，连接池。

### nginx和swoole

一个很好的思路是，仍然使用nginx作为前端服务器，然后把请求反向代理到swoole服务器。
```
root /data/code/app/static;
location / {
	proxy_http_version 1.1;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_pass http://127.0.0.1:9501;
}
```

## 协程

- 协作式
- 抢占式任务

进程和线程都是抢占式的，由操作系统调度上下文切换，保存上下文状态。

协程更像是编译器的魔法，通过一些代码使得进程中的代码段能够分段执行，每次调度都会从yield开始。

协程是用户态线程，理论上可以建起成百上千万个协程。相较于进程和线程，无需从用户态过渡到内核态，创建和切换的开销更低。

## php yield

在之前的文章中，我粗略的介绍了yield的使用，yield的语义有生成器的意思，生成器的意思是，从一端把数据传到另一端。而协程的实现正是yield的双向通信。

## swoole coroutine

swoole2.x已经完全在底层实现了协程调度，开发者在业务层无需再通过yield的关键字来显式执行调用，只需要以同步的方式编写代码，底层自动切换。

## swoft微服务

很多人由于PHP没有成熟的微服务框架而转到JAVA或者Go或者其他成熟的语言，swoole社区有一些人（swoft）把大家熟知的JAVA编程思想搬了过来，基于swoole2.x，全协程，AOP面向切面编程、服务治理，熔断、降级、负载、注册与发现，基于注解编程，看起来和JAVA非常像，通过composer把一切组件化。不过，1.0刚刚发布，需要开源社区的支持。看起来非常有前景。

## 后记

最近利用业余时间基于swoole，做了两个有趣的项目，欢迎star：）

1. [基于swoole websocket的聊天室](https://github.com/funsoul/funchat "基于swoole websocket的聊天室")
2. [基于swoole http server的小程序游戏后端服务](https://github.com/funsoul/funpinyin "基于swoole http server的小程序游戏后端服务")