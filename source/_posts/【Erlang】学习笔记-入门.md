---
title: 【Erlang】学习笔记-入门
date: 2018-05-17
tags: 
  - Erlang
  - 学习笔记
---

### 安装

#### linux

```bash
wget http://www.erlang.org/download/otp_src_20.3.tar.gz
tar -zxvf otp_src_20.3.tar.gz
cd otp_src_20.3
./configure
sudo make && make install
erl
```

#### win

可以到[官网下载地址](http://www.erlang.org/download/)下载与机器合适的版本。

安装成功后，有一个Erlang应用程序，打开即可使用erlang

### 工作目录

win操作系统下，如果没有进行自定义安装，会自动安装到c盘，可以通过以下命令查看当前工作目录

```erlang
pwd().
```

我们往往希望把工作目录改到别的磁盘，可以这样做，

```erlang
cd("d:/erlang-workplace").
```

但是，每次重启erlang程序（这里指的是Erlang.exe），目录又变回去了，为了解决这个问题可以这样做

在 /c/Users/yourname 目录下新建一个.erlang文件，文件里保存下面的代码

```erlang
c:cd("d:/erlang-workplace").
```

### Hello World

似乎这已经成为了一个习惯，不输出一个hello world，就好像没开始学习这门语言

我们在配置好的工作目录下新建一个文件 hw.erl ，输入下面的代码

```erlang
-module(hw).
-export([printHelloWorld/0]).
 
printHelloWorld() ->
    io:format("Hello World!~n").
```

然后到Erlang界面上输入

```erlang
c(hw).
```

眼力劲儿的你或许发现了，在工作目录中，生成了一个hw.beam文件，好奇的我们打开看看，发现里面是一堆二进制机器码，也就是说，c(hw).似乎做了一些事情，把源代码编译了。

在这里，有一个值得注意的点，如果**文件名**和module名不一致，会产生error，提示no match

```erlang
hw:printHelloWorld().

Hello World!
ok
```

小结一下，

- module是一个模块名，必须与文件名一致
- export把printHelloWorld导出，让外面可以使用
- printHelloWorld旁边的数字是参数个数，这个小程序不需要参数
- 箭头定义一个函数块
- io是系统内置的module，format是这个模块的方法，功能类似于c里面的printf
- ~n表示换行
- 同样，最后的英文句号"."表示语句结束

### 阶乘

```erlang
-module(hw).
-export([fac/1]).
 
fac(1) -> 1;fac(N) -> N * fac(N-1).
```

```erlang
c(hw).
hw:fac(3).

6
```

这个程序，我们使用了函数参数，函数变量必须大写，分号表示语句结束，但是还处于函数块内，所以会继续向下执行。函数体内递归。

### 常量

```erlang
-module(hw).
-export([convert/2]).
 
convert(M, inch) ->
	M / 2.54;
convert(N, centimeter) ->
	N * 2.54.
```

```erlang
c(hw).
hw:convert(1,centimeter).

2.54
```

可以看出，**常量就是一个名字，使用小写**，不像变量可以存储值。

上面的代码，更像是一个switch，根据常量的不同跳转到不同的逻辑块。