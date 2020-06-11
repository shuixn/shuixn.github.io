---
title: odbc connect return false
date: 2019-09-26
categories:
  - 技术
tags: 
  - PHP
  - ODBC
---

## 前言

odbc在connect的时候会返回``资源句柄``，但是如果返回``false``就蒙蔽了。如果有记录php错误日志，会得到下面的内容

```vim
odbc_connect(): SQL error: [unixODBC][Cloudera][ODBC] (11560) Unable to locate SQLGetPrivateProfileString function., SQL state S1000 in SQLConnect
```

google一下发现和动态库有点关系，只好看下系统调用情况

<!-- more -->

## strace php 进程

找到下面的信息，看来是这个库``libiodbcinst.so``缺失了

```vim
open("/home/.odbcinst.ini", O_RDONLY)   = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY)      = 7
fstat(7, {st_mode=S_IFREG|0644, st_size=31742, ...}) = 0
mmap(NULL, 31742, PROT_READ, MAP_PRIVATE, 7, 0) = 0x7f779b219000
close(7)                                = 0
open("/lib64/tls/libiodbcinst.so", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/lib64/libiodbcinst.so", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/usr/lib64/tls/libiodbcinst.so", O_RDONLY) = -1 ENOENT (No such file or directory)
open("/usr/lib64/libiodbcinst.so", O_RDONLY) = -1 ENOENT (No such file or directory)
munmap(0x7f779b219000, 31742)           = 0
open("/data/logs/php/php_error.log", O_WRONLY|O_CREAT|O_APPEND, 0644) = 7
write(7, "[26-Sep-2019 10:52:47 Asia/Shang"..., 304) = 304
close(7)                                = 0
```

## 安装库文件

```bash
rpm -ivh https://rpmfind.net/linux/epel/6/x86_64/Packages/l/libiodbc-3.52.7-1.el6.x86_64.rpm

ln -sv /usr/lib64/libiodbcinst.so.2 /usr/lib64/libiodbcinst.so
```