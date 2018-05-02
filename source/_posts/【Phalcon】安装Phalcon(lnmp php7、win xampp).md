---
title: 【Phalcon】安装Phalcon(lnmp php7\win xampp)
date: 2016-10-31
tags: 
  - PHP
  - Phalcon
---

## LNMP环境

- CentOS6.5
- nginx 1.6.2
- MySQL5.6
- PHP5.6 / PHP7.0.12

## 安装

1.下载安装依赖库

```bash
yum install php5-dev libpcre3-dev gcc make php5-mysql php5-fpm 
```

2.下载git库

```bash
git clone --depth=1 git://github.com/phalcon/cphalcon.git
```

3.切换到build目录

```bash
cd cphalcon/build
```

4.开始安装

```bash
sudo ./install
```

5.创建一个文件 phalcon.ini 到 /etc/php.d/ 这个目录下，内容如下:

```vim
extension=phalcon.so
```

**注意：如果使用集成环境，如oneinstack，则把上述代码加到php.ini中即可**

6.测试安装

```bash
php -r 'echo phpinfo();' | grep -i phalcon
```

![](/images/20161031161833053.jpg)

或者

```bash
php -r 'echo print_r(get_loaded_extensions());'
```

![](/images/20161031161958023.jpg)

## nginx配置

配置nginx的时候，建议用\$_SERVER[‘REQUEST_URI’]方式，这样可以防止自动加入\$_GET[‘_url’]的隐规则，在参数签名时，如果你忘记这个隐规则会导至签名验证失败。

```vim
server {
    listen   80;
    server_name _;
    index index.php index.html index.htm;
    
    root $root_path/helloworld/public;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include fastcgi_params;
    }
}
```

## phalcon开发工具

```bash
git clone git://github.com/phalcon/phalcon-devtools.git
cd phalcon-devtools/
. ./phalcon.sh
ln -s ~/phalcon-devtools/phalcon.php /usr/bin/phalcon
chmod ugo+x /usr/bin/phalcon
```

## 测试工具安装

```bash
phalcon commands
```

![](/images/20161031185149164.jpg)

## 使用phalcon工具创建项目

```bash
phalcon create-project helloworld
```

## windows xampp环境

### php5.5.15 ts(线程安全)

![](/images/20161031230347528.jpg)

[官网地址](https://www.phalconphp.com/zh/download/windows)

**注意**

1. xampp在windows下永远都是32位的版本，所以需要下载x86的
2. 查看自己的phpinfo，看看php版本以及thread safety（线程安全）

下载好后，解压php_phalcon.dll到D:\xampp\php\ext这个目录下（具体以自己的xampp安装位置而定）

回到上一级目录D:\xampp\php\，找到php.ini，在文件尾加上

```vim
extension=php_phalcon.dll
```

重新启动apache，打开phpinfo，如下图即安装成功

![](/images/20161031231218416.jpg)