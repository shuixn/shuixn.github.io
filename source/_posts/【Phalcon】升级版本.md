---
title: 【Phalcon】升级版本
date: 2016-11-04
tags: 
  - PHP
  - Phalcon
---

今天在我的PHP7环境下安装phalcon-devtools，安装完成后无法使用，提示信息是版本不兼容。查看一下，发现phalcon版本是2.1.0的...当时下载安装没去看，于是乎，只能升级版本了

1.切换到下载Phalcon的目录

```bash
cd /root/cphalcon
git fetch
```

需要等待一段时间，看到最新版本是3.0.x

```bash
cd build/
git checkout 3.0.x
./install
```

重启php-fpm和nginx，再看一下phpinfo，看到更新成功了

测试一下phalcon-devtools
```bash
phalcon commands
```

![](/images/20161104163956558.jpg)	

thanks~