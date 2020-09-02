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

每个PHP类都有 "魔术方法"。你可能已经在写PHP代码时知道这些方法：这些方法以两个下划线开头，名字像``__set()``,``__isset()``,``__call()``等等。

PHP-CPP库也支持这些魔术方法。使用一些C++编译器的技巧，C++编译器会检测你的类中是否存在方法，如果存在，它们会被编译到你的扩展中，并从PHP访问时被调用。

### 编译时检测

虽然你可能已经预料到这些魔术方法是``Php::Base``类中的虚函数，可以被重写，但其实不然。这些方法在编译时被C++编译器检测到（而且是非常正常的方法），只是碰巧有一个特定的名字。

由于编译时的检测，方法的签名有一定的灵活性。许多魔术方法的返回值都是分配给``Php::Value``对象的，这意味着只要你确保你的魔术方法返回的类型是可以分配给``Php::Value``的，你就可以在你的类中使用它。因此，你的 ``__toString()`` 方法可以返回一个 ``char*``、一个 ``std::string``、``Php::Value`` (甚至是一个整数！)，因为所有这些类型都可以分配给 ``Php::Value``。

用PHP-CPP实现的魔术方法的好处是，它们不会在PHP用户空间中变得可见。换句话说，当你在你的 C++ 类中定义了 ``__set()`` 或 ``__unset()`` 这样的函数时，这些函数不能被 PHP 脚本显式地调用，但是当一个属性被访问时，它们会被调用。

### 构造函数

一般情况下，魔术方法不需要注册就可以使用。当你在你的类中添加了一个像``__toString()``或``__get()``这样的魔术方法时，当一个对象被转换为字符串或一个属性被访问时，它将被自动调用。不需要在``get_module()``启动函数中显式启用魔术方法。

这个规则的唯一例外是``__construct()``方法。这个方法必须要明确注册。这其中的原因有很多。首先，``__construct()``方法没有固定的签名，通过显式添加到扩展中，你还可以指定它接受什么参数，以及``__construct()``方法应该是公共的、私有的还是保护的（如果你想创建不能从PHP实例化的类）。

另一个必须显式注册 ``__construct()`` 方法的原因是，与其他魔术方法不同，``__construct`` 方法必须在 PHP 中可见。在派生类的构造函数里面，经常需要对``parent::__construct()``进行调用。通过在``get_module()``函数中注册``__construct()``方法，你可以使该函数在PHP中可见。

### 克隆和析构

``__clone()``方法与``__construct()``方法非常相似。它也是在构造对象后直接调用的方法。区别在于``__clone()``是在一个对象被复制构造（克隆）后调用的，而``__construct()``是在普通构造函数之后调用的。

``__destruct()``方法会在对象被销毁之前被调用（也就是在C++的``destructor``运行之前）。

``__clone()`` 和 ``__destruct()`` 方法是常规的魔术方法（与 ``__construct()``不同），因此你不需要注册它们就可以使它们生效。如果你把这两个方法中的一个添加到你的类中，你将不必对``get_module()``启动函数做任何修改。如果有的话，PHP-CPP 库会自动调用它们。

在正常情况下，你可能不需要这些方法，也可以使用C++复制构造函数和C++析构函数。唯一不同的是，魔术方法是在处于完全初始化状态的对象上调用的，而C++复制构造函数和C++析构函数则是针对正在初始化的对象，或者是针对正在销毁的对象。

### 伪属性

通过``__get()``、``__set()``、``__unset()``和``__isset()``等方法，你可以定义伪属性。例如，它允许您创建只读属性，或在设置时检查其有效性的属性。

这些魔术方法与PHP脚本中的对应方法的工作原理完全一样，所以你可以轻松地将使用这些属性的PHP代码移植到C++中。

```cpp
#include <phpcpp.h>

/**
 *  A sample class, that has some pseudo properties that map to native types
 */
class User : public Php::Base
{
private:
    /**
     *  Name of the user
     *  @var    std::string
     */
    std::string _name;

    /**
     *  Email address of the user
     *  @var    std::string
     */
    std::string _email;

public:
    /**
     *  C++ constructor and C++ destructpr
     */
    User() = default;
    virtual ~User() = default;

    /**
     *  Get access to a property
     *  @param  name        Name of the property
     *  @return Value       Property value
     */
    Php::Value __get(const Php::Value &name)
    {
        // check if the property name is supported
        if (name == "name") return _name;
        if (name == "email") return _email;

        // property not supported, fall back on default
        return Php::Base::__get(name);
    }

    /**
     *  Overwrite a property
     *  @param  name        Name of the property
     *  @param  value       New property value
     */
    void __set(const Php::Value &name, const Php::Value &value) 
    {
        // check the property name
        if (name == "name") 
        {
            // store member
            _name = value.stringValue();
        }

        // we check emails for validity
        else if (name == "email")
        {
            // store the email in a string
            std::string email = value;

            // must have a '@' character in it
            if (email.find('@') == std::string::npos) 
            {
                // email address is invalid, throw exception
                throw Php::Exception("Invalid email address");
            }

            // store the member
            _email = email;
        }

        // other properties fall back to default
        else
        {
            // call default
            Php::Base::__set(name, value);
        }
    }

    /**
     *  Check if a property is set
     *  @param  name        Name of the property
     *  @return bool
     */
    bool __isset(const Php::Value &name) 
    {
        // true for name and email address
        if (name == "name" || name == "email") return true;

        // fallback to default
        return Php::Base::__isset(name);
    }

    /**
     *  Remove a property
     *  @param  name        Name of the property to remove
     */
    void __unset(const Php::Value &name)
    {
        // name and email can not be unset
        if (name == "name" || name == "email") 
        {
            // warn the user with an exception that this is impossible
            throw Php::Exception("Name and email address can not be removed");
        }

        // fallback to default
        Php::Base::__unset(name);
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
        Php::Class<User> user("User");

        // add the class to the extension
        myExtension.add(std::move(user));

        // return the extension
        return myExtension;
    }
}
```

上面的例子展示了如何创建一个User类，该类似乎有一个名称和电子邮件属性，但不允许你分配一个没有'@'字符的电子邮件地址，也不允许你删除属性。

```php
// initialize user and set its name and email address
$user = new User();
$user->name = "John Doe";
$user->email = "john.doe@example.com";

// show the email address
echo($user->email."\n");

// remove the email address (this will cause an exception)
unset($user->email);
```

### 魔术方法 __call(), __callStatic() and __invoke()

C++方法需要在你的扩展``get_module()``启动函数中明确注册，才能从PHP用户空间访问。然而，当你重写 ``__call()`` 方法时，你可以接受所有的调用（甚至是对不存在的方法的调用）。当有人从用户空间对一些看起来像方法的东西进行调用时，它将被传递给这个``__call()``方法。在脚本中，你可以这样使用``$object->something()``，``$object->whatever()``或者``$object->anything()``（方法的名称是什么并不重要），所有这些调用都会传递给C++类中的``__call()``方法。

``__callStatic()``方法类似于``__call()``方法，但适用于静态方法。对``YourClass::someMethod()``的静态调用可以自动传递给你的C++类的``__callStatic()``方法。

除了``__call()``和``__callStatic``函数，PHP-CPP库还支持``__invoke()``方法。这是一个当对象实例被当作函数使用时被调用的方法。这可以与C++类中的运算符``（）``重载相比。通过实现``__invoke()``方法，PHP用户空间的脚本可以创建一个对象，然后将其作为函数使用。

```cpp
#include <phpcpp.h>

/**
 *  A sample class, that accepts all thinkable method calls
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
     *  Regular method
     *  @param  params      Parameters that were passed to the method
     *  @return Value       The return value
     */
    Php::Value regular(Php::Parameters &params)
    {
        return "this is a regular method";
    }

    /**
     *  Overriden __call() method to accept all method calls
     *  @param  name        Name of the method that is called
     *  @param  params      Parameters that were passed to the method
     *  @return Value       The return value
     */
    Php::Value __call(const char *name, Php::Parameters &params)
    {
        // the return value
        std::string retval = std::string("__call ") + name;

        // loop through the parameters
        for (auto &param : params)
        {
            // append parameter string value to return value
            retval += " " + param.stringValue();
        }

        // done
        return retval;
    }

    /**
     *  Overriden __callStatic() method to accept all static method calls
     *  @param  name        Name of the method that is called
     *  @param  params      Parameters that were passed to the method
     *  @return Value       The return value
     */
    static Php::Value __callStatic(const char *name, Php::Parameters &params)
    {
        // the return value
        std::string retval = std::string("__callStatic ") + name;

        // loop through the parameters
        for (auto &param : params)
        {
            // append parameter string value to return value
            retval += " " + param.stringValue();
        }

        // done
        return retval;
    }

    /**
     *  Overridden __invoke() method so that objects can be called directly
     *  @param  params      Parameters that were passed to the method
     *  @return Value       The return value
     */
    Php::Value __invoke(Php::Parameters &params)
    {
        // the return value
        std::string retval = "invoke";

        // loop through the parameters
        for (auto &param : params)
        {
            // append parameter string value to return value
            retval += " " + param.stringValue();
        }

        // done
        return retval;
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

        // register the regular method
        myClass.method<&MyClass::regular>("regular");

        // add the class to the extension
        myExtension.add(std::move(myClass));

        // return the extension
        return myExtension;
    }
}
```

在php中使用

```php
// initialize an object
$object = new MyClass();

// call a regular method
echo($object->regular()."\n");

// call some pseudo-methods
echo($object->something()."\n");
echo($object->myMethod(1,2,3,4)."\n");
echo($object->whatever("a","b")."\n");

// call some pseudo-methods in a static context
echo(MyClass::something()."\n");
echo(MyClass::myMethod(5,6,7)."\n");
echo(MyClass::whatever("x","y")."\n");

// call the object as if it was a function
echo($object("parameter","passed","to","invoke")."\n");
```

输出如下

```bash
regular
__call something
__call myMethod 1 2 3 4
__call whatever a b
__callStatic something
__callStatic myMethod 5 6 7
__callStatic whatever x y
invoke parameter passed to invoke
```

### 转换为字符串

在PHP中，你可以在一个类中添加一个``__toString()``方法。当一个对象被转换为字符串时，或者当一个对象在字符串上下文中被使用时，这个方法会被自动调用。PHP-CPP 也支持这个 ``__toString()`` 方法。

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
     *  @return Value
     */
    Php::Value __toString()
    {
        return "abcd";
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

除了这里描述的魔术方法，你可能已经在编写PHP脚本时知道了，PHP-CPP库还引入了一些额外的魔术方法。这些方法包括额外的转换方法，以及比较对象的方法。

## 魔术接口

在PHP内核中，自带了一些特殊的"魔术"PHP接口，脚本编写者可以通过这些接口来实现为一个类添加特殊功能。这些接口的名称是'Countable'、'ArrayAccess'和'Serializable'。这些接口带来的功能，也可以用PHP-CPP来实现。

你可能会好奇为什么PHP有时会使用魔术方法（例如``__set``和``__unset``），有时会使用接口来改变一个类的行为。这种选择似乎并不统一。对我们来说，不清楚为什么有些特殊功能是用魔术方法来实现的，而有些特殊功能是通过实现接口来激活的。在我们看来，``Serializable``接口也可以用神奇的``__serialize()``和``__unserialize()``方法来实现，或者``__invoke()``方法也可以是一个"Invokable"接口。PHP 不是一种标准化的语言，有些东西看起来就是这样，因为有人觉得这样或那样的方式来实现它。

尽管如此，PHP-CPP库还是试图尽可能地接近PHP。这就是为什么在你的C++类中，你也可以使用特殊的接口（因为C++没有像PHP那样的接口），所以用纯虚函数的类来代替。

### SPL的支持

一个标准的PHP安装程序会附带标准PHP库（SPL）。这是一个建立在Zend引擎之上的扩展，它使用Zend引擎的特性来创建类和接口，如Countable、Iterator和ArrayAccess。

PHP-CPP 库也有这些名称的接口，它们的行为方式与 SPL 接口大致相同。但在内核中，PHP-CPP库不依赖于SPL。如果你实现了像``Php::ArrayAccess``或``Php::Countable``这样的C++接口，这和在PHP中写一个实现SPL接口的类是不同的。

PHP-CPP和SPL都是直接建立在Zend核心之上的，并且提供了相同的功能，但它们并不相互依赖。因此，如果没有加载SPL扩展，可以安全地使用PHP-CPP。

### Countable接口

通过实现``Php::Countable``接口，你可以创建能传递给PHP ``count()``函数的对象。

```cpp
#include <phpcpp.h>

/**
 *  The famous counter class, now also implements
 *  the Php::Countable interface
 */
class Counter : public Php::Base, public Php::Countable
{
private:
    /**
     *  The internal counter value
     *  @var int
     */
    int _value = 0;

public:
    /**
     *  C++ constructor and C++ destructor
     */
    Counter() {}
    virtual ~Counter() {}

    /**
     *  Methods to increment and decrement the counter
     */
    Php::Value increment() { return ++_value; }
    Php::Value decrement() { return --_value; }

    /**
     *  Method from the Php::Countable interface, that
     *  is used when a Counter instance is passed to the
     *  PHP count() function
     *  
     *  @return long
     */
    virtual long count() override { return _value; }
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
        Php::Class<Counter> counter("Counter");

        // add methods
        counter.method("increment", &Counter::increment);
        counter.method("decrement", &Counter::decrement);

        // add the class to the extension
        myExtension.add(std::move(counter));

        // return the extension
        return myExtension;
    }
}
```

我们之前使用的Counter类已经被修改，展示了如何制作实现``Php::Countable``接口的类。这很简单，你只需要添加``Php::Countable``类作为基类。这个``Php::Countable``类有一个纯虚函数``count()``，必须要实现。

而这就是你要做的一切。不需要在``get_module()``函数里面注册专门的``count()``函数，添加``Php::Countable``作为基类即可。

```php
// create a counter
$counter = new Counter();
$counter->increment();
$counter->increment();
$counter->increment();

// show the current value
echo(count($counter)."\n");
```

输出的结果是，正如预期的那样，数值为3。

### ArrayAccess接口

一个PHP对象可以通过实现``Php::ArrayAccess``接口变成一个变量，它的行为就像一个数组。当你这样做的时候，可以使用数组访问操作符（``$object["property"]``）访问对象。

在下面的例子中，我们使用``Php::Countable``和``Php::ArrayAccess``接口来创建一个可以用来存储字符串的关联数组类（记住：这只是一个例子，PHP已经支持关联数组了，所以这个例子有多大用处还值得商榷）。

```cpp
#include <phpcpp.h>

/**
 *  A sample Map class, that can be used to map string-to-strings
 */
class Map : public Php::Base, public Php::Countable, public Php::ArrayAccess
{
private:
    /**
     *  Internally, a C++ map is used
     *  @var    std::map<std::string,std::string>
     */
    std::map<std::string,std::string> _map;

public:
    /**
     *  C++ constructor and C++ destructpr
     */
    Map() {}
    virtual ~Map() {}

    /**
     *  Method from the Php::Countable interface that
     *  returns the number of elements in the map
     *  @return long
     */
    virtual long count() override
    {
        return _map.size();
    }

    /**
     *  Method from the Php::ArrayAccess interface that is
     *  called to check if a certain key exists in the map
     *  @param  key
     *  @return bool
     */
    virtual bool offsetExists(const Php::Value &key) override
    {
        return _map.find(key) != _map.end();
    }

    /**
     *  Set a member
     *  @param  key
     *  @param  value
     */
    virtual void offsetSet(const Php::Value &key, const Php::Value &value) override
    {
        _map[key] = value.stringValue();
    }

    /**
     *  Retrieve a member
     *  @param  key
     *  @return value
     */
    virtual Php::Value offsetGet(const Php::Value &key) override
    {
        return _map[key];
    }

    /**
     *  Remove a member
     *  @param key
     */
    virtual void offsetUnset(const Php::Value &key) override
    {
        _map.erase(key);
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
        Php::Class<Map> map("Map");

        // add the class to the extension
        myExtension.add(std::move(map));

        // return the extension
        return myExtension;
    }
}
```

``Php::ArrayAccess``有四个纯虚函数必须要实现。这些方法是用来检索和覆盖一个元素的方法，用来检查是否存在某个键的元素，以及用来删除一个元素的方法。在这个例子中，这些方法都已经实现了转发到一个常规的C++ ``std::map``对象。

在``get_module()``函数里面，Map被注册并添加到扩展中。但与其他许多例子不同的是，没有一个类方法被导出到PHP中。它只实现了``Php::Countable``接口和``Php::ArrayAccess``接口，所以它完全可以用来存储和检索属性，但是从一个PHP脚本来看，它没有任何可调用的方法。下面的脚本展示了如何使用它。

```php
// create a map
$map = new Map();

// store some values
$map["a"] = 1234;
$map["b"] = "xyz";
$map["c"] = 0;

// show the values
echo($map["a"]."\n");
echo($map["b"]."\n");
echo($map["c"]."\n");

// access a value that does not exist
echo($map["d"]."\n");

// this will result in a fatal error,
// the ArrayAccess methods are not exported to user space
echo($map->offsetGet("a")."\n");
```

输出不言而喻。该Map有三个成员，"1234"（字符串变量）、"xyz "和 "0"。

### Traversable接口

类也可以像普通数组一样，在foreach循环中使用。如果你想启用这个功能，你的类应该从``Php::Traverable``基类中扩展出来并实现``getIterator()``方法。

```php
// fill a map
$map = new Map();
$map["a"] = 1234;
$map["b"] = 5678;

// iterate over it
foreach ($map as $key => $value)
{
    // output the key and value
    echo("$key: $value\n");
}
```

PHP-CPP 库实现迭代器的方式与 SPL 略有不同，如果你一直在使用 PHP，你就会习惯这种方式。在PHP中，为了使一个类可以遍历（在foreach循环中使用），你必须实现``Iterator``接口或``IteratorAggregate``接口。这是一个奇特的架构。仔细想想，迭代器不是容器对象本身，那个容器对象才是可迭代的!。在我们上面的例子中，$map变量不是实际的迭代器，而是被迭代的容器。真正的迭代器是一个隐藏的对象，不会暴露在你的PHP脚本中，它控制着foreach循环。然而，SPL也会将该map称为迭代器。

因此，在 PHP-CPP 中，我们决定不遵循 SPL API，而是创建了一种全新的方式来实现可遍历类。要使一个类可遍历，必须从``Php::Traversable``基类中扩展出来，这就迫使你实现``getIterator()``方法。这个方法应该返回一个``Php::Iterator``实例。

``Php::Iterator``对象有五个方法是运行foreach循环所需要的。请注意，你的 ``Iterator`` 类不需要是一个可以从 PHP 中访问的类，也不需要从 ``Php::Base`` 派生。它是一个内部类，被foreach循环使用，但它并不（必须）存在于PHP用户空间。

```cpp
#include <phpcpp.h>

/**
 *  A sample iterator class that can be used to iterate
 *  over a map of strings
 */
class MapIterator : public Php::Iterator
{
private:
    /**
     *  The map that is being iterated over
     *  This is a reference to the actual map
     *  @var    std::map<std::string,std::string>
     */
    const std::map<std::string,std::string> &_map;

    /**
     *  The actual C++ iterator
     *  @var    std::map<std::string,std::string>l;::const_iterator;
     */
    std::map<std::string,std::string>::const_iterator _iter;

public:
    /**
     *  Constructor
     *  @param  object      The object that is being iterated over
     *  @param  map         The internal C++ map that is being iterated over
     */
    MapIterator(Map *object, const std::map<std::string,std::string> &map) :
        Php::Iterator(object), _map(map), _iter(map.begin()) {}

    /**
     *  Destructor
     */
    virtual ~MapIterator() {}

    /**
     *  Is the iterator on a valid position
     *  @return bool
     */
    virtual bool valid() override
    {
        return _iter != _map.end();
    }

    /**
     *  The value at the current position
     *  @return Value
     */
    virtual Php::Value current() override
    {
        return _iter->second;
    }

    /**
     *  The key at the current position
     *  @return Value
     */
    virtual Php::Value key() override
    {
        return _iter->first;
    }

    /**
     *  Move to the next position
     */
    virtual void next() override
    {
        _iter++;
    }

    /**
     *  Rewind the iterator to the front position
     */
    virtual void rewind() override
    {
        _iter = _map.begin();
    }
};

/**
 *  A sample Map class, that can be used to map string-to-strings
 */
class Map :
    public Php::Base,
    public Php::Countable,
    public Php::ArrayAccess,
    public Php::Traversable
{
private:
    /**
     *  Internally, a C++ map is used
     *  @var    std::map<std::string,std::string>
     */
    std::map<std::string,std::string> _map;

public:
    /**
     *  C++ constructor and C++ destructpr
     */
    Map() {}
    virtual ~Map() {}

    /**
     *  Method from the Php::Countable interface that
     *  returns the number of elements in the map
     *  @return long
     */
    virtual long count() override
    {
        return _map.size();
    }

    /**
     *  Method from the Php::ArrayAccess interface that is
     *  called to check if a certain key exists in the map
     *  @param  key
     *  @return bool
     */
    virtual bool offsetExists(const Php::Value &key) override
    {
        return _map.find(key) != _map.end();
    }

    /**
     *  Set a member
     *  @param  key
     *  @param  value
     */
    virtual void offsetSet(const Php::Value &key, const Php::Value &value) override
    {
        _map[key] = value.stringValue();
    }

    /**
     *  Retrieve a member
     *  @param  key
     *  @return value
     */
    virtual Php::Value offsetGet(const Php::Value &key) override
    {
        return _map[key];
    }

    /**
     *  Remove a member
     *  @param key
     */
    virtual void offsetUnset(const Php::Value &key) override
    {
        _map.erase(key);
    }

    /**
     *  Get the iterator
     *  @return Php::Iterator
     */
    virtual Php::Iterator *getIterator() override
    {
        // construct a new map iterator on the heap
        // the (PHP-CPP library will delete it when ready)
        return new MapIterator(this, _map);
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
        Php::Class<Map> map("Map");

        // add the class to the extension
        myExtension.add(std::move(map));

        // return the extension
        return myExtension;
    }
}
```

上面的例子进一步扩展了Map类。现在它实现了``Php::Countable``、``Php::ArrayAccess``和``Php::Traversable``。这意味着现在也可以在foreach循环中使用Map对象来迭代属性。

为了达到这个目的，我们必须将``Php::Traversable``类作为基类添加到Map类中，并实现``getIterator()``方法。这个方法返回一个新的``MapIterator``类，它是在堆上分配的。不用担心内存管理：PHP-CPP 库会在 foreach 循环结束的那一刻销毁迭代器。

``MapIterator``类是由``Php::Iterator``类派生出来的，实现了运行foreach循环所需的五个方法（current()、key()、next()、rewind()和valid()）。请注意，基本的``Php::Iterator``类期望将迭代过的对象传递给构造函数。这一点是必须的，这样迭代器对象才能确保只要迭代器存在，这个迭代对象就会一直在范围内。

我们内部的``MapIterator``实现只是一个C++迭代器类的小包装。当然，在需要的时候，你可以创建更复杂的迭代器。

### Serializable接口

通过实现 ``Php::Serializable`` 接口，你可以为一个类安装自定义的序列化和非序列化处理程序。PHP内置的``serialize()``函数是一个可以将数组或对象（甚至是充满数组和对象的嵌套数据结构变成简单字符串的函数。``unserialize()``方法正好相反，它将这样的字符串变回原始数据结构。

一个类的默认序列化实现将一个对象的所有公开可见的属性，并将它们连接成一个字符串。但由于你的类有一个本地实现，而且可能没有公共属性，你可能想安装一个自定义的序列化处理程序。在这个处理程序中，你就可以存储本地对象成员。

```cpp
#include <phpcpp.h>

/**
 *  Counter class that can be used for counting
 */
class Counter : public Php::Base, public Php::Serializable
{
private:
    /**
     *  The initial value
     *  @var    int
     */
    int _value = 0;

public:
    /**
     *  C++ constructor and destructor
     */
    Counter() = default;
    virtual ~Counter() = default;

    /**
     *  Update methods to increment or decrement the counter
     *  Both methods return the NEW value of the counter
     *  @return int
     */
    Php::Value increment() { return ++_value; }
    Php::Value decrement() { return --_value; }

    /**
     *  Method to retrieve the current counter value
     *  @return int
     */
    Php::Value value() const { return _value; }

    /**
     *  Serialize the object into a string
     *  @return std::string
     */
    virtual std::string serialize() override
    {
        return std::to_string(_value);
    }

    /**
     *  Unserialize the object from a string
     *  @param  buffer
     *  @param  size
     */
    virtual void unserialize(const char *buffer, size_t size) override
    {
        _value = std::atoi(buffer);
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
        Php::Class<Counter> counter("Counter");
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

上面的例子将之前看到的Counter例子，变成了一个可序列化的对象。``Php::Serializable``有两个纯虚函数，应该添加到你的类中。调用``serialize()``方法将对象变成一个字符串，对一个未初始化的对象调用``unserialize()``方法将其从一个序列化的字符串中恢复出来。请注意，如果一个对象正在使用``unserialize()``恢复，那么 ``__construct()``方法将不会被调用!

```php
// create an empty counter and increment it a few times
$counter = new Counter();
$counter->increment();
$counter->increment();

// turn the counter into a storable string
$serializedCounter = serialize($counter);

// revive the counter back into an object
$revivedCounter = unserialize($serializedCounter);

// show the counter value
echo($revivedCounter->value()."\n");
```

输出结果是2

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

## 类成员属性

当你在PHP中定义一个类时，你可以为它添加属性（成员变量）。然而，当你在一个本地C++类中添加成员变量时，你最好使用常规的本地成员变量，而不是PHP变量。原生变量的性能比PHP变量好得多，如果你也能用``int's``和``std::string``对象来存储整数或字符串，那么在``Php::Value``对象中存储这些变量就太疯狂了。

### 普通成员变量

很难想象，世界上有人愿意创建一个原生类，上面有常规的弱类型的PHP公共属性。然而，如果你坚持，你可以使用PHP-CPP库来实现。让我们以PHP中的一个类为例，看看它在C++中会是什么样子。

```php
class Example
{
    /**
     *  Define a public property
     */
    public $property1;

    /**
     *  Constructor
     */
    public function __construct()
    {
        // initialize the property
        $this->property1 = "xyz";
    }

    /**
     *  Example method
     */
    public function method()
    {
        // do something with the public property (like changing it)
        $this->property = "abc";
    }
}

// create an instance
$example = new Example();

// overwrite the public property
$example->property1 = "new value";
```

上面的例子创建了一个具有一个公共属性的类。这个属性可以被Example类访问，并且因为它是公共的，也可以被其他所有人访问，如示例中所示。如果你喜欢这样的类，你可以用PHP-CPP写一些类似的东西。

```cpp
#include <phpcpp.h>

/**
 *  C++ Example class
 */
class Example : public Php::Base
{
public:
    /**
     *  c++ constructor
     */
    Example() = default;

    /**
     *  c++ destructor
     */
    virtual ~Example() = default;

    /**
     *  php "constructor"
     *  @param  params
     */
    void __construct()
    {
        // get self reference as Php::Value object
        Php::Value self(this);

        // initialize a public property
        self["property1"] = "xyz";
    }

    /**
     *  Example method
     */
    void method()
    {
        // get self reference as Php::Value object
        Php::Value self(this);

        // overwrite the property
        self["property1"] = "abc";
    }
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
        // create static extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible
        Php::Class<Example> example("Example");

        // register the methods
        example.method<&Example::__construct>("__construct");
        example.method<&Example::method>("method");

        // the Example class has one public property
        example.property("property1", "xyz", Php::Public);

        // add the class to the extension
        myExtension.add(std::move(example));

        // return the extension
        return myExtension;
    }
}
```

该示例代码显示了如何在``get_module()``函数中初始化属性。

你也可以定义私有或受保护的属性，而不是公共属性，但即使是这样也可能不是你想要的，因为在原生C++变量中存储数据要快得多。

### 静态属性和类常量

静态属性和类常量可以用类似于属性的方式来定义。唯一不同的是，你必须传递``Php::Static``或``Php::Const``标志，而不是``Php::Public``、``Php::Private``或``Php::Protected``访问修饰符。

```cpp
#include <phpcpp.h>

// @todo your class definition

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
        // create static extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible
        Php::Class<Example> example("Example");

        // the Example class has a class constant
        example.property("MY_CONSTANT", "some value", Php::Const);

        // and a public static propertie
        example.property("my_property", "initial value", Php::Public | Php::Static);

        // add the class to the extension
        myExtension.add(std::move(example));

        // return the extension
        return myExtension;
    }
}
```

该类常量可以通过使用``Example::MY_CONSTANT``从PHP脚本中访问，静态属性可以使用``Example::$my_property``访问。

除了使用``property()``方法，你还可以使用``constant()``方法，或者使用``Php::Constant``类创建类常量。

### Smart properties

通过``get()``和``set()``魔术方法，你可以制作更高级的属性，这些属性可以直接映射到C++变量上，并且当一个属性被覆盖时，你可以执行额外的检查，从而使一个对象始终处于有效状态。

除此之外，通过 PHP-CPP 库，你还可以为属性分配 ``getter`` 和 ``setter`` 方法。每当一个属性被访问时，你的getter或setter方法就会被自动调用。

```cpp
#include <phpcpp.h>

/**
 *  C++ Example class
 */
class Example : public Php::Base
{
private:
    /**
     *  Example property
     *  @var    int
     */
    int _value = 0;

public:
    /**
     *  c++ constructor
     */
    Example() = default;

    /**
     *  c++ destructor
     */
    virtual ~Example() = default;

    /**
     *  Method to get access to the property
     *  @return Php::Value
     */
    Php::Value getValue() const
    {
        return _value;
    }

    /**
     *  Method to overwrite the property
     *  @param  value
     */
    void setValue(const Php::Value &value)
    {
        // overwrite property
        _value = value;

        // sanity check: the value should never exceed 100
        if (_value > 100) _value = 100;
    }

    /**
     *  Method to retrieve the double property value
     *  @return Php::Value
     */
    Php::Value getDouble() const
    {
        return _value * 2;
    }
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
        // create static extension object
        static Php::Extension myExtension("my_extension", "1.0");

        // description of the class so that PHP knows which methods are accessible
        Php::Class<Example> example("Example");

        // register the "value" property, with the methods to get and set it
        example.property("value", &Example::getValue, &Example::setValue);

        // register a read-only "double" property, with a method to get it
        example.property("double", &Example::getDouble);

        // add the class to the extension
        myExtension.add(std::move(example));

        // return the extension
        return myExtension;
    }
}
```

下面的PHP脚本使用了这一点。它创建了一个示例对象，将值属性设置为500（这是不允许的，高于100的值会被四舍五入到100），然后它读出双倍值。

```php
// create object
$object = new Example();

// set the value
$object->value = 500;

// show the double value
echo($object->double."\n");

// update the double value
// (this will trigger an error, this is a read-only property)
$object->double = 300;
```

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

PHP中的变量是弱类型的。因此，一个变量可以容纳任何可能的类型：整数、字符串、浮点数，甚至一个对象或数组。而C++则是一种强类型语言。在C++中，一个整数变量总是有一个数值，而一个字符串变量总是持有一个字符串值。

当你把本地代码和PHP代码混合在一起时，你需要把弱类型的PHP变量转换成本地变量，反之则是：把本地变量转换成弱类型的PHP变量。PHP-CPP库提供了``Php::Value``类，使这个任务变得非常简单。

### Zval's

如果你曾经花时间用纯C语言编写过PHP扩展，或者你曾经读过一些关于PHP内部的东西，你一定听说过``zval``的。zval是一个存储PHP变量的C结构。在内核中，这个zval保留了一个``refcount``、一个多种类型的联合体和一些其他成员。每次访问这样的zval，对它进行复制，或者对它进行写入，你都必须打破头正确更新``refcount``，或将zval分割成不同的zval，显式调用复制构造函数，分配或释放内存（使用特殊的内存分配例程），或者选择不这样做，让zval单独存在。

更糟糕的是，在Zend引擎中，有数百个不同的未被记录的宏和函数可以操作这些zval变量。有专门的宏针对zval，有通过``指针``指向zval的宏，有通过``指针的指针``指向zval的宏，甚至有通过``指针的指针的指针``指向zval的宏。

每一个PHP模块、每一个PHP扩展和每一个内置的PHP函数都在忙于处理这些zval结构。没有人花时间把这样的zval包在一个简单的C++类中，为你完成所有这些管理，这是一个很大的惊喜。C++就是这样一门不错的语言，它的构造函数、析构函数、转换运算符和运算符重载，可以封装这些复杂的zval处理。

PHP-CPP引入了``Php::Value``对象，它的接口非常简单，可以消除所有zval处理的问题。在内部，``Php::Value``对象是zval变量的一个包装器，但它完全隐藏了zval处理的复杂性。

### 标量类型

``Php::Value``对象可以用来存储标量类型。可以是整数、浮点数、字符串、布尔值和空值等变量。

```cpp
Php::Value value1 = 1234;
Php::Value value2 = "this is a string";
Php::Value value3 = std::string("another string");
Php::Value value4 = nullptr;
Php::Value value5 = 123.45;
Php::Value value6 = true;
```

``Php::Value``类有转换操作符，可以将对象转换成几乎所有可以想到的本地类型。当你可以访问一个``Php::Value``对象，但想把它存储在一个（访问速度快得多的）本地变量中时，你可以简单地赋值它。

```cpp
void myFunction(const Php::Value &value)
{
    int value1 = value;
    std::string value2 = value;
    double value3 = value;
    bool value4 = value;
}
```

如果 ``Php::Value`` 对象持有一个对象，并且你把它用成一个字符串，那么对象的 ``__toString()`` 方法就会被调用，这和你在 PHP 脚本中把变量用成字符串的情况完全一样。

许多不同的操作符也被重载，因此你可以在算术操作中直接使用``Php::Value``对象，将其与其他变量进行比较，或者将其发送到一个输出流。

```cpp
void myFunction(Php::Value &value)
{
    value += 10;
    Php::out << value << std::endl;
    if (value == "some string")
    {

    }

    int result = value - 8;
}
```

``Php::Value``对象对大多数类型都有隐式构造函数。这意味着每一个接受``Php::Value``作为参数的函数也可以用原生类型来调用，在应该返回``Php::Value``的函数中，你可以简单地指定一个标量返回值（它将被编译器自动转换为``Php::Value``对象）。

```cpp
Php::Value myFunction(const Php::Value &value)
{
    if (value == 12)
    {
        return "abc";
    }
    else if (value > 100)
    {
        return myFunction(12);
    }

    return nullptr;
}
```

正如你在例子中看到的，你几乎可以用``Php::Value``对象做任何事情。在内部，它完成了所有的zval操作，有时会变得很复杂，但对于你这个扩展程序员来说，没什么好担心的。

### 字符串

字符串可以轻松地存储在``Php::Value``对象中。将一个字符串赋给``Php::Value``，或者将一个``Php::Value``转换为一个字符串是如此的简单，以至于几乎没有任何解释的必要。通常情况下，赋值运算符和转换运算符即可。然而，当性能是一个问题时，你可以考虑直接访问``Php::Value``对象的内部缓冲区。

当一个``Php::Value``被转换为``std::string``时，整个字符串的内容会从``Php::Value``对象复制到``std::string``对象中。如果你不想做这样一个完整的拷贝，你可以把值投给一个``const char *``来代替。这使你可以直接访问``Php::Value``对象内部的缓冲区。字符串的大小可以用``size()``方法来检索。但你必须意识到，一旦``Php::Value``脱离了作用域，缓冲区的指针就不再保证有效了。

```cpp
/**
 *  Example function
 *  @param  params
 */
void myFunction(Parameters &params)
{
    // store the first parameter in a std::string (the entire string
    // buffer is copied from the Php::Value object to the std::string)
    std::string var1 = params[0];

    // it also is possible to cast the object into a const char *. This works
    // too, but the buffer is only valid for as long as the Php::Value object
    // stays in scope
    const char *var2 = params[0];
    size_t var2size = params[0].size();
}
```

也可以直接写到内核的``Php::Value``缓冲区。当你把一个字符串分配给``Php::Value``对象时，整个字符串缓冲区也会被复制。不管你赋值的字符串是``std::string``还是``char*``都会有一个拷贝。对于少量字节来说，这几乎不是问题，如果你换一种方式，会让你的代码可读性大大降低。但如果你要复制很多字节，你最好能直接访问缓冲区。

```cpp
/**
 *  Example function to read bytes from a filedescriptor, and
 *  return it as a Php::Value object
 *
 *  @param  fd          Filedescriptor
 *  @return Php::Value
 */
Php::Value readExample1(int fd)
{
    // buffer to read the bytes in
    char buffer[4096];

    // read the buffer
    ssize_t bytes = read(fd, buffer, 4096);
    if (bytes < 0) bytes = 0;

    // convert the buffer to a Php::Value object and return it
    return Php::Value(buffer, bytes);
}

/**
 *  Another example function, that does the same as the previous
 *  function, but now it reads the bytes directly into a Php::Value
 *  buffer, and does not use an intermediate buffer.
 *
 *  @param  fd          Filedescriptor
 *  @param  Php::Value
 */
Php::Value readExample2(int fd)
{
    // result variable
    Php::Value result;

    // resize the buffer to 4096 bytes, the reserve() method resizes
    // the internal buffer to the appropriate size, and returns a pointer
    // to the buffer
    char *buffer = result.reserve(4096);

    // read in the bytes directly into the just allocated buffer
    ssize_t bytes = read(fd, buffer, 4096);
    if (bytes < 0) bytes = 0;

    // resize the buffer to the actual number of bytes in it (this
    // is necessary, otherwise the PHP strlen() returns 4096 even
    // when less bytes were available
    result.reserve(bytes);

    // return the result
    return result;
}
```

第一个例子函数比较容易读懂。``read()``系统调用用于向本地缓冲区填充字节。然后将这个本地缓冲区转换为``Php::Value``对象并返回。

第二个示例函数更有效率，因为现在系统调用``read()``会立即将字节读到``Php::Value``对象的缓冲区中，而不是读到一个临时缓冲区中。作为一个程序员，你必须根据你的需求在这些算法中选择一种：**简单的代码或更高效的代码**。

### 数组

PHP支持两种数组类型：常规数组（以数字为索引）和关联数组（以字符串为索引）。``Php::Value``对象也支持数组。通过使用数组访问操作符(方括号)给``Php::Value``对象赋值，你会自动把它变成一个数组。

```cpp
// create a regular array
Php::Value array;
array[0] = "apple";
array[1] = "banana";
array[2] = "tomato";

// an initializer list can be used to create a filled array
Php::Value filled({ "a", "b", "c", "d"});

// you can cast an array to a vector, template parameter can be
// any type that a Value object is compatible with (string, int, etc)
std::vector<std::string> fruit = array;

// create an associative array
Php::Value assoc;
assoc["apple"] = "green";
assoc["banana"] = "yellow";
assoc["tomato"] = "green";

// the variables in an array do not all have to be of the same type
Php::Value assoc2;
assoc2["x"] = "info@example.com";
assoc2["y"] = nullptr;
assoc2["z"] = 123;

// nested arrays are possible too
Php::Value assoc2;
assoc2["x"] = "info@example.com";
assoc2["y"] = nullptr;
assoc2["z"][0] = "a";
assoc2["z"][1] = "b";
assoc2["z"][2] = "c";

// assoc arrays can be cast to a map, indexed by string
std::map<std::string,std::string> map = assoc2;
```

从数组中读取数据也同样简单。你也可以使用数组访问运算符（方括号）来实现。

```cpp
Php::Value array;
array["x"] = 10;
array["y"] = 20;

Php::out << array["x"] << std::endl;
Php::out << array["y"] << std::endl;
```

还有一个特殊的``Php::Array``类。这是一个扩展的``Php::Value``类，在构造时，立即以空数组开始（不像``Php::Value``对象默认构造为``NULL``值）。

```cpp
// create empty array
Php::Array array1;

// Php::Value is the base class, so you can assign Php::Array objects
Php::Value array2 = array1;

// impossible, a Php::Array must always be an array
array1 = 100;
```

### 对象

就像``Php::Array``类是一个扩展的``Php::Value``，初始化为一个空数组一样，也有一个``Php::Object``类在构造时成为一个对象。默认情况下，这是一个``stdClass``的实例（PHP最简单的类）。

```cpp
// create empty object of type stdClass
Php::Object object;

// Php::Value is the base class, so you can assign Php::Object objects
Php::Value value = object;

// impossible, a Php::Object must always be an object
object = "test";

// object properties can be accessed with square brackets
object["property1"] = "value1";
object["property2"] = "value2";

// to create an object of a different type, pass in the class name
// to the constructor with optional constructor parameters
object = Php::Object("DateTime", "now");

// methods can be called with the call() method
Php::out << object.call("format", "Y-m-d H:i:s") << std::endl;

// all these methods can be called on a Php::Value object too
Php::Value value = Php::Object("DateTime", "now");
Php::out << value.call("format", "Y-m-d H:i:s") << std::endl;
```

当你用PHP-CPP库创建了自己的类，你可以使用相同的``Php::Object``类来制作它的实例。因为PHP和C++是不同的语言，所以从函数中返回的对象实例（``Php::Value``或``Php::Object``实例）和在C++代码中内核使用的变量（普通的C++指针）是有区别的。PHP-CPP允许你轻松转换这两种类型。

```cpp
#include <phpcpp.h>

class MyClass : public Php::Base
{
    /**
     *  First factory method
     *  @return Php::Value      object holding a new MyClass instance
     */
    static Php::Value factory1()
    {
        // use the Php::Object class to create an instance (this will
        // result in __construct() being called)
        return Php::Object("MyClass");
    }

    /**
     *  Alternative factory method
     *  @return Php::Value
     */
    static Php::Value factory2()
    {
        // create an instance ourselves
        MyClass *object = new MyClass();

        // the object now only exists as C++ object, to ensure that it is also
        // registered as an object in PHP user space, we wrap it in a
        // Php::Object class (which is an extended Php::Value class). Because
        // PHP supports reflection it is necessary to also pass in the class
        // name. The __construct() method will _not_ be called - because the
        // C++ object is already instantiated.
        return Php::Object("MyClass", object);
    }

    /**
     *  Method that returns 'this' to allow chaining ($x->chain()->chain()).
     *  @return Php::Value
     */
    Php::Value chain()
    {
        // the Php::Value has an implicit constructor for Php::Base pointers.
        // This means that you can safely return 'this' from a method, which
        // will automatically be converted into a valid Php::Value object. This
        // works only for pointers to objects that already exist in PHP user
        // space.
        return this;
    }

    /**
     *  Method that gets a MyClass instance as parameter
     *  @param  params      vector holding all parameters
     */
    void process(Php::Parameters &params)
    {
        // store the first parameter in a Php::Value object
        Php::Value value = params[0];

        // if you know for sure that the 'value' variable holds a (wrapped)
        // instance of a MyClass object, you can convert the value back into
        // a pointer to the original C++ object by calling the 'implementation'
        // method.
        //
        // Note that this only works for value objects that hold instances of
        // C++ classes defined by your extension! Calling the 'implementation()'
        // method on a non-object, on an object of a user space class, or of
        // a core PHP class or a class from a different extension will probably
        // result in a crash!
        MyClass *object = (MyClass *)value.implementation();
    }
};
```

### 迭代

``Php::Value``类实现了``begin()``和``end()``方法，就像许多C++ STL容器一样。因此，你可以像遍历一个``std::map``类一样遍历一个``Php::Value``。

```cpp
/**
 *  Function that accepts an array as parameter
 *  @param  array
 */
void myFunction(const Php::Value &value)
{
    // assum the value variable holds an array or object, it then
    // is possible to iterate over the values or properties
    for (auto &iter : value)
    {
        // output key and value
        Php::out << iter.first << ": " << iter.second << std::endl;
    }
}
```

迭代值是一个``std::pair<Php::Value::Php::Value>``。你可以访问它的属性'first'来获取当前的键，而属性'second'来获取当前的值。这和你在``std::map``上迭代的方式是一样的。

你可以遍历所有持有对象或数组的``Php::Value``对象。当你在一个数组上迭代时，迭代器只是简单地迭代数组中的所有记录。

对于对象来说，有一些东西需要考虑。如果你迭代的对象实现了``Iterator``或``IteratorAggregate``接口，C++迭代器就会使用这些内置的接口并调用它的方法来遍历对象。对于常规对象（那些没有实现``Iterator``或``IteratorAggregate``的对象），迭代器只是简单地迭代对象的所有公共属性。

一个迭代器可以在两个方向上使用：操作符``++``以及操作符``--``都可以使用。但要注意使用``--``操作符。如果``Php::Value``对象持有一个实现了``Iterator``或``IteratorAggregate``的对象，反向迭代就无法进行，因为内部迭代器只有一个``next()``方法，PHP-CPP库没有办法指示内部迭代器向后移动。

同时要注意``++``后缀操作符的返回值。通常情况下，后缀增量操作会返回操作前的原始值。当你在实现了 ``Iterator`` 或 ``IteratorAggregate`` 的对象上进行迭代时，情况就不同了，因为 PHP-CPP 库不可能复制一个 PHP 迭代器。因此，``++``后缀操作符（只有在``Iterator``或``IteratorAggregate``对象上使用时）会返回一个全新的迭代器，该迭代器回到对象的前部位置。但请记住，在C++和PHP（以及许多其他编程语言）中，使用``++``前缀操作符要明智得多，因为这不需要对原始对象进行复制，所以无论如何你都不应该使用``++``后缀操作符。

### 函数

当一个``Php::Value``对象持有一个可调用的对象时，你可以使用``（）``操作符来调用这个函数或方法。

```cpp
// create a string with a function name
Php::Value date = "date";

// "date" is a built-in PHP function and thus can it be called
Php::out << date("Y-m-d H:i:s") << std::endl;

// create a date-time object
Php::Object now = Php::Object("DateTime","now");

// create an array with two members, the datetime object
// and the name of a method
Php::Array array();
array[0] = now;
array[1] = "format";

// an array with two members can be called too, the first
// member is seen as the object, and the second as the
// name of the method
Php::out << array("Y-m-d H:i:s") << std::endl;
```

### 全局变量

要读取或更新全局PHP变量，你可以使用``Php::GLOBALS``变量。这个变量的工作原理和PHP脚本中的``$GLOBALS``变量差不多。

```cpp
// set a global PHP variable
Php::GLOBALS["a"] = 12345;

// global variables can be of any type
Php::GLOBALS["b"] = Php::Array({1,2,3,4});

// nested calls are (of course) supported
Php::GLOBALS["b"][4] = 5;

// and global variables can also be read
Php::out << Php::GLOBALS["b"] << std::endl;
```

除了``$GLOBALS``变量之外，PHP还允许你使用``$_GET``、``$_POST``、``$_COOKIE``、``$_FILES``、``$_SERVER``、``$_REQUEST``和``$_ENV``变量来访问变量。在你的C++扩展中，你可以用全局变量``Php::GET``, ``Php::POST``, ``Php::COOKIE``, ``Php::FILES``, ``Php::SERVER``, ``Php::REQUEST``和``Php::ENV``做类似的事情。这些都是全局的、只读的、具有重载操作符``[]``方法的对象。因此，你可以像访问关联数组一样访问它们。

```cpp
// retrieve the value of a request variable
int age = Php::REQUEST["name"];

// or retrieve the value of a server variable
std::string referer = Php::SERVER["HTTP_REFERER"];
```

### 小心C++全局变量

与PHP脚本不同的是，PHP脚本只能处理单个请求会话，而扩展则是用来处理多个请求会话。这意味着当你在扩展中使用全局C++(!)变量时，这些变量不会在会话之间被设置回初始值。然而，``Php::GLOBALS``变量总是在每个新的会话开始时重新初始化。

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

## 扩展回调函数

``get_module()``函数在你的扩展启动时被调用。它返回一个内存地址，在那里Zend引擎可以找到关于你的扩展的所有相关信息。

在这个``get_module()``的调用之后，你的扩展就会被加载，并将被用来处理多个请求会话。这是标准PHP脚本和本地扩展之间的一个重要区别，因为标准PHP脚本只处理单个请求。但扩展服务于多个请求后。

如果你使用全局的C++变量，这种区别就显得尤为重要。这样的全局变量会在扩展加载时被初始化（而不是在每个请求开始时）。你对全局变量所做的更改会保留它们的值，因此后续的请求会看到更新后的值。

顺便说一下，这只发生在本地变量上。存储在``Php::GLOBALS``对象中的全局PHP变量，会在每次请求开始时重新初始化。你不必担心你对全局PHP变量所做的修改：在下一个请求开始时，``Php::GLOBALS``对象是全新的，你在上一个请求中所做的修改不再可见。

回到全局的C++变量。如果你想在一个新的请求开始时重置一个全局变量，你可以注册一个特殊的回调函数，这个函数在每个请求前被调用。

```cpp
#include <phpcpp.h>

/**
 *  Global variable that stores the number of times
 *  the function updateCounters() has been called in total
 *  @var    int
 */
int invokeTotalCount = 0;

/**
 *  Global variable that keeps track how many times the
 *  function updateCounters() was called during the
 *  current request
 *  @var    int
 */
int invokeDuringRequestCount = 0;

/**
 *  Native function that is callable from PHP
 *
 *  This function updates a number of global variables that count
 *  the number of times a function was called
 */
void updateCounters()
{
    // increment global counters
    invokeTotalCount++;
    invokeDuringRequestCount++;
}

/**
 *  Switch to C context, because the Zend engine expects get get_module()
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

        // install a callback that is called at the beginning
        // of each request
        extension.onRequest([]() {

            // re-initialize the counter
            invokeDuringRequestCount = 0;
        });

        // add the updateCounter method to the extension
        extension.add("updateCounters", updateCounters);

        // return the extension details
        return extension;
    }
}
```

``Php::Extension``类有一个方法``onRequest()``，在上面的例子中用来注册一个回调函数。这个回调会在每个请求之前被调用。正如你所看到的，它是允许使用C++ lambda函数的。

``onRequest()``并不是``Php::Extension``对象中唯一可以注册回调的方法。事实上，有四种不同的``on*()``方法可以使用。

- void onStartup(const std::function<void()> &callback);
- void onRequest(const std::function<void()> &callback);
- void onIdle(const std::function<void()> &callback);
- void onShutdown(const std::function<void()> &callback);

当Zend引擎已经加载了你的扩展，并且其中的所有函数和类都被注册时，启动回调被调用。如果你想在函数被调用之前初始化扩展中的其他变量，你可以使用onStartup()函数并注册一个回调来运行这个初始化代码。

Zend引擎初始化后，就可以处理请求了。在上面的例子中，我们使用了onRequest()方法来注册一个回调，这个回调会在每个请求前被调用。除此之外，你还可以安装一个回调，当Zend引擎进入空闲状态时，这个回调会在每次请求后被调用。

等待下一个请求。这可以通过Php::Extension对象中的onIdle()方法来实现。

第四个可以注册的回调是在PHP关闭前被调用的回调。如果有什么需要清理的地方，可以安装这样的回调，并从中运行清理代码。

### 预先fork的Web引擎 (如Apache)

如果你在一个``pre-forked``的web服务器（比如Apache）上运行PHP，你的扩展会在各种工作进程被fork之前被加载和初始化。这样做的后果是，``get_module()``函数和你可选的``onStartup()``回调函数被父进程调用，而所有其他回调和实际的页面处理被子进程调用。因此，对``getpid()``的调用(或其他用于检索当前进程信息的函数)将在``onStartup``回调中返回其他东西，就像在其他扩展函数中一样。

你可能要因此而小心。最好不要在启动函数中做一些在进程fork成不同子进程时可能无法工作的事情（比如打开文件描述符）。还有一点需要注意的是，启动函数只在Apache启动（或重载，见后文）时被父进程调用，而关闭函数则被每个平滑退出的子进程调用。因此，``onShutdown``不仅在Apache进程停止时被调用，而且在其中一个工作进程因为不再需要而退出时，或者因为它被一个新的工作进程取代而被调用。

``get_module()``函数在你的扩展被初始化时被调用。但不仅如此。当apache被重载时(例如通过给命令行指令 "apachectl reload")，你的``get_module()``会被第二次调用，你在``Extension::onStartup()``中注册的回调也会被再次调用。这通常不是问题，因为在第一次调用``get_module()``后，静态扩展对象处于锁定状态，在第二次调用``get_module()``时，你试图添加到扩展对象中的函数和类会被直接忽略。

### 注意多线程

如果你的扩展运行在多线程的PHP安装模式上，你需要格外小心。大多数PHP安装模式（Apache、CLI脚本等）一次只服务一个请求，按顺序进行。然而，有一些PHP安装模式使用了多线程，并且可以并行处理多个请求。如果你的扩展在这样的环境中运行，你应该知道你的全局（和静态）变量也可以被多个线程同时访问。使用``std::mutex``或``std::atomic``等技术来防止数据竞态条件和冲突是你自己的责任。

如果你的扩展是为多线程环境编译的，PHP-CPP头文件定义了宏``ZTS``。你可以使用这个宏来检查是否需要创建特殊的代码来处理线程。

```cpp
#include <phpcpp.h>

/**
 *  Global variable that store the number of times
 *  the function updateCounters() has been called in total
 *  @var    int
 */
int invokeTotalCount = 0;

#ifdef ZTS

/**
 *  Mutex so that the 'invokeTotalCount' variable is only accessed
 *  by one process at a time
 *  @var    std::mutex
 */
std::mutex invokeTotalMutex;

#endif

/**
 *  Native function that is callable from PHP
 *
 *  This function updates a number of global variables that count
 *  the number of times a function was called
 */
void updateCounters()
{
#ifdef ZTS

    // lock the mutex
    std::unique_lock<std::mutex> lock(invokeTotalMutex);

#endif

    // increment counters
    invokeTotalCount++;
}
```

另一个重要的事情是，PHP内部也做了这种锁定。如果你从你的C++代码中调用一个PHP函数（比如``Php::Value("myFunction")()``），或者当你访问``Php::GLOBALS``数组中的一个PHP变量（或者其他超级全局变量之一）时，PHP必须锁定一些东西以确保没有其他线程同时访问相同的信息。这些操作可能很昂贵。

因此，用 PHP-CPP 编写本地扩展的良好经验：

- 不要使用全局变量
- 只调用其他本地函数，不要回调到PHP中。

这些规则并不像它们看起来那样具有局限性。全局变量的使用并不被认为是优秀的软件设计，所以你可能根本就没有使用它们，你之所以要写一个本地扩展，首先是因为你想摆脱PHP。从你的扩展代码中调用（慢的）PHP函数，无论如何都应该被阻止。

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

## 动态加载扩展

从PHP转到C++的用户经常会问，如果用C++代码代替PHP代码，是否会增加系统管理的难度。我们必须在这里说实话：使用PHP比使用C++更容易。例如，要激活一个PHP脚本，你不需要root权限，你可以简单地复制脚本到Web服务器。部署原生C++扩展需要更多的工作：你需要先停止Web服务器，编译扩展，安装，然后重新启动Web服务器。

除此之外，当一个扩展被部署后，它将立即对所有托管在Web服务器上的网站进行激活。一个已部署的PHP脚本只改变了单个网站的行为，但一个已部署的C++扩展会影响所有网站。其实不可能只为特定的网站激活一个扩展，也不可能只为一个网站测试一个新版本的扩展，因为扩展是由所有PHP进程共享的。如果你真的想在不同的网站上使用不同的扩展，你需要多个服务器，都有自己的配置。

或者你可以使用动态加载。

PHP有一个内置的``dl()``函数，你可以用它来加载扩展。这允许你从PHP脚本中调用``dl("myextension.so")``函数来加载一个扩展，这样一个扩展只适用于一个特定的站点。出于安全考虑，这个内置的``dl()``函数有一些限制(否则会允许用户运行任意的本地代码)，但如果只有你一个人负责一个系统，或者当一个服务器不是由多个组织共享时，你可以使用PHP-CPP创建一个类似于``dl()``的函数，但没有这个限制。

### 为什么dl()会受到限制？

由于安全问题，``dl()``函数受到限制。当你使用``dl()``时，只能加载存储在系统范围扩展目录中的扩展，而不能用于加载用户放在其他位置的扩展。因此，调用 dl("/home/user/myextension.so")会失败，因为"/home/user "不是官方的扩展目录。为什么会有这种限制？

要理解这一点，首先必须认识到，在正常的PHP安装中，PHP脚本是由没有root权限的用户编辑的。在共享主机环境下，不同的用户都在同一个系统上运行自己的网站。在这样的设置中，如果一个用户可以编写一个可以访问他人数据的脚本或程序，那是绝对不行的。然而，如果使用一个不受限制的``dl()``函数，恰恰可以做到这一点。一个不受限制的 ``dl()`` 调用将允许 PHP 程序员编写一个本地扩展，将其存储在他们的主目录或 /tmp 目录中，并由 webserver 进程加载。然后，他们可以执行任意代码，并可能在其他人的网站内安装记录器或其他恶意代码。**通过只允许从系统范围内的扩展目录中加载扩展，PHP 保证了每一个动态加载的扩展必须至少由系统管理员安装**。

然而，当你写你自己的扩展时（无论是直接在Zend API之上，还是通过使用PHP-CPP库）你已经可以写和执行任意代码了。这里不需要安检。从你的C/C++代码中，你可以做任何你想做的事情。如果你能根据网站的要求动态加载一个扩展，那不是很酷吗？一个网站需要稳定，并加载你的扩展测试良好的1.0版本，而第二个网站更多的是实验性的，并加载2.0版本。你可能在同一台机器上运行两个版本的扩展。

### Thin loader extension

想象一下，你正在编写你自己的扩展 "MyExtension"，它有许多不同的类和功能，而且你计划一直为它推出新的版本。你不希望以 "大爆炸 "的方式部署新版本，而是希望慢慢地推出新版本，一次一个客户或一个网站。你会怎么做？

你首先要开发一个瘦加载器扩展：ExtensionLoader扩展。这个扩展只有一个函数：``enable_my_extension()``，它接收你的实际扩展的版本号，并动态地加载你的扩展的那个版本。这是一个简单的扩展（它只有一个功能），你可以在全球范围内安装，而且你可能永远不会有更新。

```cpp
/**
    *  Function to load an extension by its version number
    *
    *  It takes one argument: the version number of your extension,
    *  and returns a boolean to indicate whether the extension was 
    *  correctly loaded.
    *
    *  @param  params      Vector of parameters
    *  @return boolean
    */
Php::Value enable_my_extension(Php::Parameters &params)
{
    // get version number
    int version = params[0];

    // construct pathname to your extension (this is for example
    // /path/to/MyExtension.so.1 or /path/to/MyExtension.so.2)
    std::string path = "/path/to/MyExtension.so." + std::to_string(version);

    // load the extension
    return Php::dl(path);
}

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
        static Php::Extension myExtension("ExtensionLoader", "1.0");

        // the extension has one method
        myExtension.add("enable_my_extension", enable_my_extension, {
            Php::ByVal("version", Php::Type::Numeric)
        });

        // return the extension
        return myExtension;
    }
}
```

上面的代码保存了ExtensionLoader扩展的全部源代码。你可以在你的系统上安装这个扩展，把它复制到全局php扩展目录下并更新``php.ini``文件。

当你安装了这个thin loader扩展之后，你就可以用类和函数写满你的实际大扩展，并将这个扩展编译成*.so文件：第一个版本你编译成MyExtension.so.1，以后的版本编译成MyExtension.so.2、MyExtension.so.3等。对于每一个新的版本，你都会引入一个新的版本号，并将这些共享对象复制到/path/to目录下（与上面显示的"ExtensionLoader"扩展中的路径相同）。虽然这不是官方的PHP扩展目录，但是这些扩展可以通过enable_my_extension()函数加载。

你不需要将扩展程序复制到PHP扩展目录中，也不需要更新php.ini配置。要激活一个扩展，你只需要调用引入的enable_my_extension()函数。

```php
 // enable version 2 of the extension (this will load MyExtension.so.2)
if (!enable_my_extension(2)) die("Version 2 of extension is missing");

// from now on we can use classes and functions from version 2 of the extension
$object = new ClassFromMyExtension();
$object->methodFromMyExtension();

// you get the idea...
```

上面我们展示的ExtensionLoader还是很安全的。不能运行任意代码，也不能打开任意*.so文件。最糟糕的事情是有人用错误的版本号打开了一个扩展。

但如果你真的信任你系统上的用户，你可以很容易地调整thin loader扩展来允许其他类型的参数。在最开放的情况下，你甚至可以写一个函数，让用户从字面上打开所有可能的共享对象文件。

```cpp
/**
    *  Function to load every possible extension by pathname
    *
    *  It takes one argument: the filename of the PHP extension, and returns a 
    *  boolean to indicate whether the extension was correctly loaded.
    *
    *  This function goes further than the original PHP dl() fuction, because
    *  it does not check whether the passed in extension object is stored in the
    *  right directory. Literally every possible extension, also local ones 
    *  created by end users, can be loaded.
    *
    *  @param  params      Vector of parameters
    *  @return boolean
    */
Php::Value dl_unrestricted(Php::Parameters &params)
{
    // get extension name
    std::string pathname = params[0];

    // load the extension
    return Php::dl(pathname);
}

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
        static Php::Extension myExtension("load_extension", "1.0");

        // the extension has one method
        myExtension.add("dl_unrestricted", dl_unrestricted, {
            Php::ByVal("pathname", Php::Type::String)
        });

        // return the extension
        return myExtension;
    }
}
```

上面的代码将允许PHP脚本动态加载PHP扩展，无论它们存储在系统的哪个位置。

```php
// load the C++ extension stored in the same directory as this file
if (!dl_unrestricted(__DIR__.'/MyExtension.so')) die("Extension could not be loaded");

// from now on we can use classes and functions from the extension
$object = new ClassFromMyExtension();
$object->methodFromMyExtension();
```

``dl_unrestricted()``函数是一个很厉害的函数，但这里要注意：如果你是共享主机平台的管理员，你绝对不要安装它!

### Persistent extensions

动态加载的扩展会在请求结束时自动卸载。如果后续的请求也动态加载相同的扩展，那么它将以一个全新的环境开始。如果你想写一个使用静态数据或静态资源的扩展（比如一个持久的数据库连接，或者一个处理任务的工作线程），这不一定是你想要的行为。你要保持数据库连接的活跃性，或者线程的运行，也是在扩展被卸载之后。

为了克服这个问题，``Php::dl()``函数附带了第二个布尔参数，你可以用它来指定你是想持久地加载扩展，还是只为那个特定的请求加载。

请注意，如果你把这个参数设置为true，唯一持久化的就是扩展中的数据。在后续的请求中，你仍然需要加载扩展来激活其中的函数和类，即使你已经在之前的请求中持续加载了扩展。但由于之前已经加载了扩展，所以其中的静态数据（如数据库连接或线程）会被保存下来。

我们上面演示的``dl_unrestricted()``函数可以被修改为包含这个持久参数。

```cpp
/**
    *  Function to load every possible extension by pathname
    *
    *  It takes two arguments: the filename of the PHP extension, and a boolean to
    *  specify whether the extension data should be kept in memory. It returns a 
    *  boolean to indicate whether the extension was correctly loaded.
    *
    *  This function goes further than the original PHP dl() fuction, because
    *  it does not check whether the passed in extension object is stored in the
    *  right directory, and because it allows persistent loading of extensions. 
    *  Literally every possible extension, also local ones created by end users, 
    *  can be loaded.
    *
    *  @param  params      Vector of parameters
    *  @return boolean
    */
Php::Value dl_unrestricted(Php::Parameters &params)
{
    // get extension name
    std::string pathname = params[0];

    // persistent setting
    bool persistent = params.size() > 1 ? params[1].boolValue() : false;

    // load the extension
    return Php::dl(pathname, persistent);
}

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
        static Php::Extension myExtension("load_extension", "1.0");

        // the extension has one method
        myExtension.add("dl_unrestricted", dl_unrestricted, {
            Php::ByVal("pathname", Php::Type::String),
            Php::ByVal("persistent", Php::Type::Bool, false)
        });

        // return the extension
        return myExtension;
    }
}
```

而这个扩展允许我们动态加载扩展，同时保留扩展内部的持久化数据。

```php
// load the C++ extension stored in the same directory as this file, the
// extension is persistently loaded, so it may use persistent data like
// database connections and so on.
if (!dl_unrestricted(__DIR__.'/MyExtension.so', true)) die("Extension could not be loaded");

// from now on we can use classes and functions from the extension
$object = new ClassFromMyExtension();
$object->methodFromMyExtension();
```

## 参考

1. [php-cpp](php-cpp.com)
