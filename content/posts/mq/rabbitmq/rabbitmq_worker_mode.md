---
title: "RabbitMQ进阶使用: 多消费者的worker模式"
date: 2021-08-29T11:18:09+08:00
draft: false
toc: true
images:
tags:
- mq
---

## 项目中遇到的问题
在现有的项目架构中，我们希望使用RabbitMQ实现多个消费者同时接受同一个队列的消息。使用过RocketMQ的朋友应该都知道，RocketMQ有一个消费组的概念，其组内的每一个消费者都会消费同一个message。

那怎么在RabbitMQ中做到群组消费呢？我的答案是，RabbitMQ不适合群组消费的业务场景。或者说，可以使用RabbitMQ + 其他组件去实现群组消费。

后面我们可以说一说，如何用RabbitMQ + Redis去设计一个分布式的websocket消息网关。

## RabbitMQ worker模式
![](/img/rabbitmq_work_mode.png)

worker模式就是一种1:n的消费模式，一个生产者对应多个消费者，消费者之间存在竞争关系，只有一个消费者会获得信息，进行消费。

"如果同一个队列，有多个消费者消费这个队列。
RabbitMQ的消息的分配策略默认是按照轮询的策略发送消息，即发送的顺序是消费者1，消费者2，消费者1，消费者2，...。所以平均下来，每个消费者消费的消息数量几乎相同"，
这段说明是一篇博客上摘抄下来的，但我还是想看看官方的doc是怎么描述这个消费策略的：

"Normally, active consumers connected to a queue receive messages from it in a round-robin fashion. 
When consumer priorities are in use, messages are delivered round-robin if multiple active consumers exist with the same high priority."

官方doc告诉我们，当存在多个消费者的时候，消息的接收会使用轮询的策略。可以通过设置优先级，来为每个消费者设置权重。

## 群组消费的解决方案和问题
为每个消费者声明一个queue，利用Fanout模式绑定到同一个exchange中。当消息到达exchange，就会路由到每个queue。这种做法看似很美好，但是存在一些问题：
* 一般来说，在生产项目中，我们不会赋予消费者声明queue的权限
* 在云厂商的RabbitMQ服务套餐中（贼贵，升级套餐更贵），规定了queue的数量。假如说将declare queue的权限外放，消费应用可能会创建N个queue

有些朋友会说，我们公司有钱，多创建一些又怎么样？
我的观点是：可以接受queue的增长，但是不能接受无限制的增长。消费者的行为是不可控的，比如说它的代码不小心写了for循环，或者恶意攻击，可能会导致正常consumer不能declare queue，直接把现有的套餐打爆了，严重影响正常的业务。

所以，对外的mq, 是不适合开放权限的。

## 参考
* [官方doc, Consumer Priorities](https://www.rabbitmq.com/consumer-priority.html)
* [RabbitMQ,多消费者的WorK模式](https://blog.csdn.net/CDW2328/article/details/96700309)
* [rabbitmq 怎么实现多个消费者同时接收一个队列的消息？](https://www.zhihu.com/question/300082037)






