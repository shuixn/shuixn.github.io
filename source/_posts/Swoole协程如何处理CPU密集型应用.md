---
title: Swoole协程如何处理CPU密集型应用
date: 2019-09-26
categories:
  - 技术
tags: 
  - PHP
  - Swoole
---

## 前言

存档，待完善..

## 正文

想要做抢占式调度，对于PHP来说，有两个途径

1. 当前执行协程在一定条件下主动让出
2. 额外开线程，监控当前执行协程时间，超过设定值则让出当前执行协程

### 当前执行协程在一定条件下主动让出

Swoole版本：4.3.x

PHP是单线程运行的，可以参考这篇文章[协程 CPU 密集场景调度实现](https://wiki.swoole.com/wiki/page/p-tick_scheduler.html)，在脚本开始处注入逻辑``declare(tick=N)``，配合Swoole协程。逻辑是每次触发``tick handler``的时候，判断当前协程相对最近一次调度时间是否大于``协程最大执行时间(max_exec_msec)``，这样就可以将当前超出执行时间后的协程主动让出。

### 额外开线程，负责检查当前执行协程的执行时间

Swoole版本：4.4.x

参考这篇文章[Swoole 4.4 协程抢占式调度器详解](https://segmentfault.com/a/1190000019253487)，利用``PHP-7.1.0``引入的``VM interrupt``机制，默认每隔``5ms``检查一下当前协程是否达到最大执行时间，默认为10ms，如果超过，则让出当前协程，达到被其他协程抢占的目的。
