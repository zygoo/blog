---
title: "prometheus系列-快速入门"
date: 2021-11-11T10:18:06+08:00
draft: false
toc: true
images:
tags:
- prometheus
---

## 前言
对于Prometheus，我们一般有两种理解：
1. 指的是Prometheus本身，是一个时序数据库
2. 另外一种则是Prometheus生态圈，指的是整体的监控报警的生态圈和解决方案：prometheus + grafana + altermanager

Prometheus在2016年加入CNCF(Cloud Native Computing Foundation), 是继k8s之后的第二个托管项目，其主要特点如下：
1. 多维度的数据模型：由**指标名**和**键/值**，对标签标识的时间序列数据来组成多纬的数据模型
2. 灵活的查询语言：在prometheus中使用灵活的查询语言PromSQL来进行查询
3. 不依赖分布式存储：prometheus单个节点也可以直接工作，支持本地存储（TSDB）和远程存储的模式
4. 服务端采集数据：prometheus基于http pull的方式去对不同的端采集时间序列数据
5. 客户端主动推送：支持通过PushGateWay组件主动推送时间序列数据

## Prometheus生态组件
Prometheus生态由多个组件共同组成，其中许多组件是可以根据实际情况选择的，并且绝大部分由go语言编写，在部署和构建上比较方便，如下：
1. Prometheus Server：Prometheus服务器，用于收集指标和存储时间序列数据，并提供以一系列的查询和设置接口
2. Client Library： 客户端库，用于帮助需要监控采集的服务暴露metrics handler和Prometheus server。例如我们经常在gin直接调用promhttp暴露一个metrics接口
3. Push Gateway：推送网关，Prometheus服务端仅支持http pull的 采集方式，但是有一些值存在的时间短，Prometheus来之前pull就结束了。或者说该类指标，是需要客户端自行上报的，这时候就可以采用Push GateWay的方式。客户端将指标push到Push GateWay，再由Prometheus Server从Push GateWay上pull
4. Exporters：用于暴露已有的第三方服务（HAProxy，StatsD，Graphite）的metrics给Prometheus Server
5. AlterManager：用于处理告警，从Prometheus Server端接收到alters后，会进行去重，分组，然后路由到对应的receiver，发出报警
6. Support Tools：各种支持工具

## Prometheus 整体流程
Prometheus的整体架构和生态组件，如下图所示：
![](/img/prometheus_architecture.png)

* Prometheus Server通过从监控目标中拉取监控指标（或者通过推送网关来拉取）
* 它在本地存储所有抓取到的数据，并对这些数据执行一系列规则，以汇总和记录现有数据的新时间序列或者生成告警
* 通过Grafana或者其他工具来实现监控数据的可视化

## 利用时序数据库来存储Prometheus的数据
Prometheus所有采集的指标数据在默认情况下，都保存在本地所内置的时间序列数据库中（TSDB）当中。目前在行业中比较出名，流行度较高的时序数据库如下：

![](/img/time_series_db.jpeg)

时序数据库，简单来说就是将数据按照时间的顺序排列，它具有唯一性和可排序性，因此在Prometheus的Metrics中即使只添加一个标签，也会造成破坏，也就是说它不再是原来的那个时序数据了。关于这个破坏性，我们后面再来细说。

## 参考
* [Prometheus 快速入门](https://eddycjy.com/posts/prometheus/2020-05-16-startup/)



