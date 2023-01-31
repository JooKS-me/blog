---
title: "Nacos新版本坑，无响应，无报错，就是获取不到配置，可能是因为这个！"
date: 2023-01-30T00:12:30+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["中间件","Nacos","微服务"]
categories: ["中间件"]
draft: false
---

## 现象

我使用nacos-client，但是不使用Spring。于到nacos官方的example里面找例子：https://github.com/nacos-group/nacos-examples/blob/5d458d55411ca37660706208d997681203290258/nacos-client-example/src/main/java/com/alibaba/nacos/NacosConfigExample.java。

然后写了如下代码：

```java
				Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, "192.168.1.98:8848");
        ConfigService configService = NacosFactory.createConfigService(properties);

        // 指定配置的 DataID 和 Group
        String dataId = "nacos.cfg.dataId";
        String group = "test";
        String content = "connectTimeoutInMills=5000";

        // 查询cos配置
        boolean publishConfig = configService.publishConfig(dataId, group, content);
        System.out.println(publishConfig);
```

然后运行，日志如下：

```markdown
19:30:06.923 [main] INFO com.alibaba.nacos.plugin.auth.spi.client.ClientAuthPluginManager - [ClientAuthPluginManager] Load ClientAuthService com.alibaba.nacos.client.auth.impl.NacosClientAuthServiceImpl success.
19:30:06.923 [main] INFO com.alibaba.nacos.plugin.auth.spi.client.ClientAuthPluginManager - [ClientAuthPluginManager] Load ClientAuthService com.alibaba.nacos.client.auth.ram.RamClientAuthServiceImpl success.
false
19:30:16.656 [Thread-0] WARN com.alibaba.nacos.common.http.HttpClientBeanHolder - [HttpClientBeanHolder] Start destroying common HttpClient
19:30:16.656 [Thread-5] WARN com.alibaba.nacos.common.notify.NotifyCenter - [NotifyCenter] Start destroying Publisher
19:30:16.656 [Thread-5] WARN com.alibaba.nacos.common.notify.NotifyCenter - [NotifyCenter] Destruction of the end
19:30:16.657 [Thread-0] WARN com.alibaba.nacos.common.http.HttpClientBeanHolder - [HttpClientBeanHolder] Destruction of the end
```

发现怎么样都获取不到配置。

回退版本、关闭密码验证、上传等等都不行。

但是用curl直接请求是可以的，这说明nacos-server是好的。

## 发现问题

日志太少，在前面加一行代码：

```java
System.setProperty("nacos.logging.default.config.enabled", "false");
```

然后重新运行，会看到很多DEBUG日志。

这时候仔细观察发现这样一段日志：

```markdown
19:40:25.098 [main] ERROR com.alibaba.nacos.common.remote.client.grpc.GrpcClient - Server check fail, please check server 192.168.1.98 ,port 9848 is available , error ={}
java.util.concurrent.TimeoutException: Waited 3000 milliseconds (plus 1 milliseconds, 465958 nanoseconds delay) for com.alibaba.nacos.shaded.io.grpc.stub.ClientCalls$GrpcFuture@503f91c3[status=PENDING, info=[GrpcFuture{clientCall=ClientCallImpl{method=MethodDescriptor{fullMethodName=Request/request, type=UNARY, idempotent=false, safe=false, sampledToLocalTracing=true, requestMarshaller=com.alibaba.nacos.shaded.io.grpc.protobuf.lite.ProtoLiteUtils$MessageMarshaller@20bd8be5, responseMarshaller=com.alibaba.nacos.shaded.io.grpc.protobuf.lite.ProtoLiteUtils$MessageMarshaller@730d2164, schemaDescriptor=com.alibaba.nacos.api.grpc.auto.RequestGrpc$RequestMethodDescriptorSupplier@24959ca4}}}]]
	at com.alibaba.nacos.shaded.com.google.common.util.concurrent.AbstractFuture.get(AbstractFuture.java:508)
	at com.alibaba.nacos.common.remote.client.grpc.GrpcClient.serverCheck(GrpcClient.java:195)
	at com.alibaba.nacos.common.remote.client.grpc.GrpcClient.connectToServer(GrpcClient.java:306)
	at com.alibaba.nacos.common.remote.client.RpcClient.start(RpcClient.java:389)
	at com.alibaba.nacos.client.config.impl.ClientWorker$ConfigRpcTransportClient.ensureRpcClient(ClientWorker.java:888)
	at com.alibaba.nacos.client.config.impl.ClientWorker$ConfigRpcTransportClient.getOneRunningClient(ClientWorker.java:1035)
	at com.alibaba.nacos.client.config.impl.ClientWorker$ConfigRpcTransportClient.publishConfig(ClientWorker.java:1050)
	at com.alibaba.nacos.client.config.impl.ClientWorker.publishConfig(ClientWorker.java:293)
	at com.alibaba.nacos.client.config.NacosConfigService.publishConfigInner(NacosConfigService.java:237)
	at com.alibaba.nacos.client.config.NacosConfigService.publishConfig(NacosConfigService.java:129)
	at com.alibaba.nacos.client.config.NacosConfigService.publishConfig(NacosConfigService.java:124)
	at cn.jooks.common.utils.CosUtils.<init>(CosUtils.java:28)
	at cn.jooks.common.utils.CosUtils.main(CosUtils.java:37)
```

划重点：`please check server 192.168.1.98 ,port 9848 is available`

原来nacos 2.x增加了两个端口，用grpc通讯。（见[https://nacos.io/zh-cn/docs/v2/upgrading/2.0.0-compatibility.html](https://nacos.io/zh-cn/docs/v2/upgrading/2.0.0-compatibility.html)）

大坑！nacos官网打开默认进入nacos 1.x的文档，2.x的入口很隐蔽。。。然后快速开始也没说要用9848。

## 解决

服务器开放9848端口。

```shell
# Ubuntu
sudo ufw allow 9848
```

客户端代码改一改，SERVER_ADDR不需要带上端口号，默认就是9848，不过带了也没用。

```java
				Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, "192.168.1.98");
        ConfigService configService = NacosFactory.createConfigService(properties);
```

