---
title: 用shc保护你的敏感信息
date: 2019-02-14
categories:
  - 技术
tags: 
  - shell
---

## 背景

有个功能是在web下载大文件，但是太大的文件，在浏览器下载会崩溃。想了下，能不能改为API的方式，使用CURL来下载？

一开始不想暴露api，这个api不常用，因为绕过了用户会话（不用登陆），并不安全，如果被恶意利用：数据会被泄露，服务器会被拖慢等等安全隐患。

于是想过把通过shell文件，把api放在里面，然后使用shc来加密shell文件为二进制文件。这样一来，api就“看不到”了，有一定的安全性，只要不被反编译，应该是目前最可靠的方法。


## 安装shc

root用户

```
# wget http://www.datsi.fi.upm.es/~frosal/sources/shc-3.8.3.tgz
# tar -zxvf shc-3.8.3.tgz
# cd shc-3.8.3
# make && make install
```

**这里会让你确认，千万别直接enter，而是输入y，enter**

## 使用shc

```
shc -v -r -T -f xxx.sh
```

这里也有坑，不用-T的话，是没办法执行的。

然后会生成两个文件

```
xxx.sh.x  --- 二进制文件
xxx.sh.x.c  --- C文件，可删除
```


## 使用二进制

```
chmod +x xxx.sh.x
rname xxx.sh （重命名）
./xxx.sh
```

windows下，是执行不了这个二进制文件的，需要使用cygwin或者windows子系统才可以。想过直接用gcc编译c文件，但是会报错，应该是格式原因，还是得在windows下执行shc。

## curl注意事项

如果想要在windows下执行curl，如果链接里面有&符号是有问题的，需要把整个链接用引号包起来才可以

```
curl "https://aa.com/api/excel/download?token=xx&file=xx.csv" >> xx.csv
```