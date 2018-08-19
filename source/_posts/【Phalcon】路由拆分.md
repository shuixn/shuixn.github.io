---
title: 【Phalcon】路由拆分
date: 2016-12-15
categories:
  - 技术
tags: 
  - PHP 
  - Phlacon
---

## 写在前

去年使用过PHP的Flight框架编写api，Flight框架非常简单，只需要花一点点时间看官方文档即可上手写代码，原生支持Restful风格。刚开始没觉得什么，只要有需求则加一个路由，由于项目不大，总共也不超过10个。

最近在使用Phalcon，项目规模中等，在路由（routes.php）那里遇到了本文所关注的痛点，几十个路由放在一个文件中，而且还会继续往里加，当达到一定的数量级，如果有一天，我想改其中某一个路由（名称忘了或者记不清），则需要滚动鼠标刷半天才找到。代码混乱，所有的controller/action都放在这里管理。

如下

```php

/**
 * 首页
 */
$router->add(
    "/",
    [
        "controller" => "index",
        "action"     => "index",
    ]
);

/**
 * 用户信息
 */
$router->add(
    "/user/info",
    [
        "controller" => "user",
        "action"     => "info",
    ]
);

/**
 * 添加商品到购物车
 */
$router->add(
    "/shopcart/add",
    [
        "controller" => "shopcart",
        "action"     => "add",
    ]
);

```


## 正文

既然有痛点，那就想解决办法

Phalcon通常的项目结构大概是这样的

```php

app/                                   #应用程序
     cache/                            #缓存文件
          data/                             #缓存数据
          metaData/                         #缓存元数据
          models/                           #缓存模型
          views/                            #缓存视图
          volt/                             #缓存volt模版文件
     config/                           #配置文件
          config.example.php                #配置文件（加载环境配置，返回所有配置）
          development.example.php           #开发环境配置（定义debug模式等）
          env.php                           #配置文件（定义基路径，定义全局变量，设置环境异常模式）      
          nginx.example.conf                #nginx配置
          routes.php                        #返回路由配置
          timezones.php                     #返回时区配置
     controllers/                           #控制器
     library/                               #类库文件（工具类，启动文件）
          Badges/                           #
          Github/                           #
          Http/                             #
          Mail/                             #
          Markdown/                         #
          Mvc/                              #
          Notifications/                    #
          Paginator/                        #
          Queue/                            #
          Search/                           #
          Utils/                            #
          Bootstrap.php                     #启动文件
     logs/                                  #日志文件
     models/                                #模型类
     views/                                 #视图（volt模版）
docs/                                  #项目说明文档 
opsfiles/                              #运维工具配置文件
public/                                #公共目录
     css/                              #样式文件
     fonts/                            #字体文件
     icon/                             #图标文件
     js/                               #javascript脚本
     .htaccess                         #路由重写规则（指向该目录下的index.php）
     505.html                          #服务器异常展示文件
     index.php                         #引导文件（框架先从这里开始执行）
schemas/                               #sql文件
scripts/                               #服务器脚本
test/                                  #单元测试
.env.example                           #用户专属环境配置
.htaccess                              #路由重写规则（指向public目录下的index.php）
.htrouter.php                          #与.htaccess文件功能一样，包含public目录下的index.php
composer.json                          #依赖管理文件

```

### 加载流程

#### index.php

```php

1./public/index.php
use app\library\Bootstrap;
使用命名空间的方式，划分程序块

include_once realpath(dirname(dirname(__FILE__))) . '/app/config/env.php';
include_once BASE_DIR . 'app/library/Bootstrap.php';
第一行，包含配置文件，配置文件中有BASE_DIR的定义，APPLICATION_ENV的定义
第二行，包含引导文件

$bootstrap = new Bootstrap();

if (APPLICATION_ENV == ENV_TESTING) {
    return $bootstrap->run();
} else {
    echo $bootstrap->run();
}
声明Bootstrap对象，开始执行

```
找到Bootstrap.php，初始化路由的代码如下

```php
/**
     * Initialize the Router.
     */
    protected function initRouter()
    {
        ...
	    $router = include BASE_DIR . 'app/routes/routes.php';
	    ...
    }
```

可以发现，路由是这样包含进框架的，去到我们的routes.php，把原来的路由拆分成各自模块的文件，然后包含进routes.php，这样做的好处是，模块化管理，便于查找和修改。

#### routes.php

```php
<?php


use Phalcon\Mvc\Router;

$router = new Router(false);
$router->removeExtraSlashes(true);


/**
 * loader the urls
 */
$urls = [
    'index',
    'shopcart',
    'user',
];
foreach ($urls as $url){
    include BASE_DIR.'app/routes/'.$url.'.php';
}

return $router;
```

#### user.php

```php
<?php

/**
 * 用户信息
 */
$router->add(
    "/user/info",
    [
        "controller" => "user",
        "action"     => "info",
    ]
);
```

