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

### 内建变量类型

### 常量与枚举

### 条件语句

### 循环

### 函数

### 指针