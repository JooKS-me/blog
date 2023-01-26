---
title: "Dubbo Reactive Stream Proposal（开源之夏2022项目申请书）"
date: 2022-06-17T00:12:30+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["OSPP","Dubbo","OpenSource Internship"]
categories: ["OpenSource Internship"]
draft: false
---

## 项目详情

项目名称：Dubbo Reactive Stream 支持

项目编号：22a7f0101

项目导师：xxxx

项目链接：https://summer-ospp.ac.cn/#/org/prodetail/22a7f0101

申请学生：朱坤帅（[jookunshuai@gmail.com](mailto:jookunshuai@gmail.com), https://github.com/JooKS-me）

## 项目背景

Triple 协议为了应对大规模数据传播的场景，实现了 Streaming RPC 的支持。使用 Triple 协议的 Streaming RPC 方式，可以在consumer跟provider之间建立多条用户态的长连接，Stream。同一个TCP连接之上能同时存在多个 Stream ，其中每条 Stream 都有 StreamId 进行标识，对于一条 Stream 上的数据包会以顺序方式读写。[1]

![image-20220520212746267](https://img.jooks.cn/img/202205202127292.png)

Reactive Stream 提供了一套标准的异步流处理 API， 在能够让应用写出事件驱动的程序的同时，也通过 backpressure（反压）的方式保证了节点的稳定。Stream RPC 结合 Reactive API 实现反压策略可以最大程度保证处理大规模流数据时不会产生 Buffer Overflow。[2]

用户可以通过 compiler 将编写的 proto 文件编译为程序代码，默认会生成几种不同的 stub。stub 用一种统一的方式帮我们屏蔽和统一了不同调用方式的细节，使用户使用起来更加简单。

如果能使用 Dubbo 的 compiler 根据 proto 文件生成支持反压策略的 Reactive Stream Stub API，使得用户可以完全透明地使用 Reactive Stream API 来操作流，那么将会给用户带来非常方便的流式使用方式以及全链路异步性能提升。

## 前期调研

1. 阅读了 Reactor [官方指导](https://projectreactor.io/docs/core/release/reference/) 和 相关源码，对 Reactor、Reactive Stream API 以及 backpressure 的使用和实现原理有了一定的了解。（之前在参与其他开源项目时，对reactor有过接触）
2. 阅读了 Dubbo 中 Triple 协议的调用过程的相关代码，并将代码走读过程记录到了我的博客：https://www.jooks.cn/posts/dubbo3%E4%B9%8Btriple%E5%8D%8F%E8%AE%AE%E5%90%AF%E5%8A%A8%E5%8F%8A%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B/
2. 阅读了 reactive-grpc 项目的相关代码：https://github.com/salesforce/reactive-grpc

## 项目详细方案

### 实现目标

在原有 StreamObserver 调用方式之上，封装一层 Reactive API，对用户透明。用户通过 Dubbo Compiler 生成代码，即可在客户端和服务端使用 Reactive API 操作流。并且在封装的 Reactive API 中，实现对反压的支持。

> 以服务端流的改造为例

定义proto文件如下：

```protobuf
syntax = "proto3";

package demo;

option java_multiple_files = true;

service Numbers {
    rpc tellSomeNumber (Message) returns (stream Message) {}
}

message Message {
    int32 number = 1;
}
```

利用 Dubbo Compiler 的 ReactorTripleGenerator 根据 proto 文件，生成 stub 代码。

服务端继承 stub 接口，实现相关方法。

```java
public final class ReactorStreamDemo {
  	private static class DemoServiceImpl extends DemoServiceImplBase {
        // 产生10个数字
        @Override
        public Flux<Message> tellSomeNumber(Mono<Message> request) {
            return Flux
                    .range(1, 10)
                    .map(i -> Message.newBuilder().setNumber(i).build());
        }
    }
  
    /**
      * 开启生产者服务
      */
    private static void startServer() {
    		ServiceConfig<DemoService> service = new ServiceConfig<>();
        service.setInterface(DemoService.class);
        service.setRef(new DemoServiceImpl());
        DubboBootstrap bootstrap = DubboBootstrap.getInstance();
        bootstrap.application(new ApplicationConfig("tri-stub-server"))
                .registry(new RegistryConfig(TriSampleConstants.ZK_ADDRESS))
                .protocol(new ProtocolConfig(CommonConstants.TRIPLE, port))
                .service(service)
                .start();
    }
  
    /**
      * 开启消费者服务
      */
    private static DemoService startClient() {
        DubboBootstrap bootstrap = DubboBootstrap.getInstance();
        ReferenceConfig<DemoService> ref = new ReferenceConfig<>();
        ref.setInterface(DemoService.class);
        ref.setProtocol(CommonConstants.TRIPLE);
        ref.setProxy(CommonConstants.NATIVE_STUB);
        ref.setTimeout(3000);
        bootstrap.application(new ApplicationConfig("tri-stub-consumer"))
          			.registry(new RegistryConfig(TriSampleConstants.ZK_ADDRESS))
                .reference(ref)
                .start();
        return ref.get();
    }
  
    /**
      * 主函数
      */
    public static void main(String[] args) {
        startServer();
        DemoService service = startClient;
        service.tellSomeNumber(Mono.just(Message.getDefaultInstance()))
          		  .map(Message::getNumber)
                .subscribe(System.out::println);
    }
}
```

### 实现步骤

服务端流（One To Many）的结构图如下：

![image-20220522004207006](https://img.jooks.cn/img/202205220042039.png)

1. 定义一个 DubboPublisher ，实现 Publisher、Subscription 和 StreamObserver 接口。分别实现 subscribe、request/cancel、onNext/onError/onCompleted 方法。

2. 定义一个 DubboSubscriber，实现 Subscriber 接口，将 responseStreamObserver 绑定为下游，实现 onNext、onSubscribe、onError、onComplete 等方法。其中的 onNext 方法将调用 responseStreamObserver 的 onNext 方法。

3. 修改 Dubbo Compiler ，新增 ReactorTripleGenerator，支持生成 Dubbo Reactor API 的 stub 代码。在自动生成的 stub 中，将客户端的调用引入到 DubboPublisher 的相关调用，将服务端的逻辑引入 DubboSubscriber。

其他模式是类似的，总共需要实现四种调用：

Unary 模式（One To One，Mono -> Mono）

Server Stream 模式（One To Many，Mono -> Flux）

Client Stream 模式（Many To One，Flux -> Mono）

Bi-Stream 模式（Many To Many，Mono -> Mono）

其中，Dubbo Stream Reactor API 中的 Flux 代替了原始的流，Mono 代替了普通的参数。

### 反压策略实现

> 在 Reactive API 中，当 Subscriber 订阅 org.reactivestreams.Publisher 时，发布者将 Subscription 交给 Subscriber。 当 Subscriber 想要来自 Publisher 的更多消息时，Subscriber 可以调用 Subscription#request(long)。

反压是指下游消费端可以给上游生产端一个反馈，从而控制上游的生产速率，来实现流控。这里暂时不考虑 Triple 层面实现的反压策略。

以服务端流为例，为了实现流控，下游的客户端增加两个可配置的参数：prefetch、lowTide，其中 prefetch 一般大于等于 lowTide。

首先，我们调用 ClientStreamObserver 的 disableAutoRequest，启动手动的流控，这样每次调用 CallStreamObserver 的 request 方法时服务端才会有数据传递到下游客户端。

在客户端侧实现的 DubboPublisher 中，设置一个标志 `REQUESTED` 用来表示 DubboPublisher 当前收到下游 Reactor API 请求的个数。并实现 Subscription 的 request 方法来给下游 Reactor API 调用，request 方法接受一个参数 n，每次调用时给 `REQUESTED` 加上 n。当第一次调用时，根据设置的 prefetch 参数调用 CallStreamObserver 的 request，请求上游的服务端通过网络下发数据。

![image-20220603151131818](https://img.jooks.cn/img/202206031511881.png)

上游服务端收到请求后，通过将会连续地通过 DubboPublisher 实现的 onNext 回调方法（来自 StreamObserver 接口）来传递 prefetch 个流数据。在 DubboPublisher 的 onNext 方法中，我们首先把收到的数据存入一个队列中；然后在判断当前已经给下游消费者发送的数据有多少个（sent），再结合下游请求的个数（REQUESTED），从而决定是否要向下游继续发送数据。若发送，将之前缓存数据的队列出列一个元素，然后通过下游消费者设置的回调函数 onNext 传递数据，并为 sent 加上1。

> 这里使用队列是为了防止一种特殊情况：当上游对 DubboPublisher 的 request 不起作用时，可以先将服务端发来的数据入队缓存；当下游 request 时再发给下游。

![image-20220603151744504](https://img.jooks.cn/img/202206031517523.png)

如果发现了已经发送的数据（sent）与我们之前设置的 lowTide 相等，那么说明目前服务端发送的数据快要不够用了，并且当前客户端的 buffer 空间还是比较充裕的。这时，我们向服务侧请求 low_tide 个数据，并将 sent 置为 0。

![image-20220603160927709](https://img.jooks.cn/img/202206031609742.png)

这样，我们再根据客户端 buffer 的实际情况设置好合适的 prefetch 和 low_tide 的数值，就可以实现一种“推-拉混合”的反压模式。

### 后续

1. 在 [Dubbo Sample](https://github.com/apache/dubbo-samples) 项目中编写使用样例和集成测试
2. 在 [官网仓库](https://github.com/apache/incubator-shenyu-website) 提交使用文档

## 项目开发时间规划

|      时间段      |                     工作                     |
| :--------------: | :------------------------------------------: |
| 6月5日～6月11日  | 阅读 Reactor 源码，深入学习 Reactive Stream  |
| 6月12日～6月18日 |   阅读 Grpc 相关文档、源码，深入学习 Grpc    |
| 6月19日～6月30日 |            系统地阅读 Dubbo 源码             |
| 7月3日～7月23日  |   使用 Reactor，完成 One To Many 相关编码    |
| 7月24日～7月30日 |    使用 Reactor，完成 One To One 相关编码    |
| 7月31日～8月6日  |   使用 Reactor，完成 Many To One 相关编码    |
| 8月7日～8月13日  |   使用 Reactor，完成 Many To Many 相关编码   |
| 8月14日～8月20日 | 完成样例和集成测试的编写，以及可能的 Bug Fix |
| 8月21日～8月27日 |                完成文档的编写                |
| 9月1日～9月20日  |                   代码优化                   |
| 9月2日～9月30日  |                 完成结项报告                 |
|   9月30日之后    |        继续给 Apache Dubbo 社区做贡献        |

## 参考

[1] https://dubbo.apache.org/zh/docs/concepts/rpc-protocol/#triple-streaming

[2] https://summer-ospp.ac.cn/#/org/prodetail/22a7f0101
