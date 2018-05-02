---
title: 在PHP多版本共存下安装扩展
date: 2016-11-29
tags: 
  - PHP 
  - 扩展
---

安装PHP扩展有两种常用的安装方式

 1. 编译安装
 2. PECL

今天我为了图方便，直接使用PECL安装，后来发现，我的环境存在着两个PHP版本，一个是Ubuntu自带的php-5.5.9，另一个是集成环境oneinstack的php-5.5.38。

结果可想而知，扩展被安装到了php-5.5.9中，实际上我想安装到php-5.5.38里面。

可见PECL安装虽然方便，但是不够灵活

那么开始使用编译安装的方式，步骤一般是这样的：

 1. 下载扩展到本地（wget，git clone）
 2. 解压并进入目录
 3. phpize（如果没有安装，则须安装php-dev）
 4. ./configure --with-php-config=/usr/local/php/bin/php-config （配置指定的PHP路径）
 5. make
 6. sudo make install
 7. 修改php.ini，把扩展加入到配置中，如extension=xxx.so（同样，需要找到正确的php.ini，如/usr/local/php/etc/php.ini）
 8. 重启nginx和php-fpm
 9. 使用php -m | grep xxx，或者php -i | grep xxx，即可查看是否安装成功

总结，在PHP多版本共存下，可以使用编译配置的方式指定PHP版本，使用到的参数是--with-php-config。