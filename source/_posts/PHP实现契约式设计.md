---
title: PHP实现契约式设计
date: 2020-05-22
categories:
  - 技术
tags: 
  - PHP 
  - 契约式设计
---

## 什么是契约式设计

Eiffel语言的契约式设计介绍：[Building bug-free O-O software: An Introduction to Design by Contract™](https://www.eiffel.com/values/design-by-contract/introduction/)

不想看英文版，可以看看我翻译的中文版（如有错误，请不吝赐教）：[构建无bug面向对象软件：契约式设计简介](https://funsoul.gitbook.io/notebook/she-ji-mo-shi/yi-gou-jian-wu-bug-mian-xiang-dui-xiang-ruan-jian-qi-yue-shi-she-ji-jian-jie)

## 为什么要使用契约式设计

理解契约式设计前，有一个与之相对的概念，防御性编程。防御性编程不信任任何代码（即使是自己的代码），试图捕获一切可知的异常。

契约式设计强调“契约”，在函数中，调用方和被调用方需要履行义务和权利，如果一方不满足契约，则主动抛出异常。

契约式设计使开发人员对代码可靠性的态度更强烈

## PHP如何实现契约式设计

按照契约式设计思想，可以实现一个简易版的PHP库：[contract-php](https://github.com/funsoul/contract-php)，这个库可以很方便的通过``注解``的方式定义函数的``前置条件``和``后置条件``。