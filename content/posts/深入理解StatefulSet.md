---
title: "深入理解StatefulSet"
date: 2022-01-01T20:23:56+08:00
categories: ["云原生"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["云原生","Kubernetes"]
draft: false
---

## 引言

Deployment 实际上并不足以覆盖所有的应用编排问题，因为Deployment 对应用做了一个简单化假设：它认为，一个应用的所有 Pod，是完全一样的，所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上，可以随意创建和销毁Pod。

但是实际场景中，有很多服务不是这种情况：

1. 服务的多个实例有依赖关系，比如主从；
2. 数据存储类应用，如mysql。

这种实例间有不对等关系，以及实例对外部数据有依赖关系的应用叫 `有状态应用` (Stateful Application)。

## StatefulSet 设计思路

StatefulSet 对所有应用抽象出两种状态：

1. 拓扑状态。比如B依赖于A，B节点必须在A节点之后创建。
2. 存储状态。实例绑定了存储数据，最典型的是一个数据库应用的多个存储实例。

所以，StatefulSet 的核心功能就是，通过某种方式记录这些状态，然后在Pod被重新创建的时候，可以为Pod恢复这些状态。

## 拓扑状态

### Headless Service

Service 是 Kubernetes 用来将一组 Pod 暴露给外界的一种机制。可以通过一下方式访问：

1. VIP (Virtual IP，即：虚拟 IP)。访问VIP时，会把请求转发给真实的pod。
2. DNS。访问一个dns记录就能访问到pod。有两种处理方法：
   - Normal Service。解析到服务的VIP，然后用VIP访问。
   - Headless Service。直接解析出pod的ip地址。

### 例子

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

> ~~个人感觉作者这个地方例子有点小问题，因为上面nginx这种例子个人理解是无状态的，用来说明StatefulSet显然不太合适，让刚接触的人很难理解上面这句话。~~
>
> 二刷的时候懂了好多，作者好nb，orz

## 存储状态

### 痛点

1. 持久化存储项目（比如 Ceph、GlusterFS 等）学习成本高，volume定义文件编写有一定门槛。
2. 可能暴露公司基础设施秘密

### Persistent Volume Claim

为了解决上述痛点，Kubernetes 项目引入了一组叫作 Persistent Volume Claim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化 Volume 的门槛。

下面举个使用例子

一、开发人员定义PVC，声明想要的Volumn属性

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce #Volume 的挂载方式是可读写，并且只能被挂载在一个节点上而非被多个节点共享
  resources:
    requests:
      storage: 1Gi #Volumn大小至少为1GiB
```

>关于哪种类型的 Volume 支持哪种类型的 AccessMode，可以看官方文档的说明：[https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)

二、开发人员在应用的Pod中，声明使用这个PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```

三、运维人员编写PV

```yaml

kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    # 使用 kubectl get pods -n rook-ceph 查看 rook-ceph-mon- 开头的 POD IP 即可得下面的列表
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
```

这样，Kubernetes 就会为我们刚刚创建的 PVC 对象绑定这个 PV，实现了volume挂载的解耦。

### StatefulSet 对存储状态的管理

对于StatefulSet，我们可以新增一个 volumeClaimTemplates 字段

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

凡是被这个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC；而这个 PVC 的定义，就来自于 volumeClaimTemplates 这个模板字段。**更重要的是，这个PVC的名字，会被分配一个与这个Pod完全一致的编号**。

PVC 其实就是一种特殊的 Volume。只不过一个 PVC 具体是什么类型的 Volume，要在跟某个 PV 绑定之后才知道。

利用上面的nginx服务，启动StatefulSet之后，会发现有叫www-web-0和www-web-1的两个pvc，pvc的命名方式为：`<PVC 名字 >-<StatefulSet 名字 >-< 编号 >`。

然后我们分别在两个volume对应的目录写入web-0和web-1，然后用curl访问，可以得到相应的web-0/1.

这时如果同时删掉两个pod，两个pod会重启，然后再执行curl会发现，pod web-0 里面还是 web-0，web-1 里面还是 web-1。

> 实际上，删掉pod的时候，pvc存在，pv也存在，数据存在远程存储服务中。
>
> 重启了web-0后，会去查找叫 www-web-0 的pvc，进而找到pv，然后挂载上之前的volume。
>
> 这样，就实现了应用存储状态的管理。

## 暂时的个人理解

1. Headless Service给deployment的pod分了名字，然后StatefulSet将这种名字锁死，而这个名字就是拓扑，也就是说拓扑状态被锁死了。
2. `StatefulSet可以让pod的名字不变` + PVC，使得pod对应的volume不会变，也就是存储了存储状态。（感觉这个地方不适合翻译）



