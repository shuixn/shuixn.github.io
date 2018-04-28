---
title: PHP观察者模式之一二三木头人
date: 2018-03-23
tags: 
  - PHP 
  - 设计模式
---

## 一二三木头人

大多数小孩子都玩过这个游戏，游戏人数为N>=2就可以玩，规则很简单，考虑到小孩子的体积，粗略需要1平方米的占地面积，那么N个小孩子就是N平方米，场地出来了，在一个占地面积小于等于N平方米（为什么是小于等于？你懂的）的空地上，一个孩子王站前面背对着大家，游戏开始后，其他小孩子以风雷之势起跑试图超越孩子王所站的位置，最后拍到孩子王的肩膀，即为胜出。但是，在不定时间间隔内，孩子王有意或无意的掉头并喊出“一二三木头人”用以阻碍其他小孩子的来势汹汹，如果看到有人动了（站不稳、小动作），即可淘汰出局！

如这份草图：
![](/images/mutouren-300x131.png)

## 模式抽象

万万没想到，这个孩提时代的游戏竟体现了观察者模式，细思极恐：

- 孩子王，被观察者，负责维护一个已注册的观察者新增、删除、发通知
- 其他小孩子，观察者，负责应对被观察者所发出的事件变化而变化

### 被观察者

```php
/**
 * 抽象主题（被观察者）
 * Interface Subject
 */
interface ISubject
{
    /**
     * 新增观察者
     * @param IObserver $observer
     * @return mixed
     */
    public function attach(IObserver $observer);

    /**
     * 删除已订阅的观察者
     * @param IObserver $observer
     * @return mixed
     */
    public function detach(IObserver $observer);

    /**
     * 通知所有已订阅的观察者对象
     */
    public function notifyObservers();
}
```

### 观察者

```php
/**
 * 抽象观察者
 */
interface IObserver {

    /**
     * 更新方法
     */
    public function update();
}
```

### 被观察者实体

```php
/**
 * 孩子王（负责喊“一二三木头人”）
 * Class Boss
 */
class Boss implements ISubject
{
    private $_kids;
    public function __construct()
    {
        $this->_kids = array();
    }

    public function attach(IObserver $observer)
    {
        return array_push($this->_kids, $observer);
    }

    public function detach(IObserver $observer)
    {
        $index = array_search($observer, $this->_kids);
        if ($index === false || ! array_key_exists($index, $this->_kids)) {
            return false;
        }

        unset($this->_kids[$index]);
        return true;
    }

    /**
     * 通知所有注册过的观察者对象
     */
    public function notifyObservers() {
        if (!is_array($this->_kids)) {
            return false;
        }

        foreach ($this->_kids as $observer) {
            $observer->update();
        }

        return true;
    }
}
```

### 观察者实体

```php
/**
 * 观察者
 * Class Kid
 */
class Kid implements IObserver
{
    /**
     * 观察者的名称
     * @var <type>
     */
    private $_name;

    public function __construct($name) {
        $this->_name = $name;
    }

    /**
     * 更新方法
     */
    public function update() {
        echo $this->_name, ' 向前一步';
    }
}
```

## 游戏开始

```php
/**
 * 客户端
 */
class Client {

    /**
     * Main program.
     */
    public static function main() {

        // 游戏开始

        // 从孩子们中选出一个Boss，负责喊“一二三木头人”
        $subject = new Boss();

        // 小明参加游戏
        $observer1 = new Kid('小明');
        $subject->attach($observer1);

        // 小张参加游戏
        $observer2 = new Kid('小张');
        $subject->attach($observer2);

        // 小张他妈喊他回家吃饭，不玩了
        $subject->detach($observer2);

        $subject->notifyObservers();
    }

}

Client::main();
```