---
title: 【问题排查】PHP缓冲区满引起数据被截断，导致中文乱码
date: 2018-03-22
tags: 
  - PHP 
  - 乱码
  - 问题排查
---
## 奇怪的问题

最近发生了一个奇怪的问题，查询hadoop的后台“偶发性”的出现中文乱码，乱码和平时所遇到的形式不太一样，一般而言，如果是编码转换问题，中文乱码一般是这样的

```php
echo mb_convert_encoding("你好", "utf-8", "gb2312");
// 浣?濂
```

这表示，把一个gb2312编码的字符串转换为utf-8，能看到已经面目全非，不是原来的字，这是因为字符集的不同。还有更多这样的实例。字符集的不同导致原来的字符变为了其他的字符，但是，这篇文章遇到的情况完全不同，接着看

一开始，我也怀疑是字符集的问题，可能是数据库的编码或者是扩展的问题，又或者是php的编码问题，一一排查发现都是utf-8，我一度以为可能是扩展的问题，难道真要换一种连接数据库的扩展？

## 乱码

每个汉字都显示为向右的箭头，差不多像这样: ->

## 偶发性

另一个奇怪的问题，乱码是偶发性出现的。可能每5次就会出现2到3次，于是，我把注意力放到了fpm上，因为每次请求分配的worker不同

```bash
ps aux | grep php-fpm
```

根据worker监听的端口号查看系统调用

```bash
strace -p xxxx
```

worker非常多，需要一个一个的追踪，但是不碍事，因为我们只需要找到两组不同的数据即可。

终于，在追踪了10个worker后，我拿到了这样的两组数据，分别是，

- 没有出现乱码的系统调用ok.strace
- 出现乱码的no.strace

两组进行对比之下（人眼对比= =#），其中，我找到了一行关键的线索

```
// no.strace
recvfrom(7, 0x3f63178, 8196, 16384, 0, 0) = -1 EAGAIN (Resource temporarily unavailable)
```

在出现乱码的no.strace中，出现了这个Resource temporarily unavailable

## 缓冲区

代码如下，因为数据是生成器返回的，所以我做了一个buffer缓冲，每次写入1000条。

```php
private function export($filePath,$data,$lastWrite)
{
    $fp = null;
    if ($this->beforeFirstWrite) {
        $fp = fopen($filePath, "a");// 写入方式打开，将文件指针指向文件末尾。如果文件不存在则尝试创建之。
        fwrite($fp, "\xEF\xBB\xBF"); // 添加 BOM
        $this->beforeFirstWrite = false;
        $cellHead = array_keys($data);
        fputcsv($fp, $cellHead);
    }
    $this->buffer[] = $data;
    if(isset($this->buffer[999]) || $lastWrite){// 每次写入1000条，或者，生成器返回最后一条时，把所有数据写入
        $fp = fopen($filePath, "a");// 写入方式打开，将文件指针指向文件末尾。如果文件不存在则尝试创建之。
        foreach ($this->buffer as $fields) {
            fputcsv($fp, $fields);
        }
        unset($this->buffer);
    }
    if($fp){
        fclose($fp);
    }
}
```

#### 清除缓冲区

程序输出有两种方式：一种是即时处理方式，另一种是先暂存起来，然后再大块写入的方式，前者往往造成较高的系统负担。
我们知道，系统写入文件，并不是马上写入的，而是先写入缓冲区，然后由缓冲区写入到文件。

```php
private function export($filePath,$data,$lastWrite)
{
    $fp = null;
    if ($this->beforeFirstWrite) {
        $fp = fopen($filePath, "a");// 写入方式打开，将文件指针指向文件末尾。如果文件不存在则尝试创建之。
        fwrite($fp, "\xEF\xBB\xBF"); // 添加 BOM
        $this->beforeFirstWrite = false;
        $cellHead = array_keys($data);
        fputcsv($fp, $cellHead);
    }
    $this->buffer[] = $data;
    if(isset($this->buffer[999]) || $lastWrite){// 每次写入1000条，或者，生成器返回最后一条时，把所有数据写入
        $fp = fopen($filePath, "a");// 写入方式打开，将文件指针指向文件末尾。如果文件不存在则尝试创建之。
        foreach ($this->buffer as $fields) {
            fputcsv($fp, $fields);
        }
        unset($this->buffer);

        # 关键代码
        ob_flush();
        flush();  //刷新buffer
    }
    if($fp){
        fclose($fp);
    }
}
```