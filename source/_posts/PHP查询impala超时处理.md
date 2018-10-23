---
title: PHP查询impala超时处理
date: 2018-10-23
categories:
  - 技术
tags: 
  - PHP
  - ODBC
  - impala
  - thrift
  - 问题排查
---

## 问题还原

在[【重构Hue】大数据处理的一些总结](http://funsoul.org/2017/12/27/【重构Hue】大数据处理的一些总结/ "【重构Hue】大数据处理的一些总结") 之后，又发现了新的问题。问题出现在php-fpm的 **request_terminate_timeout** 参数，当设置为0时，worker进程在处理请求时就没有超时处理，0表示无限制，也就是说，用户在查询SQL时，如果该条SQL耗时非常久，worker也不会主动关闭该请求。这里就会出现一种常见的情况：用户看到一直不响应，那就刷新重来，重复查询操作。由于前一个worker还在等待impala服务返回数据，没执行完毕，不能恢复空闲状态，所以master只会把该请求分配给其他worker执行。长此以来，impala服务面对这么多耗时SQL，CPU和内存疯长，处理速度下降。fpm的“僵死”worker越来越多，web服务被拖慢。

优化过程如下：

## request_terminate_timeout设置一个大于0的常数值

既然worker不会主动结束与impala的连接，那就设置一个大于0的常数值，比如60s，那么当请求到了60s还没结束那就主动关闭掉。（**这里的请求指的web请求**）看起来是没问题的，既保证了impala服务稳定，又释放了worker的压力。但是，有两个小问题也很容易想到：一、worker主动关闭返回504 gateway timeout，这个异常是无法在php捕获的，所以没办法争对做处理。二、修改php-fpm的配置，可能会影响其他web程序的正常执行。

## odbc配置设置

我们使用的连接方案是**odbc**，从php到impala，整个调用栈大约如此：php->phpodbc->unixodbc->impalaodbc。由于是cgi程序，所以对进程操作会产生各种奇怪的情况，这里就不做分析了，可能性不大。

1、 odbc_setoption()

```php
// 2. Option 0 of SQLSetStmtOption() is SQL_QUERY_TIMEOUT.
//    This example sets the query to timeout after 30 seconds.
$result = odbc_prepare($conn, $sql);
odbc_setoption($result, 2, 0, 30);
odbc_execute($result);
```

看了下官网手册，发现了这个函数[odbc_setoption](http://php.net/manual/zh/function.odbc-setoption.php)，但是并没有用，因为impalaodbc并没有实现这个接口。

2、odbc.ini中SocketTimeout参数

>The number of seconds that the TCP socket waits for a response from the server before timing out the request and returning an error message

看了下[文档](https://www.cloudera.com/documentation/other/connectors/impala-odbc/latest/Cloudera-ODBC-Driver-for-Impala-Install-Guide.pdf)，这里是指TCP socket的闲置连接超时，而不是查询超时。也没用，这个参数和我们的需求没什么关系。

## thrift查询impala

前面都是用odbc来连接impala，换个思路，使用thrift来试试看。上github找了两个库，[tranch/php-thrift-impala](https://github.com/tranch/php-thrift-impala),[Automattic/php-thrift-sql](https://github.com/Automattic/php-thrift-sql)。开源万岁:)

由于我们的项目使用laravel来写，Automattic/php-thrift-sql就不太适合了（composer下载要账密是什么鬼= =），难道要我把仓库拉下来改一下？都8102年了，不支持composer，没有namespace的代码要你何用！于是，我用了这个tranch/php-thrift-impala。但是，这个仓库有bug！它自己封装的超时设置无效的！怪不得没有几个星星啊~没办法，然后，我结合Automattic/php-thrift-sql这个仓库的例子写了一个，完美执行。以后有空我封装一下开源吧~ :)

```php
$socket = new \Thrift\Transport\TSocket( config('hadoop.host'), config('hadoop.thrift_port') );
$socket->setRecvTimeout(config('hadoop.timeout'));
```

上面两行是关键！

## 注意点

thrift查询impala超时处理是可行的，在php层就可以处理超时。但是，毕竟是php代码，查询会慢一些，我简单对比了一下，同一个查询，thrift和odbc相比有一倍多的差异。所以在这个系统中，并不是只用thrift或者odbc，而是看场景，混合着用。

thanks :)