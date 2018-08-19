---
title: 基于Bootstrap+jQuery写前端分页工具
date: 2016-11-23
categories:
  - 技术
tags: 
  - js
  - css
---

## 前言

为啥名字叫【前端分页工具】？因为我实在想不到什么好名字，如果想要更加贴切的理解这个工具，应该从业务来看

业务是这样的，有一个数据从后台传到前台，因为数据量不大，因此传过来之后直接显示即可，但是=。=所谓的数据量不大，最多也达到成百上千条，不可能全部显示出来，那么就需要分页

常规的分页是利用Ajax，通过传页偏移量到后台，后台查询数据库再返回数据，可以实现无刷新分页，拿到的数据也是最新的

### 前端分页

- 优点：一次传输数据，避免用户反复请求服务器，减少网络带宽、服务器调度压力、数据库查询、缓存查询压力
- 缺点：有新数据无法实时更新，除非用户重新请求数据

### 过程

刚开始我不希望造轮子，想尽快完成，于是找了很久Bootstrap的工具，找到了一个BootstrapTable,这个插件很强大，除了可以使用常规的方式分页，还可以指定数据（json），进行前端分页

但是，这个是表格分页，也就是说，如果不是表格那就没用了，刚好...我的业务就不是表格..那么只能看插件源码修改了，改的面目全非后，上个厕所回来，想到了更好的办法，于是删除...

解决办法：先思考分页是为了什么？呈现想看的页面，隐藏不想看的。没错，可以利用CSS的display属性

## 版本

- Bootstrap-3.3.0
- jQuery-1.11.3

## 开始编写

```js
<script type="text/javascript">

    //上一页
    function previous(){
        //当前页-1
        new_page = parseInt($('#current_page').val()) - 1;
        go_to_page(new_page);
    }
    //下一页
    function next(){
        //当前页+1
        new_page = parseInt($('#current_page').val()) + 1;
        go_to_page(new_page);
    }
    //跳转某一页
    function go_to_page(page_num){
        $('.page_link[longdesc=' + page_num +']').parent().addClass('active').siblings('.active').removeClass('active');
        //获取隐藏域中页数大小（每页多少条数据）
        var show_per_page = parseInt($('#show_per_page').val());
        //得到元素从哪里开始的片数（点击页 * 页大小）如点击第5页，5条/页。则开始为25
        start_from = page_num * show_per_page;
        //得到结束片的元素数量，如果开始为25，5条/页，则结束为30
        end_on = start_from + show_per_page;
        //隐藏所有子div元素的内容,显示具体片数据，如显示25~30
        $('#datas').children().css('display', 'none').slice(start_from, end_on).css('display', 'inline-block');

        //每页显示的数目
        var show_per_page = 10;
        //获取总数据的数量
        var number_of_items = $('#datas').children().size();
        //计算页数
        var number_of_pages = Math.ceil(number_of_items/show_per_page);
        //在页数区域内则做页偏移
        if( (page_num >= 2) && (page_num <= (number_of_pages - 3)) ){
            //生成分页->上一页
            var page_info = '<li><a class="previous_link" href="javascript:previous();">&laquo;</a></li>';

            var p = page_num;
            var i = page_num - 2;
            var j = page_num + 2;
            //生成分页->前2页
            while(page_num > i){
                page_info += '<li><a class="page_link" href="javascript:go_to_page(' + i +')" longdesc="' + i +'">'+ (i + 1) +'</a></li>';
                i++;
            }
            //生成分页->当前页
            page_info += '<li><a class="page_link" href="javascript:go_to_page(' + page_num +')" longdesc="' + page_num +'">'+ (page_num + 1) +'</a></li>';
            //生成分页->后2页
            while(p < j){
                if(p == number_of_pages){
                    break;
                }
                page_info += '<li><a class="page_link" href="javascript:go_to_page(' + (p + 1) +')" longdesc="' + (p + 1) +'">'+ (p + 2) +'</a></li>';
                p++;
            }
            //生成分页->下一页
            page_info += '<li><a class="next_link" href="javascript:next();">&raquo;</a></li>';

            //加载分页
            $('.pagination').html(page_info);
            $('.page_link[longdesc=' + page_num +']').parent().addClass('active').siblings('.active').removeClass('active');
        }
        else{ //否则不偏移
            //激活某一页，使得显示某一页
            $('.page_link[longdesc=' + page_num +']').parent().addClass('active').siblings('.active').removeClass('active');
        }

        //更新隐藏域中当前页
        $('#current_page').val(page_num);
    }

    $(function(){
        //每页显示的数目
        var show_per_page = 10;
        //获取话题数据的数量
        var number_of_items = $('#datas').children().size();
        //计算页数
        var number_of_pages = Math.ceil(number_of_items/show_per_page);
        //设置隐藏域默认值
        $('#current_page').val(0);
        $('#show_per_page').val(show_per_page);

        //生成分页->上一页
        var page_info = '<li><a class="previous_link" href="javascript:previous();">&laquo;</a></li>';
        var current_link = 0;
        //生成分页->页数
        while(number_of_pages > current_link){
            if(current_link == 5){
                break;
            }
            page_info += '<li><a class="page_link" href="javascript:go_to_page(' + current_link +')" longdesc="' + current_link +'">'+ (current_link + 1) +'</a></li>';
            current_link++;
        }
        //生成分页->下一页
        page_info += '<li><a class="next_link" href="javascript:next();">&raquo;</a></li>';
        //加载分页
        $('.pagination').html(page_info);

        //生成分页->左侧总数
        $("#data-total-page").html(show_per_page+"条/页，共"+number_of_pages+"页")

        //激活第一页，使得显示第一页
        $('#data-pagination li').eq(1).addClass('active');
        //隐藏该对象下面的所有子元素
        $('#datas').children().css('display', 'none');
        //显示第n（show_per_page）元素
        $('#datas').children().slice(0, show_per_page).css('display', 'inline-block');
    });
</script>
```

HTML

```html

<!-- 数据 -->
<div id="datas">
<?php
    foreach($data as $v)
    {
        echo '<div class="data">
                <div class="data-info">
                    <div class="data-name">' + $v['name'] + '</div>
                    <div class="data-blog">' + $v['blog'] + '</div>
                </div>
            </div>
        ';
    }
?>
                  
</div>
<!-- 分页 -->
<input type="hidden" id="current_page" value="0">
<input type="hidden" id="show_per_page" value="0">
<div id="data-page-info">
    <div id="data-total-page"></div>
    <div id="data-pagination">
        <ul class="pagination"></ul>
    </div>
</div>
```
## 效果如下

![](/images/20161123170010888.jpg)

## 动态切换

![](/images/20161123170142889.jpg)