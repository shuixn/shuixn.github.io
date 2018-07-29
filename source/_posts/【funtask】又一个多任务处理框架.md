---
title: 【funtask】又一个多任务处理框架
date: 2018-07-29
tags: 
  - PHP 
  - Swoole
  - 多任务处理
---

## 框架获取

[funtask](https://github.com/funsoul/funtask "funtask")

## 使用例子

写一个简单爬虫来做简单的例子吧，现在我想获取一些页面里面的所有URL，一般会写下这样的代码：

```php
class GetURL
{
    public static function start()
    {
        $t1 = microtime(true);
        $urls = [
            'http://www.baidu.com',
            'http://qq.com',
            'http://sina.com',
            'https://www.v2ex.com',
            'https://www.csdn.net'
        ];
        $res = [];
        $count = [];
        foreach($urls as $url) {
           $res[$url] = static::getPageLink($url);
           $count[$url] = count($res[$url]);
        }
        var_dump($count);
        $t2 = microtime(true);
        echo '耗时'.round($t2-$t1,3).'秒' . chr(10);
    }

    private static function getPageLink($url){
        set_time_limit(0);
        $html=file_get_contents($url);
        preg_match_all("/<a(s*[^>]+s*)href=([\"|']?)([^\"'>\s]+)([\"|']?)/ies",$html,$out);
        $arrLink=$out[3];
        $arrUrl=parse_url($url);
        $dir='';
        if(isset($arrUrl['path'])&&!empty($arrUrl['path'])){
            $dir=str_replace("\\","/",$dir=dirname($arrUrl['path']));
            if($dir=="/"){
                $dir="";
            }
        }
        if(is_array($arrLink)&&count($arrLink)>0){
            $arrLink=array_unique($arrLink);
            foreach($arrLink as $key=>$val){
                $val=strtolower($val);
                if(preg_match('/^#*$/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(preg_match('/^\//isU',$val)){
                    $arrLink[$key]='http://'.$arrUrl['host'].$val;
                }elseif(preg_match('/^javascript/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(preg_match('/^mailto:/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(!preg_match('/^\//isU',$val)&&strpos($val,'http://')===FALSE){
                    $arrLink[$key]='http://'.$arrUrl['host'].$dir.'/'.$val;
                }
            }
        }
        sort($arrLink);
        return $arrLink;
    }
}

GetURL::start();
```

可以看到，代码按预期跑完了，可是性能有点差强人意，因为这个程序是单进程的，如果我加入更多的URL，或者说，递归地执行这个程序（爬虫），把获得的URL加入进去执行。可能会跑很久很久..

```bash
$ php ../tests/GetURL.php 
array(5) {
  ["http://www.baidu.com"]=>
  int(28)
  ["http://qq.com"]=>
  int(691)
  ["http://sina.com"]=>
  int(29)
  ["https://www.v2ex.com"]=>
  int(345)
  ["https://www.csdn.net"]=>
  int(120)
}
耗时3.145秒
```

### 使用funtask

代码和上面没有太大的差别，多了两个接口函数dispatch和consume。

- dispatch负责把任务分发给worker
- worker通过consume来消费内容

```php
namespace App\Task;
use Funsoul\Funtask\Task\Base as Task;

class FunTask implements Task
{
    private static $urls = [
        'http://www.baidu.com',
        'http://qq.com',
        'http://sina.com',
        'https://www.v2ex.com',
        'https://www.csdn.net'
    ];
    public static function dispatch()
    {
        return array_pop(static::$urls);
    }

    public static function consume($url)
    {
        $res[$url] = static::getPageLink($url);
        $count[$url] = count($res[$url]);
        var_dump($count);
    }

    private static function getPageLink($url){
        set_time_limit(0);
        $html=file_get_contents($url);
        preg_match_all("/<a(s*[^>]+s*)href=([\"|']?)([^\"'>\s]+)([\"|']?)/ies",$html,$out);
        $arrLink=$out[3];
        $arrUrl=parse_url($url);
        $dir='';
        if(isset($arrUrl['path'])&&!empty($arrUrl['path'])){
            $dir=str_replace("\\","/",$dir=dirname($arrUrl['path']));
            if($dir=="/"){
                $dir="";
            }
        }
        if(is_array($arrLink)&&count($arrLink)>0){
            $arrLink=array_unique($arrLink);
            foreach($arrLink as $key=>$val){
                $val=strtolower($val);
                if(preg_match('/^#*$/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(preg_match('/^\//isU',$val)){
                    $arrLink[$key]='http://'.$arrUrl['host'].$val;
                }elseif(preg_match('/^javascript/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(preg_match('/^mailto:/isU',$val)){
                    unset($arrLink[$key]);
                }elseif(!preg_match('/^\//isU',$val)&&strpos($val,'http://')===FALSE){
                    $arrLink[$key]='http://'.$arrUrl['host'].$dir.'/'.$val;
                }
            }
        }
        sort($arrLink);
        return $arrLink;
    }
        
}
```

通过配置可以很轻易的修改master和worker的重要属性

```env
# PROCESS
PROCESS_NAME='funtask:process:'
DAEMON=false
MAX_EXECUTE_TIME=3600   # worker max execute time, second
MIN_WORKER_NUM=3 
MAX_WORKER_NUM=6
TICKER=100            # masker ticking, microsecond
PAUSE_TIME=0.1        # pause time when worker rob key of queue, second
MAX_QUEUE_NUM=10      # starting expand process when queue is overflow
```

执行效果如下，可以看出，在1秒内就完成了所有任务。

```bash
$ php bin/funtask 
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___masterRunning() INFO: master running
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: worker[9415] started
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: worker[9414] started
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: worker[9413] started
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":1,"queue_bytes":20}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: current num of process[3] worker[9415] starting consume: https://www.csdn.net
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":1,"queue_bytes":20}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: current num of process[3] worker[9414] starting consume: https://www.v2ex.com
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":1,"queue_bytes":15}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: current num of process[3] worker[9413] starting consume: http://sina.com
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":1,"queue_bytes":13}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":2,"queue_bytes":33}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":2,"queue_bytes":33}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":2,"queue_bytes":33}
[2018-07-30 00:06:40] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":2,"queue_bytes":33}
array(1) {
  ["https://www.csdn.net"]=>
  int(120)
}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: current num of process[3] worker[9415] starting consume: http://qq.com
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":1,"queue_bytes":20}
array(1) {
  ["http://sina.com"]=>
  int(29)
}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool__workerRunning() INFO: current num of process[3] worker[9413] starting consume: http://www.baidu.com
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
array(1) {
  ["http://www.baidu.com"]=>
  int(28)
}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
array(1) {
  ["http://qq.com"]=>
  int(691)
}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
array(1) {
  ["https://www.v2ex.com"]=>
  int(345)
}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:41] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:42] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
[2018-07-30 00:06:42] TRACE: #Funsoul\Funtask\Service\ProcessPool___logQueueStatus() INFO: statQueue => {"queue_num":0,"queue_bytes":0}
```