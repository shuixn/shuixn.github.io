---
title: 【译】3 Reasons Why PHP is Not Yet Perfect for Machine Learning
date: 2019-11-01
categories:
  - 技术
  - 翻译
tags: 
  - PHP 
  - PHP-ML
  - 机器学习
  - 翻译
---

## 前言

最近做了一些数据分析、机器学习的工作，使用的是php-ml这个库，前期还算不错，不得不佩服作者[arkadiuszkondas](https://arkadiuszkondas.com/)的API设计能力和算法功底。但是，随着数据量越来越大，发现训练时长剧增、内存利用率不高，于是翻了翻作者的博客找到了这篇文章，故译之。

## 正文

为什么说目前的PHP生态还不能够普遍落地机器学习框架，我认为主要有三个原因造成当今这种状况。下面我将陈述这几个观点：

## 可视化的可能性几乎为零

> ...无论是以计算还是以图表的输出方式都应该学习，这样的呈现才是易于理解的 —— F. J. Anscombe, 1973

让我们问自己一个简单的问题：下面这些图形有什么共同点呢？(这些都是简单的集合点:x和y)

![](/images/php-ml/all-dinos-grey.jpg)

答案非常简单与令人惊讶:一套共同的指标。这些图都非常相似(小数点准确到第二位):

- X 平均值: 54.26
- Y 平均值: 47.83
- X 标准差: 16.76
- Y 标准差: 26.93
- 相关系数: -0.06

惊讶吗?还没完。向``datasaurus``问好:

![](/images/php-ml/dino.jpg)

是的，那些点在这张图上面显示是相同的指标

我希望能让你相信，机器学习过程的一个基本组成部分是数据可视化。

目前，与丰富的``Python生态``相比，在PHP中我们没有任何图表支持。下面我给你展示一下，画一个图去查看你的已预测数据是多么简单！别忘了那只恐龙～

```python
import pandas as pd
import matplotlib.pyplot as plt

data = pd.read_csv('dino.csv',header=None)
plt.scatter(data[0], data[1], color='red');
```

好了，4行代码，完美～

比分：PHP: 0 - Python: 1

## PHP不支持Jupyter Notebook

[Jupyter Notebook](https://jupyter.org/)是一个无与伦比的学习机器学习的工具。你可以只使用浏览器准备、检查、可视化数据和训练你的模型。

所有这一切都归功于这些强大的能力：混合代码（Juila、Python和R）、文档（markdown）、可视化（img）

下面是一个例子，一个Notebook片段:

![](/images/php-ml/crime-notebook.png)

在机器学习的世界里，你会花大量的时间来分析数据。你会一直想要保留笔记、实验和训练各种类型的模型。当然，最好是能可视化。

这是一个有趣的犯罪数据分析的例子：[Understanding Crime in Chicago](https://www.kaggle.com/fahd09/eda-of-crime-in-chicago-2005-2016)

对了，我不会刻意说，这个笔记本还有``版本控制``的功能。

比分：PHP: 0 - Python: 2

## 不支持GPU

在你的数据集还未达到GB级别时，你不会想到使用深度学习，你更不会使用GPU。但是，这一天终将到来不是吗，到时候我们能做些什么呢？

比如说[TensorFlow](https://www.tensorflow.org/)，相比其他环境的神经网络，它以优秀的分布式计算闻名。它给了你更多的可能性，去完全掌控计算组件（GPU、CPU），毫无疑问有了更多的选择。

例如，你可以决定把变量放在哪，下面，我们把变量``a``放在GPU上运行，把常量``b``放在``CPU``上运行。

```python
with tf.device('/gpu:0'):
    a = tf.Variable(15.12)

with tf.device('/cpu:0'):
    b = tf.constant(3.14)
```

这仅仅是开始。你可以设置一个服务器集群来运行。你还可以使用TPU（张量处理组件）。专用的计算组件，可以加速机器学习进度。

在PHP里面，我们发现一个有趣的仓库：[dnishiyama85/PHPMatrix](https://github.com/dnishiyama85/PHPMatrix)

>这是一个PHP扩展，支持浮点数矩阵运算，同时，它还支持 OpenBLAS sgemm 和 cuBLAS sgemm 函数

但是在我看来，在GPU上面的探索还任重道远(crawls like a baby)

我不知道这里有什么比较的意义，但还是让我们和平地做总结吧：

比分：PHP: 0 - Python: 3

## 总结

正如你看到的那样，上面我选择的三个主要原因足以说明，为什么PHP不是学习和使用机器学习的理想语言。但是，这不意味着情况会一直如此。

我想再一次提醒，PHP在创造之始是为了在网站上展示HTML，现在，你可以使用它来构建企业级应用。让我们看看一个例子：Spotify

>通过网页和巨大的移动设备流量，这个音乐流媒体服务使用symfony来支撑7500万活跃用户、每秒几乎60万请求

具有讽刺意味的是，Spotify有一个用于机器学习的服务器集群：[Spotify怎么如此懂你？](https://medium.com/s/story/spotifys-discover-weekly-how-machine-learning-finds-your-new-music-19a41ab76efe)

也许未来，PHP写机器学习的需求会到来，但只要上述问题不解决，我们没有什么指望。

但请不要悲伤。下面是我最喜欢的一句名言:

>Those who are crazy enough to think they can change the world usually do.
