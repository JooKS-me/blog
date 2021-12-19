---
title: "Apache Shenyu源码阅读计划（一）初印象"
date: Sat Jun 12 10:13:55 CST 2021
categories: ["中间件"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["中间件"]
draft: false
---

## 什么是Shenyu

Shenyu是一个API网关。

## 架构图

搬运官网的架构图 : )

> 目前改名尚未完全完成，图中还是soul，下面都替换成shenyu。

![soul-framework](https://img.jooks.cn/img/20210612090608.png)

从架构图上看，用户可以通过Http访问shenyu集群（因此可以说是跨语言的），来实现shenyu的一些操作。

根据上图作出如下 **猜测**：

- Shenyu Cluster：Shenyu 集群支持Zookeeper、Nacos、Http、Websocket等多种集群配置管理方式，来对shenyu单体的本地缓存做统一管理。
- Plugins：shenyu通过插件的形式来实现对其他微服务应用的支持。
- Shenyu-Admin：用户可以在这上面进行各种配置。配置完成后，会持久化进MySQL或者H2。且通过异步的方式告知注册中心。

## 跑Demo

这里先跑一下最纯净的Http吧。。。

> 参照官网文档：[https://dromara.org/zh/projects/soul/quick-start-http/](https://dromara.org/zh/projects/soul/quick-start-http/)

确实写的很「快速开始」。。。

因为application.yml写好了shenyu的配置（如下图），所以在启动时会自动往shenyu写好Selector。且controller中带了`@ShenyuSpringMvcClient`注解的接口方法，会自动往shenyu写规则。然后直接打开admin的divide插件就能看到配置了。

![image-20210612100644487](https://img.jooks.cn/img/20210612100644.png)

![image-20210612100708778](https://img.jooks.cn/img/20210612100708.png)

然后请求shenyu的9195端口，求能访问到example的api了。

![image-20210612100758171](https://img.jooks.cn/img/20210612100758.png)




