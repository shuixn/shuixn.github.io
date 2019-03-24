---
title: redis的对端心跳检测
date: 2019-03-24
categories:
  - 技术
tags: 
  - PHP 
  - Redis
  - TCP
---

## 写在前

这段时间在做``基于Redis的发布订阅``时遇到一个有意思的问题，客户端无论使用php扩展``phpredis``还是原生php写的库``predis``做subscribe时，都会在一段时间后（30min左右），当发生一次``publish``后，redis-server断开客户端的socket连接，而客户端没有任何异常（仍为ESTABLISH）

一开始我以为是客户端的问题，但是在一系列的测试后，确认应该是服务端的问题，最后在师兄在帮助下，找到了redis-server的一个配置``tcp-keepalive``，这个值当时为0，后来我们设置为300，程序正常了。

## 基于redis的发布订阅程序

![发布订阅](/images/发布订阅.png)

这个服务非常简单，从上图可知，以Redis-Server为界分为上下两部分，上部分通过Web程序下指令，发布消息到指定的Channel，下部分是一个常驻进程服务，在Redis-Server订阅了某个Channel，并阻塞在Subscribe事件回调中，当有消息来，进行消息处理。

## php实现订阅者(Subscriber.php)

```php
$channel = 'test';

function process($redis, $chan, $msg){
    var_dump(date('Y-m-d H:i:s', time()), $msg);
}

$redis = new \Redis();
$redis->connect($config['host'], $config['port']);
$redis->auth($config['password']);
$redis->select($config['db']);
$redis->setOption(Redis::OPT_READ_TIMEOUT, -1);
$redis->subscribe([$channel], 'process');
```

为了减少``变量``，说明问题，上面是一段非常简单的代码。简单说明下，这是一段cli程序，运行在前台，订阅channel为test，并阻塞在process函数中，当有消息过来，会在前台打印当前时间以及消息。

## php实现发布者(Publisher.php)

```php
$channel = 'test';
$message = 'hello';

$redis = new \Redis();
$redis->connect($config['host'], $config['port']);
$redis->auth($config['password']);
$redis->select($config['db']);
$redis->publish($channel, $message);
```

## 为什么我说不是客户端的问题

Subscriber执行后，找到Redis的连接socket

```bash
## 找到进程ID
ps -ef | grep Subscriber

## 找到进程ID占用的端口号
sudo netstat -nap | grep pid
```

由于前面设置的是``$config['port']``，所以可以看到这个端口下的状态是``ESTABLISH``。

这时，还可以做一个小操作，前面我们知道了进程ID，strace一下

```bash
sudo strace -p pid
```

这样一来，当有消息过来，我们可以更清楚地看到整个调用过程，这里可以排除一个问题 —— 客户端主动断开。有些网上的文章会告诉你，这是客户端的问题，订阅端需要设置一些长连接的参数，比如php的

```php
ini_set('default_socket_timeout', -1)
```

又比如，phpredis的

```php
$redis->setOption(Redis::OPT_READ_TIMEOUT, -1);
```

因为phpredis默认的超时时间是60秒，所以这里需要一些配置，上面两者选一种即可，都可以去掉超时限制，我这里用的就是第二种。

接下来就是等待了，前言说了，大约在30分钟后，当我发布一次消息后，Redis-Server的客户端socket会自动断开，导致没有消息到达订阅端。在此之前，我需要确保Redis-Server是有我们这个客户端存活着的，也就是说，我们需要知道事件发生前后，client的状态。这里可以通过``redis-cli``来看

假设Subscriber所在的机器与订阅端口为``192.168.8.8:6379``，这是假设的情况，实际上端口是不一样的。

```bash
redis-cli -h xxx -p 6379 -a xxx -n 1 client list | grep 192.168.8.8:6379
```

这个命令可以知道Redis-Server的客户端连接状态，包括连接时间、空闲时间、内存、事件等等。Subscriber程序的事件当然就是subscribe了，可以很容易看出来。运行这个命令后，我们发现这个client socket是非常健康的。

经过一段时间的等待，我们再回来一一检查过、确认过：

1. 客户端Subscriber并没有down掉
2. 客户端连接socket的状态是ESTABLISH，也是正常的
3. Redis-Server的client list下也能查到我们的客户端状态

然后，我们执行一次Publisher程序，问题出现了：

1. Subscriber程序没有如期打印出当前日期和消息，但是也没有报错
2. 客户端连接socket的状态是ESTABLISH，仍是正常的
3. Redis-Server的client list下，多打印几次后，我们的客户端就断开了、消失了

以上就是问题的全部了:)

## phpredis扩展的问题？

一开始我也怀疑过，于是换成了php原生写的``predis``库，写了几乎一样的代码，然而并没有用，还是一样的问题。

## 转移目标

既然客户端没问题，会不会是我们的客户端连接socket在Redis所在机器被别的程序kill掉了，有可能。重来一遍，这一次我不等30分钟了，直接在Redis服务端把我的客户端kill掉试试看

```bash
redis-cli -h xxx -p 6379 -a xxx -n 1 client kill 192.168.8.8:6379
```

这时，Subscriber报了异常，然后退出了！！这种情况说明，如果服务端有程序想要把我的客户端kill掉，客户端是会响应的，但是现在的情况并不是这样，还要继续看。

## TCP心跳检测

大胆的猜测：如果程序端有断开的行为，那么Redis-Server会做出相应的行为来告知客户端，但如果是网络层的断开呢？

后来在师兄的帮助下，逐一排查了redis.conf的配置，找到了一个参数``tcp-keepalive``。我们知道TCP的``keep-alive``和HTTP的``keepalive``是两个东西。TCP的keep-alive是``socket对端心跳检测``，用来预防客户端断开后服务端资源浪费的问题。

redis.conf中，tcp-keepalive为0，这里的意思是不做对端心跳检测，那么在一定时间后，服务端就会回收这些idle的socket资源。

后来我们改为300，意思是每隔300秒，发送一个心跳包（TCP控制包），如果客户端返回ACK，那么服务端就认为你是正常的健康的客户端，可以留着。实际上情况没那么简单，如果写过网络编程的可以知道，还有另外两个参数在控制着

```
tcp_keepalive_time // 距离上次传送数据多少时间未收到判断为开始检测
tcp_keepalive_intvl // 检测开始每多少时间发送心跳包
tcp_keepalive_probes // 发送几次心跳包对方未响应则close连接
```

## 再说发布订阅

前面我们的发布订阅写的非常简单，真实情况还需要考虑更多的问题，简单说一个，如果订阅者无故down了，还在重连中时，发布者发了一个消息，如果没有其他的订阅者，那这个消息就发布失败了，看起来非常不健壮。

这里可以考虑利用redis加一个消息队列做``持久化``，每一次发消息时，都push一个消息进队列，如果订阅者没有及时重连也没关系，重连成功后，可以去队列中把消息pop出来继续处理。

![发布订阅-消息队列](/images/发布订阅-消息队列.png)

## 小结

- redis发布订阅可能会遇到的问题
- 排查进程间通信问题的小工具
- TCP心跳机制
- 发布订阅的一点健壮性思考