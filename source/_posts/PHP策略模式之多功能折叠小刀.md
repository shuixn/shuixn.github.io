---
title: PHP策略模式之多功能折叠小刀
date: 2018-03-27
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

## 多功能折叠小刀

![](/images/xiaodao1.jpg)

从小，我们就对这个不陌生，甚至觉得，拥有一把这样的小刀，是一件很酷的事情。这样一把小刀，实在是居家旅行，必备工具，只需要一把，即可变成水果刀、剪刀或者钢锯等等，而且不占地方。

现在回想起来，这样的设计简直是大智慧有木有？一个接口响应不同策略，这正是策略模式啊~

## 同一接口

我们需要一把多功能的小刀，那么，所有的功能都是从这把小刀体现的，同一接口：

```php
/**
 * 抽象策略角色，以接口实现
 */
interface Strategy {

    /**
     * 算法接口
     */
    public function algorithmInterface();
}

```

## 不同实现

我们需要哪些实现，那就实现这个接口：

```php
/**
 * 具体策略角色：小刀
 */
class Knife implements Strategy {

    public function algorithmInterface() {
        echo '小刀';
    }
}

/**
 * 具体策略角色：剪刀
 */
class Scissors implements Strategy {

    public function algorithmInterface() {
        echo '剪刀';
    }
}

/**
 * 具体策略角色：钢锯
 */
class Hacksaw implements Strategy {

    public function algorithmInterface() {
        echo '钢锯';
    }
}
```

## 具体载体（context）

前面我们准备好了这把小刀所需要的功能，然后我们开始打造这把小刀：

```php
/**
 * 多功能折叠小刀
 */
class FoldingKnife {
    /* 引用的策略 */
    private $_strategy;

    public function __construct(Strategy $strategy) {
        $this->_strategy = $strategy;
    }

    public function contextInterface() {
        $this->_strategy->algorithmInterface();
    }

}
```

## 如何使用

有了这把利器，我们就可以按需使用想要的工具：

```php
/**
 * 客户端
 */
class Client {

    /**
     * Main program.
     */
    public static function main() {
        // 打开小刀
        $context = new FoldingKnife(new Knife());
        $context->contextInterface();

        // 打开剪刀
        $context = new FoldingKnife(new Scissors());
        $context->contextInterface();

        // 打开钢锯
        $context = new FoldingKnife(new Hacksaw());
        $context->contextInterface();
    }

}

Client::main();
```