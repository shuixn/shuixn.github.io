---
title: PHP原型模式之性能探索
date: 2018-05-10
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

### 原型模式

有些时候，我们需要创建多个类似的大对象。如果直接通过new对象，开销很大，而且new完还得进行重复的初始化工作。可能把初始化工作封装起来的，但是对于系统来说，你封不封装，初始化工作还是要执行。

原型模式则不同，原型模式是先创建好一个原型对象，然后通过clone这个原型对象来创建新的对象，这样就免去了重复的初始化工作，系统仅需内存拷贝即可。

```php
interface IPrototype {
    public function copy();
}

class Student implements IPrototype {
    private $_name;
    private $_hobbies = [];

    public function __construct($name, $hobbies)
    {
        $this->_name = $name;
        $this->_hobbies = $hobbies;
    }

    public function setName($name)
    {
        $this->_name = $name;

    }

    public function setHobby($hobbies)
    {
        $this->_hobbies = $hobbies;
    }

    public function copy()
    {
        return clone $this;
    }
}

$Alan = new Student('Alan',['basketball','swimming']);
$Bob = $Alan->copy();
$Bob->setName('Bob');

$Alan->setHobby(['traveling','chess']);

var_dump($Alan);
var_dump($Bob);

//    object(Student)#1 (2) {
//    ["_name":"Student":private]=>
//      string(4) "Alan"
//    ["_hobbies":"Student":private]=>
//      array(2) {
//        [0]=>
//        string(9) "traveling"
//        [1]=>
//        string(5) "chess"
//      }
//    }
//    object(Student)#2 (2) {
//    ["_name":"Student":private]=>
//      string(3) "Bob"
//    ["_hobbies":"Student":private]=>
//      array(2) {
//        [0]=>
//        string(10) "basketball"
//        [1]=>
//        string(8) "swimming"
//      }
//    }
```

我对这里的性能差异有些兴趣，分别对new和clone做一些性能测试

### clone方式

```
define("NEWLINE",chr(10));

interface IPrototype {
    public function copy();
}

class Student implements IPrototype {
    private $_name;
    private $_hobbies = [];

    public function __construct($name, $hobbies)
    {
        $this->_name = $name;
        $this->_hobbies = $hobbies;
    }

    public function setName($name)
    {
        $this->_name = $name;

    }

    public function setHobby($hobbies)
    {
        $this->_hobbies = $hobbies;
    }

    public function copy()
    {
        return clone $this;
    }
}

$bigArr = range(1,100000);

$m1 = memory_get_usage();

$Alan = new Student('Alan', $bigArr);
$Bob = $Alan->copy();

$m2 = memory_get_usage();
$memory = $m2 - $m1;
echo '内存占用：' . $memory . NEWLINE;
```

#### clone方式性能测试

```
$ php index.php 
内存占用：664
$ php index.php 
内存占用：664
$ php index.php 
内存占用：664
$ php index.php 
内存占用：664
```

### new方式

```php
define("NEWLINE",chr(10));

interface IPrototype {
    public function copy();
}

class Student implements IPrototype {
    private $_name;
    private $_hobbies = [];

    public function __construct($name, $hobbies)
    {
        $this->_name = $name;
        $this->_hobbies = $hobbies;
    }

    public function setName($name)
    {
        $this->_name = $name;

    }

    public function setHobby($hobbies)
    {
        $this->_hobbies = $hobbies;
    }

    public function copy()
    {
        return clone $this;
    }
}

$bigArr = range(1,100000);

$m1 = memory_get_usage();

$Alan = new Student('Alan', $bigArr);
$Alan2 = new Student('Alan', $bigArr);

$m2 = memory_get_usage();
$memory = $m2 - $m1;
echo '内存占用：' . $memory . NEWLINE;
```


#### new方式性能测试

```
$ php index.php 
内存占用：712
$ php index.php 
内存占用：712
$ php index.php 
内存占用：712
$ php index.php 
内存占用：712
```

可以看出，clone确实比new消耗的内存更小，但是，回到上面，我们真正的业务是copy完这个对象后，要对这个对象做一些修改，比如修改这个clone完后的对象的name，看看会发生什么事情

### 业务场景下的clone

```php
define("NEWLINE",chr(10));

interface IPrototype {
    public function copy();
}

class Student implements IPrototype {
    private $_name;
    private $_hobbies = [];

    public function __construct($name, $hobbies)
    {
        $this->_name = $name;
        $this->_hobbies = $hobbies;
    }

    public function setName($name)
    {
        $this->_name = $name;

    }

    public function setHobby($hobbies)
    {
        $this->_hobbies = $hobbies;
    }

    public function copy()
    {
        return clone $this;
    }
}

$bigArr = range(1,100000);

$m1 = memory_get_usage();

$Alan = new Student('Alan', $bigArr);
$Bob = $Alan->copy();
$Bob->setName('Bob');

$m2 = memory_get_usage();
$memory = $m2 - $m1;
echo '内存占用：' . $memory . NEWLINE;
```

#### 业务场景下的clone性能测试

```
$ php index.php 
内存占用：744
$ php index.php 
内存占用：744
$ php index.php 
内存占用：744
$ php index.php 
内存占用：744
```

我们发现，对新对象做完名字重置的操作后，内存占用更大了，比new的操作更大！

### 结论

- clone一个对象带来的内存消耗比new一个对象更少
- 在某些业务下，比如对新对象做属性修改时，直接new一个对象带来的内存消耗更少