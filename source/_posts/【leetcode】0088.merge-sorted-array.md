---
title: 【leetcode】0088.merge-sorted-array
date: 2019-07-03
categories:
  - 技术
tags: 
  - PHP 
  - leetcode
---

## 题目描述

给定两个**有序**整数数组 nums1 和 nums2，将 nums2 合并到 nums1 中，使得 num1 成为一个有序数组。

### 说明

初始化 nums1 和 nums2 的元素数量分别为 m 和 n。
你可以假设 nums1 有足够的空间（空间大小大于或等于 m + n）来保存 nums2 中的元素。
示例:

### 输入

nums1 = [1,2,3,0,0,0], m = 3
nums2 = [2,5,6],       n = 3

### 输出

[1,2,2,3,5,6]

## 思路

1. 将``nums2``合并到``nums1``后面，使用``插入排序``（从**第一位**开始）
2. 第一种思路的``升级版``，由于一开始nums1是**已排序**的，所以不需要从``第一位``开始，而是从``第m位``开始。

## 代码

### 插入排序

```php
class Solution {

    /**
     * @param Integer[] $nums1
     * @param Integer $m
     * @param Integer[] $nums2
     * @param Integer $n
     * @return NULL
     */
    function merge(&$nums1, $m, $nums2, $n) {
        $total = count($nums1);

        for ($i = $m, $j = 0; $i < $total && $j < $n; $i++, $j++) {
            $nums1[$i] = $nums2[$j];
        }

        for ($i = 1; $i < $total; $i++) {
            $key = $nums1[$i];
            $j = $i - 1;
            while ($j >= 0 && $nums1[$j] > $key) {
                $nums1[$j+1] = $nums1[$j];
                $j--;
            }
            $nums1[$j+1] = $key;
        }
    }
}
```

### 优化版插入排序

```php
class Solution {

    /**
     * @param Integer[] $nums1
     * @param Integer $m
     * @param Integer[] $nums2
     * @param Integer $n
     * @return NULL
     */
    function merge(&$nums1, $m, $nums2, $n) {
        $total = count($nums1);

        for ($i = $m, $j = 0; $i < $total && $j < $n; $i++, $j++) {
            $nums1[$i] = $nums2[$j];
        }

        for ($i = $m; $i < $total; $i++) { // $i = $m，从第m位开始
            $key = $nums1[$i];
            $j = $i - 1;
            while ($j >= 0 && $nums1[$j] > $key) {
                $nums1[$j+1] = $nums1[$j];
                $j--;
            }
            $nums1[$j+1] = $key;
        }
    }
}
```