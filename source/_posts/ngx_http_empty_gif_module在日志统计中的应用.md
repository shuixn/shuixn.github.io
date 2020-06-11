---
title: ngx_http_empty_gif_module在日志统计中的应用
date: 2018-04-16
categories:
  - 技术
tags: 
  - Nginx
---

## 日志统计分析架构

在日志统计分析中，通常包括以下的模块

- 前端日志上报（js上报url、user_agent等等）
- 打点服务器（nginx）
- 统计日志（access_log）
- 分析日志（后端程序）
- 保存分析结果（db）

<!-- more -->

我们通常把日志上报和日志分析解耦，利用nginx的大并发承载力，接收前端上报的日志。在这里，我们不做协议转发到后端程序（PHP或者其他cgi程序）处理日志统计，而是利用nginx的access_log，约定好日志保存格式（format），由nginx帮助我们做日志统计。

日志上报的并发量有可能非常巨大，举个例子，如果由PHP来处理，那么每一次日志上报都需要经过nginx的转发、PHP生命周期的五个阶段（MINIT、RINIT、代码处理、RSHUTDOWN、MSHUTDOWN），即使做了opcache，将预编译的脚本存储在共享内存中，也是需要不菲的消耗。1毫秒的处理时间不算什么，但是 1 X 100000n 那就需要引起关注了。

我们还需要一个分析日志的守护进程，这时候可以利用php的cli或者别的语言实现，将分析结果（pv、uv等等）保存到数据库，然后提供一个web界面来展示结果。

## ngx_http_empty_gif_module

http程序是请求-响应模式，日志上报完成后，后端需要返回一个值，告诉前端结果。一般来说，我们会使用json，上报一个日志，返回一个json串，比如

```json

{
	code: 0,
	msg: 'sucess'
}

```
正如前文所说，这是有可能是一个大并发的程序，如果每次都返回一个json串，是不是有点太大了，为什么这里会说“大”，接着看，我尝试返回一个整型数字1,

```php
echo 1;
```

在浏览器中查看返回数据的大小为 268B ，一个数字的占用就有268B，如果包含状态码，状态值，岂不是更大。

所以我们看看nginx有没有更好的解决方式，那就是ngx_http_empty_gif_module。

这是一个由nginx生成的 1x1 像素的图片，通过查看源码可以知道，这是nginx自己拼接而成的完全内存级别的图片

/path-to-nginx/src/http/modules/ngx_http_empty_gif_module.c

```c
static u_char  ngx_empty_gif[] = {

    'G', 'I', 'F', '8', '9', 'a',  /* header                                 */

                                   /* logical screen descriptor              */
    0x01, 0x00,                    /* logical screen width                   */
    0x01, 0x00,                    /* logical screen height                  */
    0x80,                          /* global 1-bit color table               */
    0x01,                          /* background color #1                    */
    0x00,                          /* no aspect ratio                        */

                                   /* global color table                     */
    0x00, 0x00, 0x00,              /* #0: black                              */
    0xff, 0xff, 0xff,              /* #1: white                              */

                                   /* graphic control extension              */
    0x21,                          /* extension introducer                   */
    0xf9,                          /* graphic control label                  */
    0x04,                          /* block size                             */
    0x01,                          /* transparent color is given,            */
                                   /*     no disposal specified,             */
                                   /*     user input is not expected         */
    0x00, 0x00,                    /* delay time                             */
    0x01,                          /* transparent color #1                   */
    0x00,                          /* block terminator                       */

                                   /* image descriptor                       */
    0x2c,                          /* image separator                        */
    0x00, 0x00,                    /* image left position                    */
    0x00, 0x00,                    /* image top position                     */
    0x01, 0x00,                    /* image width                            */
    0x01, 0x00,                    /* image height                           */
    0x00,                          /* no local color table, no interlaced    */

                                   /* table based image data                 */
    0x02,                          /* LZW minimum code size,                 */
                                   /*     must be at least 2-bit             */
    0x02,                          /* block size                             */
    0x4c, 0x01,                    /* compressed bytes 01_001_100, 0000000_1 */
                                   /* 100: clear code                        */
                                   /* 001: 1                                 */
                                   /* 101: end of information code           */
    0x00,                          /* block terminator                       */

    0x3B                           /* trailer                                */
};
```

继续刚刚的问题，前端上报日志后，后端怎么返回这个图片呢？返回这个图片有什么好处？

只需要在nginx中配置

```vim
server
{
        listen 80;
        server_name log.com;

        location = /dig {
           empty_gif;
        }
}
```

这样一来，前端只需要把url、ua等信息提交到 log.com/dig 即可，后端就会返回一个1pixel的图片。

通过查看浏览器信息可以发现，返回大小为 229B，是不是比返回json更快，占用更小呢 :)