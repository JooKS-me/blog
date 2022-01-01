---
title: "深入理解StatefulSet"
date: 2022-01-01T20:23:56+08:00
categories: ["云原生"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["云原生","Kubernetes"]
draft: false
---

### 引言

Deployment 实际上并不足以覆盖所有的应用编排问题，因为Deployment 对应用做了一个简单化假设：它认为，一个应用的所有 Pod，是完全一样的，所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上，可以随意创建和销毁Pod。

但是实际场景中，有很多服务不是这种情况：

1. 服务的多个实例有依赖关系，比如主从；
2. 数据存储类应用，如mysql。

这种实例间有不对等关系，以及实例对外部数据有依赖关系的应用叫 `有状态应用` (Stateful Application)。

### StatefulSet 设计思路

StatefulSet 对所有应用抽象出两种状态：

1. 拓扑状态。比如B依赖于A，B节点必须在A节点之后创建。
2. 存储状态。实例绑定了存储数据，最典型的是一个数据库应用的多个存储实例。

所以，StatefulSet 的核心功能就是，通过某种方式记录这些状态，然后在Pod被重新创建的时候，可以为Pod恢复这些状态。

### Headless Service

Service 是 Kubernetes 用来将一组 Pod 暴露给外界的一种机制。可以通过一下方式访问：

1. VIP (Virtual IP，即：虚拟 IP)。访问VIP时，会把请求转发给真实的pod。
2. DNS。访问一个dns记录就能访问到pod。有两种处理方法：
   - Normal Service。解析到服务的VIP，然后用VIP访问。
   - Headless Service。直接解析出pod的ip地址。

### StatefulSet 使用例子

##### 用deployment部署nginx

我们先用nginx创建两个nginx pod：

```yaml
# nginx-dep.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
```

> 创建StatefulSet必须要先创建服务。

##### 然后创建服务

```yaml
# service.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None #None表示用Headless Service的方式
  selector:
    app: nginx #会去筛选出labels为nginx的pod
```

![image-20220101220937787](https://img.jooks.cn/img/202201012209868.png)

创建成功后，这个服务所有的pod的ip都会被绑定到如下的dns记录上：

```shell
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

##### 创建StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx" #告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
          name: web
```

![image-20220101222846749](https://img.jooks.cn/img/202201012228821.png)

StatefulSet 会给它所管理的所有 Pod 的名字，进行编号，编号规则是：`<statefulset name>-<ordinal index>`

有上面的图片可以看出，StatefulSet创建pod时，web-0在进入Running状态前，web-1会一直保持Pending状态。

##### 进入容器内部查看hostname

![image-20220101223346645](https://img.jooks.cn/img/202201012233685.png)

##### 查看dns

我们安装一个busybox的pod，然后进入pod，然后用nslookup命令看一下web-0.nginx和web-1.nginx的dns记录。

![image-20220101224148014](https://img.jooks.cn/img/202201012241053.png)

##### 删除pod

删除pod后，会发现还是按照之前编号的顺序重启，且dns解析没变。

这种严格的对应规则，保证了 Pod 网络标志的稳定性。

通过这种方法，Kubernetes 就成功地将 Pod 的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照 Pod 的“名字 + 编号”的方式固定了下来。

> 个人感觉作者这个地方例子有点小问题，因为上面nginx这种例子个人理解是无状态的，用来说明StatefulSet显然不太合适，让刚接触的人很难理解上面这句话。

未完待续。。。











