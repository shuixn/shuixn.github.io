---
title: xhprof性能分析
date: 2019-07-02
categories:
  - 技术
tags: 
  - PHP 
  - 性能
---

## 安装

```bash
git clone https://github.com/longxinH/xhprof.git ./xhprof
cd xhprof/extension/
/path/to/php7/phpize
./configure --with-php-config=/path/to/php7/bin/php-config --enalbe-xhprof
make && sudo make install
```

php.ini

```vim
[xhprof]
extension=xhprof.so;
xhprof.output_dir=/var/tmp/xhprof
```

``xhprof.output_dir``是 **xhprof** 的输出目录，每次执行 xhprof 的 ``save_run`` 方法时都会生成一个 run_id.project_name.xhprof 文件

## 图形界面

```bash
brew install graphviz
```

## 使用方法

### xhprof 配置选项

- XHPROF_FLAGS_NO_BUILTINS 跳过所有内置（内部）函数。
- XHPROF_FLAGS_CPU 输出的性能数据中添加 ``CPU`` 数据。
- XHPROF_FLAGS_MEMORY 输出的性能数据中添加 ``内存`` 数据。

### 倾入性代码（可精确控制想要分析的代码段）

```php
xhprof_enable(XHPROF_FLAGS_CPU + XHPROF_FLAGS_MEMORY);

// 要检查性能的代码

$xhprof_data = xhprof_disable();
include_once  '/path/to/xhprof/xhprof_lib/utils/xhprof_lib.php';
include_once  '/path/to/xhprof/xhprof_lib/utils/xhprof_runs.php';
$xhprof_runs = new \XHProfRuns_Default();
$run_id = $xhprof_runs->save_run($xhprof_data, 'your_project');
```

可将上下两部分抽出来放在两个文件中，如下所示

```php
require_once '/path/to/xhprof/start.php';

// 要检查性能的代码

require_once '/path/to/xhprof/end.php';
```

### php代码入口（针对特定项目）

```vim
require_once '/path/to/xhprof/start.php';
register_shutdown_function(function() {
    $xhprof_data = xhprof_disable();
    if (function_exists('fastcgi_finish_request')){
        fastcgi_finish_request();
    }
    include_once "/path/to/xhprof/xhprof_lib/utils/xhprof_lib.php";
    include_once "/path/to/xhprof/xhprof_lib/utils/xhprof_runs.php";
    $xhprof_runs = new XHProfRuns_Default();
    $run_id = $xhprof_runs->save_run($xhprof_data, 'your_project');
});
```

### fpm

php-fpm.d/www.conf 添加：

```vim
php_value[auto_prepend_file] = /path/to/xhprof/start.php
php_value[auto_append_file] = /path/to/xhprof/end.php
```

### nginx

或者将上面的代码提出来，放在一个 ``xhprof.php`` 文件

```vim
location ~ \.php$ {  
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    fastcgi_param PHP_VALUE "auto_prepend_file=/path/to/xhprof/xhprof.php";
    include        fastcgi_params;
}
```

### php.ini

```vim
auto_prepend_file = /path/to/xhprof/xhprof.php
```

## 访问

### 使用nginx

```vim
server {
    listen 80;
    root /path/to/xhprof/xhprof_html;
    server_name your_host;
    location = / {
        index index.php;
    }
    location ~ \.php {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### 使用php内置服务器

```bash
cd /path/to/xhprof/xhprof_html
php -S 127.0.0.1:9000
```

## 性能指标

- funciton name ： 函数名
- calls: 调用次数
- Incl. Wall Time (microsec)： 函数运行时间（包括子函数）
- IWall%：函数运行时间（包括子函数）占比
- Excl. Wall Time(microsec)：函数运行时间（不包括子函数）
- EWall%：函数运行时间（不包括子函数）

## Done

![check_word_sim_xhprof.png](/images/xhprof/check_word_sim_xhprof.png)