---
title: socket.io-swoole-server：基于swoole实现socket.io协议
date: 2019-11-03
categories:
  - 项目
tags: 
  - PHP
  - Swoole
  - Socket.io
---

## 框架获取

[socket.io-swoole-server](https://github.com/funsoul/socket.io-swoole-server)

## 使用

![Build Status](https://travis-ci.org/funsoul/socket.io-swoole-server.svg?branch=master)
![](https://img.shields.io/badge/PHP-%3E%3D7.1.0-green)
![](https://img.shields.io/badge/Swoole-%3E%3D4.0.3-green)
![](https://img.shields.io/github/license/funsoul/socket.io-swoole-server)

## Server

### Server Config

```php
$config = new SocketIO\Engine\Server\ConfigPayload();
$config
    // server worker_num
    ->setWorkerNum(2)
    // server daemonize
    ->setDaemonize(0);
```

### Examples

```php
$io = new SocketIO\Server(9501, $config, function(SocketIO\Server $io) {
    $io->on('connection', function (SocketIO\Server $socket) {
        $socket->on('new message', function (SocketIO\Server $socket) {
            $socket->broadcast('new message', $socket->getMessage());
        });

        $socket->on('new user', function (SocketIO\Server $socket) {
            $socket->broadcast('login', $socket->getMessage());
        });

        $socket->on('disconnect', function (SocketIO\Server $socket) {
            $socket->broadcast('user left', $socket->getMessage());
        });
    });
});

$io->start();
```