---
title: PHP写快排
date: 2017-09-12
tags: 
  - PHP 
  - 数据结构与算法
---
```php
<?php

$arr = [6,3,8,7,4,2,9,5,1];

function quick_sort($arr)
{
    if(!is_array($arr)) return false;
    $length = count($arr);
    if($length <= 1) return $arr;
    $left = $right = [];
    for($i = 1;$i < $length;$i++)
    {
        if($arr[$i] < $arr[0]){
            $left[] = $arr[$i];
        }else{
            $right[] = $arr[$i];
        }
    }
    $left = quick_sort($left);
    $right = quick_sort($right);
    return array_merge($left,array($arr[0]),$right);
}

var_export(quick_sort($arr));
```