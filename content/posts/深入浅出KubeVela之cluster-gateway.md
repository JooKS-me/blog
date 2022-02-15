---
title: "深入浅出KubeVela之cluster-gateway"
date: 2022-02-14T00:23:50+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeVela"]
categories: ["云原生"]
draft: false
---

## 概述

> 这段文字翻译自cluster-gateway仓库的 [README](https://github.com/oam-dev/cluster-gateway#overall)

“Cluster Gateway”是一个gateway apiserver，用于将 **kubernetes api** 流量路由到多个 kubernetes 集群。此外，网关对于运行中的 kubernetes 集群是完全可插入的，因为它是基于名为 `apiserver-aggregation` 的原生 api 可扩展性开发的。 正确应用相应的 APIService 对象后，新的扩展资源“cluster.core.oam.dev/ClusterGateway”将注册到托管集群中，并且一个名为“proxy”的新子资源将可用于每个现有的“Cluster Gateway”资源（受原始 kubernetes “service/proxy”、“pod/proxy” 子资源的启发）。

总的来说，我们的“Cluster Gateway”作为多集群 api-gateway 解决方案具有以下优点：

- 无需 Etcd ：通常 aggregated apiserver 需要部署一个专用的 etcd 集群，这给管理员带来了额外的成本。但是，我们的“Cluster Gateway”可以在没有 etcd 实例的情况下完全运行，因为扩展的“Cluster Gateway”资源是虚拟的只读 kubernetes 资源，它是从托管集群中相应namespace的资源转换而来的。
- 可伸缩性：我们的“Cluster Gateway”可以扩展到任意数量的实例以应对不断增加的负载。

![Arch](https://img.jooks.cn/img/202202142228769.png)

> 下面都是自己写的了哈

实际上，IMO，上面写的两个优点都是aggregated apiserver所给予的特性。

cluster-gateway 项目看的出来，应该是用了 `apiserver-builder` ，由于  `apiserver-builder` 不支持arm架构（我把源码拉下来都构建不了，放弃了），所以我就没有尝试写demo了。`apiserver-builder` 代码写完之后，按照 [官方文档](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/concepts/aggregation.md)（写的非常好的一篇文章，讲了aggregated apiserver的原理）即可集成进聚合apiserver。

从上面的架构图中，我们可以提出以下几个问题：

- 集群网关是如何对 aggregation apiserver 进行拓展的？
- 集群网关是如何转发K8S API请求的？
- K8S密钥是如何存储的？

带着这几个问题，我们来看一看源码。

## 部署脚本

我们看一下官方给的部署yaml（也可以看charts，差不多）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway-deployment
  labels:
    app: gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
        - name: gateway
          image: "cluster-gateway:v0.0.0-non-etcd"
          command:
            - ./apiserver
            - --secure-port=9443
            - --secret-namespace=default
            - --feature-gates=APIPriorityAndFairness=false
          ports:
            - containerPort: 9443
---
apiVersion: v1
kind: Service
metadata:
  name: gateway-service
spec:
  selector:
    app: gateway
  ports:
    - protocol: TCP
      port: 9443
      targetPort: 9443
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.cluster.core.oam.dev
  labels:
    api: cluster-extension-apiserver
    apiserver: "true"
spec:
  version: v1alpha1
  group: cluster.core.oam.dev
  groupPriorityMinimum: 2000
  service:
    name: gateway-service
    namespace: default
    port: 9443
  versionPriority: 10
  insecureSkipTLSVerify: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: system::extension-apiserver-authentication-reader:cluster-gateway
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: cluster-gateway-secret-reader
rules:
  - apiGroups:
      - ""
    resources:
      - "secrets"
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-gateway-secret-reader
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-gateway-secret-reader
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
---

```

大致是这样：

先创建了一个deployment，容器镜像为cluster-gateway；

然后创建一个service，绑定deploy创建的几个pod；

**然后重点来了，创建一个APIService对象，然后前面创建的service就会注册进aggregation apiserver。** 然后就可以通过kubectl或者http请求从apiserver获取我们在 `pkg/apis` 目录下定义的api对象了，是不是很神奇。

剩下的是绑定权限。

## API资源

为了方便直接用curl访问rest api，我们使用kubectl的反向代理功能。

```shell
kubectl proxy --port=8080
```

然后，利用curl命令访问前面创建的api资源。

```shell
curl http://localhost:8080/apis/cluster.core.oam.dev/v1alpha1/
```

会得到如下结果：

```json
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "cluster.core.oam.dev/v1alpha1",
  "resources": [
    {
      "name": "clustergateways",
      "singularName": "",
      "namespaced": false,
      "kind": "ClusterGateway",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "clustergateways/proxy",
      "singularName": "",
      "namespaced": false,
      "kind": "ClusterGatewayProxyOptions",
      "verbs": [
        "create",
        "delete",
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

可以看到这里有一个clustergateways资源，它有一个proxy子资源。（正如 `概述` 里描述的那样）

然后我们通过如下路径就可以操作clustergateways/proxy

```
/apis/cluster.core.oam.dev/v1alpha1/clustergateways/cluster_name/proxy/<api>
```

## 如何转发K8S请求

> 看了半天还是懵的QAQ，还是太菜了

这里用到了 `apimachinery` ，而这个框架的底层核心是 go 原生的反向代理工具 `ReverseProxy#ServeHTTP`。

主要看 `clustergateway_proxy.go` 中的 ServeHTTP 方法。

```go
func (p *proxyHandler) ServeHTTP(writer http.ResponseWriter, request *http.Request) {
   cluster := p.clusterGateway
   ...

   // WithContext creates a shallow clone of the request with the same context.
   newReq := request.WithContext(request.Context())
   newReq.Header = utilnet.CloneHeader(request.Header)
   newReq.URL.Path = p.path

   urlAddr, err := GetEndpointURL(cluster)
   if err != nil {
      responsewriters.InternalError(writer, request, errors.Wrapf(err, "failed parsing endpoint for cluster %s", cluster.Name))
      return
   }
   host, _, _ := net.SplitHostPort(urlAddr.Host)
   // 1. 重写路径
   path := strings.TrimPrefix(request.URL.Path, apiPrefix+p.parentName+apiSuffix)
   // 2. 重写host
   newReq.Host = host
   newReq.URL.Path = path
   newReq.URL.RawQuery = request.URL.RawQuery
   newReq.RequestURI = newReq.URL.RequestURI()

   cfg, err := NewConfigFromCluster(cluster)
   if err != nil {
      responsewriters.InternalError(writer, request, errors.Wrapf(err, "failed creating cluster proxy client config %s", cluster.Name))
      return
   }
   if p.impersonate {
      cfg.Impersonate = getImpersonationConfig(request)
   }
   rt, err := restclient.TransportFor(cfg)
   if err != nil {
      responsewriters.InternalError(writer, request, errors.Wrapf(err, "failed creating cluster proxy client %s", cluster.Name))
      return
   }
   proxy := apiproxy.NewUpgradeAwareHandler(
      &url.URL{
         Scheme:   urlAddr.Scheme,
         Path:     path,
         Host:     urlAddr.Host,
         RawQuery: request.URL.RawQuery,
      },
      rt,
      false,
      false,
      nil)

   const defaultFlushInterval = 200 * time.Millisecond
   // ...
   // 这里是配置tls
   proxy.UpgradeTransport = apiproxy.NewUpgradeRequestRoundTripper(
      upgrading,
      RoundTripperFunc(func(req *http.Request) (*http.Response, error) {
         newReq := utilnet.CloneRequest(req)
         return upgrader.RoundTrip(newReq)
      }))
   proxy.Transport = rt
   proxy.FlushInterval = defaultFlushInterval
   proxy.Responder = ErrorResponderFunc(func(w http.ResponseWriter, req *http.Request, err error) {
      p.responder.Error(err)
   })
   proxy.ServeHTTP(writer, newReq)
}
```

1. 重写路径

   会把 `xxx/xxxx/xxx/proxy/<api>` 里的 \<api> 提出来。

2. 重写host

   根据cluster获取真实的后端

## vela-core如何与网关对接

vela-core与网关对接这块的核心逻辑在 `pkg/multicluster/proxy.go` 中

```go
// RoundTrip is the main function for the re-write API path logic
func (rt *secretMultiClusterRoundTripper) RoundTrip(req *http.Request) (*http.Response, error) {
   ctx := req.Context()
   clusterName, ok := ctx.Value(ClusterContextKey).(string)
   if !ok || clusterName == "" || clusterName == ClusterLocalName {
      return rt.rt.RoundTrip(req)
   }

   req.URL.Path = FormatProxyURL(clusterName, req.URL.Path)
   return rt.rt.RoundTrip(req)
}
```

程序会先根据context中的ClusterName是否为local来决定是否需要经过网关路由。

对于非local节点来说，会经过 `FormatProxyURL` 方法重写URL：

```go
// FormatProxyURL will format the request API path by the cluster gateway resources rule
func FormatProxyURL(clusterName, originalPath string) string {
   originalPath = strings.TrimPrefix(originalPath, "/")
   return strings.Join([]string{"/apis", clusterapi.SchemeGroupVersion.Group, clusterapi.SchemeGroupVersion.Version, "clustergateways", clusterName, "proxy", originalPath}, "/")
}
```

这个方法就是加上 `/apis/.../proxy/` 前缀，这样请求就会打到网关上了。

> 未完待续

