---
title: epoll学习笔记
date: 2018-03-02
categories:
  - 技术
tags: 
  - 网络编程
  - 学习笔记
---

## 流
流可以是文件,socket,pipe等能进行I/O操作的内核对象

## I/O操作

- read,从流中读取数据
- write,写数据到流中

设想:客户端需要从socket中读取数据,但是服务端还没有返回,这时候该怎么办?

### 阻塞
举个例子:等快递,快递没来,只能等快递打电话叫你.

### 非阻塞忙轮训
接上例子:每隔一定单位时间打电话问快递来了没.

```
while true {
	for i in stream[]; {
		if i has data
		read until unavailable
	}
}
```

### 缓冲区

- 内核缓冲区
- 用户缓冲区

## 非阻塞无差别轮训

为了避免CPU空转,可以引进了一个代理（一开始有一位叫做select的代理,后来又有一位叫做poll的代理,不过两者的本质是一样的）。这个代理比较厉害,可以同时观察许多流的I/O事件,在空闲的时候,会把当前线程阻塞掉,当有一个或多个流有I/O事件时,就从阻塞态中醒来,于是我们的程序就会轮询一遍所有的流（于是我们可以把“忙”字去掉了）
```
while true {
	select(streams[])
	for i in streams[] {
		if i has data
		read until unavailable
	}		
}
```

## epoll

epoll可以理解为event poll,不同于忙轮询和无差别轮询,epoll之会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。

### epoll_create 

创建一个epoll对象，一般epollfd = epoll_create()

### epoll_ctl （epoll_add/epoll_del的合体），

往epoll对象中增加/删除某一个流的某一个事件

比如
```
epoll_ctl(epollfd, EPOLL_CTL_ADD, socket, EPOLLIN);//注册缓冲区非空事件，即有数据流入
epoll_ctl(epollfd, EPOLL_CTL_DEL, socket, EPOLLOUT);//注册缓冲区非满事件，即流可以被写入
epoll_wait(epollfd,...)等待直到注册的事件发生
（注：当对一个非阻塞流的读写发生缓冲区满或缓冲区空，write/read会返回-1，并设置errno=EAGAIN。而epoll只关心缓冲区非满和缓冲区非空事件）。

```

### 伪代码

```
while true {
	active_stream[] = epoll_wait(epollfd)
	for i in active_stream[] {
		read or write till
	}
}
```

## epoll高效的本质

1. 减少用户态和内核态之间的文件句柄拷贝
2. 减少对可读可写文件句柄的遍历