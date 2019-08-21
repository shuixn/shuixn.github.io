---
title: 【Redis】排查 read error on connection 小记
date: 2019-08-13
categories:
  - 技术
tags: 
  - PHP
  - Redis
  - TCP/IP
---

## 从错误说起

版本信息
- php-7.1.x
- [phpredis-4.0.x](https://github.com/phpredis/phpredis/tree/4.0.2)

一个``PHP常驻内存进程``，连上``Redis``后，定时做``brpop``操作，阻塞时间为``10s``。问题出现在，几天（不定时）后，该进程就会
``僵死``，表现为：

1. ``netstat``下，php进程与redis建立的客户端连接仍在（ESTABLISHED)
2. 在客户机``tcpdump``，没有输出任何数据包信息（没有通信？)
3. ``strace``该php进程，并没有输出任何系统调用（阻塞在哪了？）
4. 查看redis-server，发现``client list``中，并不存在该client（被移除了？）

## phpredis客户端连接为何不断？

关于phpredis连接，有下面几个地方需要理解清楚

1. connect() 函数参数 timeout 为 0
2. ini_set('default_socket_timeout', -1)
3. setOption(\Redis::OPT_READ_TIMEOUT, -1)
4. pconnect

### connect 函数参数 timeout

参数：
- *host*: string. can be a host, or the path to a unix domain socket. Starting from version 5.0.0 it is possible to specify schema 
- *port*: int, optional  
- *timeout*: float, value in seconds (optional, default is 0 meaning unlimited)  
- *reserved*: should be NULL if retry_interval is specified  
- *retry_interval*: int, value in milliseconds (optional)  
- *read_timeout*: float, value in seconds (optional, default is 0 meaning unlimited)

这里的``timeout``表示``建立连接``时的超时时间，调用此函数时，客户端将与服务端进行三次握手，建立TCP连接。由于网络原因，可以指定一个超时时间，意思是，如果客户端和服务端在该``时间限制``内未能建立连接，则返回false

文件：redis.c 行：935

```c
PHP_METHOD(Redis, connect)
{
    if (redis_connect(INTERNAL_FUNCTION_PARAM_PASSTHRU, 0) == FAILURE) {
        RETURN_FALSE;
    } else {
        RETURN_TRUE;
    }
}
```

其中，redis_connect的函数原型为

```c
PHP_REDIS_API int redis_connect(INTERNAL_FUNCTION_PARAMETERS, int persistent);
```

persistent 为 ``0`` 表示不建立``持久连接``，下面会聊到``等于 1``的情况。说明``connect``函数建立的是``短连接``，当调用``close``函数时，连接就会关闭。看下面的源码确实如此，如果在建立连接前已经存在另一个连接，则关闭。

文件：redis.c 行：1011

```c
redis = PHPREDIS_GET_OBJECT(redis_object, object);
/* if there is a redis sock already we have to remove it */
if (redis->sock) {
    redis_sock_disconnect(redis->sock, 0);
    redis_free_socket(redis->sock);
}
```

### default_socket_timeout

这个配置可以在php.ini找到，文档注释很简单：``基于 socket 的流的默认超时时间（秒）``

redis是基于``tcp协议``的程序，所以这个配置也会对其造成影响。比如``read error on connection``错误，这是phpredis在执行get、brpop等操作时，如果在``default_socket_timeout``时间内不返回结果就会报这个错误。php.ini中默认为``60s``。可以在程序中使用内置函数``ini_set``在运行时修改。

### OPT_READ_TIMEOUT

phpredis版本的“default_socket_timeout”，通过这个值，一样可以达到同样的效果。那么如果同时设置了``default_socket_timeout``和``OPT_READ_TIMEOUT``，优先级是怎样的？

实测发现，**如果同时存在两个配置，优先使用``OPT_READ_TIMEOUT``的配置**，这样是合理的。

文件：redis_commands.c 行：3980

```c
case REDIS_OPT_READ_TIMEOUT:
    redis_sock->read_timeout = zval_get_double(val);
    if (redis_sock->stream) {
        read_tv.tv_sec  = (time_t)redis_sock->read_timeout;
        read_tv.tv_usec = (int)((redis_sock->read_timeout -
                                    read_tv.tv_sec) * 1000000);
        php_stream_set_option(redis_sock->stream,
                                PHP_STREAM_OPTION_READ_TIMEOUT, 0,
                                &read_tv);
    }
    RETURN_TRUE;
```

## pconnect的原理是什么？

文件：redis.c 行：947

```c
PHP_METHOD(Redis, pconnect)
{
    if (redis_connect(INTERNAL_FUNCTION_PARAM_PASSTHRU, 1) == FAILURE) {
        RETURN_FALSE;
    } else {
        RETURN_TRUE;
    }
}
```

建立连接时，先到``连接池``获取连接（最后一个），并移除最后一个连接实例。如果连接是活跃的（PHP_STREAM_OPTION_CHECK_LIVENESS），则直接返回。如果连接已失效，则建立新的连接。

文件：library.c 行：1828

```c
if (redis_sock->persistent) {
    if (INI_INT("redis.pconnect.pooling_enabled")) {
        p = redis_sock_get_connection_pool(redis_sock);
        if (zend_llist_count(&p->list) > 0) {
            redis_sock->stream = *(php_stream **)zend_llist_get_last(&p->list);
            zend_llist_remove_tail(&p->list);
            /* Check socket liveness using 0 second timeout */
            if (php_stream_set_option(redis_sock->stream, PHP_STREAM_OPTION_CHECK_LIVENESS, 0, NULL) == PHP_STREAM_OPTION_RETURN_OK) {
                redis_sock->status = REDIS_SOCK_STATUS_CONNECTED;
                return SUCCESS;
            }
            php_stream_pclose(redis_sock->stream);
            p->nb_active--;
        }

        int limit = INI_INT("redis.pconnect.connection_limit");
        if (limit > 0 && p->nb_active >= limit) {
            redis_sock_set_err(redis_sock, "Connection limit reached", sizeof("Connection limit reached") - 1);
            return FAILURE;
        }

        gettimeofday(&tv, NULL);
        persistent_id = strpprintf(0, "phpredis_%ld%ld", tv.tv_sec, tv.tv_usec);
    } else {
        if (redis_sock->persistent_id) {
            persistent_id = strpprintf(0, "phpredis:%s:%s", host, ZSTR_VAL(redis_sock->persistent_id));
        } else {
            persistent_id = strpprintf(0, "phpredis:%s:%f", host, redis_sock->timeout);
        }
    }
    
    tv.tv_sec  = (time_t)redis_sock->timeout;
    tv.tv_usec = (int)((redis_sock->timeout - tv.tv_sec) * 1000000);
    if (tv.tv_sec != 0 || tv.tv_usec != 0) {
        tv_ptr = &tv;
    }

    redis_sock->stream = php_stream_xport_create(host, host_len,
        0, STREAM_XPORT_CLIENT | STREAM_XPORT_CONNECT,
        persistent_id ? ZSTR_VAL(persistent_id) : NULL,
        tv_ptr, NULL, &estr, &err);

    if (persistent_id) {
        zend_string_release(persistent_id);
    }

    if (!redis_sock->stream) {
        if (estr) {
            redis_sock_set_err(redis_sock, ZSTR_VAL(estr), ZSTR_LEN(estr));
            zend_string_release(estr);
        }
        return FAILURE;
    }

    if (p) p->nb_active++;

    /* Attempt to set TCP_NODELAY/TCP_KEEPALIVE if we're not using a unix socket. */
    if (!usocket) {
        php_netstream_data_t *sock = (php_netstream_data_t*)redis_sock->stream->abstract;
        err = setsockopt(sock->socket, IPPROTO_TCP, TCP_NODELAY, (char*) &tcp_flag, sizeof(tcp_flag));
        PHPREDIS_NOTUSED(err);
        err = setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*) &redis_sock->tcp_keepalive, sizeof(redis_sock->tcp_keepalive));
        PHPREDIS_NOTUSED(err);
    }

    php_stream_auto_cleanup(redis_sock->stream);

    read_tv.tv_sec  = (time_t)redis_sock->read_timeout;
    read_tv.tv_usec = (int)((redis_sock->read_timeout - read_tv.tv_sec) * 1000000);

    if (read_tv.tv_sec != 0 || read_tv.tv_usec != 0) {
        php_stream_set_option(redis_sock->stream,PHP_STREAM_OPTION_READ_TIMEOUT,
            0, &read_tv);
    }
    php_stream_set_option(redis_sock->stream,
        PHP_STREAM_OPTION_WRITE_BUFFER, PHP_STREAM_BUFFER_NONE, NULL);

    redis_sock->status = REDIS_SOCK_STATUS_CONNECTED;

    return SUCCESS;
}
```

重点来了，注意看上面代码中这一段，先卖个关子，后面聊``tcp_keepalive``的时候会着重分析

```c
/* Attempt to set TCP_NODELAY/TCP_KEEPALIVE if we're not using a unix socket. */
if (!usocket) {
    php_netstream_data_t *sock = (php_netstream_data_t*)redis_sock->stream->abstract;
    err = setsockopt(sock->socket, IPPROTO_TCP, TCP_NODELAY, (char*) &tcp_flag, sizeof(tcp_flag));
    PHPREDIS_NOTUSED(err);
    err = setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*) &redis_sock->tcp_keepalive, sizeof(redis_sock->tcp_keepalive));
    PHPREDIS_NOTUSED(err);
}
```

## redis-server为什么会移除client？

先回顾一下TCP协议是怎么``keepalive``（保活）的。

### 模拟tcp keepalive

- 服务端：nc
- 客户端：[netcat-keepalive](https://github.com/cyberelf/netcat-keepalive)

### 开始通信

开启一个TCP服务端

```bash
nc -lp 9999
```

启动一个客户端，连接服务端

```bash
./nckl-linux -K -O 15 -I 5 -P 5 127.0.0.1 9999
```

netcat-keepalive的使用参数

- -K Turn on TCP Keepalive
- -O secs TCP keepalive timeout
- -I secs TCP keepalive interval
- -P count TCP keepalive probe count

如果不设置，默认为``系统的默认配置``，如linux下

```bash
sysctl -a | grep keepalive
```

- net.ipv4.tcp_keepalive_time = 7200
- net.ipv4.tcp_keepalive_probes = 9
- net.ipv4.tcp_keepalive_intvl = 75

使用tcpdump查看发包情况

```vim
18:15:24.852471 IP localhost.45698 > localhost.9999: Flags [S], seq 253066745, win 43690, options [mss 65495,sackOK,TS val 23438901 ecr 0,nop,wscale 7], length 0
18:15:24.852510 IP localhost.9999 > localhost.45698: Flags [S.], seq 2889588682, ack 253066746, win 43690, options [mss 65495,sackOK,TS val 23438901 ecr 23438901,nop,wscale 7], length 0
18:15:24.852542 IP localhost.45698 > localhost.9999: Flags [.], ack 1, win 342, options [nop,nop,TS val 23438901 ecr 23438901], length 0

18:15:32.933719 IP localhost.45698 > localhost.9999: Flags [P.], seq 1:3, ack 1, win 342, options [nop,nop,TS val 23439709 ecr 23438901], length 2
18:15:32.933814 IP localhost.9999 > localhost.45698: Flags [.], ack 3, win 342, options [nop,nop,TS val 23439709 ecr 23439709], length 0

18:15:47.962915 IP localhost.45698 > localhost.9999: Flags [.], ack 1, win 342, options [nop,nop,TS val 23441216 ecr 23439709], length 0
18:15:47.962992 IP localhost.9999 > localhost.45698: Flags [.], ack 3, win 342, options [nop,nop,TS val 23441216 ecr 23439709], length 0
18:16:03.321743 IP localhost.45698 > localhost.9999: Flags [.], ack 1, win 342, options [nop,nop,TS val 23442752 ecr 23441216], length 0
18:16:03.321802 IP localhost.9999 > localhost.45698: Flags [.], ack 3, win 342, options [nop,nop,TS val 23442752 ecr 23439709], length 0
```

分三段来看，

- 第一段：三次握手，建立连接
- 第二段：客户端发包，服务端应答（这里是我在客户端发了一个数字1）
- 第三段：每隔15秒发一个``keepalive``包

## 使用docker重现问题

### docker-compose建立本地网络

### 断开服务端容器的网络

```bash
docker network disconnect docker_network docker_redis
```

### phpredis客户端

这里出现了两种情况，分别是「已发完PSH包」和「正在发PSH包」

1. 已发完PSH包，过一段时间，然后连续发几次``FIN_WAIT1``包，最后断开与服务端的单边连接
2. 正在发PSH包，不断重试，重试几次后，如果没有得到服务端的确认，直接发一个F包，然后断开与服务端的单边连接

无论是哪一种情况，当客户端主动断开与服务端的连接时，都会返回一个异常 —— ``read error on connection``，这是可以捕获的。但是，如果在执行``brpop``操作，当断开后，的确会返回该异常，然而，下一次再执行``brpop``的时候，就不走网络了，因为连接已经断开，所以redis客户端会直接返回``false``。

### 网络恢复？

#### docker模拟

```bash
docker network connect docker_network docker_redis
```

网络恢复的时机也分为两种情况，分别对应断开的时机

1. 已发完PSH包，此时网络中断，客户端等待1分钟，然后开始发F包。这时，网络恢复了！
2. 正在发PSH包，此时网络中断，客户端不断重试，在重试结束前，网络恢复了！

第一种情况：

```bash
16:50:53.555004 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885306 ecr 19879304], length 0
16:50:53.774621 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885328 ecr 19879304], length 0
16:50:53.995675 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885350 ecr 19879304], length 0
16:50:54.425041 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885393 ecr 19879304], length 0
16:50:55.296710 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885480 ecr 19879304], length 0
16:50:57.055424 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19885656 ecr 19879304], length 0
16:51:00.495806 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 155, ack 21, win 229, options [nop,nop,TS val 19886000 ecr 19879304], length 0
16:51:00.496113 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38658: Flags [P.], seq 21:26, ack 156, win 227, options [nop,nop,TS val 19886000 ecr 19886000], length 5: RESP null
16:51:00.496207 IP 2388ad577c4b.38658 > web_docker_redis.web_docker_web_network.6379: Flags [R], seq 721889775, win 0, length 0
```

**因为客户端已经发了F包，就算这时候网络恢复了，也会断开连接，最终结果为，客户端异常**

第二种情况：

```bash
16:59:45.126281 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 123:155, ack 21, win 229, options [nop,nop,TS val 19938525 ecr 19938424], length 32: RESP "BRPOP" "test" "3"
16:59:45.126422 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [.], ack 155, win 227, options [nop,nop,TS val 19938525 ecr 19938525], length 0
16:59:48.191229 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [P.], seq 21:26, ack 155, win 227, options [nop,nop,TS val 19938831 ecr 19938525], length 5: RESP null
16:59:48.191365 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [.], ack 26, win 229, options [nop,nop,TS val 19938831 ecr 19938831], length 0
16:59:49.196785 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 155:187, ack 26, win 229, options [nop,nop,TS val 19938932 ecr 19938831], length 32: RESP "BRPOP" "test" "3"
16:59:49.196919 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [.], ack 187, win 227, options [nop,nop,TS val 19938932 ecr 19938932], length 0
16:59:52.276131 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [P.], seq 26:31, ack 187, win 227, options [nop,nop,TS val 19939240 ecr 19938932], length 5: RESP null
16:59:52.276197 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [.], ack 31, win 229, options [nop,nop,TS val 19939240 ecr 19939240], length 0
16:59:53.156963 IP 2388ad577c4b.38662 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 219, ack 31, win 229, options [nop,nop,TS val 19939328 ecr 19930202], length 0
16:59:53.279121 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939340 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
16:59:53.496082 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939362 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
16:59:53.715753 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939384 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
16:59:54.147245 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939427 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
16:59:54.997751 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939512 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
16:59:56.756647 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19939688 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
17:00:00.197701 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19940032 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
17:00:07.238143 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19940736 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
17:00:21.282035 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19942144 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
17:00:48.768290 IP 2388ad577c4b.38666 > web_docker_redis.web_docker_web_network.6379: Flags [P.], seq 187:219, ack 31, win 229, options [nop,nop,TS val 19944896 ecr 19939240], length 32: RESP "BRPOP" "test" "3"
17:00:48.768815 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [.], ack 219, win 227, options [nop,nop,TS val 19944896 ecr 19944896], length 0
17:00:51.830821 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38666: Flags [P.], seq 31:36, ack 219, win 227, options [nop,nop,TS val 19945202 ecr 19944896], length 5: RESP null
```

**在客户端重试发PSH包的时候，网络恢复了，连接还在，服务端也会继续返回结果，客户端不再阻塞，继续运行**

## 解决方案：忙连接

1. 设置connect函数的timeout为10s
2. 在客户端断开连接并报异常``read error on connection``时，进行异常捕获，开启一个阻塞循环，不断的重连redis，只有连接成功后才返回

### 代码

```php
class PopData {
    /** @var Redis */
    private $redis = null;

    public function start()
    {
        $this->newRedis();

        while (true) {
            $data = $this->popData();
            var_dump(['data' => $data, 'time' => date('Y-m-d H:i:s', time())]);
            sleep(1);
        }
    }

    /**
     * 连接Redis
     */ 
    private function newRedis()
    {
        $this->redis = new \Redis();
        $this->redis->connect('192.168.48.4', 6379, 3);
        $this->redis->auth(123456);
    }

    /**
     * brpop
     * @return array
     */
    private function popData()
    {
        try {
            // 发完Fin包后，直接从redis返回，不走网络请求。这里已经结束socket连接了，所以，即使网络情况好了也不会重连
            $data = $this->redis->brPop(['test'], 3);

            return $data;

        } catch (\Exception $e) {
            // 只打印了一次
            var_dump( $e->getMessage() );

            // 进入重连逻辑
            $this->reconnect();

            // 重连成功，返回结果
            return [];
        }
    }

    /**
     * 重连redis
     */
    private function reconnect()
    {
        $isLostConnect = true;
        while($isLostConnect) {
            try {
                $this->newRedis();

                // 重连成功
                if ($this->redis->ping() === '+PONG') {
                    $isLostConnect = false;
                }
            } catch (\Exception $e) {
                var_dump($e->getMessage());

                sleep(3);
            }
        }
    }
}
```

### 系统与网络情况

tcpdump看下，在定时重连期间，客户端的发包情况

```bash
7:33:18.326086 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140076 ecr 20134070], length 0
17:33:18.326556 IP 2388ad577c4b.38702 > web_docker_redis.web_docker_web_network.6379: Flags [S], seq 3300121440, win 29200, options [mss 1460,sackOK,TS val 20140076 ecr 0,nop,wscale 7], length 0
17:33:18.544393 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140098 ecr 20134070], length 0
17:33:18.767654 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140120 ecr 20134070], length 0
17:33:19.194564 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140163 ecr 20134070], length 0
17:33:19.404336 IP 2388ad577c4b.38702 > web_docker_redis.web_docker_web_network.6379: Flags [S], seq 3300121440, win 29200, options [mss 1460,sackOK,TS val 20140184 ecr 0,nop,wscale 7], length 0
17:33:20.044337 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140248 ecr 20134070], length 0
17:33:21.807982 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140424 ecr 20134070], length 0
17:33:24.329065 IP 2388ad577c4b.38704 > web_docker_redis.web_docker_web_network.6379: Flags [S], seq 717381143, win 29200, options [mss 1460,sackOK,TS val 20140676 ecr 0,nop,wscale 7], length 0
17:33:25.255734 IP 2388ad577c4b.38700 > web_docker_redis.web_docker_web_network.6379: Flags [F.], seq 91, ack 11, win 229, options [nop,nop,TS val 20140769 ecr 20134070], length 0
17:33:25.403884 IP 2388ad577c4b.38704 > web_docker_redis.web_docker_web_network.6379: Flags [S], seq 717381143, win 29200, options [mss 1460,sackOK,TS val 20140784 ecr 0,nop,wscale 7], length 0
17:34:59.783849 IP 2388ad577c4b.38738 > web_docker_redis.web_docker_web_network.6379: Flags [S], seq 1730851263, win 29200, options [mss 1460,sackOK,TS val 20150126 ecr 0,nop,wscale 7], length 0
17:34:59.784023 IP web_docker_redis.web_docker_web_network.6379 > 2388ad577c4b.38738: Flags [S.], seq 1414026707, ack 1730851264, win 28960, options [mss 1460,sackOK,TS val 20150232 ecr 20150126,nop,wscale 7], length 0
```

可以发现，有两个线程正在疯狂的“试探”，一个想要结束，一个想要连接。

netstat看下，在定时重连期间，客户端的连接状态

```bash
tcp        0      1 192.168.48.5:38700      192.168.48.4:6379       FIN_WAIT1   -
tcp        0      1 192.168.48.5:38728      192.168.48.4:6379       SYN_SENT    682/php
```

由于“连接线程”是通过``new Redis``来实现的，所以端口会一直变化。

## OPT_TCP_KEEPALIVE 到底是什么？怎么用？

在官方文档中，根本找不到这个选项的说明。查看源码发现，phpredis在建立连接时，``tcp_keepalive``参数默认为 ``0``

文件：library.c 行：1783

```c
redis_sock->tcp_keepalive = 0;
```

可以通过函数``setOption``来设置``tcp_keepalive``的值

文件：redis_commands.c 行：3991

```c
case REDIS_OPT_TCP_KEEPALIVE:

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
    RETURN_TRUE;
```

刚刚谈pconnect的时候，聊到下面这个地方，现在着重看看

```c
/* Attempt to set TCP_NODELAY/TCP_KEEPALIVE if we're not using a unix socket. */
if (!usocket) {
    php_netstream_data_t *sock = (php_netstream_data_t*)redis_sock->stream->abstract;
    err = setsockopt(sock->socket, IPPROTO_TCP, TCP_NODELAY, (char*) &tcp_flag, sizeof(tcp_flag));
    PHPREDIS_NOTUSED(err);
    err = setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*) &redis_sock->tcp_keepalive, sizeof(redis_sock->tcp_keepalive));
    PHPREDIS_NOTUSED(err);
}
```

在连接的时候，会通过判断``host``来看是否开启``TCP_KEEPALIVE``，前面在说connect函数的时候了解到，host由下面几种：

*host*: string. can be 
- a host（ip/域名）
- or the path to a unix domain socket. （本地域socket）
- Starting from version 5.0.0 it is possible to specify schema

我把这句话拆开来看会比较清晰，上面这段代码中可以看到，如果是``unix domain socket``，则不会启用``TCP_KEEPALIVE``。然而，在``connect``阶段，根本没有这个配置项，也就是说，真正设置该配置的地方在别处..

### docker模拟

#### 代码

**test.php**

```php

$redis = new \Redis();
$redis->connect('192.168.80.2', 6379);
$redis->auth(123456);
$redis->setOption(\Redis::OPT_TCP_KEEPALIVE, 10);
var_dump($redis->ping());

while(true) {
    
}

```

通过``host``方式连接服务端，并设置选项``OPT_TCP_KEEPALIVE``为``10s``，通过ping查看连通性，然后进行``阻塞``操作。``lsof``看下，确实使用``TCP``方式。

```bash
php     899 root    3u  IPv4 741397      0t0     TCP 2388ad577c4b:38782->web_docker_redis.web_docker_web_network:6379 (ESTABLISHED)
```

断开服务端容器的网络发现，在设定条件下，并不会发keepalive包，可能与docker的实现机制有关，自动转化为unix domain socket?目前不确定是``phpredis``的问题还是``docker网络机制``的问题。接下来，先看看phpredis究竟有没有执行到相应的逻辑。

#### 非debug模式

为了看这段代码是否被执行到，我改一下phpredis的源码，在这里打印一下日志，再重新编译。

```bash
curl -O http://pecl.php.net/get/redis-4.0.2.tgz
tar zxvf redis-4.0.2.tgz
cd redis-4.0.2
vim library.c
```

找到``redis_sock_connect``函数，在下面的代码中，加入``打印日志``的代码

```c
/* Attempt to set TCP_NODELAY/TCP_KEEPALIVE if we're not using a unix socket. */
if (!usocket) {
    printf("open keepalive");
    php_netstream_data_t *sock = (php_netstream_data_t*)redis_sock->stream->abstract;
    err = setsockopt(sock->socket, IPPROTO_TCP, TCP_NODELAY, (char*) &tcp_flag, sizeof(tcp_flag));
    PHPREDIS_NOTUSED(err);
    err = setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*) &redis_sock->tcp_keepalive, sizeof(redis_sock->tcp_keepalive));
    PHPREDIS_NOTUSED(err);
} else {
    printf("not open keepalive");
}
```

这样做发现，打印的结果是``open keepalive``。要想得到整个``调用栈``以及``打印变量``，不是很方便。下面使用gdb来调试，设置断点。

#### debug模式

为了使用``gdb``断点调试PHP扩展，需要把PHP编译为``debug``模式，然后再把phpredis重新编译一次

**编译php**

```bash
wget -c https://github.com/php/php-src/archive/php-7.1.30.tar.gz
tar zxvf php-7.1.30.tar.gz
cd php-src-php-7.1.30
./buildconf --force
./configure \
--prefix=/usr/local/php7.1.30 \
--exec-prefix=/usr/local/php7.1.30 \
--bindir=/usr/local/php7.1.30/bin \
--sbindir=/usr/local/php7.1.30/sbin \
--includedir=/usr/local/php7.1.30/include \
--libdir=/usr/local/php7.1.30/lib/php \
--mandir=/usr/local/php7.1.30/php/man \
--with-config-file-path=/usr/local/php7.1.30/etc \
--enable-pcntl \
--with-curl \
--enable-debug \
--enable-cli
make && make install
cp php-src-php-7.1.30/php.ini-development /usr/local/php7.1.30/etc/php.ini
```

**编译phpredis**

```bash
curl -O http://pecl.php.net/get/redis-4.0.2.tgz
tar zxvf redis-4.0.2.tgz
/usr/local/php7.1.30/bin/phpize
./configure --with-php-config=/usr/local/php7.1.30/bin/php-config
make && make install
vim /usr/local/php7.1.30/etc/php.ini
// 添加extension=redis.so到文件尾
```

编译完成后，会发现安装目录为 ``/usr/local/php7.1.30/lib/php/extensions/debug-non-zts-20160303``

**开始gdb调试**

```bash
gdb /usr/local/php7.1.30/bin/php

Reading symbols from /usr/local/php7.1.30/bin/php...done.
(gdb) b redis_sock_connect
Function "redis_sock_connect" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (redis_sock_connect) pending.
(gdb) run test.php
Starting program: /usr/local/php7.1.30/bin/php test.php
95337
Breakpoint 1, redis_sock_connect (redis_sock=0x7ffff687e0e0) at /data/tools/redis-4.0.2/library.c:1416
1416	{
(gdb) n
1417	    struct timeval tv, read_tv, *tv_ptr = NULL;
(gdb) n
1418	    char host[1024], *persistent_id = NULL;
(gdb) n
1419	    const char *fmtstr = "%s:%d";
(gdb) n
1420	    int host_len, usocket = 0, err = 0;
(gdb) n
1422	    int tcp_flag = 1;
(gdb) n
1426	    zend_string *estr = NULL;
(gdb) n
1429	    if (redis_sock->stream != NULL) {
(gdb) n
1433	    tv.tv_sec  = (time_t)redis_sock->timeout;
(gdb) n
1434	    tv.tv_usec = (int)((redis_sock->timeout - tv.tv_sec) * 1000000);
(gdb) n
1435	    if(tv.tv_sec != 0 || tv.tv_usec != 0) {
(gdb) n
1439	    read_tv.tv_sec  = (time_t)redis_sock->read_timeout;
(gdb) n
1440	    read_tv.tv_usec = (int)((redis_sock->read_timeout-read_tv.tv_sec)*1000000);
(gdb) n
1442	    if (ZSTR_VAL(redis_sock->host)[0] == '/' && redis_sock->port < 1) {
(gdb) n
1446	        if(redis_sock->port == 0)
(gdb) n
1452	        if (strchr(ZSTR_VAL(redis_sock->host), ':') != NULL) {
(gdb) n
1456	        host_len = snprintf(host, sizeof(host), fmtstr, ZSTR_VAL(redis_sock->host), redis_sock->port);
(gdb) n
1459	    if (redis_sock->persistent) {
(gdb) n
1469	    redis_sock->stream = php_stream_xport_create(host, host_len,
(gdb) n
1473	    if (persistent_id) {
(gdb) n
1477	    if (!redis_sock->stream) {
(gdb) n
1491	    sock = (php_netstream_data_t*)redis_sock->stream->abstract;
(gdb) p persistent_id
$1 = 0x0
(gdb) n
1492	    if (!usocket) {
(gdb) p usocket
$2 = 0
(gdb) n
1493		printf("open keepalive");
(gdb) n
1494	        err = setsockopt(sock->socket, IPPROTO_TCP, TCP_NODELAY, (char*) &tcp_flag, sizeof(tcp_flag));
(gdb) n
1496	        err = setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*) &redis_sock->tcp_keepalive, sizeof(redis_sock->tcp_keepalive));
(gdb) p redis_sock->tcp_keepalive
$5 = 0
```

通过上面的gdb调试纪录可以发现，
1. ``usocket``的值为``0``，说明docker没有做什么“小动作”，host模式没问题。
2. 在``connect``阶段，``tcp_keepalive``默认为``0``

```bash
(gdb) b redis_setoption_handler
Function "redis_setoption_handler" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (redis_setoption_handler) pending.
(gdb) run /data/webapp/test/test.php
Starting program: /usr/local/php7.1.30/bin/php /data/webapp/test/test.php
6
open keepalive
Breakpoint 1, redis_setoption_handler (execute_data=0x7ffff6814160, return_value=0x7fffffffb000, redis_sock=0x7ffff687e0e0, c=0x0)
    at /data/tools/redis-4.0.2/redis_commands.c:3089
3089	{
(gdb) n
3095	    int tcp_keepalive = 0;
(gdb) n
3098	    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ls", &option,
(gdb) n
3104	    switch(option) {
(gdb) n
3150	            if (ZSTR_VAL(redis_sock->host)[0] == '/' && redis_sock->port < 1) {
(gdb) p option
$1 = 6
(gdb) n
3153	            tcp_keepalive = atol(val_str) > 0 ? 1 : 0;
(gdb) p val_str
$2 = 0x7ffff6802bd8 "10"
(gdb) n
3154	            if (redis_sock->tcp_keepalive == tcp_keepalive) {
(gdb) p tcp_keepalive
$3 = 1
(gdb) p redis_sock->tcp_keepalive
$4 = 0
(gdb) n
3157	            if (redis_sock->stream) {
(gdb) n
3159	                sock = (php_netstream_data_t*)redis_sock->stream->abstract;
(gdb) n
3160	                if (setsockopt(sock->socket, SOL_SOCKET, SO_KEEPALIVE, (char*)&tcp_keepalive,
(gdb) n
3164	                redis_sock->tcp_keepalive = tcp_keepalive;
(gdb) p redis_sock->tcp_keepalive
$5 = 0
(gdb) p tcp_keepalive
$6 = 1
(gdb) n
3166	            RETURN_TRUE;
```

通过上面的调试可以知道，
- 在调用``setOption``函数阶段，成功设置了``tcp_keepalive``为``1``。

### 疑问

前面我们通过docker模拟，gdb断点排查，现在进行小结：

1. 版本问题：一开始怀疑是phpredis没有``TCP_KEEPALIVE``的配置项，查看源码发现4.0以上的版本都支持了。
2. 环境问题：通过gdb断点发现，host是没问题的，并没有采用``unix domain socket``模式，在docker环境下模拟没问题。
3. 逻辑问题：通过gdb断点发现，在``connect``阶段，sock->tcp_keepalive默认为``0``，在``setOption``阶段，sock->tcp_keepalive被设置为``1``，逻辑也没问题

到现在，几乎任何关于代码的地方都“似乎”没问题，所以走不通了，只能回头再看看，有什么细节遗漏了。前面，我们在``setOption``阶段，把``OPT_TCP_KEEPALIVE``设置为``10``，当时我说，把时间设置为``10s``，因为我把这里理所当然的理解为``tcp_keepalive_time``，我希望在断网后10秒内，能给服务端发``keepalive``包。可是，查看源码发现，

```c
tcp_keepalive = zval_get_long(val) > 0 ? 1 : 0;
```

这里传入的值，似乎被当作了另一种用法，``只要是正整数，就把tcp_keepalive设置为1，否则设置为0``。也就是说，这里并没有``tcp_keepalive_time``的功能，仅作为开关！！！

但是，我找不到任何提供的API可以设置了...

### 设置系统默认TCP_KEEPALIVE各参数值

前面我们知道，系统有一个全局默认的TCP_KEEPALIVE配置

```bash
sysctl -a | grep keepalive

net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200
```

上面这个配置是两个小时（7200s）后才发包，现在我把这些设置改一下，改短一点

```bash
sysctl -w net.ipv4.tcp_keepalive_time=15 net.ipv4.tcp_keepalive_probes=3 net.ipv4.tcp_keepalive_intvl=10
```

- net.ipv4.tcp_keepalive_time：15
- net.ipv4.tcp_keepalive_probes：3
- net.ipv4.tcp_keepalive_intvl：10

重新跑一遍代码，断开服务端网络，tcpdump看发包情况。

```bash
15:38:24.862503 IP web_docker_php.web_docker_web_network.42480 > ce6e2fa39930.6379: Flags [.], ack 13, win 229, options [nop,nop,TS val 35239808 ecr 35238270], length 0
15:38:24.862592 IP ce6e2fa39930.6379 > web_docker_php.web_docker_web_network.42480: Flags [.], ack 41, win 227, options [nop,nop,TS val 35239808 ecr 35238275], length 0
15:38:39.866247 IP web_docker_php.web_docker_web_network.42480 > ce6e2fa39930.6379: Flags [.], ack 13, win 229, options [nop,nop,TS val 35241312 ecr 35239808], length 0
15:38:39.866290 IP ce6e2fa39930.6379 > web_docker_php.web_docker_web_network.42480: Flags [.], ack 41, win 227, options [nop,nop,TS val 35241312 ecr 35238275], length 0
15:38:54.907073 IP web_docker_php.web_docker_web_network.42480 > ce6e2fa39930.6379: Flags [.], ack 13, win 229, options [nop,nop,TS val 35242816 ecr 35241312], length 0
15:38:54.907178 IP ce6e2fa39930.6379 > web_docker_php.web_docker_web_network.42480: Flags [.], ack 41, win 227, options [nop,nop,TS val 35242816 ecr 35238275], length 0
```

重新试一下发现，竟然没问题了！确实每隔15秒发一次``keepalive包``。也就是说，我一直对phpredis的``TCP_KEEPALIVE``用法理解错了。先入为主的认为这个就是``tcp_keepalive_time``。其实，之前的程序一直没有问题，只不过，因为系统默认的时间太久了，程序一直阻塞着，所以我才觉得这个参数没有正确被设置。

## 更简单的方案?

前面讨论了解决``brpop``在网络抖动的情况下，使用``忙连接``的方案。后来，我们了解了``OPT_TCP_KEEPALIVE``的用法，能不能有更简单的方案？要是phpredis客户端能定时发``keepalive包``，如果网络中断，直接报异常，然后进行异常捕获，重新连接。岂不是更佳？

然而，在实测过程中（使用test.php），当网络中断后，客户端便不再发送``keepalive包``，通过netstat看，客户端在**短时间内自动断开客户端与服务端的单边连接**，然后也没有报异常:(