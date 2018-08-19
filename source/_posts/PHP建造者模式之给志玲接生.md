---
title: PHP建造者模式之给志玲接生
date: 2018-04-09
categories:
  - 技术
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

## 女主角范志玲

志玲是一只银渐层，她那几天待产（问题一：猫的平均生产时间是多少天呢？）
![](/images/zhiling2-283x300.jpg)

## 故事

那天说来话长...

### 11:30

我刚想出门去买菜，志玲便横在门口，仰着大肚皮看着我（问题二：生产期如何安抚母猫？），好像在呢喃着什么~看着她老是仰着个大肚皮睡觉，也不知道肚子里到底有几只小猫，想必志玲也是相当难受吧...我也没注意，以为她只是肚子不太舒服...

### 20:00

志玲冲着我们大叫几声，声音中如泣如诉，似乎想告诉我们什么，作为21世纪的上进小青年，吃完饭当然要义不容辞地抢着洗碗啊，所以也没怎么在意...


### 21:30

Qi不太放心过去看了看纸皮箱（问题三：母猫生产环境如何准备？），起初没觉得什么，志玲很安静的躺在纸皮箱里，忽然发现，志玲躺着的毛巾上有一滩淡红色的“污渍”，我过去看了看，这滩淡红色“污渍”正在志玲的肚皮下方、纸皮箱的外出口前。由于纸皮箱放在阳台门靠里边的角落，通风凉爽而且那天也比较干燥，给志玲喝水的碗和纸皮箱有一定的距离，不可能产生这么大面积的“污渍”。Qi说，这张毛巾之前是干净的（没有染色），那么说明，这滩“污渍”是刚刚半小时到1小时左右志玲自己产生的！

### 22:00

这滩“污渍”就是羊水，羊水破了！我们翻开志玲，看到志玲胯下有个羊水球露出来一点点，周边的毛都是淡红色的，这里有个疑问，为什么第一个羊水破了，却不见小猫出来，难道被志玲压着？后来我们发现小猫在纸皮箱里面，我穿上手套（一定要用手套），把小猫弄出来，像只“小老鼠”似的，闭着眼睛。我们怕它死了（问题四：如何处理刚出生的小猫？），弄来一盆温水（40℃左右），把小猫放进去，轻轻搓着它的身体。这时候，志玲也过来了，它似乎看到了我们的努力，也在尽力生第二个小孩。可是，我们觉得这样不是办法，毕竟我们也是外行人，从来没有碰过这种事。后来，我们把它和小猫送到了宠物医院...

### 喜讯

第二天，我们得知，志玲一共生了三只小猫，母子平安，后来分别被取名叫“佩奇”、“佐治”和“八千”

![](/images/zhiling1-225x300.jpg)

### 问答环节

#### 问题一：猫的平均生产时间是多少天呢？

63天（60~68天）

#### 问题二：生产期如何安抚母猫？

尽量不要着急，大喊大叫，跑来跑去，轻轻抚摸母猫，揉肚子，陪伴左右，给予信心和安全感。

#### 问题三：母猫生产环境如何准备？

- 温暖舒适的产箱（家里可以用纸皮箱加毛巾，纸箱要注意通风）
- 剪刀（猫生产的时候会自己咬断脐带并吃掉胎盘，有备无患）
- 生理盐水（给母猫擦身）
- 一次性手套（全程带着）
- 加热垫，有条件最好有，那天去到医院，医生第一时间就给志玲垫了
- 热水，毛巾肯定要有

#### 问题四：如何处理刚出生的小猫？

温水，把小猫放进水里按摩心脏，搓身子

### 设计模式

如果把医生接生的过程抽象出来看，那就是“建造者模式”。

- 医生：Director
- 志玲：具体建造者
- 小猫：产品

```php
<?php

/**
 * 产品类（小猫）
 */
class Kitten {
    /**
     * 所有小猫
     */
    private $_babies;

    public function __construct() {
        $this->_babies = array();
    }

    public function add($part) {
        return array_push($this->_babies, $part);
    }

    public function show() {
        echo "有这些小猫:";
        array_map('printf', $this->_babies);
    }
}

/**
 * 抽象建造者
 */
abstract class Builder {

    /**
     * 产品零件构造方法1
     */
    public abstract function buildPart1();

    /**
     * 产品零件构造方法2
     */
    public abstract function buildPart2();

    /**
     * 产品零件构造方法3
     */
    public abstract function buildPart3();

    /**
     * 产品返还方法
     */
    public abstract function getResult();
}

/**
 * 具体建造者（母猫：志玲）
 */
class MotherCat extends Builder {

    private $_babies;

    public function __construct() {
        $this->_babies = new Kitten();
    }

    public function buildPart1() {
        $this->_babies->add("佩奇");
    }

    public function buildPart2() {
        $this->_babies->add("佐治");
    }

    public function buildPart3() {
        $this->_babies->add("八千");
    }

    public function getResult() {
        return $this->_babies;
    }
}

/**
 * 接生员
 */
class Director {

    public function __construct(Builder $builder) {
        $builder->buildPart1();
        $builder->buildPart2();
        $builder->buildPart3();
    }
}



class Client {

    /**
     * Main program.
     */
    public static function main() {
        $zhiling = new MotherCat();
        $director = new Director($zhiling);
        $kitten = $zhiling->getResult();
        $kitten->show();
    }

}

Client::main();
```