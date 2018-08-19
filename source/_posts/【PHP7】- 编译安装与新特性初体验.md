---
title: 【PHP7】- 编译安装与新特性初体验
date: 2015-12-03
categories:
  - 技术
tags: 
  - PHP
  - PHP7
---

## 前言

今天是世界上最好的语言革命性的一天，PHP7问世了！实际上我使用php只有一年半多一点，但是却被深深地吸引住，所以也容我感慨一下吧。

大一下学期，我们学校开始教Java（我的专业方向就是Java），那时班上的很多同学早在上学期就自学了Java，而且学的非常好。我跟着课程的进度一步步的下载，安装，配置环境，然后写一些简单的程序，调试。这个过程非常煎熬，因为在配置环境的时候就出现了很多问题，使用eclipse来写代码，由于当时电脑配置并不好，所以写代码的时候总是会出现“未响应”，然后卡顿很长时间，更不用说MyEclipse了。苦闷的心情还历历在目~

当时学校还为我们开设了一门PHP课程，我第一次使用的时候感觉很神奇，弱类型的概念还不是很清晰，只是觉得不需要再去考虑变量的类型，php为我们都处理好了。而且网上PHP的环境非常多，其中xampp就是一个著名的跨平台环境，一键安装，可以马上看到自己的代码运行的结果，我可以更关注自己写的逻辑和正确性，不得不说实在是太好用了。最后这门课需要结合另一门jQuery课做一个课程设计，我一个星期就敲出来了，还为我们赢得了不少掌声。就这样，一直用到现在，与php牵连在了一起。

## 正文

### 环境
    
- Ubuntu 14.04

### 准备PHP源码

http://php.net/get/php-7.0.0.tar.gz/from/a/mirror

1.解压源码并重命名为php-7
2.安装依赖项

```bash
sudo apt-get update
sudo apt-get install libxml2-dev
```

#### 安装依赖项

```bash
sudo apt-get  install  build-essential
sudo apt-get install openssl 
sudo apt-get install libssl-dev 
sudo apt-get install make
sudo apt-get install curl
sudo apt-get install libcurl4-gnutls-dev
sudo apt-get install libjpeg-dev
sudo apt-get install libpng-dev
sudo apt-get install libmcrypt-dev
sudo apt-get install libreadline6 libreadline6-dev
```

#### 编译配置

```bash
./configure（注意斜杠前的点）

./configure --prefix=/usr/local/php --with-config-file-path=/usr/local/php/etc --enable-fpm --with-fpm-user=www --with-fpm-group=www --with-mysqli --with-pdo-mysql --with-iconv-dir --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir=/usr --enable-xml --disable-rpath --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --with-curl --enable-mbregex --enable-mbstring --with-mcrypt --enable-ftp --with-gd --enable-gd-native-ttf --with-openssl --with-mhash --enable-pcntl --enable-sockets --with-xmlrpc --enable-zip --enable-soap --without-pear --with-gettext --disable-fileinfo --enable-maintainer-zts  

./configure --prefix=/usr/local/php --enable-fpm --enable-inline-optimization --disable-debug --disable-rpath --enable-shared --enable-opcache  --with-mysql --with-mysqli --with-mysql-sock  --enable-pdo --with-pdo-mysql --with-gettext --enable-mbstring --with-iconv --with-mcrypt --with-mhash --with-openssl --enable-bcmath --enable-soap --with-libxml-dir --enable-pcntl --enable-shmop --enable-sysvmsg --enable-sysvsem --enable-sysvshm --enable-sockets --with-curl --with-zlib --enable-zip --enable-bz2 --with-readline --without-sqlite3 --without-pdo-sqlite --with-pear --with-libdir=/lib/x86_64-linux-gnu --with-gd --with-jpeg-dir=/usr/lib --enable-gd-native-ttf --enable-xml
```

#### 安装 PHP

```bash
make && make test      （这个过程可能会慢一点，视配置而定，我的虚拟机为单处理器+2G内存）
```

![](/images/20151203232056485.jpg)

这里可能会出现这个问题，save一下就好了，也就是输入字母s即可，如果开源精神爆棚（我我我），可以输入y，然后输入自己的邮箱。啦啦啦~

```bash
make && sudo make install   （这里可能需要输入你的root账户密码）
php -v
```

![](/images/20151203232255760.jpg)

到这里已经完成php7的安装，接下来就可以尽情的使用PHP7了！

## PHP7新特性

php7与以前的版本相比有哪些让人兴奋的新特性，我们来体验一番~

### 宇宙飞船运算符

新增运算符Combined comparison Operator '<=>' 吼吼~

![](/images/20151203231855632.jpg)

代码变少了，小于则返回-1，大于返回1，等于返回0，简单易懂，非常爽~

### Return Type Declarations  

返回类型声明 和Scalar Type Declarations 标量类型声明

![](/images/20151203231720195.jpg)

对于C/C++/java系的看到这个肯定开心爆了，还可以通过declare(strict_type = 1);开启严格模式，这时php将不会帮你自动转换类型，而是报错。

### 更多的Error变为可捕获的Exception

![](/images/20151203231410865.jpg)

在php7前未定义函数是致命错误Fatal error，这样会使整个脚本中止，程序无法继续运行下去。但是现在可以通过捕获错误->处理错误，让程序继续执行下去，大大提高了异常处理能力。


### 性能提升

![](/images/20151203231219865.png)

可以看出，php7比php5提高不止100%的性能。升级后该省多少机器啊！

## 最后再说两句

听说PHP7.1将会重启JIT计划，PHP本来就不适合密集计算，开启JIT后，性能，效率又会有多大的提升？很难想象~再加上swoole，yaf等等优秀的开源框架，PHP社区只会越来越繁荣。简单，门槛低，性能好，这应该就是未来编程语言的走向，人人编程的设想也不会是梦吧~

感谢开源，感谢自己的坚持~

本文所有例子皆得到验证，亲测可用。

本文安装教程参考了http://www.oschina.net/question/2010961_242272

本文PHP7新特性参考了鸟哥的分享与博客http://www.laruence.com/