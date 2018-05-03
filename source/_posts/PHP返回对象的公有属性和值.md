---
title: PHP返回对象的公有属性和值
date: 2018-05-03
tags: 
  - PHP 
---

```php
trait PrintPublic {
    public function publics()
    {
    	// 在类内部，单纯使用 get_object_vars 会拿到所有属性
        $varArray = call_user_func('get_object_vars', $this);
        return $varArray;
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
//     [name] => xxx
// )
```