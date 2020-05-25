---
title: 可靠的PHP：契约式设计
date: 2020-05-22
categories:
  - 项目
tags: 
  - PHP 
  - 契约式设计
---

前段时间读了云风翻译的《程序员修炼之道》，里面提到了契约式设计（Design by Contract），在多人合作项目中，规范是大家共同履行的契约，但是大多数时候，我们总是在不经意间忽视规范的重要性。如果能在开发过程中，通过契约设计，也许能免掉不少麻烦，最重要的是，提高软件的可靠性和健壮性。

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

在非面向对象编程语言，不变式可视为一种``状态``

## 为什么要使用契约式设计

理解契约式设计前，先看另一种安全编程方法——防御性编程。其做法是，客户在调用供应方代码前，先做好前置检验。契约式设计和防御性编程中，如果客户违反了前置条件，供应方都会抛出异常，然后给到客户。但是契约式设计的做法更简单，不仅如此，还有以下明显的好处：

1. 面向对象：在编写继承、多态等重用性代码时，不满足契约会提前终止，解决隐性覆盖的问题
2. 文档化：例程的契约是很好的说明文档
3. 调试体：契约模块是一种单元测试
4. 质量保证：容易在调试阶段找出不满足契约的情况
5. 测试文化：面向契约设计可以引导供应方思考例程的边界条件

## PHP如何实现契约式设计

按照契约式设计思想，可以实现一个简易版的PHP库：[contract-php](https://github.com/funsoul/contract-php)，这个库可以很方便的通过``注解``的方式定义函数的``前置条件``、``不变式``和``后置条件``。

### 通过代理和切面

在Python中，在不改变原功能代码的前提下，我们可以通过装饰器来对类或函数进行``切面``，提供诸如日志收集、缓存等功能。但PHP就没有那么好用的语法糖了，如果想要管理对象的生命周期，可以像Laravel框架，提供一个依赖管理容器，把对象的控制权交给容器，而容器在程序运行过程中，扮演着``代理``的角色。这样一来，容器就能在对象的生命周期上“做手脚”，比如在对象方法运行时提供切面功能，在方法执行前、执行后做一些增强的``装饰``。

``contract-php``库的存在意义只为说明契约式设计，无意实现一个DI容器，所以仅通过一个代理类来托管原类，在运行时反射原类方法，进而获取契约注解中的前置条件、不变式和后置条件，最后通过切面的方式，进行契约检查。

**目前这个库是实验性的玩具，请不要在生产环境中使用，借助这个库，可以对契约式设计进行窥探，丰富我们的软件开发思路。**

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

## 类比

### 断言

断言是一种对逻辑条件运行时检查，属于契约式设计方法的子集，并不能完全实现契约式设计。理由如下

1. 面向对象编程的继承结构中，断言无法将契约传播，需要手动复制断言逻辑的代码。
2. 在某些语言中，断言也许会被全局关闭。
3. 传统的运行时系统和库不是为支持契约而设计的，因此这些调用不会检查。

### 测试驱动开发

1. 测试方法一次只针对一种特定的情况，契约式设计可以定义出参数何时成立、何时失败的所有情况。
2. 契约式设计和断言在软件开发周期（设计、开发、部署）中永远存在，而测试只发生在“测试环节”。
3. 测试更多只检查公共接口，没有重点考虑不变式。

**契约式设计和测试都有必要，不可完全替代。**

### 尽早崩溃理念

契约式设计非常符合尽早崩溃理念，因为死掉的程序不会说谎，不要让异常传播，而是尽早崩溃，让问题及早暴露出来，这是更务实的做法。如Erlang、Elixir语言的设计哲学：

> 防御式编程是在浪费时间，让它崩溃！

故障由专门的``监控程序``掌控，在程序崩溃时进行特定的``善后``工作。从而形成一种由``监管程序树``构成的设计。

## 模式？规范？

契约式设计不是一种设计模式，而是规范。除了Eiffel类语言，大多数编程语言（C/C++/C#/Java/PHP/Golang/Python..）都没有实现契约式设计，而是把异常、断言、返回值和程序终止指令交给开发人员，由开发人员自己制定规范。

喜欢防御性编程的开发人员，会在任何地方编写校验代码，这会编写许多一致的代码（违反DRY原则）。

契约式设计规范可以防止巧合式编程，通过契约设计，可以在程序运行之前考虑边界条件，对于不符合契约的客户，提前抛出问题。

## 尾

在《程序员修炼之道》一书的第4章“务实的偏执”中，对契约式设计有很好的利弊分析，推荐阅读。

>文档化及对主张进行检验是契约式设计的核心
