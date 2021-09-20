---
title: "RabbitMQ断线重连的正确姿势"
date: 2021-09-13T12:18:09+08:00
draft: false
toc: true
images:
tags:
- mq
---

## 背景/问题
在项目中，使用RabbitMQ来做消息的异步分发，但是每隔一段时间都会出现发布订阅的报错， 服务重启后恢复。

### 问题分析
服务故障之后，首先查看了发布方和消费方的日志，发现了"channel/connection is not open"的错误日志。而为什么channel会被关闭，先来看看下面错误码：
| Description              | Reason                   | Type                     |
| ------------------------ | ------------------------ | ------------------------ |
| 应用程序主动断开连接，网络异常被动断开连接    | 连接中断                     | 连接关闭                     |
| 重新声明现有队列或具有不匹配属性的交换将失败   | 406 PRECONDITION_FAILED | 协议异常                     |
| 访问不允许用户访问的资源将失败          | 403 ACCESS_REFUSED错误    | 协议异常                     |
| 绑定不存在的队列或不存在的交换将失败       | 404 NOT_FOUND错误         | 协议异常                     |
| 从不存在的队列中使用将失败            | 404 NOT_FOUND错误         | 协议异常                     |
| 发布到不退出的交换将失败             | 404 NOT_FOUND错误         | 协议异常                     |
| 从除声明连接以外的连接访问独占队列将失败     | 405 RESOURCE_LOCKED     | 协议异常                     |

得到结论：由于网络波动、生产者消费者使用不当会导致channel被关闭。对于AMQP的重连，社区里面其实已经做过了很多讨论。AMQP的主要开发者认为重连是业务的事情，所以不愿意在AMQP底层中加入更多自动重连的逻辑。

没有重连的策略，导致消息无法消费订阅，服务中断，这在业务上是无法接受的。所以我们在业务SDK中，加入了重连的逻辑。

## 重连原理
```cgo
https://ms2008.github.io/2019/06/16/golang-rabbitmq/ 
https://github.com/streadway/amqp/issues/133
```
简而言之，channel.NotifyClose和connection.NotifyClose可以接收到错误消息，那就可以以此为重连的触发器。函数原型：func NotifyClose(c chan *Error) chan *Error。流程图如下：

![](/img/rabbitmq_reconnect.png)

## 结合code食用

```go
func (m *AmqpClient) reconnect() {
	for{
		//延时启动，确保消费方订阅之后，再启动重连的监听
		time.Sleep(reconnectDetectDur)
		select{
		// 生产环境中没有观察到channel close err，目前只是做了简单的日志打印
		case err := <-m.channelNotify:
		    m.notifyFunc(fmt.Sprintf("channel close notify: %v", err))
		    fmt.Printf("channel close notify: %v", err)
		case err := <-m.connNotify:
		    m.notifyFunc(fmt.Sprintf("connection close notify: %v", err))
            fmt.Printf("connection close notify: %v", err)
            m.isConnected = false
        }
 
        if !m.isConnected {
        	if m.consumerFunc != nil {
        		if err := m.consumeChannel.Cancel(m.queueName, true); err != nil {
        			fmt.Printf("channel canclel err: %v", err)
        	}
        	// 限制重连接次数
            for cnt := 0; cnt < reconnectTimes; cnt++ {
                if _, err := m.Connect(m.notifyFunc); err == nil {
                   if m.consumerFunc != nil {
                      // 消费者重连之后需要重新订阅
                   	  err := m.Subscribe(m.queueName, m.consumerFunc)
                      if err != nil {
                   	     fmt.Printf("reconnect subscribe error")
                      }
                   break   
                }
                // 重连时留一些时间间隔，后期最好可以做到指数退避算法
                time.Sleep(reconnectDelay)
            }
            break
        }
    }        
}
```

## 参考
* [Golang RabbitMQ 故障排查一例](https://ms2008.github.io/2019/06/16/golang-rabbitmq/)
* [AMQP重连机制实现 GO](https://yeqown.xyz/2019/10/10/AMQP%E9%87%8D%E8%BF%9E%E6%9C%BA%E5%88%B6%E5%AE%9E%E7%8E%B0-Go/)



