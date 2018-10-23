---
title: 【字符编码】PHP导出CSV中文乱码问题研究
date: 2018-02-02
categories:
  - 技术
tags: 
  - PHP 
  - 乱码
---

没有踩过字符编码问题的程序生涯是不完整的，还记得曾经还踩过Apache+PHP+MySQL的编码问题，不过那时候没总结下来，今天遇到了导出文件的编码问题，一起来好好研究一下:)

推荐一下这篇文章[十分钟搞清字符集和字符编码](http://cenalulu.github.io/linux/character-encoding/ "十分钟搞清字符集和字符编码")，可以快速了解一下字符编码的知识

### 业务背景

把数据查询结果导出到CSV，由于Laravel Excel内部实现方法的问题，载入大数据量时，容易爆内存，因此这里分了两种实现，小数据量则采用Laravel Excel，大数据量则使用无缓冲查询+Yield，详情可参考这篇文章[【Yield】大数据下的应用](http://funsoul.org/2018/02/01/【Yield】大数据下的应用/ "【Yield】大数据下的应用")

由于使用了Laravel Excel工具集，先来看看这个工具内部是如何实现编码兼容的
```php
# ..\vendor\maatwebsite\excel\src\Maatwebsite\Excel\Writers\LaravelExcelWriter.php
# 约347行
protected function _download(Array $headers = [])
{
    // Set the headers
    $this->_setHeaders(
        $headers,
        [
            'Content-Type'        => $this->contentType,
            'Content-Disposition' => 'attachment; filename="' . $this->filename . '.' . $this->ext . '"',
            'Expires'             => 'Mon, 26 Jul 1997 05:00:00 GMT', // Date in the past
            'Last-Modified'       => Carbon::now()->format('D, d M Y H:i:s'),
            'Cache-Control'       => 'cache, must-revalidate',
            'Pragma'              => 'public'
        ]
    );

    ...
    ..
    .
}
```
通过区分不同的文件类型设置$this->contentType，但是这里是用来设置浏览器下载的header的，并不是保存为服务器文件，继续找。


### 查看PHPExcel对CSV格式的兼容性实现
```php
# ..\vendor\phpoffice\phpexcel\Classes\PHPExcel\Writer\CSV.php
# 约116行

if ($this->_excelCompatibility) {
	fwrite($fileHandle, "\xEF\xBB\xBF");	//	Enforce UTF-8 BOM Header
	$this->setEnclosure('"');				//	Set enclosure to "
	$this->setDelimiter(";");			    //	Set delimiter to a semi-colon
    $this->setLineEnding("\r\n");
	fwrite($fileHandle, 'sep=' . $this->getDelimiter() . $this->_lineEnding);
} elseif ($this->_useBOM) {
	// Write the UTF-8 BOM code if required
	fwrite($fileHandle, "\xEF\xBB\xBF");
}
```

发现$this->_useBOM这个配置项，默认是没有开启的。查看使用LaravelExcel下载的CSV文件的BOM确实没有支持UTF-8

### laravel Excel配置开启use_bom
config/excel.php，设置csv的use_bom为true，默认为false

### 查看BOM
```bash
]$ head -c 3 file | hexdump -C

00000000  ef bb bf                                          |...|
00000003
```

找到问题的关键了，在需要写入文件前，添加UTF-8的BOM即可
```php
fwrite($fp, "\xEF\xBB\xBF"); // 添加 UTF-8 BOM
```