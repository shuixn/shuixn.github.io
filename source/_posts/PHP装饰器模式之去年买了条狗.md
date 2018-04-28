---
title: PHP装饰器模式之去年买了条狗
date: 2018-03-23
tags: 
  - PHP 
  - 设计模式
---

## 去年买了条狗

小明去年想买条狗，于是他搭公交去宠物店逛了逛，看到了很多种狗。

```php
abstract class Dog {}
```

他看了看，哈士奇要888，秋田犬比哈士奇还要更贵，需要1298。

![哈士奇](/images/Husky.jpg)
[ 哈士奇 - 888￥ ]

![秋田犬](/images/Akitas.jpg)
[ 秋田犬 - 1298￥ ]

```php
class Husky extends Dog {
    public function cost(){ return 888;}
    public function getDescribe(){ return '灰白';}
}

class Akitas extends Dog {
    public function cost(){ return 1298;}
    public function getDescribe(){ return '黄白';}
}
```

看着自己腰包上仅有的1024.5，一阵凉风吹过，充满了唏嘘和无奈...

交完钱，顺手买了一根狗链10块，还有一包狗粮20块。

其实，他内心更喜欢黄色的秋田犬，无奈太贵了，**这狗自己就长这样，没办法自己变成黄色啊~**

正要回家，看到旁边的宠物美容店，嘴角划过一丝微笑...

```php
class DogDecorator extends Dog {
    protected $dog;
    public function __construct(Dog $dog)
    {
        $this->dog = $dog;
    }
}
```
咨询了下，原来染毛只需要100块钱，嘿嘿

```php
class DyeYellow extends DogDecorator {
    public function cost(){ return $this->dog->cost() + 100;}
    public function getDescribe(){ return '黄白';}
}
```
于是，他给这只哈士奇染成了黄色...这样一来，虽然没买到秋田犬，但是只需要998就可以买到和秋田犬差不多的狗，还是很开心的...

```php
$a = new DyeYellow(new Husky());
echo $a->cost();
echo $a->getDescribe();

// 998
// 黄白
```
![](/images/yellowHusky-1.jpg)

感到非常开心的他，在家附近的士多店买一根五羊2块钱，正要开始吃时，哈士奇一个飞扑，把他弄摔在地，手中的五羊以一个美妙的抛物线划过天际，口袋里仅剩的5毛钱掉了出来，望着哈士奇，哭笑不得~

## 完整代码

```php
<?php

abstract class Dog {}

class Husky extends Dog {
    public function cost(){ return 1000;}
    public function getDescribe(){ return '灰白';}
}

class Akitas extends Dog {
    public function cost(){ return 1500;}
    public function getDescribe(){ return '黄白';}
}

class DogDecorator extends Dog {
    protected $dog;
    public function __construct(Dog $dog)
    {
        $this->dog = $dog;
    }
}

class DyeYellow extends DogDecorator {
    public function cost(){ return $this->dog->cost() + 100;}
    public function getDescribe(){ return '黄白';}
}

$a = new DyeYellow(new Husky());
echo $a->cost();
echo $a->getDescribe();

```