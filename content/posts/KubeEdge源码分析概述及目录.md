---
title: "KubeEdge源码分析概述及目录"
date: 2022-02-02T14:03:17+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeEdge"]
categories: ["云原生"]
draft: false
---

之前的一篇 [Kubeedge源码分析汇总](https://www.jooks.cn/posts/kubeedge%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E6%B1%87%E6%80%BB/) ，汇总了之江实验室几年前的源码分析文章。

现在打算自己读一读源码，顺便把相关的分析写成文章。

于是就在这里开一个目录。

下面是KubeEdge的架构图。

![KubeEdge 架构](https://img.jooks.cn/img/202202021446700.png)

KubeEdge大体分成了Cloud侧和Edge侧，其中：

- **[Edged](https://kubeedge.io/zh/docs/architecture/edge/edged):** 在边缘节点上运行并管理容器化应用程序的代理。
- **[EdgeHub](https://kubeedge.io/zh/docs/architecture/edge/edgehub):** Web套接字客户端，负责与Cloud Service进行交互以进行边缘计算（例如KubeEdge体系结构中的Edge Controller）。这包括将云侧资源更新同步到边缘，并将边缘侧主机和设备状态变更报告给云。
- **[CloudHub](https://kubeedge.io/zh/docs/architecture/cloud/cloudhub):** Web套接字服务器，负责在云端缓存信息、监视变更，并向EdgeHub端发送消息。
- **[EdgeController](https://kubeedge.io/zh/docs/architecture/cloud/edge_controller):** kubernetes的扩展控制器，用于管理边缘节点和pod的元数据，以便可以将数据定位到对应的边缘节点。
- **[EventBus](https://kubeedge.io/zh/docs/architecture/edge/eventbus):** 一个与MQTT服务器（mosquitto）进行交互的MQTT客户端，为其他组件提供发布和订阅功能。
- **[DeviceTwin](https://kubeedge.io/zh/docs/architecture/edge/devicetwin):** 负责存储设备状态并将设备状态同步到云端。它还为应用程序提供查询接口。
- **[MetaManager](https://kubeedge.io/zh/docs/architecture/edge/metamanager):** Edged端和Edgehub端之间的消息处理器。它还负责将元数据存储到轻量级数据库（SQLite）或从轻量级数据库（SQLite）检索元数据。

K8S拉起应用

![image-20220202145609335](https://img.jooks.cn/img/202202021456361.png)

KubeEdge拉起应用

![image-20220202145444323](https://img.jooks.cn/img/202202021454352.png)

对比两幅图，可以发现KubeEdge其实是对Kubelet的相关行为做了等价替换，来适配边缘场景。

## 目录

1. CloudCore启动流程分析
2. EdgeCore启动流程分析
3. KubeEdge源码分析之Edged
4. KubeEdge源码分析之CloudHub
5. KubeEdge源码分析之EdgeHub
6. KubeEdge源码分析之EventBus
7. KubeEdge源码分析之DeviceTwin
8. KubeEdge源码分析之MetaManager

