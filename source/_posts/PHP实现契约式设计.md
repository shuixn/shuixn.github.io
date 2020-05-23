---
title: PHP实现契约式设计
date: 2020-05-22
categories:
  - 项目
tags: 
  - PHP 
  - 契约式设计
---

## 什么是契约

>契约，最初是指双方或多方共同协议订立的有关买卖、抵押、租赁等关系的文书，可以理解为“守信用”。形式有精神契约和文字合同契约，对象多样，可以是：生意伙伴、挚友、爱人、国家、世界、全人类，以及对自己的契约等，可以用“文字合同”来约定，可以用“语言”来约定，还可以是“无言”的契约。

—— 来自百度百科

由上可知，关键词有：信用、双方（或多方）、约定。

## 什么是契约式设计

>契约式设计（英语：Design by Contract，缩写为 DbC），一种设计计算机软件的方法。这种方法要求软件设计者为软件组件定义正式的，精确的并且可验证的接口，这样，为传统的抽象数据类型又增加了先验条件、后验条件和不变式。这种方法的名字里用到的“契约”或者说“契约”是一种比喻，因为它和商业契约的情况有点类似。

—— 来自WIKI

由上可知，关键词有：软件方法、可验证、先验条件、后验条件、不变式。

``契约式设计``最早由伯特兰·迈耶于1986年提出，并在Eiffel语言实现了该设计。

这是Eiffel语言对契约式设计的官方介绍：[Building bug-free O-O software: An Introduction to Design by Contract™](https://www.eiffel.com/values/design-by-contract/introduction/)，不想看英文版，可以看看我翻译的中文版（如有错误，请不吝赐教）：[构建无bug面向对象软件：契约式设计简介](https://funsoul.gitbook.io/notebook/she-ji-mo-shi/yi-gou-jian-wu-bug-mian-xiang-dui-xiang-ruan-jian-qi-yue-shi-she-ji-jian-jie)

契约一词来源自商业，在客户和供应商之间产生。双方必须针对某一协议（契约）履行义务，如有一方不履行相应的义务（责任），则视为毁约。可见契约在商业社会代表着``可靠``、``信任``。

在面向对象编程范式中，契约式设计由几部分组成

- 前置条件（对输入参数的值进行检验，如果不符合契约，则不会进入函数体，而是抛异常）
- 后置条件（进入函数体后，针对例程参数做进一步检验，或处理资源释放等情况）
- 类的不变式（针对类的整体属性做断言）

## 为什么要使用契约式设计

理解契约式设计前，先看另一种安全编程方法——防御性编程。其做法是，客户在调用供应方代码前，先做好前置检验。契约式设计和防御性编程中，如果客户违反了前置条件，供应方都会抛出异常，然后给到客户。但是契约式设计的做法更简单，不仅如此，还有以下明显的好处：

1. 面向对象：在编写继承、多态等重用性代码时，不满足契约会提前终止，解决隐性覆盖的问题
2. 文档化：例程的契约是很好的说明文档
3. 调试体：契约模块是一种单元测试
4. 质量保证：容易在调试阶段找出不满足契约的情况
5. 测试文化：面向契约设计可以引导供应方思考例程的边界条件

## PHP如何实现契约式设计

按照契约式设计思想，可以实现一个简易版的PHP库：[contract-php](https://github.com/funsoul/contract-php)，这个库可以很方便的通过``注解``的方式定义函数的``前置条件``、``不变式``和``后置条件``。

### 安装

```bash
git clone https://github.com/funsoul/contract-php.git
cd contract-php
composer install
```

### 使用

#### 通过注解来制定契约

```php
@DbcRequire(condition="a >= 1, a < 10, b >= 1")
@DbcInvariant(condition="discount = 0.6")
@DbcEnsure(callback="ContractExamples\MyEnsureCallback")
```

#### 目前支持的条件

- gt >
- ge >=
- lt <
- le <=
- e =
- ne !=

#### 自定义回调 (如果条件不满足你的需求)

MyRequireCallback.php

```php
use Contract\ContractCallbackInterface;

class MyRequireCallback implements ContractCallbackInterface
{
    public function match(array $arguments): bool
    {
        list($a, $b) = $arguments;

        return $a >= 1 || $b >= 1;
    }
}
```

#### 供应商

Test.php

```php
class Test {
    /** @var float */
    private $discount = 0.5;

    /**
     * @DbcRequire(condition="a >= 1, a < 10, b >= 1")
     * @param int $a
     * @param int $b
     * @return int
     */
    public function addTwoNums(int $a, int $b): int
    {
        return $a + $b;
    }

    /**
     * @DbcRequire(callback="ContractExamples\MyRequireCallback")
     * @DbcEnsure(callback="ContractExamples\MyEnsureCallback")
     * @param int $a
     * @param int $b
     * @return int
     */
    public function addTwoNumsCallback(int $a, int $b): int
    {
        return $a + $b;
    }

    /**
     * @DbcRequire(callback="ContractExamples\MyRequireCallback")
     * @DbcEnsure(callback="ContractExamples\MyEnsureCallback")
     * @DbcInvariant(condition="discount = 0.6")
     * @param int $a
     * @param int $b
     * @return float
     */
    public function multiplyDiscount(int $a, int $b): float
    {
        return ($a + $b) * $this->discount;
    }
}
```

#### 客户

```php
/** @var ContractExamples\Test $proxy */
$proxy = new Contract\Proxy(new ContractExamples\Test());

$res1 = $proxy->addTwoNums(1, 1);

$res2 = $proxy->addTwoNumsCallback(1, 1);

$res3 = $proxy->multiplyDiscount(2, 2);

var_dump($res1, $res2, $res3);
```