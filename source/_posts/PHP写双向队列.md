---
title: PHP写双向队列
date: 2017-11-21
tags: 
  - PHP 
  - 数据结构与算法
  - 学习笔记
---
```php
<?php

/**
 * Class BidirectionalQueue
 */
class BidirectionalQueue{
    /**
     * 队列
     * @var array
     */
    private $queue = [];

    /**
     * 从头部入队
     * @param $item
     * @return int
     */
    public function enqueueFromHead($item){
        return array_unshift($this->queue,$item);
    }

    /**
     * 从尾部入队
     * @param $item
     * @return int
     */
    public function enqueueFromEnd($item){
        return array_push($this->queue,$item);
    }

    /**
     * 从头部出队
     * @return mixed
     */
    public function dequeueFromHead(){
        return empty($this->queue) ? false : array_shift($this->queue);
    }

    /**
     * 从尾部出队
     * @return mixed
     */
    public function dequeueFromEnd(){
        return empty($this->queue) ? false : array_pop($this->queue);
    }

    /**
     * 获取队列
     * @return array
     */
    public function getQueue(){
        return $this->queue;
    }
}

$queue = new BidirectionalQueue();
$queue->enqueueFromHead(1);
$queue->enqueueFromHead(2);
$queue->enqueueFromEnd(3);
print_r($queue->getQueue());
$queue->dequeueFromHead();
$queue->dequeueFromEnd();
print_r($queue->getQueue());
Output

Array
(
    [0] => 2
    [1] => 1
    [2] => 3
)
Array
(
    [0] => 1
)
```