---
title: Swoole加速结巴分词
date: 2019-08-14
categories:
  - 技术
tags: 
  - PHP
  - Swoole
  - 结巴分词
---

## 中文分词

对于英文句子来说，可以通过``空格``来切分单词，如

```bash
// 今天天气不错
the weather is nice today
```

可以很简单的把该句子中的单词区分出来

```bash
the/weather/is/nice/today
```

<!-- more -->

在中文里面，就没有那么方便的区分方法了。当然，如果你习惯这样说话：

```bash
今天 天气 不错
```

大家也不会打你，只会觉得你像个``“结巴”``（点题了！）

### 为什么需要分词？

在中文里面的``字``和``英文单词``是两个不同的东西。在读书的时候，最痛苦的一件事就是学习文言文，我想了一下，有大于等于三个原因：

1. 不知道在哪里断句
2. 字或词的含义很多
3. 这个字是通假字（它不是这个它，它是那个它），或者说纯粹就是写错了，但是细想一下也能读的通。

我们常说中文博大精深，历史原因就不细究了，简单来说就是，我们的祖先在中文上的造诣非常高，好几层楼那么高，研究非常透彻，一句话能说出几个意思。我们自小在中文环境下成长，经过千锤百炼，读写是没问题的，但是计算机要怎么理解一句话呢？先从分词开始。

计算机学习分词的过程，和人类是很像的（或许这是局限性），目前有几种：

1. 基于字符串匹配。按一定的策略在一个已分好词的大词典中搜寻，若找到则视为一个词。
2. 统计。大量已经分词的文本，利用统计机器学习模型学习词语切分的规律（训练），从而实现对未知文本的切分。
3. 组合。结合1、2点，如``结巴分词``。

我们学习中文的时候，也有这样的过程，

1. 积累词语（建立词典）
2. 训练不同词语在不同句子中的含义的概率（权重），选择具有最大概率（权重）的含义的词语（动态规划，寻找切分组合）

## 结巴分词是什么？

``结巴分词``是国内程序员用``python``开发的一个``中文``分词模块, 源码被托管在[Github](https://github.com/fxsjy/jieba)

为了方便说明，下面截取了部分文档和例子。

### 特点

- 精确模式，试图将句子最精确地切开，适合文本分析
- 全模式，把句子中所有可以成词的词语都扫描出来, 速度非常快，但是不能解决歧义
- 搜索引擎模式，在精确模式的基础上，对长词再次切分，提高召回率，适合用于搜索引擎分词
- 支持繁体分词
- 支持自定义词典
- MIT 授权协议

### 例子

```python
# encoding=utf-8
import jieba

seg_list = jieba.cut("我来到北京清华大学", cut_all=True)
print("Full Mode: " + "/ ".join(seg_list))  # 全模式

seg_list = jieba.cut("我来到北京清华大学", cut_all=False)
print("Default Mode: " + "/ ".join(seg_list))  # 精确模式

seg_list = jieba.cut("他来到了网易杭研大厦")  # 默认是精确模式
print(", ".join(seg_list))

seg_list = jieba.cut_for_search("小明硕士毕业于中国科学院计算所，后在日本京都大学深造")  # 搜索引擎模式
print(", ".join(seg_list))
```

输出：

```vim
【全模式】: 我/ 来到/ 北京/ 清华/ 清华大学/ 华大/ 大学

【精确模式】: 我/ 来到/ 北京/ 清华大学

【新词识别】：他, 来到, 了, 网易, 杭研, 大厦    (此处，“杭研”并没有在词典中，但是也被Viterbi算法识别出来了)

【搜索引擎模式】： 小明, 硕士, 毕业, 于, 中国, 科学, 学院, 科学院, 中国科学院, 计算, 计算所, 后, 在, 日本, 京都, 大学, 日本京都大学, 深造
```

### 算法实现

1. 基于前缀词典实现高效的词图扫描，生成句子中汉字所有可能成词情况所构成的有向无环图 (DAG)
2. 采用了动态规划查找最大概率路径, 找出基于词频的最大切分组合
3. 对于未登录词，采用了基于汉字成词能力的 ``HMM`` 模型，使用了 ``Viterbi`` 算法

针对结巴分词的原理，网上的文章写的非常详细了，这里就不再赘述了。有兴趣的读者可以看看

1. [中文分词的基本原理以及jieba分词的用法](https://blog.csdn.net/John_xyz/article/details/54645527)
2. [一个隐马尔科夫模型的应用实例：中文分词](https://blog.csdn.net/u014365862/article/details/54891582)

## PHP结巴分词库（扩展）

有国人实现了PHP版本：

1. PHP扩展：[jonnywang/phpjieba](https://github.com/jonnywang/phpjieba)
2. PHP类库：[fukuball/jieba-php](https://github.com/fukuball/jieba-php)

尤其是这个扩展``jonnywang/phpjieba``实现，**支持PHP7**，果断安装了。

## 使用方法

### PHP-FPM模式

PHP的LNMP架构在Web开发领域常年占据一定的市场，那么是否可以使用结巴分词呢？当然可以，不过，我们知道在FPM模式下，PHP的生命周期非常短，前面我们了解到，结巴分词使用``前缀字典树``建立词库，该操作需要一定的时间和耗费内存（默认词典dict.txt占用将近1G）。那么，在常规FPM模式下，假设开启8个worker，那就需要大约8G内存分配。而且，在应对大量请求时，频繁的申请/销毁操作并不合理。所以，在FPM模式下，使用结巴分词不合适。

### CLI模式

我们想到，和应用强耦合在一起不是个好办法，把结巴分词独立出来作为一个公共服务，通过不同的接口（HTTP，unixsocket）给其他应用提供服务是个不错的方案。

在考察该方案前，我们需要解决几个问题：

1. 进程拉起才初始化词典
2. 为其他应用提供分词服务，需要应对高并发
3. 更新用户自定义词库

我们第一时间想到了[Swoole](https://www.swoole.com/)，有下面的优势：

1. 假设提供HTTP服务，可以在Worker进程启动时（onWorkerStart）初始化词典，当服务启动后，字典树就完全载入到内存中了，由于常驻内存，后面我们只需要处理请求（onRequest）即可。
2. 使用HTTP服务，可以为其他应用提供服务，而不需要每一个需要分词服务的应用都写一个类似的分词库。
3. 用户自定义词库需要在初始化词典阶段载入，因此，如果需要添加/删除自定义词库，需要做下面几步：
    - Process模式
        1. 服务启动时，记录Master进程ID到本地文件
        2. 提供给外部应用增加/删除词的接口，写入用户自定义词库（user_dict.txt）文件
        3. Worker进程给Master进程发``SIGUSR1``信号，进行**柔性重启**（重启Worker进程）
    - Base模式
        * 只有一个Worker进程，默认不开启Manager进程，所以需要自己终止掉，由外部来重启，如Supervisor
        * 大于等于两个Worker进程
            1. 服务启动时，记录Manager进程ID到本地文件
            2. 同Process模式第2点
            3. 同Process模式第3点

Base模式比Process模式少了两次ipc的过程，性能会更好些。

## 性能测试

- 4c
- 2g

### Base 模式、1 Worker

- 请求：10000
- 并发：1000
- api：a=2&s=我爱中华民族、广东、美食

```bash
Benchmarking 127.0.0.1 (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        swoole-http-server
Server Hostname:        127.0.0.1
Server Port:            9501

Document Path:          /
Document Length:        204 bytes

Concurrency Level:      1000
Time taken for tests:   8.499 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      3580000 bytes
Total body sent:        2180000
HTML transferred:       2040000 bytes
Requests per second:    1176.63 [#/sec] (mean)
Time per request:       849.883 [ms] (mean)
Time per request:       0.850 [ms] (mean, across all concurrent requests)
Transfer rate:          411.36 [Kbytes/sec] received
                        250.49 kb/s sent
                        661.86 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    5  14.1      0      69
Processing:    29  800 181.2    840    1260
Waiting:        4  800 181.2    840    1260
Total:         30  805 174.0    840    1275

Percentage of the requests served within a certain time (ms)
  50%    840
  66%    855
  75%    866
  80%    870
  90%    894
  95%    912
  98%   1139
  99%   1214
 100%   1275 (longest request)
```

### Base 模式、2 Worker

```bash
Benchmarking 127.0.0.1 (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        swoole-http-server
Server Hostname:        127.0.0.1
Server Port:            9501

Document Path:          /
Document Length:        204 bytes

Concurrency Level:      1000
Time taken for tests:   4.746 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      3580000 bytes
Total body sent:        2180000
HTML transferred:       2040000 bytes
Requests per second:    2106.85 [#/sec] (mean)
Time per request:       474.643 [ms] (mean)
Time per request:       0.475 [ms] (mean, across all concurrent requests)
Transfer rate:          736.57 [Kbytes/sec] received
                        448.53 kb/s sent
                        1185.10 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    9  28.0      0     148
Processing:     0  415 407.8    421    1270
Waiting:        0  415 407.8    421    1270
Total:          0  423 409.2    443    1282

Percentage of the requests served within a certain time (ms)
  50%    443
  66%    822
  75%    827
  80%    830
  90%    838
  95%    850
  98%   1157
  99%   1225
 100%   1282 (longest request)
```