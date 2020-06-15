---
title: 使用C++扩展PHP
date: 2020-06-15
categories:
  - 技术
tags: 
  - PHP 
  - C/C++
---

## 概述

[PHP-CPP](https://github.com/CopernicaMarketingSoftware/PHP-CPP)是一个用来开发PHP扩展的C++库，它提供了一个文档完善且易于使用的类集合，这些类可以用于为PHP构建本地扩展。有完整的[文档](http://www.php-cpp.com/documentation/introduction)。

<!-- more -->

注意：仅适用于PHP7。这个库已经更新为适用于 PHP 7.0 及以上版本。如果你想为旧版本的 PHP 创建扩展，请使用 [PHP-CPP-LEGACY](https://github.com/CopernicaMarketingSoftware/PHP-CPP-LEGACY) 库。PHP-CPP 和 PHP-CPP-LEGACY 库有（几乎）相同的 API，所以你可以很容易地将 PHP 5.*的扩展移植到 PHP 7，反之亦然。

## 安装

1. 注意：目前，PHP-CPP仅适用于Linux或OSX系统
2. 本文实验环境为OSX

### 下载源码

```bash
git clone -b v2.1.0 https://github.com/CopernicaMarketingSoftware/PHP-CPP.git
```

打开``Makefile``文件（Makefile是一个保存编译器设置和指令的文件），大多数情况下，该文件中的默认配置已经足够好，但是你或许需要针对自己的环境做一些轻微的改动，比如改变安装目录或者选择自己的编译器。

### 开始构建PHP-CPP库

```bash
make
```

### 常见错误

1. 如果你使用OSX来编译构建，可能会遇到``链接``和``unresolved symbol``错误，如果你正面临此问题，那么需要对``Makefile``文件做一些改动，在这个 Makefile的某个地方有一个选项``LINKER_FLAGS``。修改为``-shared -undefined dynamic_lookup``。
2. [ld: unknown option: -soname clang: error: linker command failed with exit code 1](https://github.com/CopernicaMarketingSoftware/PHP-CPP/issues/368)

### 安装PHP-CPP库到系统中

```bash
sudo make install
```

## PHP如何载入扩展

你可能知道在类unix的系统中，本地的PHP扩展名被编译成``.so``文件，在Windows环境中，编译成``.dll``文件，而全局的``php.ini``文件保存了系统中所有可用的扩展的列表，这意味着如果你正在创建自己的扩展，你也要创建这样的``.so``或``.dll``文件，并且你必须更新PHP配置文件，以便你自己的扩展被PHP加载。

### get_module启动函数

在解释如何创建自己的扩展之前，我们先解释一下 PHP 如何加载一个扩展。当 PHP 启动时，它从配置目录中加载 ``*.ini`` 配置文件，对于这些配置文件中的每一行 ``extension=name.so``，它都会打开相应的库，并调用其中的``get_module()``函数。因此，每个扩展库（你的扩展也是）都必须定义并实现这个``get_module()``函数。这个函数在库加载后就被 PHP 调用（因此在处理 pageviews 之前就被调用了），它应该返回一个指向一个结构的内存地址，这个结构保存了所有扩展库提供的函数、类、变量和常量的信息。

``get_module()``返回的结构是在Zend引擎的头文件中定义的，但它是一个相当复杂的结构，而且没有很好的文档。幸运的是，PHP-CPP库让你的生活变得更简单，并提供了一个扩展类，可以用来代替。

```c++
#include <phpcpp.h>

/**
 *  tell the compiler that the get_module is a pure C function
 */
extern "C" {

    /**
     *  Function that is called by PHP right after the PHP process
     *  has started, and that returns an address of an internal PHP
     *  strucure with all the details and features of your extension
     *
     *  @return void*   a pointer to an address that is understood by PHP
     */
    PHPCPP_EXPORT void *get_module() 
    {
        // static(!) Php::Extension object that should stay in memory
        // for the entire duration of the process (that's why it's static)
        static Php::Extension myExtension("my_extension", "1.0");

        // @todo    add your own functions, classes, namespaces to the extension

        // return the extension
        return myExtension;
    }
}
```

在上面的例子中，你看到了``get_module()``函数的一个非常直接的实现。每个使用 PHP-CPP 库的 PHP 扩展都或多或少地实现了这个函数，它是每个扩展的起点。有一些元素需要特别注意，首先，你看到的唯一的头文件是 phpcpp.h 头文件。如果你使用PHP-CPP库来构建你自己的扩展，你不需要包含Zend引擎的那些复杂的、非结构化的、大部分没有文档的头文件——你需要的只是PHP-CPP库的这个单一的phpcpp.h头文件。如果你坚持的话，你当然也可以包含核心 PHP 引擎的头文件——但你不必这样做。PHP-CPP 负责处理 PHP 引擎的内部，并提供给你一个简单易用的 API。

接下来你会注意到，我们将``get_module()``函数放在了一个 ``extern "C"``的代码块中。正如库的名字所透露的那样，PHP-CPP 是一个 C++ 库。然而，PHP 希望你的库，尤其是 ``get_module()`` 函数是用 C 而不是 C++ 实现的。这就是为什么我们把 ``get_module()`` 函数包装在一个 ``extern "C"`` 块中。这将指示 C++ 编译器 ``get_module()`` 是一个常规的 C 函数，并且它不应该对它进行任何 C++ 名称的篡改。

PHP-CPP 库定义了一个 ``PHPCPP_EXPORT`` 宏，它应该放在 ``get_module()`` 函数的前面。这个宏确保``get_module()``函数是公开导出的，因此可以被PHP调用。这个宏根据编译器和操作系统的不同有不同的实现。

顺便说一下，这也是 PHP-CPP 提供的唯一一个宏。PHP-CPP打算成为一个普通的C++库，不使用魔术或预处理器的技巧。你所看到的就是你所得到的。如果某些东西看起来像函数，你可以肯定它实际上就是一个函数，而当某些东西看起来像一个变量，你可以肯定它也是一个变量。

我们继续往下看。在``get_module()``函数里面，``Php::Extension``对象被实例化，并被返回。至关重要的是，你必须为这个``Php::Extension``类创建一个静态实例，因为这个对象必须在PHP进程的整个生命周期内存在，而不仅仅是在调用``get_module()``的期间。构造函数有两个参数：扩展名和版本号。

``get_module()`` 函数的最后一步是返回扩展对象。这看起来很奇怪，因为get_module()函数应该返回一个指向void的指针，而不是一个完整的Php::Extension对象。为什么编译器没有报告这个问题呢？那是因为``Php::Extension``类有一个``cast-to-void-pointer-operator``。因此，虽然看起来你返回的是完整的扩展对象，但实际上你只是返回了一个指向一个数据结构的内存地址，这个数据结构被 PHP 核心引擎所理解，并且保存了你的扩展的所有细节。

请注意，上面的例子还没有导出任何``本地函数``或``本地类``到PHP中——它只是创建了扩展。

## 编写第一个扩展

当你创建你自己的 PHP-CPP 扩展时，你也必须编译和部署它。一个普通的PHP脚本只需要复制到web服务器上就可以部署，但是部署一个扩展需要花费更多的精力：你需要一个``Makefile``，一个扩展专用的``php.ini``文件，当然还有实现扩展的``*.cpp``文件。

为了帮助你完成这些步骤，我们创建了一个几乎是空的扩展，包含了所有需要的文件。它包含了一个示例``Makefile``，一个示例配置文件，以及第一个``main.cpp``文件，其中的``get_module()``调用已经被实现。这为你开发扩展提供了一个良好的开端。

该扩展代码在PHP-CPP源码的``Example``目录下，本文后面的所有扩展源码都在这里可以找到。

```bash
├── CallPhpFunctions
│   ├── 30-callphpfunction.ini
│   ├── Makefile
│   ├── callphpfunction.cpp
│   └── callphpfunction.php
├── ConstStaticProp
│   ├── cpp
│   ├── readme
│   └── test.php
├── CppClassesInPhp
│   ├── Makefile
│   ├── check_map.php
│   ├── cppclassinphp.cpp
│   ├── cppclassinphp.ini
│   ├── cppclassinphp.php
│   └── includeMyCustomClass.h
├── DlUnrestricted
│   ├── Makefile
│   ├── dlunrestricted.cpp
│   ├── dlunrestricted.ini
│   └── dlunrestricted.php
├── EmptyExtension
│   ├── Makefile
│   ├── main.cpp
│   └── yourextension.ini
├── Exceptions
│   ├── ExceptionCatch
│   └── ExceptionThrow
├── Extension
│   ├── 30-phpcpp.ini
│   ├── Makefile
│   ├── extension.cpp
│   ├── extension.o
│   ├── extension.php
│   └── extension.so
├── FunctionNoParameters
│   ├── 30-functionnoparameters.ini
│   ├── Makefile
│   ├── functionnoparameters.cpp
│   └── functionnoparameters.php
├── FunctionReturnValue
│   ├── 30-functionreturnvalue.ini
│   ├── Makefile
│   ├── functionreturnvalue.cpp
│   └── functionreturnvalue.php
├── FunctionVoid
│   ├── 30-functionvoid.ini
│   ├── Makefile
│   ├── functionvoid.cpp
│   ├── functionvoid.o
│   ├── functionvoid.php
│   └── functionvoid.so
├── FunctionWithParameters
│   ├── 30-functionwithparameters.ini
│   ├── Makefile
│   ├── functionwithparameters.cpp
│   └── functionwithparameters.php
├── Globals
│   ├── 30-globals.ini
│   ├── Makefile
│   ├── globals.cpp
│   └── globals.php
├── Makefile
├── README.md
├── ReturnObject
│   ├── Makefile
│   ├── child.h
│   ├── main.cpp
│   ├── master.h
│   ├── returnobject.ini
│   └── test.php
└── simple
    ├── 30-phpcpp.ini
    ├── Makefile
    ├── simple.cpp
    └── simple.php
```

### 查看例子：Extension

```bash
├── 30-phpcpp.ini   # 扩展声明文件
├── Makefile        # 构建文件
├── extension.cpp   # 扩展源码文件
└── extension.php   # 测试文件
```

修改``Makefile``文件中下面两行内容。

```makefile
LIBRARY_DIR		= $(shell php-config --extension-dir)
PHP_CONFIG_DIR	= /usr/local/etc/php/7.1/conf.d
```

- 第一行用于获取扩展的目录
- 第二行为你的PHP配置目录，用于存放你的扩展声明``extension=name.so``

通过以下步骤安装扩展

```bash
>>make
g++ -Wall -c -I. -O2 -std=c++11 -fpic -o extension.o extension.cpp
g++ -Wall -shared -O2  -o extension.so extension.o -lphpcpp
>>sudo make install
cp -f extension.so /usr/local/Cellar/php@7.1/7.1.30/pecl/20160303
cp -f 30-phpcpp.ini     /usr/local/etc/php/7.1/conf.d
```

测试扩展

```bash
>>php -m | grep extension
my_simple_extension
>>php extension.php
Array
(
    [67] => my_simple_extension
)
```

## 输出和错误



## 注册原生函数

## 函数参数

## 调用函数

## Lambda函数

## 类和对象

## 构造与析构

## 继承

## 魔术方法

## 魔术接口

## 特性

## 类成员

## 异常

## 变量

## 全局和类常量

## 从php.ini载入配置

## 扩展回调

## 命名空间

## 动态加载

