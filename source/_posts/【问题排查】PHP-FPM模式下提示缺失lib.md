---
title: 【问题排查】PHP-FPM模式下提示缺失lib
date: 2017-12-14
categories:
  - 技术
tags: 
  - PHP 
  - 问题排查
---

## php-fpm.conf

设置worker为1，方便strace

<!-- more -->

```vim
[global]
pid = /usr/local/php/var/run/php-fpm.pid
error_log = /usr/local/php/var/log/php-fpm.log
log_level = notice
[www]
listen = /tmp/php-cgi.sock
listen.backlog = -1
listen.allowed_clients = 127.0.0.1
listen.owner = www
listen.group = www
listen.mode = 0666
user = www
group = www
pm = static
pm.max_children = 1
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 1
request_terminate_timeout = 100
request_slowlog_timeout = 0
slowlog = var/log/slow.log
```
## ps -ef | grep php-fpm

```bash
root       6598      1  0 10:45 ?        00:00:00 php-fpm: master process (/usr/local/php/etc/php-fpm.conf)                                                                    
www        6599   6598  0 10:45 ?        00:00:00 php-fpm: pool www

```

## sudo strace -p 6599

查看worker进程系统调用，找到问题行

```vim
open("/lib64/tls/libssl.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory) 
open("/lib64/libssl.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory) 
open("/usr/lib64/tls/libssl.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory) 
open("/usr/lib64/libssl.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory
```

发现系统中确实没有libssl.so.1.0.0，只有libssl.so.1.0.1e，一般而言都会向下兼容，设置软链接

```bash
sudo ln -s /usr/lib64/libssl.so.1.0.1e /usr/lib64/tls/libssl.so.1.0.0
```

再次执行，刚刚出现的“libssl”缺失已经不见了，出现了新的lib缺失

```vim
open("/lib64/tls/libcrypto.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/lib64/libcrypto.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/usr/lib64/tls/libcrypto.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/usr/lib64/libcrypto.so.1.0.0", O_RDONLY) = -1 ENOENT (No such file or directory)
```

同上，找到系统中存在的libcrypto.so.1.0.1e，并设置软链接

```bash
sudo ln -s /usr/lib64/libcrypto.so.1.0.1e /usr/lib64/libcrypto.so.1.0.0
```

问题解决~ :)