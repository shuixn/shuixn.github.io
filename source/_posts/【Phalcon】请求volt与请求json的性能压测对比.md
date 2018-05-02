---
title: 【Phalcon】请求volt与请求json的性能压测对比
date: 2017-03-07
tags: 
  - PHP 
  - Phlacon
  - Apache Bench
---

## 前言

最近比较忙，没有太多时间更新博客。不过，公司要求实习生每天都写日报，一周完结的时候还要写周报。我便把每天遇到的问题和思考都记在了日志中，也算是小型博客了。在最近的工作中，遇到了一些痛点。我想把解决这些痛点的经过写下来。

使用Phalcon，模板引擎Volt是一个亮点，我们不需要写原生的PHP代码，当然如果逻辑很复杂，Volt无法满足我们的需求，可以使用原生。

## Phalcon的模板渲染流程是这样的，

### 路由

 => 控制器(如indexController) ，使用setVars把变量发到视图

 => views，存放所有视图

 => views/index.volt
 
 => views/index/

 => views/index/index.volt，视图文件

这样一来，只需要在最终的视图文件中写HTML即可，但是调试起来并不是非常友好，至少在PHPStorm里面不会语法高亮。

回到渲染过程，可以把PHP逻辑写成Volt模板语句。一个页面就生成并返回到浏览器了。

但是，这里的痛点也开始出现了。有时候，在页面DOM生成的时候，整个页面还未加载完成，我们希望根据服务端传来的数据，使用JS改变页面的呈现。比如，我们希望保存中国的省份信息在一个JS变量中，然后把它循环输出到页面中。以前我们可以直接把PHP变量发送到JS中保存，现在不奏效了。使用setVars保存的变量，并不能发送到JS中。有人说，可以使用PHP输出JS变量，这个还未尝试过，而且我会告诉你，即使这样做能行，也不能很好的解决我所描述的业务。

## 请求JSON

这是一个类似微博时间线的业务，如果我使用刚才说的方法，使用PHP输出JS变量（实现起来应该很简单），那么，第一次加载的时候，可以做成一个包含时间线数据的JSON，以及一个空的HTML和一堆JS模板，在页面加载的时候，JS读取变量数据，填充到预先写好的JS模板，再渲染到Body中，看似没问题，渲染页面的工作交给了JS，服务端只负责取数据，不负责渲染HTML。那么，SEO呢？搜索引擎是不知道JSON数据里面的信息有什么关系的，所以，对爬虫并不友好。尽管对开发人员来说，只需要写一套JS模板就可以了，是一件多么开心的事情~

## 结合Volt和JSON

第二种方法，大多数人都会这样做。还是使用Volt在服务端渲染HTML，输出到浏览器，当我请求下一页的时候，使用JSON返回数据，前端有一套由JS写的逻辑一样的模板，然后使用高效一点的模板引擎，填充到模板中，最后输出到页面（局部动态刷新）。这里有一个痛点，那就是，同一套逻辑，PHP写一遍，JS一遍，维护起来也相当麻烦。不过效率很高，看下面的压测数据。

## 完全使用Volt

第三种方法，AJAX这一步，我请求一个webapi，而不是ajaxapi，什么意思？其实返回数据是一个包含HTML标签的页面。具体怎么做？在indexController里面定义一个nextPageAction，然后在views/index/中新建一个nextPage.volt，里面是index中抽出来的时间线业务模板。我只需要在nextPageAction中获取数据，并setVars到nextPage.volt中，返回。我就能得到一个渲染好的模板，直接append到页面中即可。是不是非常简单！这样一来，只有一套模板代码，维护简单，也很开心。但是，正如我刚刚说，返回数据原本应该只有JSON，现在多了HTML标签，体积大了，是否会造成一些影响？想不出来，用数据说话把！

## 环境

- ubuntu 16.04
- 64位
- CPU:2
- 内存：4G
- 硬盘(SCSI)：20G
- Apache
- PHP 7.0.14
- MySQL 5.6
- Apache Bench 2.3

## 压测前准备：

使用浏览器登录网站，到审查元素中 -> network -> cookies
获取所需cookie。

请求的时间线数据是相同的第二页。

## 开始压测：

### 一、请求Volt

```bash
ab -n 100 -H "Cookie:key=value;key=value" http://localhost/index/timeline
```

![](/images/20170307235446889.jpg)

在上一个基础上加10个并发

```bash
ab -n 100 -c 10 -H "Cookie:key=value;key=value" http://localhost/index/timeline
```

![](/images/20170307235503952.jpg)

2000请求，200并发

```bash
ab -n 2000 -c 200 -H "Cookie:key=value;key=value" http://localhost/index/timeline
```

![](/images/20170307235516635.jpg)

### 二、请求Json

100请求，10并发

```bash
ab -n 100 -c 10 -H "Cookie:key=value;key=value" http://localhost/index/timeline
```

![](/images/20170307235529197.jpg)

2000请求，200并发

```bash
ab -n 2000 -c 200 -H "Cookie:key=value;key=value" http://localhost/index/timeline
```

![](/images/20170307235552891.jpg)

### 2000请求，并发200对比


|         | 请求Volt    |  请求json  |
| --------   | -----:   | :----: |
| Time taken for tests        (seconds)	| 70.096      |   63.901    |
| Total transferred           (bytes)	| 43890000	| 11228000 | 
| HTML transferred        ( bytes )	| 43298000	| 10578000 | 
| Requests per second    [#/sec] (mean)	| 28.53 | 	31.30 | 
| Time per request          [ms] (mean)	| 7009.527	| 6390.073 | 
| Time per request       [ms] (mean,across all concurrent requests)	| 35.048 | 	31.950 | 
| Transfer rate            [Kbytes/sec] received	| 611.47 | 	171.59 | 


## 结论

1. 数据（包含HTML）比原来（JSON串）大4倍以上；
2. 网络传输效率比原来低三倍以上；

考虑到网络延时情况，实时性，数据量大、效率要求高的地方，如时间线，应该优先使用JSON传输数据。其他的可以考虑使用请求volt方法，提高开发效率。