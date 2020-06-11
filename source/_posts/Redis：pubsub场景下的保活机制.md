---
title: Redis：pubsub场景下的保活机制
date: 2020-04-13
categories:
  - 技术
tags: 
  - PHP
  - Redis
  - TCP/IP
---

上一篇[{% post_link 【Redis】brpop保活机制 %}]，排查了redis pop list场景下断网引发``read error on connection``错误的情况，文末留下一个疑问，就是在pubsub场景下，该如何应对断网？（客户端和服务端不可达，触发保活机制）

**在这篇文章，我们假设网络在某时会中断十几分钟**

<!-- more -->

## 问题复现

### 我的环境

- php=7.1.22
- phpredis=5.0.2
- redis-server=4.0.14

可以通过服务端 ``client kill ip:port``，客户端报异常``read error on connection``。

这里抛出疑问：服务端主动kill的？何时会kill？

### 因为系统的tcp保活机制被kill？

先看系统的tcp_keepalive配置

```
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 75
```

即30+9*75=705s=11min45s，如果网络在``12分钟内``保持断开状态，服务端就会kill掉客户端。**可以解释的通，但是实测过程中发现，并没有使用系统的设置定时发ack**

### 因为redis服务端的保活机制被kill?

查看服务端redis.conf配置，tcp_keepalive被设置为``600``

注释：
>TCP keepalive.
>If non-zero, use SO_KEEPALIVE to send TCP ACKs to clients in absence
>of communication. This is useful for two reasons:
>1) Detect dead peers.
>2) Take the connection alive from the point of view of network
>   equipment in the middle.
>On Linux, the specified value (in seconds) is the period used to send ACKs.
>Note that to close the connection the double of the time is needed.
>On other kernels the period depends on the kernel configuration.
>A reasonable value for this option is 300 seconds, which is the new
>Redis default starting with Redis 3.2.1.

含义（[官网](https://redis.io/topics/clients)）
>Recent versions of Redis (3.2 or greater) have TCP keepalive (SO_KEEPALIVE socket option) enabled by default and set to about 300 seconds. This option is useful in order to detect dead peers (clients that cannot be reached even if they look connected). Moreover, if there is network equipment between clients and servers that need to see some traffic in order to take the connection open, the option will prevent unexpected connection closed events.

由上面的含义，可以提出三种可能的解释：
1. 这个参数覆盖系统的tcp_keepalive_time，并结合系统的tcp_keepalive_probes和tcp_keepalive_intvl，即600+9*75=1275s=21min15s（时间长了点）
2. 每隔600s，进行一次系统的保活机制（600s+11min45s=21min45s，时间长了点）
3. **服务端在600s后会给客户端发ack。也就是10min。（可以解释，实测中发现是这种）**

## 客户端选项的``OPT_KEEP_ALIVE``到底是怎么使用的？

上篇文章我们已经发现了，它是一个开关，但是有两个疑问：
1. 它用来启动服务端对客户端的保活？
2. 它用来启动客户端对服务端的保活？(实测发现，是这个。**使用系统的tcp_keepalive配置，与redis.conf无关**)

实测发现，如果在12分钟内网络保持断开状态（系统的tcp_keepalive配置为11min45s），客户端subscribe则会抛出``read error on connection``异常。这很好，我们可以捕获这个异常，重启客户端。

但是，由于不清楚网络何时会恢复，如果断开时间较短，可以借助``supervisor``等管理工具，在程序异常退出后不断拉启即可。如果断开时间较长，或许需要像上一篇文章[{% post_link 【Redis】brpop保活机制 %}]那样，做一个``忙连接``程序，不断去尝试重连服务端。

### pubsub场景下的忙连接程序

```php

function main()
{
    $redis = new Redis();
    $redis->connect($host, $port);

    while (true) {
        try {
            $redis->subscribe($chans, [$this, 'process']);
        } catch (Exception $exception) {
            // 忙连接等待网络恢复
            $redis = $this->reconnect();
        }
    }
}

function reconnect() 
{
    $redis = null;
    $isLostConnect = true;
    while ($isLostConnect) {
        try {
            $redis = new Redis();
            $redis->connect($host, $port);

            // 重连成功
            if ($redis->ping()) {
                $isLostConnect = false;
            }
        } catch (\Exception $e) {
            // 阻塞等待几秒钟，再次尝试重连
            sleep(10);
        }
    }

    return $redis;
}

main();
```


## phpredis保活机制原理

phpredis源码：

```c
/* Don't set TCP_KEEPALIVE if we're using a unix socket. */
if (ZSTR_VAL(redis_sock->host)[0] == '/' && redis_sock->port < 1) {
    RETURN_FALSE;
}
tcp_keepalive = zval_get_long(val) > 0 ? 1 : 0;
if (redis_sock->tcp_keepalive == tcp_keepalive) {
    RETURN_TRUE;
}
if (redis_sock->stream) {
    /* set TCP_KEEPALIVE */
    sock = (php_netstream_data_t*)redis_sock->stream->abstract;
    if (setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*)&tcp_keepalive,
                sizeof(tcp_keepalive)) == -1) {
        RETURN_FALSE;
    }
    redis_sock->tcp_keepalive = tcp_keepalive;
}
```

可以发现，phpredis并没有提供``SOL_TCP``字段选项的配置，即

1. TCP_KEEPIDLE（等同于tcp_keepalive_time）
2. TCP_KEEPINTVL（等同于tcp_keepalive_intvl）
3. TCP_KEEPCNT（等同于tcp_keepalive_probes）

所以，**phpredis默认使用系统内核提供的tcp_keepalive配置**，这意味着，如果需要修改保活时间，需要更改系统配置...

## 小结

1. 服务端保活，通过设置redis.conf里面的tcp-keepalive配置，即可定时检测连接是否有效（新版本默认为300s），如果不可达，则移除客户端连接（kill client），移除连接的时候，客户端则保持"Established"(``半连接状态``)。对pubsub程序来说，客户端无法接收到服务端的消息。
2. 客户端保活，通过setOption配置``OPT_KEEP_ALIVE``开启，使用系统tcp_keepalive配置，定时发送ack包验证连接有效，如果不可达，则同样抛出``read error on connection``异常，程序需要捕获异常并重连服务端，连接成功后，打开subscribe。
3. phpredis的客户端保活不支持自定义``SOL_TCP``，使用系统内核的网络配置。

## 参考

[TCP Keepalive HOWTO](https://zzyongx.github.io/blogs/tcp-keepalive-howto.html)