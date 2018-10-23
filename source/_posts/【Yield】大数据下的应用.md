---
title: 【Yield】大数据下的应用
date: 2018-02-01
categories:
  - 技术
tags: 
  - PHP 
  - 大数据
---

继上一篇文章[【重构Hue】大数据处理的一些总结](http://funsoul.org/2017/12/27/【重构Hue】大数据处理的一些总结/ "【重构Hue】大数据处理的一些总结")后,引起了一些思考.上篇文章提出了在大数据查询的情况下,分次读取是一种方案,但是这种方案并不完善,接下来,看看这样的情况吧. :)

#### **SQL子查询嵌套**
什么是SQL子查询？类似这样的:
```sql
SELECT * from
    (SELECT user_id,name,age
     FROM user
     WHERE age = 18)a
LEFT JOIN
    (SELECT user_id,good_id
     FROM cart)b 
ON a.user_id = b.user_id
```
上面的SQL查询出"18岁的用户购买了哪些商品",直接查询是没有丝毫问题的.但是如果在这条SQL的外层再加一层SELECT呢?比如这样:
```sql
SELECT * from
	(SELECT * from
		(SELECT user_id,name,age
		 FROM user
		 WHERE age = 18) AS a
	LEFT JOIN
		(SELECT user_id,good_id
		 FROM cart) AS b
	ON a.user_id = b.user_id) AS c limit 0,1000
```

或许你已经察觉到了什么,是的,这条SQL会引发错误（[HY000] : AnalysisException: duplicated inline view colum
n alias: 'user_id' in inline view 'c'）

也就是说,如果想要分次查询大数据,必须在原始SQL的基础上多加一层限制（查询区间）.而对于子查询来说,这样极容易造成错误.有人说了,使用者给SELECT加查询字段而不是\*,或者给相同的字段加别名都可以避免这个问题,是的,你没说错.但是,既然原本的语句没有问题,为何要多写一个版本呢?给使用者也新增了工作量.

#### **缓冲与无缓冲**
在技术角度,有没有查询大数据更好的方案,这值得思考,我们知道,查询分为缓冲与无缓冲,默认情况下,使用缓冲模式,这意味着查询结果会立即从数据库服务器转移到PHP,然后保存在PHP的内存中.这种模式有一定的好处

- 计算行数
- 移动(查找)当前的结果指针
- 在处理结果集时对相同的连接发出进一步的查询

缺点显而易见,遇到大数据查询,需要大量的内存.相对之下,无缓冲模式执行查询时返回一个资源句柄,然后等待数据库服务器的获取,在PHP进程内占用内存较少,但是缺点也是相对的

- 维持资源句柄,增加服务器负载
- 获得完整结果集前,无法对相同进一步连接发送查询

因此,根据不同的业务场景可以选择.

#### **odbc计算行数**
刚开始,我找到了这个[odbc_num_rows]函数,后来发现,怎么查都是返回-1,看看手册才知道,

```
odbc_num_rows be reliable for INSERT, UPDATE, and DELETE queries only.
Using odbc_num_rows() to determine the number of rows available after a SELECT will return -1 with many drivers.
```

那么怎么得到SELECT返回的数据行数呢?接着看

```php
function best_odbc_num_rows($r1)  {
	ob_start(); // block printing table with results
	$number = (int)odbc_result_all($r1); 
	ob_clean(); // block printing table with results
	return $number;
}

```
哈哈,是不是相当机智:)

#### **PDO查询每条记录最后几个字段为null**

```php
$pdo = new \PDO($dsn, $username,$password);
$pdo->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, false);
$result = $pdo->query($sql);
```
类似这种情况：https://stackoverflow.com/questions/22197884/pdo-select-fetch-last-columns-as-null
一直找不到原因,可能是bug,也可能是用的姿势不对吧

#### **Yield生成器**

- PHP5.5开始支持
- 是一个函数,返回值依次返回.更方便的实现Iterator接口
- 可中断的函数,为了继续执行生成器中yield后的代码,你就需要调用$range->next()方法.这将再次启动生成器, 直到下一次yield语句出现.因此,连续调用next()和current()方法, 你就能从生成器里获得所有的值, 直到再没有yield语句出现.

从数据库服务器到PHP进程,如果直接放进内存中,很容易造成内存不够,比如下面的代码

```php
$rs = odbc_exec($conn, $sql);
while ($row=odbc_fetch_array($rs)) {
    $result[] = $row;
}
```

那么,可不可以一点一点的取出来呢？Yield可以做到,
```php
function query($conn, $sql){
	$rs = odbc_exec($conn, $sql);
	while ($row=odbc_fetch_array($rs)) {
	    yield $row;
	}
}
```
然后,在需要的地方进行迭代
```php
foreach(query($conn, $sql) as $row){
	// 业务处理
}
```

如果需要把数据导出到文件,不建议直接写文件,细想一下很容易明白,生成器是逐条返回的,如果逐条写入文件,那么频繁的打开关闭文件,性能消耗巨大,这时候可以使用一个$bucket装起来,达到一定数量后才做写文件操作.

Yield在这篇文章中仅当生成器来用,我们知道,它的另一个概念【协程】更有意思,值得深入研究 :)

#### **协程**

- 抢占式（操作系统控制、目前大多数操作系统，防止恶意程序占用）
- 协作式（程序主动控制）
- 协程的支持是在迭代生成器的基础上, 增加了可以回送数据给生成器的功能(调用者发送数据给被调用的生成器函数). 这就把生成器到调用者的单向通信转变为两者之间的双向通信.

这里引用一下鸟哥的例子做简单的探讨,详情可以看看【[在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html "在PHP中使用协程实现多任务调度")】
```php
<?php

function gen() {
    $ret = (yield 'yield1');
    var_dump($ret);
    $ret = (yield 'yield2');
    var_dump($ret);
}
 
$gen = gen();

// 1、接收第一个yield返回的值
var_dump($gen->current());    // string(6) "yield1"

// 2、发送值回去给第一个yield并替代，执行到第二个yield，进行接收
var_dump($gen->send('ret1')); // string(4) "ret1"   (the first var_dump in gen)
                              // string(6) "yield2" (the var_dump of the ->send() return value)

// 3、发送值回去给第二个yield并替代，没有返回
var_dump($gen->send('ret2')); // string(4) "ret2"   (again from within gen)
                              // NULL               (the return value of ->send())
```