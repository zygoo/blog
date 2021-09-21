---
title: "RabbitMQ的Channel池"
date: 2021-09-15T12:18:09+08:00
draft: false
toc: true
images:
tags:
- mq
---

## Issue
"channel id space exhausted"，部分消息无法进行正常地发布，这便是我们在生产环境中遇到的问题。虽然对connection进行了复用，
但在高并发时，channel id还是耗尽了。在当前的AMQP的releases版本中，打开channel的最大默认值是2047。基于这个情况，我们决定复用channel, 构造一个channel pool。

## 方案预研
### 开源库
Java的组件中，早就引入了channel pool的概念。所以刚开始的思路，肯定是去找开源的库。
[houseofcat/turbocookedrabbit](https://github.com/houseofcat/turbocookedrabbit)是golang中star较高的一个库，
但是看了源码之后，发现其并没有re-use channel, 跟我们的建设思路严重不符，遂抛弃之。对这个库有兴趣的朋友，可以看一下。

### sql.driver的连接池
golang的"database/sql"是操作数据库时常用的包，这个包定义了一些sql操作的接口，具体的实现还是需要不同的数据库来实现。
"database/sql"除了定义接口之外还有一个重要的功能：连接池，在其他网络通信时可以借鉴其实现。
我们RabbitMQ的channel pool就是参考database connection pool来实现的，可以说是超级简化版的pool。虽然说channel和connection是完全不一样的概念，但是不妨碍我们学习上层的封装。

## channel pool的实现
### 流程图
![](/img/rabbitmq_channel_pool.jpg)

### getChannelFromPool
```go
func (m *AmqpClient) getChannelFromPool() (*channelHost, error) {
	m.mu.Lock()
	numFree := len(m.freeChannel)
	if numFree > 0 {
		ch := m.freeChannel[0]
		copy(m.freeChannel, m.freeChannel[1:])
		m.freeChannel = m.freeChannel[:numFree-1]
		m.mu.Unlock()
		return ch, nil
	}

	if m.numOpenChannel >= m.maxChannel {
		req := make(chan channelRequest)
		reqKey := m.nextRequestsKeyLocked()
		m.channelRequests[reqKey] = req
		m.mu.Unlock()
		ret, ok := <-req
		if !ok {
			return nil, errors.New("error request")
		}
		return ret.cHost, nil
	}

	m.numOpenChannel++
	channel, err := m.conn.Channel()
	if err != nil {
		m.numOpenChannel--
		m.mu.Unlock()
		return nil, errors.New("create channel error")
	}
	return &channelHost{channel: channel, isUse: false}, nil
}
```
1. 首先通过len(m.freeChannel)判断pool中是否有空闲的channel, 如果存在空闲的channel, 直接返回给client，用于发布消息
2. numOpenChannel已经建立或者等待建立的channel数，如果numOpenChannel>maxChannel, 证明当前的存在的channel数已经到达设置的上限，client的请求被加入等待队列中，阻塞等待
3. 创建一个channel，首先将numOpenChannel加1，然后再创建连接。如果创建失败，将numOpenChannel减1

### returnChannel
```go
func (m *AmqpClient) returnChannel(cHost *channelHost) {
	m.mu.Lock()
	defer func() {
		m.mu.Unlock()
	}()

	c := len(m.channelRequests)
	if c > 0 {
		var req chan channelRequest
		var reqKey uint64
		for reqKey, req = range m.channelRequests {
			break
		}
		delete(m.channelRequests, reqKey)
		cHost.isUse = true
		req <- channelRequest{
			cHost: cHost,
		}
	}

	if m.maxIdleChannel > len(m.freeChannel) {
		m.freeChannel = append(m.freeChannel, cHost)
	} else {
		err := cHost.channel.Close()
		if err != nil {
			return
		}
	}
	m.numOpenChannel--
}
```
1. len(m.channelRequests)>0, 证明队列中有等待的请求，channel re-use应该优先考虑等待队列
2. 注意，代码中使用map作为等待队列，range方法帮助我们随机选择一个等待的请求
3. m.maxIdleChannel > len(m.freeChannel)， 说明channel pool中连接池还有剩余空间，这时候可以将channel归还到pool
4. 如果channel pool已经满了，直接close该channel

## 参考
* [golang sql连接池的实现方法详解](https://www.jb51.net/article/147071.htm)

