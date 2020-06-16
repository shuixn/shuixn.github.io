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

你可以使用常规的C++流来进行IO，使用常规的``<<``操作符和特殊的函数，如``std::endl``。但是使用``std::cout``和``std::cerr``流并不是一个好主意。

当 PHP 作为 webserver 模块运行时，stdout 被重定向到 webserver 进程最初启动的终端。在生产服务器上，这样的终端是不活动的，所以任何发送到stdout的输出都会丢失。因此，在webserver模块中运行的扩展中使用``std::cout``是不行的。但是即使 PHP 以 ``CLI`` 脚本的形式运行（并且 ``std::cout`` 也能工作），也不应该直接向 ``stdout`` 写入。写入``stdout``会绕过所有PHP用户空间脚本可能设置的输出处理程序。

PHP-CPP 库提供了一个 ``Php::out`` 流用来替代标准输出。这个``Php::out``变量是众所周知的``std::ostream``类的一个实例，并且尊重PHP中所有的输出缓冲设置。它的作用与PHP脚本中的echo()函数基本相同。

``Php::out``是一个普通的 ``std::ostream`` 对象。其结果是它使用了一个需要刷新的内部缓冲区。当你在输出中添加``std::endl``或明确添加``std::flush``时，刷新会自动发生。

```c++
/**
 *  Example function that shows how to generate output
 */
void example()
{
    // 等同于php中的echo()
    Php::out << "example output" << std::endl;

    // 不带换行符，刷新输出缓冲
    Php::out << "example output" << std::flush;

    // 也可以直接调用flush
    Php::out << "example output";
    Php::out.flush();

    // 也可以直接使用echo
    Php::echo("Example output\n");
}   
```

当你想触发一个PHP错误（相当于PHP trigger_error()的C++函数），你可以使用``Php::error``、``Php::notice``、``Php::warning``和``Php::deprecated``流中的一个。这些也是``std::ostream``类的实例。

```c++
/**
 *  Example function that shows how to generate output
 */
void example()
{
    // 输出一个 PHP notice
    Php::notice << "this is a notice" << std::flush;

    // 输出一个 PHP warning
    Php::warning << "this is a warning" << std::flush;

    // 通知用户，该函数已经不推荐使用
    Php::deprecated << "this method is deprecated" << std::flush;

    // 输出一个 PHP fatal error
    Php::error << "fatal error" << std::flush;

    // 当输出 fatal error 后，这一行不会执行
    Php::out << "regular output" << std::endl;
}  
```

在上面的例子中，你可以看到我们使用了 ``std::flush`` 而不是 ``std::endl``。原因是``std::endl``内部做了两件事：它附加了一个换行符，以及它刷新了缓冲区。对于错误、通知和警告，我们不需要换行，但我们仍然需要刷新缓冲区来实际生成输出。

``Php::error``流有一个非常奇特的地方：当你刷新它时，PHP脚本以一个致命的错误结束，而你的C++代码立即退出！！在引擎下面，PHP引擎做了一个``longjump``，到了Zend引擎深处的一个地方。在这个例子中，``Php::out << "regular output";`` 语句从未被执行。

这一切都很不寻常，而且（根据我们的说法）与软件工程的一般规则相冲突。一个输出生成函数的行为不应该像抛出一个异常。看起来像正常代码的代码，也应该表现得像正常代码一样，而不应该做意想不到的事情，比如跳出当前的调用栈。因此，我们建议不要使用``Php::error``，或者在使用它时要格外小心。

## 注册原生函数

### 无返回值：FunctionVoid

在``get_module``中声明扩展信息

```c++
// create extension
static Php::Extension extension("my_function_void","1.0");

// add function to extension
extension.add<my_function_void>("my_void_function");
```

编写一个直接打印字符串的函数

```c++
void my_function_void()
{
    cout << "In my_function_void()" << endl;
}
```

### 有返回值：FunctionReturnValue

在``get_module``中声明扩展信息

```c++
// create extension
static Php::Extension extension("my_function_return_value","1.0");

// add function to extension
extension.add<my_return_value_function>("my_return_value_function");
```

通过``Php::Value``来标示返回值类型

```c++
/**
 *  my_return_value_function()
 *  @return Php::Value
 */
Php::Value my_return_value_function()
{
    return "42";
}
```

``Php::Value``是存储在Zend引擎中的值的基类。value类的一个实例代表了在PHP环境用户空间中存在的一个变量，例如作为全局变量、函数中的局部变量、对象或数组的成员。可以是标量类型也可以是更复杂的数组或对象类型。

在内核中，Zend引擎使用``zval``对象来实现。这些``zval``对象持有引用计数和引用配置。PHP-CPP的``Value``类负责处理这些工作，所以你需要做的就是使用这个类的对象。

## 函数参数

### 查看例子：functionwithparameters

#### 如何获取未定义参数

添加一个获取未定义参数的函数

```c++
// add function, with undefined parameters, to extension
extension.add<my_with_undefined_parameters_function>("my_with_undefined_parameters_function");
```

可以通过``Php::Parameters``来获取函数参数

```c++
void my_with_undefined_parameters_function(Php::Parameters &params)
{
    for (unsigned int i = 0; i < params.size(); i++)
    {
        cout << "Parameter " << i << ": " << params[i] << endl;
    }
}
```

上面这个例子，尽管在定义函数时没有定义参数，但是也可以通过``Php::Parameters``来获取，非常神奇。

#### 如何写一个加法运算函数

添加一个有参数的函数

```c++
// add function, with defined numeric parameters, to extension
extension.add<my_with_defined_parameters_function>("my_with_defined_parameters_function", {
    Php::ByVal("x", Php::Type::Numeric),
    Php::ByVal("y", Php::Type::Numeric)
});
```

编写函数定义

```c++
/**
 *  my_with_defined_parameters_function()
 *  @param  Php::Parameters     the given parameters
 *  @return Php::Value          Param[0] and Param[1] added
 */
Php::Value my_with_defined_parameters_function(Php::Parameters &params)
{
    for (unsigned int i = 0; i < params.size(); i++)
    {
        cout << "Parameter " << i << ": " << params[i] << endl;
    }
    
    return params[0] + params[1];
}
```

这个函数的含义是，接收两个整型数字，并返回求和结果。

#### 如何传递引用

```c++
// add function, with defined parameter by reference, to extension
extension.add<my_with_defined_parameters_reference_function>("my_with_defined_parameters_reference_function", {
    Php::ByRef("string", Php::Type::String)
});
```

修改传递进来的参数值

```c++
/**
 *  This functions receives a reference to a variable. When the variable is altered,
 *  so is the value in the php script.
 *  my_with_defined_parameters_reference_function()
 *  @param  Php::Parameters     the given parameters
 */
void my_with_defined_parameters_reference_function(Php::Parameters &params)
{
    params[0] = "I changed!";
}
```

#### 如何接收数组

```c++
// add function, with defined array parameter, to extension
extension.add<my_with_defined_array_parameters_function>("my_with_defined_array_parameters_function", {
    Php::ByVal("array", Php::Type::Array)
});
```

#### 如何接收对象

```c++
// add function, with defined object parameter, to extension
extension.add<my_with_defined_object_parameters_function>("my_with_defined_object_parameters_function", {
    Php::ByVal("myClassObjVar", "MyPhpClass")
});
```

### Php::Type支持情况

```c++
/**
 *  Supported types for variables
 *  The values are the same as the ones used internally in Zend
 */
enum class Type : unsigned char {
    Undefined       =   0,  // Variable is not set
    Null            =   1,  // Null will allow any type
    False           =   2,  // Boolean false
    True            =   3,  // Boolean true
    Numeric         =   4,  // Integer type
    Float           =   5,  // Floating point type
    String          =   6,  // A string obviously
    Array           =   7,  // An array of things
    Object          =   8,  // An object
    Resource        =   9,  // A resource
    Reference       =  10,  // Reference to another value (can be any type!)
    Constant        =  11,  // A constant value
    ConstantAST     =  12,  // I think an Abstract Syntax tree, not quite sure

    // "fake types", not quite sure what that means
    Bool            = 13,   // You will never get this back as a type
    Callable        = 14,   // I don't know why this is a "fake" type
};
```

### 小结

1. 使用``Php::ByVal``定义接收参数（值传递）
2. 使用``Php::ByRef``定义接收引用
3. ``Php::Parameters``是一个数组，用来获取参数

## 调用函数

首先让我们弄清楚一件事。 运行编译后的机器码比运行PHP代码快得多。 因此，一旦最终调用了C++函数或C++方法，通常就将参数转换为本地变量，然后开始运行自己的快速算法。从那时起，您就不想调用其他PHP函数。

但是，如果您要调用PHP函数（无论是Zend内置的函数，在扩展中定义的函数，还是来自PHP用户空间的函数），也是可以做到的。

### 查看例子：callphpfunction

添加一个含有两个参数的函数，第一个参数是回调函数，第二个参数是数字

```c++
// add function to extension
extension.add<call_php_function>("call_php_function", {
    Php::ByVal("addFunc", Php::Type::Callable),
    Php::ByVal("x", Php::Type::Numeric)
});
```

函数实现

```c++
/**
 *  call_php_function()
 *  Calls a function in PHP space.
 *  @param      &params
 *  @return     Php::Value
 */
Php::Value call_php_function(Php::Parameters &params)
{
    // check whether the parameter is callable
    if (!params[0].isCallable()) throw Php::Exception("Not a callable type.");
        
    // perform the callback
    return params[0](params[1]);
}
```

## Lambda（匿名）函数

C++和PHP都支持lambda函数或匿名函数（在C++世界里，"lambda "这个词用得最多，PHPer讲的是 "匿名函数"）。使用 PHP-CPP 可以将这些函数从一种语言传递到另一种语言。可以从C++代码中调用一个匿名的PHP函数，也可以从PHP脚本中调用一个C++ lambda函数。

让我们从一个非常简单的PHP例子开始。在PHP中，你可以创建匿名函数，并将它们赋值给一个变量（或者直接将它们传递给一个函数）。

```php
<?php
// 使用变量$f保存匿名函数
$f = function($a, $b) {

    // return the sum of the parameters
    return $a + $b;
};

// 把这个变量传递给另一个函数
other_function($f);

// 或者直接传递匿名函数
other_function(function() {

    // return the product of the parameters
    return $a * $b;
});

?>
```

上面的代码对于大多数PHP程序员来说应该是很熟悉的，当然'other_function'也可以在PHP用户空间中实现，但是我们要用C++来演示如何用PHP-CPP来实现。'other_function'当然可以在PHP用户空间中实现，但是为了演示如何用PHP-CPP来实现，我们将用C++来构建它。就像你在前面的例子中看到的所有其他函数一样，这样的C++函数函数接收一个``Php::Parameters``对象作为参数，它是一个由``Php::Value``对象组成的``std::vector``。

```c++
#include <phpcpp.h>
/**
 *  Native function that is callable from PHP
 *
 *  This function gets one parameter that holds a callable anonymous
 *  PHP function.
 *
 *  @param  params      The parameters passed to the function
 */
void other_function(Php::Parameters &params)
{
    // make sure the function was really called with at least one parameter
    if (params.size() == 0) return nullptr;

    // this function is called from PHP user space, and it is called
    // with a anonymous function as its first parameter
    Php::Value func = params[0];

    // the Php::Value class has implemented the operator (), which allows
    // us to use the object just as if it is a real function
    Php::Value result = func(3, 4);

    // @todo do something with the result
}

/**
 *  Switch to C context, because the Zend engine expects the get_module()
 *  to have a C style function signature
 */
extern "C" {
    /**
     *  Startup function that is automatically called by the Zend engine
     *  when PHP starts, and that should return the extension details
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() 
    {
        // the extension object
        static Php::Extension extension("my_extension", "1.0");

        // add the example function so that it can be called from PHP scripts
        extension.add<other_function>("other_function");

        // return the extension details
        return extension;
    }
}
```

就是这么简单。但是反过来说也是可以的。想象一下，我们在PHP用户空间代码中有一个接受回调函数的函数，下面的函数是PHP array_map()函数的简单版本。

```php
<?php
// function that iterates over an array, and calls a function on every
// element in that array, it returns a new array with every item
// replaced by the result of the callback
function my_array_map($array, $callback) {

    // initial result variable
    $result = array();

    // loop through the array
    foreach ($array as $index => $item) {

        // call the callback on the item
        $result[$index] = $callback($item);
    }

    // done
    return $result;
}
?>
```

想象一下，我们想从你的C++代码中调用这个PHP函数，使用一个C++ lambda函数作为回调。这是有可能的，而且很简单。

```cpp
#include <phpcpp.h>
/**
 *  Native function that is callable from PHP
 */
void run_test()
{
    // create the anonymous function
    Php::Function multiply_by_two([](Php::Parameters &params) -> Php::Value {

        // make sure the function was really called with at least one parameter
        if (params.empty()) return nullptr;

        // one parameter is passed to the function
        Php::Value param = params[0];

        // multiple the parameter by two
        return param * 2;
    });

    // the function now is callable
    Php::Value four = multiply_by_two(2);

    // a Php::Function object is a derived Php::Value, and its value can 
    // also be stored in a normal Php::Value object, it will then still 
    // be a callback function then
    Php::Value value = multiply_by_two;

    // the value object now also holds the function
    Php::Value six = value(3);

    // create an array
    Php::Value array;
    array[0] = 1;
    array[1] = 2;
    array[2] = 3;
    array[3] = 4;

    // call the user-space function
    Php::Value result = Php::call("my_array_map", array, multiply_by_two);

    // @todo do something with the result variable (which now holds
    // an array with values 2, 4, 6 and 8).
}

/**
 *  Switch to C context, because the Zend engine expects the get_module()
 *  to have a C style function signature
 */
extern "C" {
    /**
     *  Startup function that is automatically called by the Zend engine
     *  when PHP starts, and that should return the extension details
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() 
    {
        // the extension object
        static Php::Extension extension("my_extension", "1.0");

        // add the example function so that it can be called from PHP scripts
        extension.add<run_test>("run_test");

        // return the extension details
        return extension;
    }
}
```

在这个例子中，我们将一个C++ lambda函数分配给一个``Php::Function``对象。``Php::Function``类是由``Php::Value``类派生出来的。``Php::Value``和``Php::Function``的唯一区别是``Php::Function``的构造函数接受一个函数。尽管有这个区别，这两个类是完全相同的。事实上，我们更希望能够让C++函数直接赋值给``Php::Value``对象，而跳过``Php::Function``构造函数，但这是不可能的，因为存在调用歧义。

``Php::Function``类可以像普通的``Php::Value``对象一样使用：你可以把它赋值给其他``Php::Value``对象，也可以在调用用户空间PHP函数时把它作为参数使用。在上面的例子中，我们正是这样做的：我们用我们自己的 "乘以二 "C++函数调用用户空间的my_iterate()函数。

### C++ 函数签名

你可以向``Php::Function``构造函数传递不同类型的C++函数，只要它们与以下两个函数签名兼容：

```cpp
Php::Value function();
Php::Value function(Php::Parameters &params);
```

在内核，``Php::Function``类使用一个C++的``std::function``对象来存储函数，所以凡是可以存储在这样一个``std::function``对象中的东西，都可以分配给``Php::Function``类。

## 类和对象

C++和PHP都是面向对象的编程语言，你可以在其中创建类和对象。PHP-CPP 库为你提供了将这两种语言结合起来的工具，并使本地 C++ 类可以从 PHP 中访问。

遗憾的是（但如果你考虑一下，也是符合逻辑的），并不是每一个可以想到的C++类都可以直接导出到PHP中。这需要更多的工作（虽然不是那么多）。首先，你必须确保你的类是从``Php::Base``派生出来的，其次，当你把你的类添加到扩展对象中时，你还必须指定所有你想从PHP中访问的方法。

1. 必须公开继承自``Php::Base``
2. 指定访问控制

### 查看例子：cppclassinphp

```cpp
// we are going to define a class
Php::Class<MyCustomClass> customClass("MyClass");

// add methods to it
customClass.method<&MyCustomClass::myMethod>("myMethod", Php::Final, {});
customClass.method<&MyCustomClass::myMethod>("myMethod2");
customClass.property("property1", "prop1");
customClass.property("property2", "prop2", Php::Protected);

customClass.method<&MyCustomClass::loop>("loopArray", {
    Php::ByVal("arr", Php::Type::Array)
});
customClass.method<&MyCustomClass::loop>("loopObject", {
    Php::ByVal("obj", Php::Type::Object)
});
```

在扩展对象中，

1. 通过``Php::Class``定义类;
2. 使用``method``方法来指定需要php代码访问的方法，和普通函数一样，也可以定义参数;
3. 使用``property``来指定类成员，并设置访问权限

静态方法也支持。静态方法是指一个不能访问``this``指针的方法。因此，在C++中，这种静态方法和普通函数是一样的，普通函数也不能访问``this``指针。静态C++方法与普通C++函数的唯一区别是在编译时：编译器允许静态方法访问私有数据。然而，静态方法的签名与普通函数的签名完全相同。

**PHP-CPP允许你注册静态方法。但是由于静态方法的签名与普通函数的签名完全相同，所以你注册的方法甚至不一定是同一个类的方法。普通函数和其他类的静态方法的签名完全一样，也可以注册! 从软件架构的角度来看，最好只使用同一类的静态方法，但C++允许你做的更多。**

```cpp
#include <phpcpp.h>

/**
 *  普通函数
 *
 *  因为普通函数没有this指针,
 *  所以它和静态方法拥有相同的签名
 *
 *  @param  params      Parameters passed to the function
 */
void regularFunction(Php::Parameters &params)
{
    // @todo add implementation
}

/**
 *  不会暴露给php调用的类
 */
class PrivateClass
{
public:
    /**
     *  C++ constructor and destructor
     */
    PrivateClass() = default;
    virtual ~PrivateClass() = default;

    /** 
     *  静态方法
     *
     *  静态方法没有this指针
     *  因此它的签名和普通函数相同
     *
     *  @param  params      Parameters passed to the method
     */
    static void staticMethod(Php::Parameters &params)
    {
        // @todo add implementation
    }
};

/**
 *  暴露给php使用的类
 */
class PublicClass : public Php::Base
{
public:
    /**
     *  C++ constructor and destructor
     */
    PublicClass() = default;
    virtual ~PublicClass() = default;

    /** 
     *  另一个静态方法
     *
     *  这个静态方法的签名和前面提到的普通函数和静态方法完全一样。
     *
     *  @param  params      Parameters passed to the method
     */
    static void staticMethod(Php::Parameters &params)
    {
        // @todo add implementation
    }
};

/**
 *  Switch to C context to ensure that the get_module() function
 *  is callable by C programs (which the Zend engine is)
 */
extern "C" {
    /**
     *  Startup function that is called by the Zend engine 
     *  to retrieve all information about the extension
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {
        // create static instance of the extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible
        Php::Class<PublicClass> myClass("MyClass");

        // 将PublicClass::staticMethod注册为一个可在PHP中调用的静态方法
        myClass.method<&PublicClass::staticMethod>("static1");

        // 普通函数与静态方法具有相同的签名。所以，没有什么能禁止你把普通函数也注册为静态方法。
        myClass.method<regularFunction>("static2");

        // 甚至来自完全不同类的静态方法也有相同的函数签名，因此可以注册
        myClass.method<&PrivateClass::staticMethod>("static3");

        // add the class to the extension
        myExtension.add(std::move(myClass));

        // 事实上，由于静态方法的签名与普通函数相同，你也可以将静态C++方法注册为普通的全局PHP函数。
        myExtension.add("myFunction", &PrivateClass::staticMethod);

        // return the extension
        return myExtension;
    }
}
```

在PHP代码中使用扩展的功能

```php
<?php
// this will call PublicClass::staticMethod()
MyClass::static1();

// this will call PrivateClass::staticMethod()
MyClass::static2();

// this will call regularFunction()
MyClass::static3();

// this will also call PrivateClass::staticMethod()
myFunction();
?>
```

### 访问修饰符

在PHP中（在C++中也是），你可以将方法标记为``public``、``private``或``protected``。为了使你的本地类也能实现这一点，你应该在向``Php::Class``对象添加方法时传递一个额外的flags参数。

```cpp
// description of the class so that PHP knows which methods are accessible
Php::Class<Counter> counter("Counter");

// register the increment method, and specify its parameters
counter.method<&Counter::increment>("increment", Php::Protected, { 
    Php::ByVal("change", Php::Type::Numeric, false) 
});

// register the decrement, and specify its parameters
counter.method<&Counter::decrement>("decrement", Php::Protected, { 
    Php::ByVal("change", Php::Type::Numeric, false) 
});

// register the value method
counter.method<&Counter::value>("value", Php::Public | Php::Final);
```

默认情况下，每一个方法 (还有每一个属性，但我们稍后会处理) 都是``公开``的。如果你想把一个方法标记为受保护的或私有的，你可以传递一个额外的 ``Php::Protected`` 或 ``Php::Private`` 标志。如果你也想把你的方法标记为抽象的或最终的，那么可以用``Php::Abstract``或``Php::Final``来对flag参数进行``位或``。PHP-CPP对value()方法做了这样的处理，这样在派生类中就不可能覆盖这个方法了。

请记住，C++ 类中的导出方法必须始终是公共的(即使在 PHP 中标记为私有或保护)。这是有道理的，因为毕竟你的方法会被 PHP-CPP 库调用，如果你把它们变成私有的，它们就会被库所忽略。

### Abstract and final

在上一节中，我们展示了如何使用``Php::Final``和``Php::Abstract``标志来创建一个final或抽象方法。如果你想让你的整个类成为抽象的或最终的，你可以通过把这个标志传递给``Php::Class``构造函数来实现。

```cpp
// description of the class so that PHP knows which methods are accessible
Php::Class<Counter> counter("Counter", Php::Final);
```

就像我们之前解释的那样，当你想注册一个抽象方法时，你应该在调用Php::Class::method()时传递一个Php::Abstract标志。然而，可能看起来很奇怪，这个方法也需要你传入一个真正的C++方法的地址。抽象方法通常没有实现，那么你需要提供一个方法的指针干什么呢？幸运的是，也有一种不同的方法来注册抽象方法。

```cpp
// register the decrement, and specify its parameters
counter.method("decrement", { 
    Php::ByVal("change", Php::Type::Numeric, false) 
});
```

要注册抽象方法，你可以简单地使用Counter::method()方法的另一种形式，它不接受指向C++方法的指针。

## 构造与析构

在 C++ 中的构造函数和析构函数与 PHP 中的 __construct() 和 __destruct() 方法之间有一个很小但非常重要的区别。

C++ 中的构造函数是在一个正在初始化的对象上调用的，但这个对象还没有处于初始化状态。你可以通过调用构造函数中的一个虚拟方法来体验这种情况。即使这个虚拟方法在派生类中被重写，这也将始终执行类本身的方法，而不是重写的实现。原因是在调用C++构造函数的过程中，对象还没有完全初始化，对象还不知道自己在类层次结构中的位置。因此对虚拟方法的调用不能传递给派生对象。

然而在 PHP 中，__construct() 方法有不同的行为。当它被调用时，对象已经被初始化了，因此对派生类中实现的抽象方法的调用是完全合法的。下面的 PHP 脚本是完全有效的，但是在 C++ 中不可能做类似的事情。

```php
// base class in PHP, in which the an abstract method is called
abstract class BASE 
{
    // constructor
    public function __construct() 
    {
        // call abstract method
        $this->doSomething();
    }

    // abstract method to be implemented by derived classes
    public abstract function doSomething();
}

// the derived class
class DERIVED extends BASE 
{
    // implement the abstract method
    public function doSomething() 
    {
        echo("doSomething()\n");
    }
}

// create an instance of the derived class
$d = new DERIVED();
```

这个脚本输出的是'doSomething()'。原因是``__construct()``根本就不是一个构造函数，而是一个很普通的方法，只是恰好是第一个被调用的方法，而且是在对象被构造后自动调用的。

这个区别对于作为一个C++程序员的你来说是很重要的，因为你千万不要把你的C++构造函数和PHP的``__construct()``方法混淆。在C++构造函数中，对象正在被构造，而且还不是所有的数据都可用。虚拟方法不能被调用，对象也还不存在于 PHP 用户空间中。

在构造函数完成后，PHP引擎接管控制并创建PHP对象，然后PHP-CPP库将该PHP对象链接到你的C++对象。只有在PHP对象和C++对象都完全构造完成之后，才会调用__construct()方法（就像普通方法一样）。因此，在你的类中同时拥有 C++ 构造函数和 __construct() 方法是很常见的。**C++ 构造函数用来初始化成员变量，而 __construct() 方法用来激活对象。**

```cpp
#include <phpcpp.h>

/**
 *  Simple counter class
 */
class Counter : public Php::Base
{
private:
    /**
     *  Internal value
     *  @var int
     */
    int _value = 0;

public:
    /**
     *  c++ constructor
     */
    Counter() = default;

    /**
     *  c++ destructor
     */
    virtual ~Counter() = default;

    /**
     *  php "constructor"
     *  @param  params
     */
    void __construct(Php::Parameters &params)
    {
        // copy first parameter (if available)
        if (!params.empty()) _value = params[0];
    }

    /**
     *  functions to increment and decrement
     */
    Php::Value increment() { return ++_value; }
    Php::Value decrement() { return --_value; }
    Php::Value value() const { return _value; }
};

/**
 *  Switch to C context so that the get_module() function can be
 *  called by C programs (which the Zend engine is)
 */
extern "C" {
    /**
     *  Startup function for the extension
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible
        Php::Class<Counter> counter("Counter");
        counter.method<&Counter::__construct>("__construct");
        counter.method<&Counter::increment>("increment");
        counter.method<&Counter::decrement>("decrement");
        counter.method<&Counter::value>("value");

        // add the class to the extension
        myExtension.add(std::move(counter));

        // return the extension
        return myExtension;
    }
}
```

上面的代码显示 ``__construct()`` 被注册为一个普通的方法。我们之前使用的例子（有Counter类的例子）现在被扩展了，这样就可以通过向 "构造函数"传递一个值来给它一个计数器的初始值。

```php
$counter = new Counter(10);
$counter->increment();
echo($counter->value()."\n");
```

因为``__construct()``方法被看作是一个普通的方法，所以你也可以指定它的参数，以及该方法是公共的、私有的还是保护的。``__construct()``也可以从PHP用户空间直接调用，所以派生方法可以显式调用``parent::__construct()``。

### 私有构造函数

就像其他方法一样，__construct()方法也可以被标记为私有或保护。如果你这样做，你将使你的类无法从PHP脚本中创建实例。重要的是要意识到，在这种情况下，C++ 构造函数和 C++ 解构函数仍然会被调用，因为会失败的是``__construct()``调用，而不是实际的对象构造。

是的，如果你把``__construct()``方法设为私有，并且在 PHP 脚本中执行了``new Counter()``调用，PHP-CPP 库将首先实例化你的类的一个新实例，然后报告一个错误，因为``__construct()``方法是私有的，然后立即析构对象（并调用 C++ 析构函数）。

```cpp
// add a private __construct method to the class, so that objects can 
// not be constructed from PHP scripts. Be aware that the C++ constructer 
// does get called - it will be the call to the first __construct() 
// function that will fail, and not the actual object construction.
counter.method<&Counter::__construct>("__construct", Php::Private);
```

### 克隆对象

如果你的类有一个复制构造函数，它就会自动成为可克隆的类。如果你不希望你的类可以被 PHP 脚本克隆，你可以做两件事：

1. 你可以从你的类中删除复制构造函数;
2. 你可以注册一个私有的 ``__clone()`` 方法，就像我们之前注册一个私有的 ``__construct()`` 方法一样。

删除复制构造函数

```cpp
/**
*  Remove the copy constructor
*  
*  By removing the copy constructor, the PHP clone operator will
*  automatically be deactivated. PHP will trigger an error if 
*  an object is attempted to be cloned.
*  
*  @param  counter
*/
Counter(const Counter &counter) = delete;
```

把克隆方法注册为私有

```cpp
// alternative way to make an object unclonable
counter.method("__clone", Php::Private);
```

### 构造对象

``Php::Value``类可以作为一个常规的PHP ``$variable``使用，因此你也可以用它来存储对象实例。但是如何创建全新的对象呢？为此，我们有 ``Php::Object`` 类，它是一个简单的重写的 ``Php::Value`` 类，带有可供选择的构造函数，还有一些额外的检查，以防止你使用 ``Php::Object`` 对象来存储对象以外的值。

```cpp
// new variable holding the string "Counter"
Php::Value counter0("Counter");

// new variable holding a newly created object of type "Counter",
// the __construct() gets called without parameters
Php::Object counter1("Counter");

// new variable holding a newly created object, and 
// the __construct() is being called with value 10
Php::Object counter2("Counter", 10);

// new built-in DateTime object, constructed with "now"
Php::Object time("DateTime", "now");

// valid, a Php::Object is an extended Php::Value, and 
// can thus be assigned to a base Php::Value object
Php::Value copy1 = counter1;

// invalid statement, a Php::Object can only be used for storing objects
Php::Object copy2 = counter0;
```

``Php::Object`` 的构造函数接收一个类的名称，以及一个可选的参数列表，这些参数将被传递给 ``__construct()`` 函数。你可以使用内置的 PHP 类和其他扩展的名称（如 DateTime），你的扩展的类（如 Counter），甚至是 PHP 用户空间的类。

如果你想在不调用 ``__construct()`` 函数的情况下构造一个你自己的 C++ 类的实例，也可以使用 ``Php::Object`` 类。例如，当 ``__construct()`` 方法是私有的，或者当你想绕过对你自己的 ``__construct()`` 方法的调用时，这就很有用。

```cpp
#include <phpcpp.h>

// actual class implementation
class Counter : public Php::Base
{
private:
    int _value = 0;

public:
    // c++ constructor
    Counter(int value) : _value(value) {}

    // c++ destructor
    virtual ~Counter() = default;

    // php "constructor"
    void __construct() {}

    // functions to increment and decrement
    Php::Value value() const { return _value; }
};

// function to create a new timer
Php::Value createTimer()
{
    return Php::Object("Counter", new Counter(100));
}

extern "C" {
    PHPCPP_EXPORT void *get_module() {
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible,
        // the __construct method is private because PHP scripts are not allowed
        // to create Counter instances
        Php::Class<Counter> counter("Counter");
        counter.method<&Counter::__construct>("__construct", Php::Private);
        counter.method<&Counter::value>("value");

        // add the class to the extension
        myExtension.add(std::move(counter));

        // add the factory function to create a timer to the extension
        myExtension.add("createTimer", createTimer);

        // return the extension
        return myExtension;
    }
}
```

在上面的代码中，我们将 Counter 类的 ``__construct()`` 函数设为私有。这使得不可能创建这个类的实例（无论是从 PHP 用户脚本中，还是通过调用 ``Php::Object("Counter")``），因为用这些方法构造对象最终会导致一个被禁止的 ``__construct()`` 调用。

``Php::Object`` 确实有一种替代的语法，它可以接受一个指向 C++ 类的指针（在堆上分配，使用运算符 ``new``！），并将这个指针变成一个 PHP 变量，而无需调用 ``__construct()`` 方法。请注意，你还必须指定类名，因为 C++ 类不保存任何关于它们自己的信息（比如它们的名字），而在 PHP 中，这样的信息是处理``反射``和 ``get_class()`` 等函数所需要的。

## 继承

PHP和C++都是支持类继承的面向对象编程语言。有一些区别。C++支持多继承，而PHP类只能有一个基类。为了弥补没有多重继承的不足，PHP支持接口和``traits``。

PHP-CPP库还允许你定义PHP接口，并创建PHP类和PHP接口的层次结构。

### 定义接口

如果你想让你的扩展定义一个接口，这样接口就可以从 PHP 用户空间脚本中实现，你可以用类似于定义类的方式来实现。唯一不同的是，你不使用``Php::Class<YourClass>``，而是使用``Php::Interface``实例。

```cpp
// description of the interface so that PHP knows which methods 
// are defined by it
Php::Interface interface("MyInterface");

// define an interface method
interface.method("myMethod", { 
    Php::ByVal("value", Php::Type::String, true) 
});
```

### 派生和实现

PHP-CPP 库试图使 PHP 和 C++ 的工作尽可能的透明。C++函数可以从PHP用户空间脚本中调用，C++类可以从PHP中访问。然而，归根结底PHP和C++还是不同的语言，由于C++没有PHP那样的反射功能，所以你必须显式地告诉PHP引擎该类实现了哪些基类和接口。

``Php::Class<YourClass>``对象有一个方法 ``extends()``和一个方法 ``implements()``，可以用来指定基类和实现的接口。你需要传入一个你之前配置的类或接口。我们来看一个例子。

```cpp
/**
 *  Switch to C context to ensure that the get_module() function
 *  is callable by C programs (which the Zend engine is)
 */
extern "C" {
    /**
     *  Startup function that is called by the Zend engine 
     *  to retrieve all information about the extension
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {
        // create static instance of the extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the interface so that PHP knows which methods 
        // are defined by it
        Php::Interface myInterface("MyInterface");

        // define an interface method
        myInterface.method("myMethod", { 
            Php::ByVal("value", Php::Type::String, true) 
        });

        // register our own class
        Php::Class<MyClass> myClass("MyClass");

        // from PHP user space scripts, it must look like the myClass implements
        // the MyInterface interface
        myClass.implements(myInterface);

        // the interface requires that the myMethod method is implemented
        myClass.method<&MyClass::myMethod>("myMethod", {
            Php::ByVal("value", Php::Type::String, true) 
        });

        // create a third class
        Php::Class<DerivedClass> derivedClass("DerivedClass");

        // in PHP scripts, it should look like DerivedClass has "MyClass" 
        // as its base
        derivedClass.extends(myClass);

        // add the interface and the classes to the extension
        myExtension.add(myInterface);
        myExtension.add(myClass);
        myExtension.add(derivedClass);

        // return the extension
        return myExtension;
    }
}
```

请注意，在 ``get_module()`` 函数中定义的 PHP 类的层次结构不一定要和 C++ 类的层次结构一致。你的 C++ 类 ``DerivedClass`` 根本不需要以 "MyClass"为基础，尽管在 PHP 脚本中它看起来像这样。为了代码的可维护性，当然最好让 PHP 的签名与 C++ 的实现多少有些相似。

## 魔术方法

## 魔术接口

## 特性

当我们开发 PHP-CPP 库时，我们不得不问自己一个问题，那就是我们应该遵循 PHP 惯例还是遵循 C++ 惯例来实现库中的许多功能。

在 PHP 脚本中，你可以使用``魔术方法``和``魔术接口``来为类添加特殊的行为。在C++类中，你也可以实现同样的功能，不过是通过使用``操作符重载``、``隐式构造函数``和``转换操作符``等技术。例如PHP的``__invoke()``方法，与C++中的``operator()``多少有些相同。我们问自己的问题是，我们是否应该自动将 PHP 的 ``__invoke`` 方法传递给 C++ 的 ``operator()`` 调用，还是在 C++ 中也使用同样的 ``__invoke()`` 方法名？

我们决定遵循PHP的惯例，在C++中也使用``魔术方法``和``魔术接口``（尽管我们必须承认，以两个下划线开头的方法并不能使代码看起来非常漂亮），但是通过使用魔术方法，对于初学C++的程序员来说，从PHP到C++的转换保持了更简单的状态。而且最重要的是，并不是所有的魔术方法和接口都能用C++的核心特性来实现（比如运算符重载），所以我们不得不使用一些魔术方法或接口。这就是为什么我们决定，既然我们必须在C++中使用一些魔术方法，那么我们也可以完全遵循PHP，在C++中也支持所有的PHP魔术方法。

除了PHP用户空间中的魔术方法和接口外，Zend引擎还有一些额外的功能是PHP用户空间脚本无法接触到的。这些功能只有扩展程序员才能使用。PHP-CPP库也支持这些特殊功能。这意味着，如果使用PHP-CPP来编写函数和类，可以实现编写纯PHP代码无法实现的事情。

### 额外的转换函数

在内部，Zend引擎有特殊的转换例程来将对象转换为整数、布尔值和浮点值。由于这样或那样的原因，一个PHP脚本只能实现``__toString()``方法，而其他所有的转换操作都远离它。PHP-CPP 库解决了这一限制，并允许实现其他的转换函数。

PHP-CPP 库的设计目标之一是尽可能地接近 PHP。出于这个原因，转换函数被赋予了与 ``__toString()`` 方法相匹配的名称：``__toInteger()``, ``__toFloat()`` 和 ``__toBool()``。

```cpp
#include <phpcpp.h>

/**
 *  A sample class, with methods to cast objects to scalars
 */
class MyClass : public Php::Base
{
public:
    /**
     *  C++ constructor and C++ destructpr
     */
    MyClass() = default;
    virtual ~MyClass() = default;

    /**
     *  Cast to a string
     *
     *  Note that now we use const char* as return value, and not Php::Value.
     *  The __toString function is detected at compile time, and it does
     *  not have a fixed signature. You can return any value that can be picked
     *  up by a Php::Value object.
     *
     *  @return const char *
     */
    const char *__toString()
    {
        return "abcd";
    }

    /**
     *  Cast to a integer
     *  @return long
     */
    long __toInteger()
    {
        return 1234;
    }

    /**
     *  Cast to a floating point number
     *  @return double
     */
    double __toFloat()
    {
        return 88.88;
    }

    /**
     *  Cast to a boolean
     *  @return bool
     */
    bool __toBool()
    {
        return true;
    }
};

/**
 *  Switch to C context to ensure that the get_module() function
 *  is callable by C programs (which the Zend engine is)
 */
extern "C" {
    /**
     *  Startup function that is called by the Zend engine
     *  to retrieve all information about the extension
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {

        // extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows
        // which methods are accessible
        Php::Class<MyClass> myClass("MyClass");

        // add the class to the extension
        myExtension.add(std::move(myClass));

        // return the extension
        return myExtension;
    }
}
```

当一个对象被转换为标量类型时，或者在标量上下文中使用时，会自动调用转换方法。下面的例子说明了这一点。

```php
// initialize an object
$object = new MyClass();

// cast it
echo((string)$object."\n");
echo((int)$object."\n");
echo((bool)$object."\n");
echo((float)$object."\n");
```

### 对象比较

如果你在PHP中用``<``, ``==``, ``!=``, ``>``等比较运算符比较两个对象，Zend引擎会运行一个对象比较函数。PHP-CPP库会拦截这个方法，并将比较方法传递给你的类的``__compare``方法。换句话说，如果你想安装一个自定义的比较操作符，你可以通过实现``__compare()``来实现。

```cpp
#include <phpcpp.h>
/**
 *  A sample class, that shows how objects can be compared
 */
class MyClass : public Php::Base
{
private:
    /**
     *  Internal value of the class
     *  @var    int
     */
    int _value;

public:
    /**
     *  C++ constructor
     */
    MyClass()
    {
        // start with random value
        _value = rand();
    }

    /**
     *  C++ destructor
     */
    virtual ~MyClass() = default;

    /**
     *  Cast the object to a string
     *  @return std::string
     */
    std::string __toString()
    {
        return std::to_string(_value);
    }

    /**
     *  Compare with a different object
     *  @param  that
     *  @return int
     */
    int __compare(const MyClass &that) const
    {
        return _value - that._value;
    }
};
```

当你在PHP脚本中尝试比较对象时，比较函数会被自动调用。当两个对象相同时，它应该返回0，当'this'对象较小时，返回小于0的值，当'this'对象较大时，返回大于0的值。

```php
// initialize a couple of objects
$object1 = new MyClass();
$object2 = new MyClass();
$object3 = new MyClass();

// compare the objects
if ($object1 < $object2)
{
    echo("$object1 is smaller than $object2\n");
}
else
{
    echo("$object1 is bigger than $object2\n");
}

if ($object1 == $object3)
{
    echo("$object1 is equal to $object3\n");
}
else
{
    echo("$object1 is not equal to $object3\n");
}
```

## 类成员

## 异常

PHP和C++都支持异常，通过PHP-CPP库，这两种语言之间的异常处理是完全透明的。在 C++ 中抛出的异常会自动传递给 PHP 脚本，而 PHP 脚本抛出的异常可以被 C++ 代码捕获，就像一个普通的 C++ 异常一样。

让我们从一个简单的抛出异常的C++函数开始。

```cpp
#include <phpcpp.h>

/**
 *  Simple function that takes two numeric parameters,
 *  and that divides them. Division by zero is of course
 *  not permitted - it will throw an exception then
 */
Php::Value myDiv(Php::Parameters &params)
{
    // division by zero is not permitted, throw an exception when this happens
    if (params[1] == 0) throw Php::Exception("Division by zero");

    // divide the two parameters
    return params[0] / params[1];
}

extern "C" {
    PHPCPP_EXPORT void *get_module() {
        static Php::Extension extension("my_extension", "1.0");
        extension.add<myDiv>("myDiv", {
            Php::ByVal("a", Php::Type::Numeric, true),
            Php::ByVal("b", Php::Type::Numeric, true)
        });
        return extension;
    }
}
```

你又一次看到了一个非常简单的扩展。在这个扩展中，我们创建了一个 "myDiv "函数，用来除以两个数字。但是除以零当然是不允许的，所以当试图除以零时，会产生一个异常。下面的 PHP 脚本就使用了这个功能。

```php
try
{
    echo(myDiv(10,2)."\n");
    echo(myDiv(8,4)."\n");
    echo(myDiv(5,0)."\n");
    echo(myDiv(100,10)."\n");
}
catch (Exception $exception)
{
    echo("exception caught\n");
}
```

这个例子显示了从C++代码中抛出异常并在PHP脚本中捕获异常是多么的简单。PHP-CPP 库会在内部捕获你的 C++ 异常并将其转换为 PHP 异常，但这一切都发生在引擎盖下。对于你这个扩展程序员来说，就好像你根本没有在两种不同的语言中工作，你可以简单地抛出一个Php::Exception对象，就好像它是一个普通的PHP异常一样。

### 在C++中捕获异常

反过来，如果你的扩展调用了一个PHP函数，而这个PHP函数恰好抛出了一个异常，你可以像捕获一个普通的C++异常一样捕获它。

```cpp
#include <phpcpp.h>

Php::Value callMe(Php::Parameters &params)
{
    // prevent that exceptions bubble up
    try
    {
        // call the function that was supplied by the user
        return params[0]();
    }
    catch (Php::Exception &exception)
    {
        return "Exception caught!\n";
    }
}

extern "C" {
    PHPCPP_EXPORT void *get_module() {
        static Php::Extension extension("my_extension", "1.0");
        extension.add<callMe>("callMe", {
            Php::ByVal("callback", Php::Type::Callable, true)
        });
        return extension;
    }
}
```

这段代码需要解释一下。正如我们之前提到的，``Php::Value`` 对象可以像使用普通的 PHP ``$variable`` 一样使用，因此你可以在其中存储整数、字符串、对象、数组等等。但这也意味着你可以用它来存储函数（因为 PHP 变量也可以用来存储函数）！而这正是我们要做的。

本例扩展中的 ``callMe()`` 函数只接收一个参数：一个它将立即调用的回调函数，回调函数的返回值也由 ``callMe()`` 函数返回。如果这个回调函数以某种方式抛出一个异常，它将被``callMe()``函数捕获，并返回一个替代的字符串("Exception caught!")。

```php
// call "callMe" for the first time, and supply a function that returns "first call"
$output = callMe(function() {
    return "First call";
});

// show output (this will be "First call")
echo("$output\n");

// call "callMe" for the second time, but throw an exception this time
$output = callMe(function() {
    throw new Exception("Sorry...\n");
    return "Second call\n";
});

// show output (this will be "Exception caught")
echo("$output\n");
```

这个 PHP 脚本使用了我们的扩展，并连续两次调用 callMe() 函数。首先用一个普通函数返回一个字符串，然后用一个抛出异常的函数（扩展会捕捉到这个异常）。输出结果正如你所期望的那样。

## 变量

## 全局常量和类级常量

### 常量

在 PHP 脚本中可以定义常量（包括全局常量和类级常量）。这也可以通过 PHP-CPP 来实现。如果你想把常量暴露在用户空间的PHP代码中，你可以通过在``get_module()``调用中添加常量到``Php::Extension``对象中来实现。

```cpp
// add integer constants
myExtension.add(Php::Constant("MY_CONSTANT_1", 1));
myExtension.add(Php::Constant("MY_CONSTANT_2", 2));

// floating point constants
myExtension.add(Php::Constant("MY_CONSTANT_3", 3.1415927));
myExtension.add(Php::Constant("MY_CONSTANT_4", 4.932843));

// string constants
myExtension.add(Php::Constant("MY_CONSTANT_5", "This is a constant value"));
myExtension.add(Php::Constant("MY_CONSTANT_6", "Another constant value"));

// null constants
myExtension.add(Php::Constant("MY_CONSTANT_7", nullptr));
```

在php中使用常量

```php
echo(MY_CONSTANT_1."\n");
echo(MY_CONSTANT_2."\n");
echo(MY_CONSTANT_3."\n");
echo(MY_CONSTANT_4."\n");
echo(MY_CONSTANT_5."\n");
echo(MY_CONSTANT_6."\n");
echo(MY_CONSTANT_7."\n");
```

PHP也支持类级常量的概念。在内部，在Zend引擎中，类级常量被实现为常规的类成员，但是常量属性没有 "public "或 "private "标志，而是用 "constant "标志来标记。PHP-CPP也暴露了这一点。你可以用``Php::Const``标志来注册类属性。

除此之外，一个``Php::Class``实例也有一个 "constant"方法，你可以将``Php::Constant``的实例添加到类中。从语义上看，这三种创建类级常量的方法都是相同的。

```cpp
/**
 *  The C++ class that we're going to expose
 *
 *  (For this example we use a completely empty class, as only examples
 *  are given on how to use constants)
 */
class Dummy : public Php::Base
{
};

/**
 *  Switch to C context so that the get_module() function can be
 *  called by C programs (which the Zend engine is)
 */
extern "C" {
    /**
     *  Startup function for the extension
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {
        static Php::Extension myExtension("my_extension", "1.0");

        // create a class objects
        Php::Class<Dummy> dummy("Dummy");

        // there are many different ways to add constants, but semantically,
        // they're all the same
        dummy.property("MY_CONSTANT_1", 1, Php::Const);
        dummy.property("MY_CONSTANT_2", "abcd", Php::Const);
        dummy.constant("MY_CONSTANT_3", "xyz");
        dummy.constant("MY_CONSTANT_4", 3.1415);
        dummy.add(Php::Constant("MY_CONSTANT_5", "constant string"));
        dummy.add(Php::Constant("MY_CONSTANT_5", true));

        // add the class to the extension
        myExtension.add(std::move(dummy));

        // return the extension
        return myExtension;
    }
}
```

### 运行时常量

如果你想在运行时从你的C++代码中找出一个用户空间常量的值，或者当你想找出一个常量是否被定义时，你可以简单地使用``Php::constant()``或``Php::defined()``函数。要在运行时定义常量，请使用``Php::define()``。

```cpp
/**
 *  Function that can be called from a PHP script
 */
void example_function()
{
    // check if a certain user space constant is defined
    if (Php::defined("USER_SPACE_CONSTANT"))
    {
        // retrieve the value of a constant
        Php::Value constant = Php::constant("ANOTHER_CONSTANT");

        // define other constants at runtime
        Php::define("DYNAMIC_CONSTANT", 12345);
    }
}
```

## 从php.ini读取配置

从php.ini文件中读取设置就像从普通PHP脚本中获取设置一样简单。在PHP脚本中，你可以使用内置的``ini_get()``函数从php.ini文件中读取设置，而在你的C++扩展中，你可以使用``Php::ini_get()``函数。

```cpp
/**
 *  Simple function that is used to demonstrate how settings from the
 *  php.ini file can be read
 */
void myFunction()
{
    // read in the "output_buffering" variable from the php.ini file
    int output_buffering = Php::ini_get("output_buffering");

    // read in the "variables_order" variable
    std::string variables_order = Php::ini_get("variables_order");
}
```

``Php::ini_get()``函数返回一个可以分配给字符串、整数和浮点数的对象。在上面的例子中，我们使用这个函数将设置直接分配给一个整数和一个``std::string``。

你只能从php.ini中获取预定义的变量。因此不可能用随机字符串调用``Php::ini_get()``. 如果你想使用你自己的变量，你必须先在get_module()函数中注册它们，然后才能调用``Php::ini_get()``来获取当前值。

```cpp
#include <phpcpp.h>

/**
 *  Simple function that is used to demonstrate how settings from the
 *  php.ini file can be read
 */
void myFunction()
{
    // read in a variable defined for this extension
    int var1 = Php::ini_get("my_extension.var1");

    // read in a string variable
    std::string var2 = Php::ini_get("my_extension.var2");
}

/**
 *  Switch to C contect so that the get_module() function can be
 *  called by the Zend engine
 */
extern "C" {
    /**
     *  The get_module() startup function
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() {

        // create extension object
        static Php::Extension extension("my_extension", "1.0");

        // export one function
        extension.add("myFunction", myFunction);

        // tell the PHP engine that the php.ini variables my_extension.var1
        // and my_extension.var2 are usable
        extension.add(Php::Ini("my_extension.var1", "default-value"));
        extension.add(Php::Ini("my_extension.var2", 12345));

        // return a pointer to the extension object
        return extension;
    }
}
```

## 扩展回调

## 命名空间

尽管在 PHP 脚本中，命名空间有非常丰富的实现方式，有特殊的关键字，如 ``use`` 和 ``namespace`` 以及特殊的常量，如 ``__NAMESPACE__``，但它们内部非常简单。

命名空间无非就是一个类或函数的前缀。如果你想让你的类或函数出现在一个特定的命名空间中，你只需要在类或函数名中添加一个前缀。下面的代码在 "myNamespace"命名空间中创建了一个函数 "myFunction"。

```cpp
// add the myFunction function to the extension, 
// and put it in namespace "myNamespace"
extension.add("myNamespace\\myFunction", myFunction);
```

如果你愿意，你可以使用``Php::Namespace``实用类来实现。这个类的签名和``Php::Extension``类完全一样，你也可以用它来注册类和函数。

```cpp
// create a namespace
Php::Namespace myNamespace("myNamespace");

// add the myFunction function to the namespace
myNamespace.add("myFunction", myFunction);

// @todo add more functions and classes to the namespace

// create a nested namespace
Php::Namespace nestedNamespace("nestedNamespace");

// @todo add functions and classes to the nested namespace

// add the nested namespace to the first namespace
myNamespace.add(std::move(nestedNamespace));

// add the namespace to the extension
extension.add(std::move(myNamespace));
```

``Php::Namespace``类只是一个容器，它会自动为你添加的所有类和函数添加一个前缀。正如你在例子中看到的那样，嵌套命名空间也是可能的。

在这个例子中，我们使用std::move()函数将嵌套的命名空间移动到父命名空间中，并将第一个命名空间移动到扩展中。移动比添加更有效率，尽管常规的``extension.add(myNamespace)``也是有效的。

## 动态加载

## 参考

1. [php-cpp](php-cpp.com)