---
title: PHP返回对象的公有属性和值
date: 2018-05-03
tags: 
  - PHP 
---


获取对象的公有属性和值，很自然的会想到 **get_object_vars** 这个函数，可以这样写，

```php
class User {
    private $age = 18;
    public $name = 'xxx';
}

$user = new User();
$item = get_object_vars($user);
print_r($item);

// Array
// (
//     [name] => xxx
// )
```

上面的代码是写在对象的外部的，可不可以提供一个 **trait** 使得每个对象都复用一个接口，供外部使用呢？这样一来，看上去更像是对象自己的方法，我们可以通过这个方法来取得对象的公有属性与其值。

上面我们用 **get_object_vars** 函数在外部实现了这个想法，放到内部试试看，当然我们要结合**trait**来实现：

```php
trait PrintPublic {
    public function publics()
    {
        return get_object_vars($this);
    }
}

class User {
    private $age = 18;
    public $name = 'xxx';

    use PrintPublic;
}

$user = new User();
$item = $user->publics();
print_r($item);

// Array
// (
//     [age] => 18
//     [name] => xxx
// )
```

我们发现，把所有的属性都取出来了，无论是公有的还是私有的，这不对。为什么会这样呢？不难想象，因为这个方法是放在类内部的，作用域在内部。而之前这个方法是放在类的外部，作用域是全局的。思路有了，我们只需要想办法让作用域在类的外部就可以了，我们想到了 **call_user_func** 这个方法。

```php
trait PrintPublic {
    public function publics()
    {
        return call_user_func('get_object_vars', $this);
    }
}

class User {
    private $age = 18;
    public $name = 'xxx';

    use PrintPublic;
}

$user = new User();
$item = $user->publics();
print_r($item);

// version < php 7
// Array
// (
//     [name] => xxx
// )

// version > php 7
// Array
// (
//     [name] => xxx
//     [age] => 18
// )
```

我们又发现了一个问题，这个方法在php7不能奏效，具体原因未知，有空探讨一下，既然这样也不是最好的办法，那就再换一个函数 **create_function**，试试看

```php
trait PrintPublic {
    public function publics() {
        $publicVars = create_function('$obj', 'return get_object_vars($obj);');
        return $publicVars($this);
    }
}

class User
{
    public $name = "xxx";
    private $_age = 30;

    use PrintPublic;
}

$User = new User();
$data = $User->publics();
print_r($data);

// Array
// (
//     [name] => xxx
// )
```

没问题了，问题解决 :)