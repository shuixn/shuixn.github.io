---
title: 【打脸】php5解决laravel队列中的超时重试问题
date: 2019-01-19
categories:
  - 技术
tags: 
  - PHP
  - Laravel
---

## 写在前

上一篇文章{% post_link 【Laravel】laravel队列中的timeout参数需要使用php7.1 %}，最后结论时，我说这个问题只能升级php7.1才能解决，在今天的思考和实验中发现，还可以“曲线救国”！！啪啪打脸。

上一篇文章，在读了laravel源码知道，在异步队列中，laravel使用了一个php7.1才有的函数[pcntl_async_signals](http://php.net/manual/en/function.pcntl-async-signals.php)，这让我瞬间失去了所有想法，虽然升级PHP7是大趋势，但是有些依赖库可能在支持上还不完善。当然，大部分时候建议是升级的，PHP5很快就不进行安全维护了啊。

## 再探索

这个离线队列到底是怎么运行的？有兴趣的朋友可以看下陈昊写的[《laravel框架关键技术解析》](https://blogoss.yinghualuo.cn/blog/2017/08/Laravel框架关键技术解析-陈昊.pdf)中13章【消息队列】，能够大致明白laravel程序消息队列的“前半部分”，为什么我说前半部分？接着看

这部书讲了**同步类型**和**数据库类型**消息队列，接下来我要讲的是数据库类型里面的**redis驱动类型**以及**worker处理程序**这里面到底发生了什么事。

laravel优秀的设计与机制给我们提供了很多最佳实践，这也意味着**隐藏了不少黑盒子**，有时候让人不痛快、不明所以。只能一边看源码、一边做实验、一边感叹XX。

如果你已经了解了laravel队列的机制或者看完了[《laravel框架关键技术解析》](https://blogoss.yinghualuo.cn/blog/2017/08/Laravel框架关键技术解析-陈昊.pdf)这本书，我想你已经明白消息的生成和发送。着重看下消息的处理

如果你使用supervisor来管理队列程序，一般会开启多个worker

```
numprocs=8
```

假设有8个worker(1000，1001，...，1007)，后面再说下只有一个worker会发生什么

当一个消息来了，会先在redis生成一个list，假设你的队列名称为queueA，就会生成一个名为**queue:queueA**的list，此时，worker1000就会从list中pop一个消息出来处理，把这个消息放入一个名为**queues:queueA:reserved**的zset中，即“保留消息有序集”。

这时会有一堆信号注册，其中就包括timeout超时信号检测，如上篇文章所说，由于背景是php5，所以并不会注册异步信号，timeout参数就没用了，看下面的源码得知

```php
/**
     * Enable async signals for the process.
     *
     * @return void
     */
    protected function listenForSignals()
    {
        if ($this->supportsAsyncSignals()) {
            pcntl_async_signals(true);

            pcntl_signal(SIGTERM, function () {
                $this->shouldQuit = true;
            });

            pcntl_signal(SIGUSR2, function () {
                $this->paused = true;
            });

            pcntl_signal(SIGCONT, function () {
                $this->paused = false;
            });
        }
    }

    /**
     * Register the worker timeout handler (PHP 7.1+).
     *
     * @param  \Illuminate\Contracts\Queue\Job|null  $job
     * @param  WorkerOptions  $options
     * @return void
     */
    protected function registerTimeoutHandler($job, WorkerOptions $options)
    {
        if ($this->supportsAsyncSignals()) {
            // We will register a signal handler for the alarm signal so that we can kill this
            // process if it is running too long because it has frozen. This uses the async
            // signals supported in recent versions of PHP to accomplish it conveniently.
            pcntl_signal(SIGALRM, function () {
                $this->kill(1);
            });

            pcntl_alarm(
                max($this->timeoutForJob($job, $options), 0)
            );
        }
    }

    /**
     * Determine if "async" signals are supported.
     *
     * @return bool
     */
    protected function supportsAsyncSignals()
    {
        return version_compare(PHP_VERSION, '7.1.0') >= 0 &&
               extension_loaded('pcntl');
    }
```

上篇文章就讲到了这里，我们接着看，既然timeout信号无效，任务过时了自然就不会终止，可是时间还在继续，走到了另一个参数retry_after到期要做的事情。

>为了避免任务被执行多次，retry_after参数要比timeout参数的值大一些

这里假设**timeout为10s，retry_after为15s，tries为3次**

>tries是max_tries，即任务执行次数

worker1000检测到tries为3，也就是说，这任务能执行3次，就把任务信息中**attempts改为1，原先是0**，重新释放回队列中，看下面的代码。还没完，这任务还在执行呢....

```php
/**
     * Get the Lua script for releasing reserved jobs.
     *
     * KEYS[1] - The "delayed" queue we release jobs onto, for example: queues:foo:delayed
     * KEYS[2] - The queue the jobs are currently on, for example: queues:foo:reserved
     * ARGV[1] - The raw payload of the job to add to the "delayed" queue
     * ARGV[2] - The UNIX timestamp at which the job should become available
     *
     * @return string
     */
    public static function release()
    {
        return <<<'LUA'
-- Remove the job from the current queue...
redis.call('zrem', KEYS[2], ARGV[1])

-- Add the job onto the "delayed" queue...
redis.call('zadd', KEYS[1], ARGV[2], ARGV[1])

return true
LUA;
    }
```

worker1001等呀等，终于等到了一个消息，就拿了执行了，和worker1000一样，到了retry_after的时候，把attempts改为2，又释放回去。

worker1002同上，把attempts改为3，又释放回去。

worker1003拿到后，一看，tries为3啊，已经不能再执行了，就抛异常**Illuminate\Queue\MaxAttemptsExceededException**，并说

>A queued job has been attempted too many times. The job may have previously timed out.

然后，worker1003回调任务里面的failed方法，并终止任务逻辑

```php
	/**
	 * 任务失败回调方法
     * @param  Exception  $exception
     * @return void
     */
    public function failed(Exception $exception)
    {

    }
```

注意，此时worker1000、worker1001、worker1002还在执行....这是很可怕的，很容易引起**资源泄露、依赖服务负载过大**等问题。

## “曲线救国”

上面介绍了laravel队列中worker运行原理，但是问题并没有得到解决。这么说来，升级PHP7应该是最好的办法，确实是的，但是根据laravel worker的运行机制，还能这么做，接着看

利用laravel的**重试**机制和**异常**机制，我们可以这样设置

- tries=1
- timeout=10 (不变)
- retry_after=15 (不变)

这样一来，worker1000在执行任务时，我们记录该进程的ID，保存起来，能够让别的进程获取到。在worker1001重试的时候，会报异常，因为tries已经是最大值了。此时，在异常回调函数中，我们把worker1000的进程ID取出来，kill掉，supervisor会重新拉起一个新的worker，问题就解决了！

这个解决方案是不是特别有趣？:)

## 特殊情况：只有一个worker

上面的解决方案是可行的，但是，这是建立在supervisor能维护worker池的前提下，如果打从一开始，就只有一个worker，这怎么办呢？

这个问题让人吐血，因为真实的情况是，只有一个worker的情况下，会忽略掉retry_after，并不会释放任务，更别提报异常了，一股脑的往下执行....