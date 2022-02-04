---
title: "KubeEdge源码分析之CloudHub"
date: 2022-02-04T00:12:30+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeEdge"]
categories: ["云原生"]
draft: false
---

### 注册

- 初始化配置

  ```go
  hubconfig.InitConfigure(hub)
  ```

  里面包含了读取认证证书。

- 初始化cloudHub对象

  ```go
  newCloudHub(hub.Enable)
  ```

  newCloudHub方法中，声明了使用的informer，初始化了内置的消息队列，并封装进cloudHub对象中。

  ```go
  func newCloudHub(enable bool) *cloudHub {
     crdFactory := informers.GetInformersManager().GetCRDInformerFactory()
     // declare used informer
     clusterObjectSyncInformer := crdFactory.Reliablesyncs().V1alpha1().ClusterObjectSyncs()
     objectSyncInformer := crdFactory.Reliablesyncs().V1alpha1().ObjectSyncs()
     messageq := channelq.NewChannelMessageQueue(objectSyncInformer.Lister(), clusterObjectSyncInformer.Lister())
     ch := &cloudHub{
        enable:   enable,
        messageq: messageq,
     }
     ch.informersSyncedFuncs = append(ch.informersSyncedFuncs, clusterObjectSyncInformer.Informer().HasSynced)
     ch.informersSyncedFuncs = append(ch.informersSyncedFuncs, objectSyncInformer.Informer().HasSynced)
     return ch
  }
  ```

接下来就是启动cloudHub了

### 等待同步完成

```go
if !cache.WaitForCacheSync(beehiveContext.Done(), a.informersSyncedFuncs...) {
   klog.Errorf("unable to sync caches for objectSyncController")
   os.Exit(1)
}
```

### 分发消息到边端

```go
// start dispatch message from the cloud to edge node
go a.messageq.DispatchMessage()
```

DispatchMessage函数内部是个大for循环。

然后判断是否结束。

```go
select {
case <-beehiveContext.Done():
   klog.Warning("Cloudhub channel eventqueue dispatch message loop stopped")
   return
default:
}
```

从beehiveContext中拿消息，然后再发出去。

```go
msg, err := beehiveContext.Receive(model.SrcCloudHub)
klog.V(4).Infof("[cloudhub] dispatchMessage to edge: %+v", msg)
if err != nil {
   klog.Info("receive not Message format message")
   continue
}
nodeID, err := GetNodeID(&msg)
if nodeID == "" || err != nil {
   klog.Warning("node id is not found in the message")
   continue
}
if isListResource(&msg) {
   q.addListMessageToQueue(nodeID, &msg)
} else {
   q.addMessageToQueue(nodeID, &msg)
}
```

那么，为什么调用addListMessageToQueue或者addMessageToQueue就会下发消息到边端呢🤔（请看下面的 `启动cloudHub`）

下面，我们详细剖析一下addMessageToQueue。

```go
func (q *ChannelMessageQueue) addMessageToQueue(nodeID string, msg *beehiveModel.Message) {
   if msg.GetResourceVersion() == "" && !isDeleteMessage(msg) {
      return
   }
   
   // 拿到nodeID对应的queue，从queuePool中取，如果没有就新建
   nodeQueue := q.GetNodeQueue(nodeID)
   // 拿到nodeID对应的store，从storePool中取，如果没有就新建
   nodeStore := q.GetNodeStore(nodeID)

   messageKey, err := getMsgKey(msg)
   if err != nil {
      klog.Errorf("fail to get message key for message: %s", msg.Header.ID)
      return
   }

   // 如果是删除或回应操作，不进入if
   if !isDeleteMessage(msg) && msg.GetOperation() != beehiveModel.ResponseOperation {
      item, exist, _ := nodeStore.GetByKey(messageKey)
      // 如果nodeStore中不存在这个messageKey，则将它跟数据库中的版本进行对比，如果数据库中的版本>=消息的版本，就忽略，否则下发消息。
      if !exist {
         resourceNamespace, _ := edgemessagelayer.GetNamespace(*msg)
         resourceUID, err := GetMessageUID(*msg)
         if err != nil {
            klog.Errorf("fail to get message UID for message: %s", msg.Header.ID)
            return
         }
         objectSync, err := q.objectSyncLister.ObjectSyncs(resourceNamespace).Get(synccontroller.BuildObjectSyncName(nodeID, resourceUID))
         if err == nil && objectSync.Status.ObjectResourceVersion != "" && synccontroller.CompareResourceVersion(msg.GetResourceVersion(), objectSync.Status.ObjectResourceVersion) <= 0 {
            return
         }
      }

      // 如果存在，就比较一下版本，判断是否需要下发。
      if exist {
         msgInStore := item.(*beehiveModel.Message)
         if isDeleteMessage(msgInStore) || synccontroller.CompareResourceVersion(msg.GetResourceVersion(), msgInStore.GetResourceVersion()) <= 0 {
            return
         }
      }
   }

   if err := nodeStore.Add(msg); err != nil {
      klog.Errorf("fail to add message %v nodeStore, err: %v", msg, err)
      return
   }
   nodeQueue.Add(messageKey)
}
```

### 准备证书

```go
// check whether the certificates exist in the local directory,
// and then check whether certificates exist in the secret, generate if they don't exist
if err := httpserver.PrepareAllCerts(); err != nil {
   klog.Exit(err)
}
// TODO: Will improve in the future
DoneTLSTunnelCerts <- true
close(DoneTLSTunnelCerts)
```

检查ca是否存在，如果不存在则创建，然后生成token

```go
// generate Token
if err := httpserver.GenerateToken(); err != nil {
   klog.Exit(err)
}
```

### 启动HTTP服务器

```go
// HttpServer mainly used to issue certificates for the edge
go httpserver.StartHTTPServer()
```

这里http服务器主要用来解决边缘ca验证的问题。

### 启动cloudHub

```go
servers.StartCloudHub(a.messageq)
```

这里的StartCloudHub非常关键。

```go
// StartCloudHub starts the cloud hub service
func StartCloudHub(messageq *channelq.ChannelMessageQueue) {
   handler.InitHandler(messageq)
   // start websocket server
   if hubconfig.Config.WebSocket.Enable {
      go startWebsocketServer()
   }
   // start quic server
   if hubconfig.Config.Quic.Enable {
      go startQuicServer()
   }
}
```

首先是初始化了消息队列的handler。

```go
// InitHandler create a handler for websocket and quic servers
func InitHandler(eventq *channelq.ChannelMessageQueue) {
   once.Do(func() {
      CloudhubHandler = &MessageHandle{
         KeepaliveInterval: int(hubconfig.Config.KeepaliveInterval),
         WriteTimeout:      int(hubconfig.Config.WriteTimeout),
         MessageQueue:      eventq,
         NodeLimit:         hubconfig.Config.NodeLimit,
         crdClient:         client.GetCRDClient(),
      }

      CloudhubHandler.Handlers = []HandleFunc{
         CloudhubHandler.KeepaliveCheckLoop,
         CloudhubHandler.MessageWriteLoop,
         CloudhubHandler.ListMessageWriteLoop,
      }

      CloudhubHandler.initServerEntries()
   })
}
```

这里创建了三个loop，分别是

- KeepaliveCheckLoop

  用于检查边缘节点是否存活

  ```go
  // KeepaliveCheckLoop checks whether the edge node is still alive
  func (mh *MessageHandle) KeepaliveCheckLoop(info *model.HubInfo, stopServe chan ExitCode) {
     keepaliveTicker := time.NewTimer(time.Duration(mh.KeepaliveInterval) * time.Second)
     nodeKeepaliveChannel, ok := mh.KeepaliveChannel.Load(info.NodeID)
     if !ok {
        klog.Errorf("fail to load node %s", info.NodeID)
        return
     }
  
     for {
        select {
        // 如果收到keepalive信号，就进入第一个分支，然后重置计时器；否则，如果在一个keepalive时间段中没有收到，就会进入第二个分支，被任务节点已经挂了。
        case _, ok := <-nodeKeepaliveChannel.(chan struct{}):
           if !ok {
              klog.Warningf("Stop keepalive check for node: %s", info.NodeID)
              return
           }
  
           // Reset is called after Stop or expired timer
           if !keepaliveTicker.Stop() {
              select {
              case <-keepaliveTicker.C:
              default:
              }
           }
           klog.V(4).Infof("Node %s is still alive", info.NodeID)
           keepaliveTicker.Reset(time.Duration(mh.KeepaliveInterval) * time.Second)
        case <-keepaliveTicker.C:
           if conn, ok := mh.nodeConns.Load(info.NodeID); ok {
              klog.Warningf("Timeout to receive heart beat from edge node %s for project %s", info.NodeID, info.ProjectID)
              if err := conn.(hubio.CloudHubIO).Close(); err != nil {
                 klog.Errorf("failed to close connection %v, err is %v", conn, err)
              }
              mh.nodeConns.Delete(info.NodeID)
           }
        }
     }
  }
  ```

- MessageWriteLoop

  用于发消息，loop会先从nodeQueue里面拿到要发送的消息的key，然后根据key从nodeStore里面拿到具体的消息，然后发出去。

  这样一来，其他地方只要把消息放入nodeQueue和nodeStore中即可，且是异步的。

- ListMessageWriteLoop

  MessageWriteLoop的批量下发版

然后开启了消息应答服务

```go
CloudhubHandler.initServerEntries()
```

```go
// initServerEntries register handler func
func (mh *MessageHandle) initServerEntries() {
   mux.Entry(mux.NewPattern("*").Op("*"), mh.HandleServer)
}
```

这里会注册一个handler，用于响应所有url

其中包含了

- keepalive消息，收到后将其放入nodeKeepalive channel中，交给前面的loop
- volume消息，收到后将其塞进beehiveContext，交给其他模块处理
- 。。。

最后，开启websocket或者quic服务

```go
// start websocket server
if hubconfig.Config.WebSocket.Enable {
   go startWebsocketServer()
}
// start quic server
if hubconfig.Config.Quic.Enable {
   go startQuicServer()
}
```