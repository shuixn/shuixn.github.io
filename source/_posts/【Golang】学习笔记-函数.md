---
title: 【Golang】学习笔记-函数
date: 2018-05-09
tags: 
  - Golang
  - 学习笔记
---

### 变参

```go
func computeSum(nums ...int) int{
	sum := 0
	for _,item := range nums {
		sum += item
	}
	return sum
}

func main(){
	fmt.Println(computeSum(1,2,3,4))
}

// 10
```

使用 ** ...type ** 来接收不定数量的参数。在上面的代码中，函数内部的变参为一个类型为int的slice

### 多返回值

```go
func swap(a,b int) (int,int) {
	return b,a
}

func main(){
	fmt.Println(swap(1,2))
}

// 2 1
```

- 函数参数如果类型一样，可以把类型声明放在最后，如a,b int
- 函数支持多返回值

返回值可以命名

```go
func computeNum(a, b int) (add, cal int){
	add = a+b
	cal = a*b
	return
}

func main(){
	fmt.Println(computeNum(1,2))
}

// 3 2
```

### 传指针

上面的 swap 函数传参，函数体内部的a,b是函数外的一份copy，也就是说，改变函数体内a,b的值不会影响外部的，如果变量所占的内存非常大，拷贝变量是很浪费内存和时间的一件事，这时候可以使用指针传递变量的地址

```go
func swapByPoint(a,b *int) {
	tmp := *a
	*a = *b
	*b = tmp
}

func main(){
	a,b := 1,2
	swapByPoint(&a,&b)
	fmt.Println(a,b)
}

// 2 1
```

go的指针非常简单，看起来就是一个引用。不像C那样，可以做指针运算，go的设计者考虑到指针运算可能带来巨大的维护难度，只保留了最简单的指针用法

同样，使用取址符 & 来获取变量的地址，星号 * 获取地址真实的值

### 传slice、map、channel

slice、map、channel都是类似指针的传递，无需取地址

##### slice

```go
func editSlice(arr []int) {
	arr[0] = 5
}

func main(){
	arr := []int{1,2,3}
	editSlice(arr)
	for _,item := range arr {
		fmt.Println(item)
	}
}

// 5 2 3
```

##### map

```go
func editMap(m1 map[int]string){
	m1[100] = "hello"
}

func main(){
	var m1 map[int]string
	m1 = make(map[int]string)
	m1[100] = "aaa"
	m1[200] = "bbb"

	editMap(m1)
	for k,v := range m1 {
		fmt.Println(k,v)
	}
}

// 100 hello
// 200 bbb
```

