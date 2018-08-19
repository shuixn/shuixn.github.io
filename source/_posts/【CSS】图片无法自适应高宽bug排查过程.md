---
title: 【CSS】图片无法自适应高宽bug排查过程
date: 2016-11-29
categories:
  - 技术
tags: 
  - css
  - 问题排查
---

## 前置知识

### css如何做图片自适应高宽？

- <div>块设置固定高宽
- <div>子元素设置百分比

如下

```css
#container{
     border:1px solid;
     height:180px;
     width:380px;   
}
#container img{
     height:100%;
     width:100%;  
}

```

```html
<div id="container">
    <a href="#"><img src="imgs/p2.jpg" /></a>
</div>
```

如图所示

![p1](/images/20161110164135461.jpg)

## bug描述

业务代码和以上demo有一点区别：

1. container宽度固定
2. container高度按内容垂直延伸

按照上面的写法，开始修改我的代码，一切进行的很顺利。然而，当我打开浏览器一看，吓一跳！

大概长这样：

![](/images/20161110165150270.jpg)

刚开始，我把问题定位在标签style的优先级大于style样式表，开始改。
后来，应该是别的元素样式冲突了？（使用的是bootstrap）
最后，我在火狐的调试工具中发现了一个单词

container->static

![](/images/20161110170007159.jpg)

img->static

![](/images/20161110170032534.jpg)

a->absolute

![](/images/20161110170050712.jpg)

原来，在样式表中，有这么一个定义

```css
#container a{
  position: absolute;
}
```

这是啥意思？回顾一下**absolute**的含义


>生成绝对定位的元素，相对于 static 定位以外的第一个父元素进行定位。
>元素的位置通过 "left", "top", "right" 以及 "bottom" 属性进行规定。


结果很明了了，由于我的container的高度是按内容垂直延伸的，所以图片的高度等于内容的高度。