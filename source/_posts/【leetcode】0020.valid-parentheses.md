---
title: 【leetcode】0020.Valid Parentheses
date: 2019-07-01
categories:
  - 技术
tags: 
  - PHP 
  - leetcode
---

## 题目描述

给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。

有效字符串需满足：

- 左括号必须用相同类型的右括号闭合。
- 左括号必须以正确的顺序闭合。
- 注意空字符串可被认为是有效字符串。

示例 1:

输入: "()"
输出: true
示例 2:

输入: "()[]{}"
输出: true
示例 3:

输入: "(]"
输出: false
示例 4:

输入: "([)]"
输出: false
示例 5:

输入: "{[]}"
输出: true

## 思路

- 哈希表存储括号
- 栈存储``左半边``括号集

遍历输入字符串，如果当前字符为左半边括号时，则将其压入栈中

如果遇到右半边括号时：

1. 若此时为空栈，则直接返回false
2. 若栈不为空且为对应的左半边括号，则取出栈顶元素，继续循环
3. 若不为对应的左半边括号，反之返回false
4. 遍历完后，若栈已清空，返回true，否则返回false

## 代码

```php
class Solution {

    /**
    * @param String $s
    * @return Boolean
    */
    function isValid($s) {
        if (empty($s)) {
            return true;
        }

        $map = [
            '(' => ')',
            '[' => ']',
            '{' => '}'
        ];

        $stack = [];
        $lefts = array_keys($map);

        $len = strlen($s);
        for ($i = 0; $i < $len; $i++) {
            if (in_array($s[$i], $lefts)) {
                $stack[] = $s[$i];
            } else {
                $left = array_pop($stack);
                if (isset($map[$left]) && $map[$left] == $s[$i]) {
                    continue;
                } else {
                    return false;
                }
            }
        }
        return empty($stack) ? true : false;
    }
}
```

## 复杂度分析

- 时间复杂度O(n)，主要时间花在遍历输入字符串的长度n。
- 空间复杂度O(n)，主要使用了一个一维数组结构来存储左半边括号。