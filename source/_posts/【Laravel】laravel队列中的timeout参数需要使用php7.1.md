---
title: laravel队列中的timeout参数需要使用php7.1
date: 2018-12-29
categories:
  - 技术
tags: 
  - PHP
  - Laravel
---


## timeout解释

[job-expirations-and-timeouts](https://laravel-china.org/docs/laravel/5.4/queues/1256#job-expirations-and-timeouts)

<!-- more -->

### 任务过期 & 超时

#### 任务过期

>config/queue.php 配置文件里，每一个队列连接都定义了一个 retry_after 选项。这个选项指定了任务最多处理多少秒后就被当做失败重试了。比如说，如果这个选项设置为 90，那么当这个任务持续执行了 90 秒而没有被删除，那么它将被释放回队列。通常情况下，你应该把 retry_after 设置为最长耗时的任务所对应的时间。
>{note} 唯一没有 retry_after 选项的连接是 Amazon SQS。当用 Amazon SQS 时，你必须通过 Amazon 命令行来配置这个重试阈值。


## 版本

- php 5.6.30
- laravel 5.4.*

## laravel5.4源码

[Worker](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Queue/Worker.php)

```php
/**

* Register the worker timeout handler (PHP 7.1+).
*
* @param \Illuminate\Contracts\Queue\Job|null $job
* @param WorkerOptions $options
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


## 测试结论

1. php5.6，无法使用timeout属性
2. 任务进入重试后，相关参数就全部失效了，如tries，因此长耗时任务会一直跑
3. tries为0时，会陷入无限重试。此时，必须要把redis中“queues:【这里是你的queue名称】:reserved”删除才能终止
3. 上面的设置中，retry_after比timeout大一些，在timeout无效后，任务会进行retry（丢回队列，由其他worker接管执行，这时会出现第2点的问题），当重试次数达到设定值时，原进程就会异常退出，报【A queued job has been attempted too many times. The job may have previously timed out】错误。


上面的问题都是因为，php5不支持timeout........看来要使用这个参数，必须要升级到php7.1了....