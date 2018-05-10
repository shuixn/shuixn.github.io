---
title: PHP状态模式之天气热了把空调开开
date: 2018-05-10
tags: 
  - PHP 
  - 设计模式
  - 学习笔记
---

### 写在前

夏天不可辜负的东西有三样：西瓜、海滩和冰淇淋。然而，单靠这三样还不足以度过炎炎夏日啊~

### 空调开开

夏天到了，对于南方的同学来说，回到家第一件事就是找**空调遥控器**，这个遥控器是个好东西，按一下开关就可以让我们清凉一夏~

那么，我们会想，遥控器是怎么保存开关的状态的呢？为什么我们按一下按钮就开，按一下就关，同一个按钮，两种状态，有点神奇~

### 状态

无论是开，还是关。它们都是一种状态，所以，我们有一个状态接口，这个状态是遥控器控制的，所以它有一个handle方法。

```php
interface IState
{
    public function handle(RemoteController $controller);
}
```

### 开关状态

开、关状态都实现了状态接口，所以它们都要实现handle方法，这是具体的状态。

```php
/**
 * 开机状态
 * Class OnState
 */
class OnState implements IState
{
    public function handle(RemoteController $controller)
    {
        echo '开空调';
        $controller->setState(new self);
    }
}


/**
 * 关机状态
 * Class OffState
 */
class OffState implements IState
{
    public function handle(RemoteController $controller)
    {
        echo '关空调';
        $controller->setState(new self);
    }
}
```

### 遥控器

我们还需要一个遥控器来保存这些状态，当然，这个遥控器有一个按钮给我们点击

```php
/**
 * 遥控器
 * Class RemoteController
 */
class RemoteController
{
    private $_state = null;

    public function __construct(IState $state)
    {
        $this->_state = $state;
    }

    public function setState(IState $state)
    {
        $this->_state = $state;
    }

    public function onClick()
    {
        $this->_state->handle($this);
    }
}
```

### 代码实现

```php
define("NEWLINE",chr(10));

interface IState
{
    public function handle(RemoteController $controller);
}

/**
 * 开机状态
 * Class OnState
 */
class OnState implements IState
{
    public function handle(RemoteController $controller)
    {
        echo '开空调' . NEWLINE;
        $controller->setState(new self);
    }
}


/**
 * 关机状态
 * Class OffState
 */
class OffState implements IState
{
    public function handle(RemoteController $controller)
    {
        echo '关空调' . NEWLINE;
        $controller->setState(new self);
    }
}


/**
 * 遥控器
 * Class RemoteController
 */
class RemoteController
{
    private $_state = null;

    public function __construct(IState $state)
    {
        $this->_state = $state;
    }

    public function setState(IState $state)
    {
        $this->_state = $state;
    }

    public function onClick()
    {
        $this->_state->handle($this);
    }
}

// 默认开空调
$controller = new RemoteController(new OnState());
$controller->onClick();
$controller->onClick();
$controller->onClick();
$controller->onClick();

// 开空调
// 关空调
// 开空调
// 关空调
```

### 最佳实践

上面，我们实现了一个遥控器来控制开关状态，但是细心的你会发现，每次点击按钮，我们都要重新做一个状态（new OnState() 或者 new OffState()）出来，这不太好。无论是开还是关状态，都应该在开机的时候就**存在**的，不需要我们重复去做。所以，我们使用**单例模式**，让开、关状态都只定义一次。

```php
define("NEWLINE",chr(10));

interface IState
{
    public function handle(RemoteController $controller);
}

/**
 * 开机状态
 * Class OnState
 */
class OnState implements IState
{
    private static $_instance = null;

    private function __construct() {}

    public function getInstance()
    {
        if (is_null(self::$_instance)) {
            self::$_instance = new self;
        }

        return self::$_instance;
    }

    public function handle(RemoteController $controller)
    {
        echo '开空调' . NEWLINE;
        $controller->setState(OffState::getInstance());
    }
}


/**
 * 关机状态
 * Class OffState
 */
class OffState implements IState
{
    private static $_instance = null;

    private function __construct() {}

    public function getInstance()
    {
        if (is_null(self::$_instance)) {
            self::$_instance = new self;
        }

        return self::$_instance;
    }

    public function handle(RemoteController $controller)
    {
        echo '关空调' . NEWLINE;
        $controller->setState(OnState::getInstance());
    }
}


/**
 * 遥控器
 * Class RemoteController
 */
class RemoteController
{
    private $_state = null;

    public function __construct(IState $state)
    {
        $this->_state = $state;
    }

    public function setState(IState $state)
    {
        $this->_state = $state;
    }

    public function onClick()
    {
        $this->_state->handle($this);
    }
}

// 默认开空调
$controller = new RemoteController(OnState::getInstance());
$controller->onClick();
$controller->onClick();
$controller->onClick();
$controller->onClick();

// 开空调
// 关空调
// 开空调
// 关空调
```

done~