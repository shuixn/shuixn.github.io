---
title: 【Golang】学习笔记-基础语法
date: 2018-05-04
tags: 
  - Golang
  - 学习笔记
---

### 安装

- 环境：Ubuntu 16.04

```bash
sudo apt-get install git golang-go vim sublime-text-installer
```
如果出现安装错误，进行修复安装：

```bash
sudo apt-get -f install
```

安装完成后，查看安装版本：

```bash
go version
```

### 环境变量

安装后go之后，要配置几个环境变量，

- GOPATH  go的开发路径
- GOBIN   go工具程序存放路径
- GOROOT  go的安装路径，默认情况下，系统已经自动配置了GOROOT

```bash
sudo vi ~/.bashrc

export GOPATH=/home/ubuntu/go-workplace
export GOBIN=$GOPATH/bin
export PATH=$PATH:${GOPATH//://bin:}/bin

source ~/.bashrc
```

查看环境变量是否生效:

```bash
go env
```

开发目录

```bash
cd /home/ubuntu
mkdir bin src pkg
```

### Hello World

```go
package main

import "fmt"

func main(){
	fmt.Println("hello");
}
```

### 升级

```bash
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt-get install golang-go
```

### 变量定义

```go
// 变量声明
var i int

// 变量声明并赋值
var i int = 1

// 多变量声明与赋值
var i,j int = 1,2

// 根据值类型自动推导类型
var i,j = 1,2

// 只能用在函数内部；在函数外部使用则会无法编译通过
i,j := 1,2 

// 只声明不使用的变量，无法通过编译期检查，使用'_'下划线来舍弃值
_,j := 1,2
```

与常见的C、C++、Java、PHP7等语言不太一样，类型放在变量的后面，其实这是更人性的写法，由于先入为主的理念，可能会觉得很奇怪。不过不要紧，声明的时候注意一下就好了，慢慢会熟悉这种写法。

不需要分号结束语句。

### 内建变量类型

### 常量与枚举

### 条件语句

### 循环

### 函数

### 指针