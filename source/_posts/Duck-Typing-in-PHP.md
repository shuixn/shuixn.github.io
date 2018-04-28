---
title: Duck Typing in PHP
date: 2018-04-23
tags: 
  - PHP 
---

先看看wiki是怎么描述Duck Typing的：
Duck Typing in PHP
>在程序设计中，鸭子类型（duck typing）是动态类型的一种风格。在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由"当前方法和属性的集合"决定。这个概念的名字来源于由James Whitcomb Riley提出的鸭子测试（见下面的“历史”章节），“鸭子测试”可以这样表述：

>“当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。”
在鸭子类型中，关注点在于对象的行为，能作什么；而不是关注对象所属的类型。例如，在不使用鸭子类型的语言中，我们可以编写一个函数，它接受一个类型为"鸭子"的对象，并调用它的"走"和"叫"方法。在使用鸭子类型的语言中，这样的一个函数可以接受一个任意类型的对象，并调用它的"走"和"叫"方法。如果这些需要被调用的方法不存在，那么将引发一个运行时错误。任何拥有这样的正确的"走"和"叫"方法的对象都可被函数接受的这种行为引出了以上表述，这种决定类型的方式因此得名。

## 鸭子是什么？

鸭子：脊索动物门，脊椎动物亚门，鸟纲雁形目，鸭科鸭属动物...

这是从百度百科中查询到的，可见，鸭子有自己专门的定义，因此，如果我们想“正统”地描述一个鸭子，或许应该这样写：

```php

trait 脊索动物门 {}
trait 脊椎动物亚门 {}
trait 鸟纲雁形目 {}
class 鸭科鸭属动物 {}

class 鸭子 extends 鸭科鸭属动物 {
	use 脊索动物门,脊椎动物亚门,鸟纲雁形目;
	...
}

```

我们知道PHP是不支持多继承的，但是我们可以通过5.4起出现的新语法trait来复用代码。这样的代码不容易编写，更多时候，我们希望使用组合的方式来复用代码。

## 怎么才算一只鸭子？

前面我们以正统的方式来追寻鸭子的定义，其实，**鸭子在不同的人眼里有不同的呈现。**

前段时间，广东出现了一只“大黄鸭”飘在水中，引起不少人去一睹真面目，这里就不放图了，大家可以借助搜索引擎查看相关资料。

那么，对于我们来说，这只“大黄鸭”算是鸭子吗？前面说了，**鸭子在不同的人眼里有不同的呈现。**小孩子觉得，这就是鸭子，因为它拥有鸭子的形态，而且它可以飘在水中。有些吃货觉得，这不是鸭子，因为它煮熟了不能吃，没有鸭子的味道。

## 动态语言的鸭子

PHP是动态语言，它不像C、C++、Java那样在编译期间就可以知道变量的类型，在代码编写阶段，这样的特性可以为我们很快的排查出错误，在运行时也就可以避免出现类型相关的错误。PHP是解释型的，在运行时才能知道变量类型。

通过下面一个简单的例子，学习Duck typing在PHP中是怎么编写的。

```php

define('NEWLINE',chr(10)); // <br /> on web

class Duck {
    public function quack(){
        echo get_class($this)." 呱呱".NEWLINE;
    }

    public function feathers(){
        echo get_class($this)." 白色与灰色羽毛".NEWLINE;
    }
}

class Person {
    public function quack(){
        echo get_class($this)." 人模仿鸭子呱呱".NEWLINE;
    }

    public function feathers(){
        echo get_class($this)." 拿起一根羽毛".NEWLINE;
    }
}

class Client {
    public static function inTheForest($duck){
        $duck->quack();
        $duck->feathers();
    }

    public static function main(){
        $duck = new Duck();
        $person = new Person();

        static::inTheForest($duck);
        static::inTheForest($person);
    }
}

Client::main();


// Duck 呱呱
// Duck 白色与灰色羽毛
// Person 人模仿鸭子呱呱
// Person 拿起一根羽毛
```

我们前面讲了这么多概念，一看代码，其实不就是一种**编码风格**吗？是的，如果我们使用接口来实现duck和person，使用的时候（inTheForest）通过接口来约束，不就没这些事情了。比如这样写：

```php

define('NEWLINE',chr(10));

interface IType {
    public function quack();
    public function feathers();
}

class Duck implements IType {
    public function quack(){
        echo get_class($this)." 呱呱".NEWLINE;
    }

    public function feathers(){
        echo get_class($this)." 白色与灰色羽毛".NEWLINE;
    }
}

class Person implements IType {
    public function quack(){
        echo get_class($this)." 人模仿鸭子呱呱".NEWLINE;
    }

    public function feathers(){
        echo get_class($this)." 拿起一根羽毛".NEWLINE;
    }
}

class Client {
    public static function inTheForest(IType $obj){
        $obj->quack();
        $obj->feathers();
    }

    public static function main(){
        static::inTheForest(new Duck());
        static::inTheForest(new Person());
    }
}

Client::main();
```

这样的风格是可选的，需要**依赖文档、清晰的代码和测试**来确保正确使用。

回到前面实现Duck Typing的代码，前面说了，小孩子觉得“大黄鸭”是鸭子，因为它可以飘在水中，并且拥有鸭子的形态，所以它就等同于鸭子。例子中，我们的实现者是inTheForest，它拥有quack和feathers方法，那么，只要调用者也拥有这两个方法，那么它等同于这个“类”。这就是所谓的Duck Typing。代码编写阶段，我们并不知道它是否拥有这两个方法，所以需要借助文档和测试，才能确保运行时无误。

以上。 :)