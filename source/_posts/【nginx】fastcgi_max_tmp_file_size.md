---
title: 下载大文件时不得不了解nginx的一个配置
date: 2019-02-14
categories:
  - 技术
tags: 
  - nginx
---

## 问题背景

laravel下载大文件服务，浏览器无法承载大文件下载，会崩溃。改为提供api，外部使用curl或者wget等工具下载。为什么没用scp呢？因为服务器权限问题，这个功能是为了解决浏览器下载大文件崩溃而生的。

<!-- more -->

## 问题

大文件下载遇到的问题，每次下载到1024M的时候，就自动停了。

## 解决过程

1. curl的问题，加上-C参数，提供断点续传。无效
2. wget，同上，到了1024M就自动停了
3. laravel的问题，并不是，symfony有很好的文件响应组件
4. php的问题，没有这样的配置
5. php-fpm的问题，同上，没有这样的配置
6. nginx的问题，是的！

## fastcgi_max_temp_file_size默认为1024M

配置说明

```
When buffering of responses from the FastCGI server is enabled, and the whole response does not fit into the buffers set by the fastcgi_buffer_size and fastcgi_buffers directives, a part of the response can be saved to a temporary file. This directive sets the maximum size of the temporary file. The size of data written to the temporary file at a time is set by the fastcgi_temp_file_write_size directive.
The zero value disables buffering of responses to temporary files.
This restriction does not apply to responses that will be cached or stored on disk.
```

改大一点，解决~
