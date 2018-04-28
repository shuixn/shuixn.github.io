---
title: PHP中两个数组怎样才算相等？
date: 2017-11-15
tags: 
  - PHP 
---
今天遇到一个有趣的问题，两个数组，判断他们是否是相等的（元素个数、元素均一样），至于元素顺序是否也要一样？下面的判断给出了答案。

```php
$a = [1,2,3,4];
$b = [2,1,3,4];
$c = [1,2,3,4];
var_dump($a==$c); // true
var_dump($a==$b); // false
var_dump(array_diff($a,$b)); // empty => true
```

可以发现，如果使用“==”做判断，如果两个数组的顺序不等，那么这两个数组竟然不是一样的了，这时可以使用array_diff来做判断。
array_diff是以a数组为基准，找出a数组中不存在于b数组的差值，既然两个数组元素是一样的，那当然没问题。
可以认为他们是除了顺序之外相等的两个数组。可是这么处理也是有问题的，接着看

```php
$a = [1,2,3];
$b = [2,1,3,4];
var_dump(array_diff($a,$b)); // empty => true
```

是的，这样也是emtpy。严谨起见，可以这样，

```php
$res = empty(array_diff($a,$b)) && empty(array_diff($b,$a));
```