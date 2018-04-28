---
title: PHP适配器模式之找不到合适的接口
date: 2018-03-28
tags: 
  - PHP 
  - 设计模式
---

## 诡异的现象

不知道大家是否有过一个疑问，当你想把USB插进接口的时候，总是会这样：

1. 第一次插进去，不行
2. 反过来（旋转180度），再插，还是不行
3. 再反过来（回到最开始），进去了！

这个接口结合的过程，是建立在双方遵循一致协议的情况下才会发生的，今天讨论的情况是，

1. 第一次插进去，不行
2. 反过来（旋转180度），再插，还是不行
3. 再反过来（回到最开始），仍旧不行！
4. 我擦，是不是拿错接口了！

## 接口适配

- 既然接口不合适，那怎么办？买一个？可以~ 请关闭此文章~
- 利用现有接口进行适配

![](/images/adapter-300x159.jpg)


**可以看出，现有插座只有三口的，而我的插头是两口的，这表示，接口不兼容。这种情况下，可以通过第三方（三口中间件）来进行适配。这个适配器其实就是在原来的基础上增加了额外的功能，是对原有功能的一种扩展，在不改变原来接口的基础上，使得两个互不兼容的接口可以一起工作。**

适配器模式有两种实现方式，一种是**类适配器**，另一种是**对象适配器**。

## 类适配器

```php
/**
 * 目标角色
 */
interface Target {

    /**
     * 两口
     */
    public function echo2plug();

    /**
     * 一口
     */
    public function echo1plug();
}

/**
 * 两口插头
 */
class TwoPlug {

    /**
     * 两口
     */
    public function echo2plug() {
        echo '2 plug';
    }
}

/**
 * 三口适配器
 */
class ThreePlugAdapter extends TwoPlug implements Target {

    /**
     * 新增一口
     */
    public function echo1plug() {
        echo '1 plug';
    }

}

class Client {

    /**
     * Main program.
     */
    public static function main() {
        $adapter = new ThreePlugAdapter();
        $adapter->echo2plug();
        $adapter->echo1plug();
    }

}

Client::main();
```

如上代码所示，类适配器使用继承，三口插头继承于两口插头。

##  对象适配器

```php
/**
 * 目标角色
 */
interface Target {

    /**
     * 两口
     */
    public function echo2plug();

    /**
     * 一口
     */
    public function echo1plug();
}

/**
 * 两口插头
 */
class TwoPlug {

    /**
     * 两口
     */
    public function echo2plug() {
        echo '2 plug';
    }
}

/**
 * 三口适配器
 */
class ThreePlugAdapter implements Target {
    private $_twoPlug;

    public function __construct(TwoPlug $twoPlug) {
        $this->_twoPlug = $twoPlug;
    }

    /**
     * 两口
     */
    public function echo2plug() {
        $this->_twoPlug->echo2plug();
    }

    /**
     *
     */
    public function echo1plug() {
        echo '1 plug';
    }

}

class Client {

    /**
     * Main program.
     */
    public static function main() {
        $adapter = new ThreePlugAdapter(new TwoPlug);
        $adapter->echo2plug();
        $adapter->echo1plug();
    }

}

Client::main();

```

如上代码所示，对象适配器使用的是依赖注入，对于三口插头适配器而言，可一次性给多个两口插头添加一口（添加功能）。