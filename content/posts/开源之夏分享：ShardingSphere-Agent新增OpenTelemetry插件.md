---
title: "开源之夏分享：ShardingSphere-Agent新增OpenTelemetry插件"
date: Mon Aug 02 01:36:04 CST 2021
categories: ["中间件"]
tags: ["中间件"]
draft: false
---

> 个人介绍
>
> 大家好，我是jooks，一个来自浙江的准大三学生。技术栈上对Java和数据库比较感兴趣。
>
> 我在这次开源之夏所承担的任务是：shardingsphere-Agent 新增OpenTelemetry插件
>
> 这篇文章整理了我在做该任务过程中所接触的技术。

## 目录

- ShardingSphere简介
- ShardingSphere-Porxy简介
- 分布式可观测性简介
- OpenTelemerty简介
- 分布式链路追踪（Traces）
- Java Agent技术
- ShardingSphere的agent

## ShardingSphere简介

> 官网搬运 ^_^

Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

![shardingsphere-scope_cn](https://img.jooks.cn/img/20210802093038.png)

## ShardingSphere-Porxy简介

Proxy是SS的一款产品，也是我个人最喜欢的。把配置文件写好之后，启动proxy，然后你就可以像使用一个数据库一样，来使用数据库集群。

举个例子，如果你把proxy开在本机的13307端口，那么你只需要在终端输入`mysql -u root -P 13307 -h 127.0.0.1 -p`即可直接跟proxy交互。你在这里输入的sql将会被proxy加工、计算，然后打到真实的数据库上。

![image-20210801135639263](https://img.jooks.cn/img/20210801135639.png)

## 分布式可观测性简介

可观测性 (Observability) 的三种数据模型：追踪 (Traces)、度量 (Metrics)、日志 (Logs)。

> PS: 这三种数据模型不是相互独立的，之间有一定联系。

![image-20210801140227276](https://img.jooks.cn/img/20210801140227.png)

Traces：在微服务时代，追踪不只是单体程序调用栈的追踪；一个外部请求导致的内部服务的调用轨迹的收集，更是它的重要内容。因此，分布式系统中的追踪在国内通常被称为“全链路追踪”，许多资料中也把它叫做是“分布式追踪”（Distributed Tracing）。

Metrics：度量是指对系统中某一类信息的统计聚合。这里的信息可以包括：JVM、CPU、数据库连接池等里面的具体指标。

Logs：日志我们应该都知道。

这些数据的产生可能不难，但是把这些信息收集起来，并按照一定规则展示，可能是非常复杂的。

## OpenTelemerty简介

OpenTelemetry（简称OTel）是2019年，由CNCF的OpenTracing和谷歌主导的OpenCencus合并而来。OpenTelemerty本身不负责数据的存储、展示等，自身定位是：数据采集和标准规范的统一。

OpenTelemetry涵盖了分布式可观测性 (Observability) 的三大部分：追踪 、度量、日志。

个人理解，OTel的作用可以一句话概括：收集服务的数据，然后发送到后端。

收集：OTel提供了一系列的API和SDK可以在服务中使用。

后端：一些存储和分析的工具，比如Zipkin、Jaeger、Prometheus等。

![image-20210801142650737](https://img.jooks.cn/img/20210801142650.png)

## 分布式链路追踪（Traces）

>  因为我这次开源之夏的任务是把traces信息用OpenTelemetry导出出来，所以这里来详细介绍一下traces。

现代分布式链路追踪公认的起源，应该是 Google 在 2010 年发表的论文《[Dapper : a Large-Scale Distributed Systems Tracing Infrastructure](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf)》，这篇论文介绍了 Google 从 2004 年开始使用的分布式追踪系统 Dapper 的实现原理。

traces的存在可以在出现故障时快速定位问题所在，提高研发效率。

一个trace，可以看成是一棵树的结构，其中节点是一个个Span，然后Span之间有关联，如下图。

![image-20210801143752961](https://img.jooks.cn/img/20210801143752.png)

在Otel中，一个Span封装了以下内容：

- 名称
- 开始和结束的时间戳
- 属性（K-V结构）
- 事件，零个或多个，是一个元组结构
- 父SpanID
- 链接
- SpanContext，表示在Trace中标识Span的所有信息，并且必须传播到子 Span 并跨越进程边界。里面包含了TraceId、SpanId等信息。

Otel对traces的收集，可以用一张图来概括。（来自腾源会 SkyWalkingDay）

![image-20210712172402395](https://img.jooks.cn/img/20210801144104.png)

Traces的采样策略有以下几种：

1. 头部采样。

   在网关或者前端确定这条链路是要采样的。

2. 尾部采样。

   将整个系统的数据缓存下来，到了尾部决定是否要采样上传。这种注定内存开销是很大的。

3. 单元采样。

   由每个采样点决定是否采样上传。比如某个点超时了，不管上下游是否决定采样，直接在这个点进行采样。

4. 多维度染色采样。

   指定某个信息（比如某用户或文章）进行采样。

而traces的收集方式又有以下几种：

1. 基于日志的追踪
2. 基于服务的追踪
3. 基于边车代理的追踪

而ShardingSphere使用的应该是头部采样策略，且是基于服务的追踪。

服务追踪的实现思路是通过某些手段给目标应用注入追踪探针，而针对 Java 应用，一般就是通过 Java Agent 注入的。

## Java Agent技术简介

如果我们要无侵入地添加一些可选的功能，而不是与原有的代码放在一起，java agent是一个比较好的选择。

“无侵入”很容易让人想到还有一种比较流行的方案，就是使用aop。但是无论是Spring AOP还是AspectJ，跟agent比起来还是没那么彻底，至少前者还需要跟原有的程序一起打包。

### 那么什么是Java Agent呢？

举一个小例子，首先写好主程序，比如这样。

![image-20210801213351932](https://img.jooks.cn/img/20210801213351.png)

然后将其打成jar包。

![image-20210802005023419](https://img.jooks.cn/img/20210802005023.png)

然后执行`java -jar MainClass.jar`

终端会输出`Hello World!`

然后写好要注入的agent程序。

![image-20210801221108926](https://img.jooks.cn/img/20210801221108.png)

在打包之前需要修改一下maven的配置，来自动生成需要的MANIFEST.MF文件（也可以手动写）。

![image-20210801221210307](https://img.jooks.cn/img/20210801221210.png)

打成jar包后，输入`java -javaagent:/Users/jooks/agent/agent-instr/target/AgentClass.jar -jar MainClass.jar`即可触发魔法，如下图所示。

![image-20210801221040116](https://img.jooks.cn/img/20210801221040.png)

实际上，java agent还可以动态注入程序，也就是在程序运行过程中注入。

另外，`premain`方法里面的参数`instrumentation`是一个非常重要的参数，配合ASM、Javassist等工具，实现`ClassFileTransformer`接口，可以实现对原字节码文件的修改或者替换。ShardingSphere-Agent模块的实现就是利用了字节码框架ByteBuddy。

### Java Agent的原理

这里只介绍前面演示的启动前注入。

实际上，当我们使用参数启动`-javaagent`时，`InvocationAdapter.c`里面的`DEF_Agent_OnLoad`函数会被调用。而`Agent_OnLoad`做了如下事情：

1. 创建并初始化JPLISAgent
2. 监听VMInit，当vm完成初始化时：
   - 创建InstrumentationImpl对象
   - 监听ClassFileLoadHook事件（在字节码文件加载时，调用`ClassFileTransformer#transform`，实现字节码的修改）
   - 调用InstrumentationImpl的`loadClassAndCallPremain`方法，在这个方法里会调用javaagent里MANIFEST.MF里指定的`Premain-Class`类的premain方法
3. 解析javaagent里MANIFEST.MF里的参数，设置JPLISAgent

画一张图：

![未命名文件 (1)](https://img.jooks.cn/img/20210802000709.jpg)

## ShardingSphere的agent

ShardingSphere-Agent 是独立自主设计，基于`Bytebuddy`字节码增加的项目，基于插件化的设计，可以无缝隙的与ShardingSphere集成，目前有提供 log, metrics, traces 等可观测性功能，接下来还将实现使用distSQL来查询运行时的数据。（插件化的设计很漂亮～具体可以看我之前写的这篇文章：[https://www.jooks.cn/article/56](https://www.jooks.cn/article/56)）

![image-20210802002904042](https://img.jooks.cn/img/20210802002904.png)

如下图所示，ShardingSphere通过agent方式设置拦截点，接入apm系统。目前做的链路追踪是在几个地方：执行命令、解析sql、用jdbc执行sql。

![image-20210802003316104](https://img.jooks.cn/img/20210802003316.png)

---

over~

我的微信号：`jooks-me`，欢迎加好友交流\^o\^


