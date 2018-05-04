---
title: 【Swoole入门】异步毫秒定时器
date: 2015-11-28
tags: 
  - PHP
  - Swoole
  - 学习笔记
---

## 前言

前几天在朋友圈看到一个俄罗斯工程师为了给他老婆实时报到情况写了一个自动化脚本，通过定时任务来触发。比如晚上9点了，他的服务器上还有正在运行的SSH进程，就给他老婆发一条短信，“今晚加班晚点回家”，多么温馨的故事啊。碰巧我正在学习swoole扩展，特此记录一下。

## 定时器

一般的定时器是怎么实现的呢？我总结如下：

1. 使用Crontab工具，写一个shell脚本，在脚本中调用PHP文件，然后定期执行该脚本；
2. ignore_user_abort()和set_time_limit()配合使用；
3. pcntl_alarm;
4. swoole异步毫秒定时器

## swoole异步毫秒定时器

### Timer定时器

swoole内置定时器，通过函数addtimer即可在server中添加一个定时器，参数单位为毫秒，该定时器会在建立之后，按照预先设定好的时间间隔，每到对应的时间就会调用一次回调函数onTimer。看代码：

![](/images/20151128020847375.jpg)

执行过程：

![](/images/20151128021004126.jpg)

### swoole_timer_add定时器

swoole_timer_add($interval,$callback)

$interval：时间间隔，单位毫秒，不能同时存在同样时间间隔的两个定时器（据官方文档，即将废弃）

**注意点**

1. swoole_server中不能使用。
2. 定时器必须在全异步模式下使用，同步阻塞代码下不可使用，比如sleep,file_get_contents，这些会阻塞timer，使之不能在预定时间间隔下执行。

![](/images/20151128021727238.jpg)

![](/images/20151128021830709.jpg)

### Tick定时器
swoole_timer_tick函数

设置一个间隔时钟定时器，与after定时器不同的是tick定时器会持续触发，直到调用swoole_timer_clear清除。与swoole_timer_add不同的是tick定时器可以存在多个相同间隔时间的定时器。

![](/images/20151128021727238.jpg)

![](/images/20151128023519283.jpg)

定时器的作用非常重要，在实际工作中常常通过定时器任务检测服务器状况，数据库备份，心跳检测，消息订阅等等。



