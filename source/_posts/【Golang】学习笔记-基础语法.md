---
title: 【Golang】学习笔记-起步与基础语法
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

#### bool

true或false，默认为false

#### string

UTF-8编码，以双引号或反引号定义
不可变，不能直接修改字符串中的字符，如

```go
str := "hello world"
str[0] = 'a'
```

可通过将字符串转换为字符数组实现，如

```go
str := "hello world"
chars := []byte(s)
chars[0] = 'a'
str2 := string(chars)
fmt.Printf("%s\n", str2)
```

#### (u)int,(u)int8,(u)int16,(u)int32,(u)int64,uintptr,rune

没有所谓的long或者long long int，不规定长度时根据操作系统来定

其中rune是int32的别称，byte是uint8的别称

这些类型间不可以互相赋值或者操作，如

```go
var a int8
var b int32
c := a + b
```

尽管int的长度是32bit, 但int与int32并不可以互用

#### float32,float64

浮点数的类型有float32和float64两种（没有float类型），默认是float64

#### 原生支持复数，complex64(32位实数+32位虚数),complex128(64位实数+64位虚数)

复数回顾

```
i = √-1
i*i = -1
i*i*i = -i
i*i*i*i = 1
...
```

3+4i，3表示实部，4i表示虚部，可在平面直角坐标系中画出来，X轴等于3，Y轴等于4i，那么有

```
|3+4i| = √(3*3 + 4*4) = 5
```

```go
// import "math/cmplx"
c := 3+4i
fmt.Println(cmplx.Abs(c))
// 5
```

### 常量与枚举

常量，在程序编译时就确定下来的值，而程序在运行时无法改变该值。

```go
// import "math"
const a,b = 3,4
var c int
c = int(math.Sqrt(a*a + b*b))
fmt.Println(c)
```

上面的代码片段中，math.Sqrt期望得到一个float64，然而我们的常量看起来是int。其实a,b既可以做int也可以做float，这只是一个文本替换，但是，如果我们显示定义了a,b int，那么在上面的程序中，就必须强转为float64。

可定义在包内部，函数体外

一组常量可定义在一起，如

```go
const (
	file = "a.txt"
	a,b = 3,4
)
```

在其他语言的编码习惯中，我们习惯于把常量写成大写字母，在go中，不能这样写，因为首字母大写的变量、函数或常量表示public，后面会讲到相关的内容

#### 枚举

```go
const (
	Monday = iota
	Tuesday
	Wednesday
)
fmt.Println(Monday,Tuesday,Wednesday)
// 0,1,2

const (
	Monday = iota
	_
	Tuesday
	Wednesday
)
fmt.Println(Monday,Tuesday,Wednesday)
// 0,2,3
```

可加入表达式，如

```go
const (
	b = 1 << (10 * iota)
	kb
	mb
	gb
	tb
	pb
)
fmt.Println(b,kb,mb,gb,tb,pb)
```

### 条件语句

#### if

##### 不需要括号

```go
i := 1
if i == 1 {
	fmt.Println(1)
}else{
	fmt.Println("no 1")
}
// 1
```

##### 允许在条件中声明变量，作用域仅在条件语句的逻辑块内

```go
func compute(i int) int {
	return i * i
}

func conditionVal(i int){
	if j := compute(i) ; j == 1 {
		fmt.Println(1)
	}else{
		fmt.Println("no 1")
	}
}
```

##### 多条件判断

```go
if j := compute(i) ; j == 1 {
	fmt.Println(1)
} else if j < 1 {
	fmt.Println("< 1")
} else {
	fmt.Println("> 1")
}
```

#### switch

switch 可接收一个表达式，case没有break关键字，自动中断条件，跳出switch

使用**fallthrough**关键字可以使条件继续向下执行

```go
i := 1
switch i {
case 1:
	fmt.Println("i = 1")
	fallthrough
case 2:
	fmt.Println("i = 2")
case 3:
	fmt.Println("i = 3")
default:
	fmt.Println("mismatch")
}

// i = 1
// i <= 2
```

#### goto

- 标签名大小写敏感

```go
Here:   // 标签：第一个词，以冒号结束
	fmt.Println(i)
	i++
	if i > 5 {
		fmt.Println("end at",i)
	}else{
		goto Here   // goto Here
	}

// 1
// 2
// 3
// 4
// 5
// end at 6
```

### 循环

#### for

```go
sum := 0;
for i := 0; i <= 10 ; i++ {
    sum += i
}
fmt.Println(sum)
// 55
```

for 包含三个表达式，以分号隔开，第一段在for开始时执行，第二段为条件判断，第三段在每层for结束时执行

如果忽略第一段和第三段，那便是while循环

因此，可以把上面的代码简写为下面这样：

```go
i,sum := 0,0;
for i <= 10 {
	sum += i
	i++
}
fmt.Println(sum)
```

可以使用for和range来遍历array,slice,map

- 遍历array

```go
arr := [2]int{1,2}
for i,item := range arr {
	fmt.Println(i,item)
}
// 0 1
// 1 2
```

- 遍历slice

```go
arr := []int{1,2,3}
for index,item := range arr  {
	fmt.Println(index,item)
}
// 0 1
// 1 2
// 2 3
```

- 遍历map

```go
var m1 map[int]string
m1 = make(map[int]string)
m1[100] = "aa"
m1[200] = "bb"
for k,v := range m1 {
	fmt.Println(k,v)
}
// 100 aa
// 200 bb
```