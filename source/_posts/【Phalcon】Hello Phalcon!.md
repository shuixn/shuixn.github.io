---
title: 【Phalcon】Hello Phalcon!
date: 2016-11-01
tags: 
  - PHP
  - Phalcon
---

在上一篇文章中已经分别介绍了Phalcon在linux和windows下安装的步骤，接下来就是熟悉的hello world，为了方便，本次学习在windows xampp下进行。

## 文件结构

在xampp\htdocs下，我的习惯是建立一个www目录作为根目录，把所有项目放在一起管理，接下来，新建一个“hellophalcon”项目，然后接着建立所需的文件夹，看起来像这样：

```
hellophalcon/
	app/
		controllers/
		models/
		views/
	public/
		css/
		img/
		js/
```

我使用的是apache，然后hellophalcon目录下新建一个.htaccess文件，编写重写规则，内容如下

```vim

	<IfModule
	 mod_rewrite.c
	>

	    RewriteEngine on
	    RewriteRule  ^$ public/    [L]
	    RewriteRule  (.*) public/$1 [L]
	</IfModule>
```

说明：对该项目的所有请求都将被重定向到为public/文档根目录。此步骤可确保内部项目的文件夹仍然对公共访客隐藏，从而消除了一些安全威胁。

同样，在public目录下新建一个.htaccess文件。第二组规则（内容如下）将检查是否存在所请求的文件，如果存在所要请求的文件，就不需要Web服务器模块来重写：

```vim
<IfModule
 mod_rewrite.c
>

    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ index.php?_url=/$1 [QSA,L]
</IfModule>
```

刚刚写好的重写规则是把服务请求全部重定向到public目录下，所以，我们需要在public目录下做一些事情，在大多数框架中，被称为引导程序，新建一个index.php，内容如下

```php

try {

    //Register an autoloader
    $loader = new \Phalcon\Loader();
    $loader->registerDirs(array(
        '../app/controllers/',
        '../app/models/'
    ))->register();

    //Create a DI
    $di = new Phalcon\DI\FactoryDefault();

    //Setup the view component
    $di->set('view', function(){
        $view = new \Phalcon\Mvc\View();
        $view->setViewsDir('../app/views/');
        return $view;
    });

    //Setup a base URI so that all generated URIs include the "tutorial" folder
    $di->set('url', function(){
        $url = new \Phalcon\Mvc\Url();
        $url->setBaseUri('/hellophalcon/');
        return $url;
    });

    //Handle the request
    $application = new \Phalcon\Mvc\Application($di);

    echo $application->handle()->getContent();

} catch(\Phalcon\Exception $e) {
     echo "PhalconException: ", $e->getMessage();
}
```

然后，在app/controllers目录下新建一个控制器，命名规则应该是这样的，IndexController.php，内容如下

```php

class IndexController extends \Phalcon\Mvc\Controller
{

    public function indexAction()
    {
        echo "<h1>Hello Phalcon!</h1>";
    }

}
```

开启服务器，输入http://localhost/www/hellophalcon/

```
Hello Phalcon!	
```