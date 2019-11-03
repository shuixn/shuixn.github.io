---
title: funtask：又一个多任务处理框架
date: 2018-07-29
categories:
  - 项目
tags: 
  - PHP
  - Swoole
  - 多任务处理
---

## 框架获取

[funtask](https://github.com/funsoul/funtask "funtask")

## 更新日志

1. 2018-07-29 Swoole1.0简单版生产消费模型，采用多进程模式
2. 2019-04-25 采用Swoole4重构进程池（退出重启），增加进程组（退出不重启）
3. 2019-05-06 支持协程Coroutine

## 使用例子

### 耗时任务例子

```php
class Consumer implements \Funsoul\Funtask\Process\JobInterface {

    /**
     * business job
     *
     * @return bool [exit process or not]
     */
    public function handle(): bool
    {
        $pid = getmypid();

        $i = 0;
        $running = true;
        while ($running) {
            echo "{$pid}: " . $i++ . PHP_EOL;

            if ($i == 5)
                $running = false;
        }

        // exit current process
        return false;
    }
}
```

### ProcessPool进程池

```php

$task = new \Funsoul\Funtask\Funtask();
$task->setType('POOL')
    ->setJob(new Consumer())
    ->setWorkerNum(3)
    ->setWorkerName('myWorker')
    ->start();
```

### ProcessGroup进程组

```php
$task = new \Funsoul\Funtask\Funtask();
$task->setType('GROUP')
    ->setJob(new Consumer())
    ->setWorkerNum(3)
    ->setWorkerName('myWorker')
    ->start();
```

### Coroutine协程

#### 协程任务例子

```php
class ConsumerCo implements \Funsoul\Funtask\Coroutine\CoJobInterface {

    /**
     * @param \Swoole\Http\Request $request
     * @return mixed|void
     */
    public function handle(\Swoole\Http\Request $request)
    {
        $cid = Co::getuid();

        $i = 0;
        $running = true;
        while ($running) {
            echo "{$cid}: " . $i++ . PHP_EOL;

            Co::sleep(1);

            if ($i == 5)
                $running = false;
        }
    }
}
```

#### funtask统一接口

```php
$task = new \Funsoul\Funtask\Funtask();
$task->setType('CO')->setCoJob(new ConsumerCo())->setWorkerNum(3);
/** var \Swoole\Http\Response $response */
$task->setFinishCallback(function ($response) {
	$response->end('finished');
});
$task->start();
```

#### coroutine接口

```php
$co = new \Funsoul\Funtask\Coroutine\Coroutine();
$co->setHost('127.0.0.1')
    ->setPort(9501)
    ->setWorkerNum(1)
    ->setCoNum(3)
    ->setJob(new ConsumerCo());

/** var \Swoole\Http\Response $response */
$co->start(function ($response) {
	$response->end("finished!\n");
});
```