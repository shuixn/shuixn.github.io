---
title: array_diff 遇到的问题
date: 2017-08-21
tags: 
  - PHP 
---
```php
array array_diff ( array $array1 , array $array2 [, array $... ] )
```

返回一个数组，该数组包括了所有在 array1 中但是不在任何其它参数数组中的值。注意键名保留不变。

这个函数很好用，帮助我解决了不少了业务问题，但是上周遇到一个业务，发现这个函数有缺陷，也找不到更合适的函数。先看看这个函数能做什么（例子来自手册）：

```php
$array1 = array("a" => "green", "red", "blue", "red");
$array2 = array("b" => "green", "yellow", "red");
$result = array_diff($array1, $array2);
```

output:

```php
Array
(
[1] => blue
)
```

显然，基于array1,通过和array2的比较，得出了“blue”不在array2中。
这个例子反过来，

```php
$result = array_diff($array2, $array1);
```

output:

```php
Array
(
[1] => yellow
)
```

因为，yellow不在array1中。

用这个函数可以处理什么类型的业务？答案就是数组差异集，如果我想知道一个数组经过一系列处理，删掉了哪些，又新增了哪些元素，可以这样：

```php
$deletes = array_diff($array1, $array2);
$inserts = array_diff($array2, $array1);
```

array1表示原数组，array2表示处理后的数组。这样一来，就可以很清楚的知道数据的前后差异。

### 遗漏的业务

```php
$array1 = array(1, 1);
$array2 = array(1);
$result = array_diff($array1, $array2);
```

如果是这种情况，得到的是“空”。我们的本意是取出差异集，按理说，这两个数组是有差异的，但是array_diff不负责取出长度不同的差异，只判断存在与否。

遇到这种业务，找不到任何可用的内置函数，只能扩展了。

```php
/**
 * 判断数组中是否含有重复的值
 * @param $arr
 * @return bool
 */
function is_array_repeat($arr)
{
    if(count($arr) != count(array_unique($arr))){
        return true;
    }else{
        return false;
    }
}

/**
 * 计算差异集（包含数组含有相同值的情况）
 * @param $arr1
 * @param $arr2
 * @return array
 */
function array_xdiff($arr1,$arr2)
{
    if(is_array_repeat($arr1) || is_array_repeat($arr2)) {
        foreach ($arr1 as $key1 => &$value1) {
            foreach ($arr2 as $key2 => &$value2) {
                if($value1 == $value2) {
                    unset($arr1[$key1]);
                    unset($arr2[$key2]);
                }
            }
        }
        sort($arr1);
        return $arr1;
    }else{
        return array_diff($arr1,$arr2);
    }
}
```

其实，后来我想到了更简单的办法规避了这个业务，毕竟代码是给人看的，这种情况不好理解，写出来别人看了也是一脸蒙蔽。

### 后记

后来发现array_diff_assoc函数

array array_diff_assoc ( array $array1 , array $array2 [, array $... ] )

返回一个数组，该数组包括了所有在 array1 中但是不在任何其它参数数组中的值。

这样一来，原来的写法可以简单很多

```php
/**
 * 计算差异集（包含数组含有相同值的情况）
 * @param $arr1
 * @param $arr2
 * @return array
 */
function array_xdiff($arr1,$arr2)
{
    if(is_array_repeat($arr1) || is_array_repeat($arr2)) {
        return array_diff_assoc($arr1,$arr2); 
    }else{ 
        return array_diff($arr1,$arr2); 
    } 
}
```