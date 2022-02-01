---
title: "KubeVela中APIServer和Catalog架构设计"
date: 2022-02-01T17:03:17+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeVela"]
categories: ["云原生"]
draft: false
---

> 本文是对KubeVela官方文档的翻译。
>
> 原文链接：[https://github.com/oam-dev/kubevela/blob/master/design/vela-core/APIServer-Catalog.md](https://github.com/oam-dev/kubevela/blob/master/design/vela-core/APIServer-Catalog.md)

## 概要

在 KubeVela 中，APIServer 为外部系统（例如 UI）提供 RESTful API 来管理 Vela 抽象，如应用程序、定义； Catalog 存储 templates 以在 Kubernetes 上安装 common-off-the-shell (COTS) 功能。

本文档为 Vela APIServer 和 Catalog 提供了自上而下的架构设计。它阐明了平台构建者构建集成解决方案的 API 接口，并详细描述了传入路线图的架构设计。有些接口可能还没有实现，但我们会在未来的项目路线图中遵循这个设计。

## 动机

此设计基于并尝试解决以下用例：

1. UI 组件想要发现 API 以与 Vela APIServer 集成。
2. 用户希望在一个地方管理多个集群、Catalog、配置环境。
3. 管理数据可以存储在 MySQL 等云数据库中，而不是 k8s 控制平面。
   - 因为这些数据没有控制逻辑。这与在 K8s 控制平面中存储为 CR 的其他 Vela 资源不同。
   - 托管 k8s 控制平面比云上的 MySQL 数据库更昂贵。
4. 用户希望通过定义明确的标准 Catalog API 接口和包格式来管理第三方功能。
5. 以下是创建应用程序的工作流程：
   - 用户选择环境
   - 用户在环境中选择一个或多个集群
   - 用户配置副本数、实例标签、容器镜像、域名、端口号等服务部署配置。

## Proposal

### 1. 自上而下的架构

整体架构图：

![alt](https://img.jooks.cn/img/202202011154689.jpg)

以下是对图表的一些解释：

- UI 向 APIServer 请求数据以呈现仪表板。
- Vela APIServer 聚合来自不同来源的数据。
  - Vela APIServer 可以与多个 k8s 集群同步。
  - Vela APIServer 可以与多个 catalog 服务器同步。
  - 对于那些不在 k8s 或 catalog 服务器中的数据，Vela APIServer 将它们同步到 MySQL 数据库中。

上述架构意味着 Vela APIServer 可以用于多个 k8s 集群和 catalog。下面是 Vela 平台的部署情况：

![alt](https://img.jooks.cn/img/202202011158585.png)

### 2. API 设计

下面是API分组和存储的整体架构：

![alt](https://img.jooks.cn/img/202202011200178.jpg)

有两个不同的层：

- **API 层**：它定义了 Vela APIServer 实现必须遵循的 API 发现和服务端点。这是外部系统组件（例如 UI）联系的集成点。
- **存储层**：描述 Vela APIServer 在后台同步数据的存储系统和对象。存储方式分为三种：
  - **K8s 集群**：Vela APIServer 管理多个关于应用程序和定义自定义资源的 k8s 集群。
  - **Catalog服务器**：Vela APIServer 管理包含 COTS 应用程序包的多个 Catalogs。目前在我们的用例中，Catalogs 位于 Git 存储库中。将来我们可以将其扩展到其他 Catalogs 存储，如文件服务器、对象存储。
  - **MySQL 数据库**：Vela APIServer 将全局、跨集群、跨 Catalog 信息存储在 MySQL 数据库中。这些数据不存在于 k8s 或 Catalog 中，因此需要由 APIServer 在单独的数据库中进行管理。数据库通常托管在云上。

#### Environment API

Environment 是应用程序和依赖资源的配置集合。例如，开发人员将定义 `prod` 和 `stage` 环境，其中每个环境都有不同的路由、缩放、数据库连接凭据等配置。

- ```
  /environments
  ```

  - 描述：所有的环境

    ```
    [{"id": "env-1"}, {"id": "env-2"}]
    ```

- ```
  /environments/<env>
  ```

  - 描述: 某个环境的CRUD信息

    ```
    {
      "id": "env-1",
    
       // The clusters bound to this environment
      "clusters": [
        {
          "id": "cluster-1"
        }
      ],
    
      "config": {
        "foo": "bar"
      }
    }
    ```

#### Cluster API

略

#### Catalog API

略

### 3. Catalog 设计

本节将描述目录结构的设计，它如何与APIServer交互，以及工作流用户安装包。

#### Catalog 架构

所有包都放在 `/catalog/` dir下面（这是可配置的）。目录结构遵循预定义的格式：

```
/catalog/ # a catalog consists of multiple packages 
|-- <package>
    |-- v1.0 # a package consists of multiple versions
        |-- metadata.yaml
        |-- definitions/
            |-- xxx-workload.yaml
            |-- xxx-trait.yaml
        |-- conditions/
            |-- check-crd.yaml
        |-- hooks/
            |-- pre-install.yaml
        |-- modules.yaml
    |-- v2.0
|-- <package>
```

一个包版本的结构包含：

- `metadata.yaml`: 包的元数据。

  ```yaml
  name: "foo"
  version: 1.0
  description: |
    More details about this package.
  maintainer:
  - "@xxx"
  license: "Apache License Version 2.0"
  url: "https://..."
  label:
    category: foo
  ```

- `definitions`: 定义文件，描述此包中要在集群上启用的功能。 请注意，这些定义将与 APIServer 端的集群进行比较，以查看集群是否可以安装或升级此包。

- `conditions/`: 在部署此包之前定义条件检查。 例如，检查是否存在具有特定版本的 CRD，如果不存在，则部署应该失败。

  ```yaml
  # check-crd.yaml
  conditions:
  - target:
      apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: test.example.com
      fieldPath: spec.versions[0].name
    op: eq
    value: "v1"
  ```

- `hooks/`: 部署此包的生命周期挂钩。 这些是 k8s 作业，包括安装前、安装后、卸载前、卸载后。

  ```yaml
  # pre-install.yaml
  pre-install:
  - job:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
            - name: pre-install-job
              image: "pre-install:v1"
              command: ["/bin/pre-install"]
  ```

- `modules.yaml`: 定义包含实际资源的模块，例如 Helm Charts 或 Terraform 模块。 请注意，我们选择与现有的社区解决方案集成，而不是发明我们自己的格式。通过这种方式，我们可以使用社区的库，并使设计可扩展到更多的内部格式。

  ```yaml
  modules:
  - chart:
      path: ./charts/ingress-nginx/ # local path in this package
      remote:
        repo: https://kubernetes.github.io/ingress-nginx
        name: ingress-nginx
  - terraform:
      path: ./tf_modules/rds/ # local path in this package
      remote:
        source: terraform-aws-modules/rds/aws
        version: "~> 2.0"
  ```

#### 在 APIServer 中注册 Catalog

请参考上面的`/catalogs/<catalog>` API。

在后台，APIServer 将根据预定义的结构扫描目录存储库以解析每个包和版本。

#### 在 APIServer 中同步 Catalog

请参考上面的 `/catalogs/<catalog>/sync` API。

在后台，APIServer 将重新扫描目录。

#### 从 Catalog 下载包

Vela APIServer 聚合来自多个目录服务器的包信息。 要下载一个包，用户首先请求 APIServer 找到目录和包的位置。 然后用户直接访问目录 repo 以下载包数据。 工作流程如下图所示：

![alt](https://img.jooks.cn/img/202202011831691.jpg)

在我们未来的路线图中，我们将为每个 k8s 集群构建一个 Catalog 控制器。然后我们将添加 API 端点以在 APIServer 中安装包，这基本上创建了一个 CR 以触发控制器将包安装协调到集群中。 我们选择这个而不是 APIServer 安装包，因为这样我们可以绕过包数据传输路径中的 APIServer，避免 APIServer 成为单点故障。

## 注意事项

### 封装参数

我们可以从 Helm Chart 或 Terraform 解析参数的模式。 例如，Helm 支持用于输入验证的值模式文件，并且有一个自动化工具来生成模式。

### 包依赖

我们可以定义一个包只与一个定义相关，而不是在一个包中有多个定义。 但是有些用户需要一组定义而不是一个。 例如，一个完全可观察性特征可能包括一个 prometheus 包、一个 grafana 包、一个 loki 包和一个 jaeger 包。 这就是我们所说的“打包捆绑”。

为了提供一组定义，我们可以定义包依赖。 因此，父包可以依赖多个原子包来提供完整的功能。

包依赖解决方案将简化结构并提供更多原子包。 但这不是一个简单的问题，超出了目前的范围。 我们将在未来的路线图中添加这一点。

### 多租户

对于初始版本，我们计划在没有多租户的情况下实现 APIServer。 但作为一个应用平台，我们期望多租户是 Vela 的必要组成部分。 我们将保持 API 兼容性，并且将来可能会添加某种身份验证令牌（例如 JWT）作为查询参数。

