---
title: PHP单例模式探索
date: 2018-03-22
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

## 设计初衷

在讨论每个设计模式时，我们会反复思考设计的初衷是什么？真正理解后，用起来才能更加得心应手。单例模式是最早开始接触的模式之一，这篇文章将基于PHP语言做一些探索。

在讨论单例模式之前，先聊聊面向对象，编程语言一开始是过程式，代表语言有C，后来出现了C++、Java等等面向对象式语言，对象的含义由此而生。说到面向对象，必谈三大特性，封装、继承和多态，这里不深入讨论，以后会着重谈谈。把数据和方法进行封装，我们得到类，类是抽象的，我们需要能做点事的，于是把类实例化得到对象。哪里需要这个类的数据，那就在哪里实例一个对象。看起来没什么问题，但是，在内存没那么多的时候，能不能在进程中只使用一个对象，能省则省嘛，于是，我们找到了单例模式。

## 模式

在Java中，我们知道单例模式可以写成两种，分别是懒汉模式和饿汉模式，他们的区别在于对象实例化的时间，一个java类的完整的生命周期会经历加载、连接、初始化、使用、和卸载五个阶段，懒汉模式正如其名，在显示调用类的getInstance方法时（使用阶段），才会实例化对象。而饿汉模式早在java初始化时就已经实例化好了对象，故名饿汉。

那么在PHP中如何使用单例模式呢？

### 懒汉模式

```php
class Husky
{
	// 定义私有静态变量
	private static $_instance = null;

	// 私有化构成方法
	private function __construct(){}

	// 提供获取实例的公共方法  
	public static function getInstance(){  
	    if(!(self::$_instance instanceof self)){  
	        self::$_instance = new self();  
	    }  
	    return self::$_instance;  
	}
	// 私有__clone方法，禁止复制对象  
	private function __clone(){}

	// 防止反序列化后创建对象
	private function __wakeup(){}
}

```

### 饿汉模式

我们知道，在PHP中，静态变量只能被初始化为文字或常量，不能使用表达式。所以可以把静态属性初始化为整数或数组，但不能初始化为另一个变量或函数返回值，也不能指向一个对象。

所以，饿汉模式也没办法写了:)

## 线程安全（web模式/cli模式）

PHP在web模式下可以粗略的认为是单进程单线程的，当然，nginx/php-fpm确是多进程多线程。单次请求对于PHP来说，可以认为是单线程的，那么不存在线程安全问题。但是对于其他线程来说，比如读取数据库，文件，session来说，则需要加锁处理。

在cli模式下，每次执行脚本都会新建一个进程，所以也不需要考虑线程安全问题。

## 单例复用-后期静态绑定

设计模式的流行在很大一方面是因为人是懒惰的，能复用就复用，能少写就少写，于是，在我想创建下一个相同的单例类时，我想到了继承，于是有了下面的代码

```php
class Akitas extends Husky{

}
```

我们期望得到的是

```php
object(Husky)#1 (0) {}
object(Akitas)#2 (0) {}
```

但是，结果不尽人意，得到了

```php
object(Husky)#1 (0) {}
object(Husky)#1 (0) {}
```

那么，可以怎么修改这个类呢，我们想到了php5.3出的特性，后期静态绑定。修改一下代码

```php
class Husky
{
    // 定义私有静态变量
    protected static $_instance = null;

    // 私有化构成方法
    protected function __construct(){}

    // 提供获取实例的公共方法
    public static function getInstance(){
        if(static::$_instance === null){
            static::$_instance = new static;
        }
        return static::$_instance;
    }
    // 私有__clone方法，禁止复制对象
    protected function __clone(){}

    // 防止反序列化后创建对象
    protected function __wakeup(){}
}

class Akitas extends Husky{
    protected static $_instance = null;
}
```

结果如预期所想，
```php
object(Husky)#1 (0) {}
object(Akitas)#2 (0) {}
```

thanks :)