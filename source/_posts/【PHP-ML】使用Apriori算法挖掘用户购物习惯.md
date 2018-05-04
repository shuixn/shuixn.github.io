---
title: 【PHP-ML】使用Apriori算法挖掘用户购物习惯
date: 2018-01-12
tags: 
  - PHP 
  - PHP-ML
  - 机器学习
  - 学习笔记
---
```php
<?php
require_once __DIR__ . '/vendor/autoload.php';
use Phpml\Association\Apriori;

// 4位用户购买清单
$samples = [
    ['啤酒', '尿布', '儿童玩具'],
    ['尿布', '儿童玩具','笔记本电脑'],
    ['啤酒','耳机', '唇膏'],
    ['啤酒','唇膏', '高跟鞋']
];
$labels  = [];

/*
 * Apriori 参数
 * support支持度
 * confidence 自信度
*/
$associator = new Apriori($support = 0.5, $confidence = 0.5);
$associator->train($samples, $labels);

$res = $associator->predict(['啤酒']);
print_r($res);
// Array([0] => Array( [0] => 唇膏) )

```

<h1>实践</h1>
数据挖掘是从一个数据集中提取信息，并将其转换成可理解的结构，以进一步使用。本次实践模拟用户购买的商品清单，提取关联性，最后根据用户购买欲望推荐关联商品。该实例告诉我们，买啤酒的人想去买唇膏。
<h1>总结</h1>
<strong>定律</strong>

<strong>1、如果一个集合是频繁项集，则它的所有子集都是频繁项集。</strong>

频繁项集的意思是，假设集合{'啤酒', '尿布'}是频繁项集，那么其中的‘啤酒’、‘尿布’同时出现在一条清单记录中的次数是大于或等于最小支持度的（构造参数中的support），则该集合的子集{'啤酒'}、{‘尿布’}也必定大于或等于support，也就是说，它的子集也都是频繁项集。

<strong>2、如果一个集合不是频繁项集，则它的所有超集都不是频繁项集。</strong>

结合第一定律，反之亦然，如果集合{‘啤酒’}不是频繁项集，那么它的超集{'啤酒', '尿布'}也不是频繁项集。

&nbsp;

可以通过<span style="color: #ff0000;">apriori</span>方法获取所有的频繁项集。

```php
$res = $associator->apriori();
print_r($res);
Array
(
    [1] => Array
    (
        [0] => Array
        (
            [0] => 啤酒
        )
        [1] => Array
        (
            [0] => 尿布
        )
        [2] => Array
        (
            [0] => 儿童玩具
        )
        [5] => Array
        (
            [0] => 唇膏
        )
    )

    [2] => Array
    (
        [2] => Array
        (
            [0] => 啤酒
            [1] => 唇膏
        )
        [3] => Array
        (
            [0] => 尿布
            [1] => 儿童玩具
        )
    )
    [3] => Array
    (
    )
)

```

<h3>关联规则（为什么买啤酒的人想去买唇膏？）</h3>
```php
// 获取生成的关联规则
$rules = $associator->getRules();
print_r($rules);

Array
(
    [0] => Array
    (
        [antecedent] => Array
        (
            [0] => 啤酒
        )
        [consequent] => Array
        (
            [0] => 唇膏
        )
        [support] => 0.5
        [confidence] => 0.66666666666667
    )
    [1] => Array
    (
        [antecedent] => Array
        (
            [0] => 唇膏
        )
        [consequent] => Array
        (
            [0] => 啤酒
        )
    
        [support] => 0.75
        [confidence] => 1
    )
    [2] => Array
    (
        [antecedent] => Array
        (
            [0] => 尿布
        )
        [consequent] => Array
        (
            [0] => 儿童玩具
        )
        [support] => 0.5
        [confidence] => 1
    )
    [3] => Array
    (
        [antecedent] => Array
        (
            [0] => 儿童玩具
        )
        [consequent] => Array
        (
            [0] => 尿布
        )
        [support] => 0.5
        [confidence] => 1
    )
)

```

可以通过<span style="color: #ff0000;">getRules</span>方法获取生成的关联规则。由上可知，一共生成了4组关联，每一组有4个属性，意思很明确，

从以上的购物清单可知，存在4组关联，
<ul>
 	<li>买啤酒会去买唇膏</li>
 	<li>买唇膏会去买啤酒</li>
 	<li>买尿布会去买儿童玩具</li>
 	<li>买儿童玩具会去买尿布</li>
</ul>
因为我们输入了“<strong>啤酒</strong>”，所以，推荐给我们“<strong>唇膏</strong>”。
<h3>大数据挖掘存在的局限</h3>
wiki：
<blockquote>先验算法采用<a title="广度优先搜索" href="https://zh.wikipedia.org/wiki/%E5%B9%BF%E5%BA%A6%E4%BC%98%E5%85%88%E6%90%9C%E7%B4%A2">广度优先搜索</a>算法进行搜索并采用<a title="树 (数据结构)" href="https://zh.wikipedia.org/wiki/%E6%A0%91_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)">树</a>结构来对候选项目集进行高效计数。它通过长度为<img src="https://wikimedia.org/api/rest_v1/media/math/render/svg/21363ebd7038c93aae93127e7d910fc1b2e2c745" alt="k-1" />的候选项目集来产生长度为<img src="https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40" alt="k" />的候选项目集，然后从中删除包含不常见子模式的候选项。根据<a title="向下封闭性引理（页面不存在）" href="https://zh.wikipedia.org/w/index.php?title=%E5%90%91%E4%B8%8B%E5%B0%81%E9%97%AD%E6%80%A7%E5%BC%95%E7%90%86&amp;action=edit&amp;redlink=1">向下封闭性引理</a>,该候选项目集包含所有长度为<img src="https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40" alt="k" />的频繁项目集。之后，就可以通过扫描交易数据库来决定候选项目集中的频繁项目集。</blockquote>
在写demo的过程中也发现了这个问题，当商品数量多，清单条数大，生成的候选项集非常多，在筛选效率上存在一定的局限性。这时候或许就要选择别的算法了：）