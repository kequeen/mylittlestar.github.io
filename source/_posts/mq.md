---
title: 小雨伞消息队列进阶之路
date: 2019-01-20 20:17:23
tags:
---
wiki上的定义，消息队列（Message queue，下面简称MQ）是一种进程间通信或同一进程的不同线程间的通信方式。


## 背景介绍
### 消息队列1.0
小雨伞原有消息队列底层是使用redis的list数据结构来实现的，通过rpush与lpop操作实现消息的入栈与出栈。
![](old_mq.png)
消息队列1.0优点：
1. 实现简单
2. 调用方便
但是随着业务的增长，消息队列1.0在功能和管理上越来越难以满足我们的业务需求。

### 业务痛点
原有消息队列1.0有以下业务痛点：
1. 不支持发布订阅，只支持了点对点模型
2. 不支持消费应答，即无法通过消息系统保证消息一定被成功处理，只能通过业务的最终一致性来进行保证
3. 无法直观看到当前消息系统的消费情况，只能通过业务告警来发现消息系统的异常

### 方案选择
我们比对了当前市场上的诸多方案，与我们的业务进行适配，并最终选择了rabbitmq来搭建我们的消息队列2.0
下面说一下我们为什么没有选择其他方案：
redis:
redis高版本推出了发布订阅功能，但是无法解决上面提出的业务痛点2、3
kafka:
官方不支持php
阿里云的MQ:
按照topic与消息流量进行双重收费，而我们的业务类型 topic 会比较多，但是流量初期可能并不多，价格上不划算；不开源，相对于我们是黑盒，很难进行二次开发，满足我们的个性化需求

## 消息系统2.0
底层基于 rabbitmq 实现，完美满足我们的业务痛点，开源，并且官方提供了 http 接口来对消息进行管理，我们可以方便地进行二次开发，满足我们的一些特殊业务需求。

### 系统模型
rabbitmq的基本模型如下
![](mq_model.png)
其本身提供了丰富的功能，包含Work queues(消息系统1.0具备的)，Publish/Subscribe，Routing，Topics，RPC。[具体详情](https://www.rabbitmq.com/getstarted.html)
我们目前只使用了其Publish/Subscribe功能，做了一定的封装。其它的功能待以后扩展使用。

### 时序图
我们重新封装了rabbitmq的接口，以满足我们的业务需求。生产者与消费者的使用时序如下。

![生产者时序图](producer.png)

![消费者时序图](consumer.png)

### 一些改进
1. 增加了业务的管理后台
订阅关系能够在管理后台进行管理和查看，包括topic与队列管理和查看。
![管理后台topic与队列绑定图](mq_bind.png)
并且能够在管理后台启动与关闭消费进程，不需要到具体机器上进行手动关闭。

2. 增加了关于消费进程的相关配置
如管理后台topic与队列绑定图中所示，我们能够指定机器群组来启动进程，比如群组有M台机器，配置的进程数为N，那么将同时有M x N个进程来消费队列中的消息。

3. 增加了消费异常的主动告警
![新增队列](mq_queue.png)
选项中未消费消息数告警阈值，当队列中未消费消息数超过设定的阈值时，将触发我们的告警，会给相关人发信息来及时处理。而不是和原来一样，仅仅通过业务上的异常来进行告警，系统将更加可靠。

4. 对手动确认与自动确认进行了封装，对业务方来说更方便。
```php
    /**
     * Starts a queue consumer
     *
     * @param string $queue
     * @param string $consumer_tag
     * @param bool $no_local
     * @param bool $no_ack
     * @param bool $exclusive
     * @param bool $nowait
     * @param callable|null $callback
     * @param int|null $ticket
     * @param array $arguments
     * @return mixed|string
     */
    public function basic_consume(
        $queue = '',
        $consumer_tag = '',
        $no_local = false,
        $no_ack = false,
        $exclusive = false,
        $nowait = false,
        $callback = null,
        $ticket = null,
        $arguments = array()
    ) {
    	...
    }
```
原本的调用方式
```php
$callback = function ($msg) {
  echo ' [x] Received ', $msg->body, "\n";
  sleep(substr_count($msg->body, '.'));
  echo " [x] Done\n";
  $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};

$channel->basic_consume('task_queue', '', false, false, false, false, $callback);
```
我们做了封装之后
![消费者消费示例](mq_ackconsumer.png)

##遇到的困难
### 1.MQ的崩溃问题
测试环境下搭建的mq系统时常容易崩溃，因为使用docker，并且对rabbitmq不够熟悉，定位了很久。最终通过docker logs 容器名
命令查到了erlang oom的报错，定位到了vm_memory_high_watermark.relative（Memory threshold at which the flow control is triggered）的设置有问题，其默认是0.4，




