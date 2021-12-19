---
title: "OpenTelemetry和分布式链路追踪小调研"
date: Mon Jul 12 17:57:37 CST 2021
categories: ["中间件"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["中间件"]
draft: false
---

## 云原生可观测性

可观测性有三种数据模型：Metrics、Logs、Traces

> PS: 这三种数据模型不是相互独立的，之间有一定联系，比如可以从Metrics跳转到Traces，也可以从Logs跳转到Traces。

## OpenTelemetry的历史

| 标准          | 概述                                      | Traces | Metrics | Logs |   状态   |
| ------------- | ----------------------------------------- | ------ | ------- | ---- | :------: |
| OpenTracing   | 2015年发起，2016年进入CNCF                | 支持   |         |      | 停止维护 |
| OpenCencus    | 2017年发起，负责人来自谷歌、微软          | 支持   | 支持    |      | 停止维护 |
| OpenTelemetry | 2019年，由OpenTracing和OpenCencus合并而来 | 支持   | 支持    | 支持 | 正常维护 |

![image-20210712160814695](https://img.jooks.cn/img/20210712160814.png)

OpenTelemetry是为了统一目前可观察性的规范。

## OpenTelemetry功能

一句话概括Telemetry的作用：创建并收集服务的数据，然后发送到后端。

创建并收集：Telemetry提供了一系列的API和SDK来在服务中使用。

后端：一些存储和分析的工具，比如Zipkin、Jaeger、Prometheus等。

## 架构

按照是否使用OTEL Collector可以分成两种架构。

首先是不使用Collector，这种可能跟使用OpenTracing类似。

![image-20210712163609516](https://img.jooks.cn/img/20210712163609.png)

然后是使用Collector。Collector的存在可以对收集的数据进行预先处理，提高观测系统的效率。

![image-20210712165110829](https://img.jooks.cn/img/20210712165110.png)

But，也许你会看到这么一张图。

![image-20210712165159999](https://img.jooks.cn/img/20210712165200.png)

事实上，OT提供了两种运行Collector的方式，一种是 **Agent** ，一种是 **Gateway** 。

Agent大家应该都知道是可以无侵入地注入到应用中，跟应用一起运行。

而这个Gateway方式是什么鬼呢，按官网的说法应该是指一个独立运行的服务。我上面画的Collector就是指后者，因为我在使用过程中感受不到前者的存在（雾）。

## OpenTelemtry的组成

从前面的架构图大概可以看出来，

OT包含了：API package、SDK package、Semantic Conventions package、Plugin package。

API package：略

SDK package：对API的实现，可以自己去写SDK

Semantic Conventions package：语义规范

Plugin package：插件建设

## Traces

traces的存在可以在出现故障时快速定位问题所在，提高研发效率。

一个trace，可以看成是一棵树的结构，其中节点是一个个Span，然后Span之间有关联，如下图。

![image-20210712172620332](https://img.jooks.cn/img/20210712172620.png)

OT对traces的收集如下图所示（图片来自腾讯云 X 腾源会 X Skywalking）。

![image-20210712172402395](https://img.jooks.cn/img/20210712172402.png)

#### Trace Detail

其中，Trace Detail是指Span，包含了traceId, spanId, traceState, parentSpanId, name, startTime, endTime, attributes, events, links, status。

其中attributes是一个键值对的形式，有点类似OpenTracing里面的Tags；

events是内嵌日志；

links可以链接其它Trace，实现一些特殊功能。

#### Trace Context

Trace context实际上是遵循了W3C的格式规范，如下图，下面的规范都是兼容的。

![image-20210712174601789](https://img.jooks.cn/img/20210712174601.png)

context内容如下：

![截屏2021-07-12 17.38.29](https://img.jooks.cn/img/20210712174107.png)

![image-20210712174815317](https://img.jooks.cn/img/20210712174815.png)

## Traces采样策略

1. 头部采样。

   在网关或者前端确定这条链路是要采样的。

2. 尾部采样。

   将整个系统的数据缓存下来，到了尾部决定是否要采样上传。这种注定内存开销是很大的。

3. 单元采样。

   由每个采样点决定是否采样上传。比如某个点超时了，不管上下游是否决定采样，直接在这个点进行采样。

4. 多维度染色采样。（腾讯内部流传？）

   指定某个信息（比如某用户或文章）进行采样。

   

> Metrics以及Log，有时间再更。


