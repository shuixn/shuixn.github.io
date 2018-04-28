---
title: PHP门面模式之网红进阶之路
date: 2018-03-29
tags: 
  - PHP 
  - 设计模式
---

## 写在前

门面模式又称外观模式，既然如此，最容易联想到的就是化妆了。不要问我为什么知道怎么化妆 = =#

有人说，关注我的文字是为了学习技术，我觉得这是对我赤果果的侮辱！今天我就教你如何化妆！

## 化妆

我简单总结了一下，化妆大致分为五大模块

- 基础护肤
- 隔离保护
- 脸部美容
- 眼部美容
- 唇部美容

**题外话（划重点）**：这里的化妆步骤看起来很像模版方法模式，也是给定一个算法架构，然后让客户端去实现细节。其实，化妆除了前面两步（基础护肤和隔离保护）需要严格按顺序执行，其他部分并不需要强制按顺序，如果你有多个化妆师，也可以一遍画眉一遍抹口红，或者你觉得你不需要抹口红，纯天然水嫩！那少了唇部美容这一部分也无碍。所以，外观模式是最适合的，对于整个化妆而言，每个模块都是相对独立的。

#### 基础护肤

众说周知（真的假的），化妆前不护肤？这脸是要毁掉了呀！

>化妆品主要成分是溶剂，其次是表面活性剂，另外就是林林总总的防腐剂、色素、香精、增色剂、稠化剂、络合剂等添加剂。
>1. 溶剂、表面活性剂会软化表皮细胞，褪去角质层，一次当然不行，天天累积的效果，一定程度上弱化皮肤的防护作用（主要是表面活性剂的效果）;
>2. 被弱化的表皮细胞和毛孔部位会渗透进各种添加剂，有些存在特殊结构的物质有可能会被细胞识别并拖进去，完成吸收——主要是小分子有机物，对于蛋白类，呵呵，你又不是欧米巴变形虫~;
>3. 从来没有报道或研究指出此种途径对人体的影响，但是整体影响的相关报道时有见报，一般是医学杂志——其实是体系太复杂，不好研究，又没人会支持这项研究;
>4. 过多的添加剂，在前两条途径存在的情况下，会整体上增大过敏的可能性，尤其是使防护能力减弱的皮肤暴露在空气中，会使皮肤变得敏感;
>5. 重金属化合物往往是化妆品增色剂的主要成分，依靠其物理色泽产生美白等效果，在前两点的存在下，重金属积累的效果很有可能会产生，而且由于生物体缺乏对这种物质的代谢能力，其影响是深远的。另外，由于重金属运输效率其实不高，在皮肤积累会使得皮肤变得越来越差！

有人又说了，生活中到处都是化学品，防不胜防啊，是啊，现在教你怎么防~

1. 洗脸
2. 爽肤
3. 精华 （促进吸收、美白、保湿）
4. 乳液

```php
/**
 * 基础护肤
 * Class BasicSkinCare
 */
class BasicSkinCare
{
    public function washFace(){ echo '洗脸'; }
    public function toning(){ echo '爽肤'; }
    public function eliteFluid(){ echo '精华'; }
    public function emulsion(){ echo '乳液'; }
}
```

#### 隔离保护

做完基础护肤，还不能直接往脸上抹化妆品，需要做隔离

1. 隔离霜
2. 粉底液

```php
/**
 * 隔离保护
 * Class IsolationProtection
 */
class IsolationProtection
{
    public function segregationFrost(){ echo '隔离霜'; }
    public function liquidFoundation(){ echo '粉底液'; }
}
```

#### 脸部美容

昨晚隔离后，可以开始化妆了，从脸部开始，

1. 定妆粉饼
2. 腮红

```php
/**
 * 脸部美容
 * Class FacialBeauty
 */
class FacialBeauty
{
    public function calmMakeup(){ echo '定妆粉饼'; }
    public function cheekRed(){ echo '腮红'; }
}
```

#### 眼部美容

眼睛是心灵的窗户，拥有一双明眸会让你增色不少

1. 画眉
2. 眼影
3. 眼线
4. 睫毛膏

这里还可以加美瞳，因人而异啦吼吼~

```php
/**
 * 眼部美容
 * Class EyeBeauty
 */
class EyeBeauty
{
    public function thrush(){ echo '画眉'; }
    public function eyeShadow(){ echo '眼影'; }
    public function eyeliner(){ echo '眼线'; }
    public function eyelashCream(){ echo '睫毛膏'; }
}
```

#### 唇部美容

在某些男生眼中，判断一个女生是否化妆的标准是 —— 有没有口红....我的兄dei...

```php
/**
 * 唇部美容
 * Class LipsBeauty
 */
class LipsBeauty
{
    public function lipstick(){ echo '唇彩或唇膏'; }
}
```

## 化妆门面类

刚刚介绍了化妆的五大模块，把这个五个模块组装在一起，就形成了化妆门面类。当然，你也可以少一些模块，并不强制要求，也不需要严格按照步骤执行。

```php
/**
 * 化妆门面类
 * Class SecurityFacade
 */
class MakeupFacade
{
    private $_basicSkinCare;
    private $_isolationProtection;
    private $_facialBeauty;
    private $_eyeBeauty;
    private $_lipsBeauty;

    public function __construct()
    {
        $this->_basicSkinCare = new BasicSkinCare();
        $this->_isolationProtection = new IsolationProtection();
        $this->_facialBeauty = new FacialBeauty();
        $this->_eyeBeauty = new EyeBeauty();
        $this->_lipsBeauty = new LipsBeauty();
    }

    public function start()
    {
        $this->_basicSkinCare->washFace();
        $this->_basicSkinCare->toning();
        $this->_basicSkinCare->eliteFluid();
        $this->_basicSkinCare->emulsion();

        $this->_isolationProtection->segregationFrost();
        $this->_isolationProtection->liquidFoundation();

        $this->_facialBeauty->calmMakeup();
        $this->_facialBeauty->cheekRed();

        $this->_eyeBeauty->thrush();
        $this->_eyeBeauty->eyeShadow();
        $this->_eyeBeauty->eyeliner();
        $this->_eyeBeauty->eyelashCream();

        $this->_lipsBeauty->lipstick();
    }
}
```

## 开始化妆

一般而言，门面只需要一个对象，所以这里声明为静态变量，下次使用不需要再重新申请。

```php
/**
 * 客户端
 */
class Client {

    private static $_makeupFacade;
    /**
     * Main program.
     */
    public static function main() {
        self::$_makeupFacade = new MakeupFacade();
        self::$_makeupFacade->start();
    }
}

Client::main();
```