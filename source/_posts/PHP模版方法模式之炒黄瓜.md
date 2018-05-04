---
title: PHP模版方法模式之炒黄瓜
date: 2018-03-26
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

## 生命不息，学习不止！

又到了让人充满感激的周末，和往常一样，我一如既往地被“请”去学做菜，作为21世纪的五好青年（吃喝拉撒睡？），我表示毫无压力可言，区区一道炒黄瓜，还不是“洒洒水”。

## 炒菜的步骤

在拿起铲子那一刻，脑子里回顾了一遍老祖宗传下来的“陈家菜”菜谱 —— 要煮熟，少放盐。心里已经大定，其实炒菜和写程序一样：

- 输入指令：放食材放进去锅内
- 处理指令：煎、炒、烹、炸 ......
- 输出结果：熟了、出锅

于是，我们建立一套炒菜的模版

```php
/**
* 抽象模板角色
* 定义抽象方法作为顶层逻辑的组成部分，由子类实现
* 定义模板方法作为顶层逻辑的架子，调用基本方法组装顶层逻辑
*/
abstract class AbstractCooking {

  /**
  * 模板方法 调用基本方法组装顶层逻辑
  */
  public function cook() {
	  echo 'begin';
	  $this->PutinMaterials();
	  $this->ProcessingMaterial();
	  $this->TakeoutProduct();
	  echo 'end';
  }

  /**
  * 投放材料
  */
  abstract protected function PutinMaterials();

  /**
  * 处理材料
  */
  abstract protected function ProcessingMaterial();

  /**
  * 取出产品
  */
  abstract protected function TakeoutProduct();

}
```

## 炒黄瓜

有了这套模版，可以说，已经掌握了做菜的最高心法（呸！），无论是煎、炒、烹、炸，还是做饭、煲汤，都遵循这套模版。

今天拿炒黄瓜来举个例子：

```php
/**
* 具体模板角色
* 实现父类的抽象方法
*/
class FriedCucumber extends AbstractCooking{
  protected function PutinMaterials(){
	  echo '开火';
	  echo '滤干锅里的水';
	  echo '倒油';
	  echo '投入蒜蓉并爆香';
	  echo '投入准备好的黄瓜切片';
  }

  protected function ProcessingMaterial(){
	  for ($i = 0;$i < 10;$i++){
	  	echo '翻炒';
	  }
	  echo '放调味料'；
  }

  protected function TakeoutProduct(){
  	echo '装盘、上菜';
  }
}
```

## 开始
```php
/**
* 客户端
*/
class Client {

  /**
  * Main program.
  */
  public static function main() {
	  $class = new FriedCucumber();
	  $class->cook();
  }
}

Client::main();
```

## 过程
```
begin
开火
滤干锅里的水
倒油
投入蒜蓉并爆香
投入准备好的黄瓜切片
翻炒
翻炒
翻炒
...
...
放调味料
装盘、上菜
end
```
