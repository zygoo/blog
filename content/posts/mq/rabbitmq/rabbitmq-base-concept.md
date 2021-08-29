---
title: "聊一聊RabbitMQ的基本原理"
date: 2021-08-27T12:18:09+08:00
draft: false
toc: true
images:
tags:
- mq
---

## RabbitMQ的核心概念
* 生产者: 发送消息的应用
* 消费者: 接受消息的应用
* exchange: 将消息路由到queue的组件
* queue: 存储信息的缓存区，消费者对接的是queue, 并不关心别的配置项

## RabbitMQ信息流
1. 生产者向exchange发送消息
2. exchange根据消息属性路由到queue中进行存储
3. 消费者从queue拉取消息进行消费

![](/img/rabbitmq信息流.png)

## Exchange的基本的概念
生产者向RabbitMQ发送消息时，消息不会直接到达Queue, 而是先到达Exchange。Exchange将消息路由到一个或者多个Queue中。Exchange根据Binding Key，Routing Key以及Headers属性路由消息，下面我们来介绍三种常见的Exchange:
* Direct Exchange
* Topic Exchange
* Fanout Exchange

### Direct Exchange
Direct Exchange根据Binding Key和Routing Key完全匹配的规则去路由信息，这是最常见的配置方式。我们尝试着用简单的思路来说明这两个Key：
* Routing Key，用于指定消息路由到具体的Exchange, 在Direct Exchange中，这个key一般设置为Queue Name
* Binding Key，顾名思义，这个Key用于Exchange和Queue的Binding

简单一点来说，在Direct Exchange中，Routing Key，Binding Key, Queue是一一对应的，其次是Routing Key和Queue的命名是一致的。

![](/img/rabbitmq_direct_exchange.png)

### Topic Exchange
Topic Exchange和Direct Exchange类似，也需要通过Routing Key来路由消息。区别在于Direct Exchange对Routing Key是精确匹配，Topic Exchange支持模糊匹配，也就是通配符匹配。*表示匹配一个单词，#表示匹配零个, 一个或者多个单词。在这个类型的Exchange中，Routing Key一般是具体的，通配符一般放在Binding Key中。

![](/img/rabbitmq_topic_exchange.png)

### Fanout Exchange
还有一种的特殊的Exchange, 就是Fanout Exchange。 Fanout Exchange忽略Routing Key和Binding Key的匹配规则，会将消息路由到所有的的绑定的queue中。Fanout Exchange适用于广播这种消息的场景，比如说分发系统适用Fanout Exchange来广播各种状态和配置的更新。

## Connection和Channel
![](/img/rabbitmq_channel.png)

### Connection
connection就是物理连接，connection会将应用与消息队列连接在一起。Connection会执行认证，IP解析，路由等底层网络任务。应用与消息队列完成connection建立大约需要15个报文交互，因此会消耗大量的网络资源。大量connection会对消息队列造成巨大的压力，甚至触发RabbitMQ的SYN洪水攻击保护，导致消息队列无响应，进而影响业务。

### Channel
* channel是物理TCP连接中的虚拟连接，当应用通过connection与消息网关建立连接后，所有的AMQP协议操作（比如说创建队列，发送消息，接收消息）都会通过connection中的channel完成
* 保持connection长连接，请勿频繁开启或关闭Connection。如果确实需要频繁开启或关闭连接，请使用Channel，我们现在lib库就是这样实现的
* producer和consumer分别使用不同的connection进行消息发送和消费

### 到底什么是Channel?
当我们去搜索RabbitMQ概念的时候，几乎所有的文章都会告诉我们，channel是一个virtual connection, 仅此而已。那到底什么是channel? 答案还是很抽象，这个问题困扰了我很久。

直到有一天，我在看http2.0的多路复用的时候，终于找到了对这个问题的简单回答，这种发现答案的方式令人兴奋。入行的这几年，我一直都在寻找成为合格程序员的入门法则（没办法，底子太薄了，脑子也不是太好使）。看了很多博客，公众号，知乎等等，方法论多如牛毛，令我印象深刻的总结有两个：
1. 程序员要构建完整的计算机知识体系，不要一叶障目，不见森林。很多时候，我们可能会钻技术的牛角尖，比如说偏僻的语法糖，极致的封装，这种炫技一般的代码反而会加大后期维护的难度。跳出这种局限性，从网络，os, 数据库，分布式，微服务等更全面的角度去设计架构，这才是可行之道
2. 精于基础，广于工具。正如上文所说的，我从http2.0的概念中找到了关于RabbitMQ的channel的答案，这就是基础的力量。良好的基础可以帮助我们快速地将知识体系串联起来，举一反三。九层之台，起于累土

废话说得太多，回到问题本身。http2提出了流的概念，每一次请求对应一个流，有一个唯一的ID, 用来区分不同的请求。基于流的概念，进一步提出了帧。一个请求的数据会被分为多个帧，方便进行数据分割传输。每个帧都唯一属于某一个流ID，将帧按照流ID进行分组，即可分离出不同的请求。

这样同一个TCP连接中就可以同时并发多个请求，不同的请求的帧数据可以穿插在一起。根据流的分组即可。这种方式直接解决了http1.1的核心痛点，通过复用TCP连接，不用再同时建多个连接，提升了TCP的利用效率。这也是多路复用思想的一种落地方式，在很多消息队列协议中也广泛存在，比如AMQP, 其channel的概念和流如出一辙，大道相通。

## 参考
* [阿里云RabbitMQ文档](https://help.aliyun.com/document_detail/178124.html?spm=a2c4g.11186623.6.546.5d697340qd92FD)
* [理解RabbitMQ Exchange](https://zhuanlan.zhihu.com/p/37198933)
* [如何借助HTTP2实现传输](https://zhuanlan.zhihu.com/p/161577635)




