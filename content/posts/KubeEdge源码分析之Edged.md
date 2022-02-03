---
title: "KubeEdge源码分析之Edged"
date: 2022-02-03T00:00:01+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeEdge"]
categories: ["云原生"]
draft: false
---

首先是注册，注册是在EdgeCore的启动流程中包含的。

### 注册

```go
// Register register edged
func Register(e *v1alpha1.Edged) {
   edgedconfig.InitConfigure(e)
   edged, err := newEdged(e.Enable)
   if err != nil {
      klog.Errorf("init new edged error, %v", err)
      os.Exit(1)
   }
   core.Register(edged)
}
```

这里就是初始化了edged对象，其中的newEdged比较重要，下面列出里面的一些操作。

- 初始化backoff

- 初始化podManager

- 初始化ImageGCPolicy，镜像回收策略

- 初始化EventRecorder

- 初始化metaclient

- 初始化runtimeClassManager，这个manager的作用是缓存RuntimeClass API对象，并向kubelet提供访问

- 初始化containerGCPolicy，容器回收策略

- 开启docker服务（startDockerServer方法）

  `RemoteRuntimeEndpoint` 必须是 `unix:///var/run/dockershim.sock` 或者 `/var/run/dockershim.sock` ，默认值是前者。

  首先是初始化各种配置，然后调用了 `dockershim` 的API初始化了一个CRIService对象

  ```go
  ds, err := dockershim.NewDockerService(DockerClientConfig,
     edgedconfig.Config.PodSandboxImage,
     streamingConfig,
     &pluginConfigs,
     cgroupName,
     cgroupDriver,
     DockershimRootDir)
  ```

  然后初始化了DockerServer，这一步创建了dockershim的grpc服务

  ```go
  server := dockerremote.NewDockerServer(edgedconfig.Config.RemoteRuntimeEndpoint, ds)
  ```

- 初始化kubedns

- 根据配置中的endpoint，初始化runtimeService和imageService

  ```go
  runtimeService, imageService, err := getRuntimeAndImageServices(
     edgedconfig.Config.RemoteRuntimeEndpoint,
     edgedconfig.Config.RemoteImageEndpoint,
     metav1.Duration{
        Duration: time.Duration(edgedconfig.Config.RuntimeRequestTimeout) * time.Minute,
     })
  ```

- 初始化ContainerLifecycleManager

- 初始化cadvisor

- 初始化machineInfo

- 初始化logManager

- 初始化kubeGenericRuntimeManager

- 初始化ContainerManager

- 初始化RuntimeCache

- 初始化imageGCManager

- 初始化containerGCManager

- 初始化podKiller

注册完成后就是启动了，启动也包含在EdgeCore的启动流程中。

### 初始化volumePluginMgr

```go
e.volumePluginMgr = NewInitializedVolumePluginMgr(e, ProbeVolumePlugins(""))
```

这个地方具体是先生成了一个插件列表，然后调用VolumePluginMgr#InitPlugins初始化列表中的volumn插件。

### 启动模块

```go
if err := e.initializeModules(); err != nil {
   klog.Errorf("initialize module error: %v", err)
   os.Exit(1)
}
```

initializeModules方法的代码如下，分析请看注释

```go
func (e *edged) initializeModules() error {
   // 可观测性的逻辑
   if edgedconfig.Config.EnableMetrics {
      // Start resource analyzer
      e.resourceAnalyzer.Start()

      if err := e.cadvisor.Start(); err != nil {
         // Fail kubelet and rely on the babysitter to retry starting kubelet.
         // TODO(random-liu): Add backoff logic in the babysitter
         klog.Exitf("Failed to start cAdvisor %v", err)
      }

      // trigger on-demand stats collection once so that we have capacity information for ephemeral storage.
      // ignore any errors, since if stats collection is not successful, the container manager will fail to start below.
      e.Provider.GetCgroupStats("/", true)
   }
  
   // 下面的代码的核心是e.containerManager.Start
   // Start container manager.
   node, err := e.initialNode()
   if err != nil {
      klog.Errorf("Failed to initialNode %v", err)
      return err
   }
   // If the container logs directory does not exist, create it.
   if _, err := os.Stat(ContainerLogsDir); err != nil {
      if err := e.os.MkdirAll(ContainerLogsDir, 0755); err != nil {
         klog.Errorf("Failed to create directory %q: %v", ContainerLogsDir, err)
      }
   }
   // containerManager must start after cAdvisor because it needs filesystem capacity information
   err = e.containerManager.Start(node, e.GetActivePods, edgedutil.NewSourcesReady(e.isInitPodReady), e.statusManager, e.runtimeService)
   if err != nil {
      klog.Errorf("Failed to start container manager, err: %v", err)
      return err
   }

   return nil
}
```

### 初始化volumeManager

```go
e.volumeManager = volumemanager.NewVolumeManager(
   true,
   types.NodeName(e.nodeName),
   e.podManager,
   e.statusManager,
   e.kubeClient,
   e.volumePluginMgr,
   e.containerRuntime,
   e.mounter,
   e.hostUtil,
   e.getPodsDir(),
   record.NewEventRecorder(),
   false,
   false,
   volumepathhandler.NewBlockVolumePathHandler(),
)
go e.volumeManager.Run(edgedutil.NewSourcesReady(e.isInitPodReady), utilwait.NeverStop)
```

这个地方调用了kubelet的API进行初始化

### 创建一个goroutine不断查询node状态

```go
go utilwait.Until(e.syncNodeStatus, e.nodeStatusUpdateFrequency, utilwait.NeverStop)
```

### 创建一个goroutine根据channel杀掉对应的pod

```go
go utilwait.Until(e.podKiller.PerformPodKillingWork, 5*time.Second, utilwait.NeverStop)
```

### 更新节点label

```go
// update node label
node, _ := e.GetNode()
node.Labels = e.labels
if err := e.metaClient.Nodes(e.namespace).Update(node); err != nil {
   klog.Errorf("update node failed, error: %v", err)
}
```

### 创建pod工作循环

```go
e.podAddWorkerRun(e.concurrentConsumers)
e.podRemoveWorkerRun(e.concurrentConsumers)
```

这里会根据concurrentConsumers来生成相应个数的goroutine，然后一直循环，从podAdditionQueue和podDeletionQueue中获取新增或删除的pod的信息，然后执行pod的增加和删除操作。

### 创建pod同步循环

```go
go e.syncLoopIteration(e.pleg.Watch(), housekeepingTicker.C, syncWorkQueueCh.C)
```

感觉这个地方的syncLoopIteration太重要了，虽然代码很长，还是贴一下吧，分析请看注释。

```go
func (e *edged) syncLoopIteration(plegCh <-chan *pleg.PodLifecycleEvent, housekeepingCh <-chan time.Time, syncWorkQueueCh <-chan time.Time) {
   for {
      select {
      case update := <-e.livenessManager.Updates():
         // 如果发现有pod创建错误了，就将pod的信息重新放入podAdditionQueue
         if update.Result == proberesults.Failure {
            pod, ok := e.podManager.GetPodByUID(update.PodUID)
            ...
            klog.Infof("Will restart pod [%s]", pod.Name)
            key := types.NamespacedName{
               Namespace: pod.Namespace,
               Name:      pod.Name,
            }
            e.podAdditionQueue.Add(key.String())
         }
      case plegEvent := <-plegCh:
         if pod, ok := e.podManager.GetPodByUID(plegEvent.ID); ok {
            // 更新pod状态
            if err := e.updatePodStatus(pod); err != nil {
               klog.Errorf("update pod %s status error", pod.Name)
               break
            }
            // 如果容器die了
            if plegEvent.Type == pleg.ContainerDied {
               // 如果restart策略为Never，直接忽略
               if pod.Spec.RestartPolicy == v1.RestartPolicyNever {
                  break
               }
               var containerCompleted bool
               // 如果restart策略为OnFailure，检查是否为启动失败
               if pod.Spec.RestartPolicy == v1.RestartPolicyOnFailure {
                  for _, containerStatus := range pod.Status.ContainerStatuses {
                     if containerStatus.State.Terminated != nil && containerStatus.State.Terminated.ExitCode == 0 {
                        containerCompleted = true
                        break
                     }
                  }
                  if containerCompleted {
                     break
                  }
               }
               klog.Infof("sync loop get event container died, restart pod [%s]", pod.Name)
               key := types.NamespacedName{
                  Namespace: pod.Namespace,
                  Name:      pod.Name,
               }
               // 否则添加至podAdditionQueue
               e.podAdditionQueue.Add(key.String())
            } else {
               klog.Infof("sync loop get event [%s], ignore it now.", plegEvent.Type)
            }
         } else {
            klog.Infof("sync loop ignore event: [%s], with pod [%s] not found", plegEvent.Type, plegEvent.ID)
         }
      case <-housekeepingCh:
         // 清理所有pod
         err := e.HandlePodCleanups()
         if err != nil {
            klog.Errorf("Handle Pod Cleanup Failed: %v", err)
         }
      case <-syncWorkQueueCh:
         // 重新同步pod（ready状态的pod和内部模块需要重新同步）
         podsToSync := e.getPodsToSync()
         if len(podsToSync) == 0 {
            break
         }
         for _, pod := range podsToSync {
            if !e.podIsTerminated(pod) {
               key := types.NamespacedName{
                  Namespace: pod.Namespace,
                  Name:      pod.Name,
               }
               e.podAdditionQueue.Add(key.String())
            }
         }
      }
   }
}
```

### 启动垃圾回收

```go
e.imageGCManager.Start()
e.StartGarbageCollection()
```

其中，StartGarbageCollection主要是创建了两个goroutine，不断地调用CRI的api，每2分钟进行一次containerGC，每5分钟进行一次imageGC

### 初始化并启动pluginManager

```go
e.pluginManager = pluginmanager.NewPluginManager(
   e.getPluginsRegistrationDir(), /* sockDir */
   nil,
)

// Adding Registration Callback function for CSI Driver
e.pluginManager.AddHandler(pluginwatcherapi.CSIPlugin, plugincache.PluginHandler(csiplugin.PluginHandler))
// Start the plugin manager
klog.Infof("starting plugin manager")
go e.pluginManager.Run(edgedutil.NewSourcesReady(e.isInitPodReady), utilwait.NeverStop)
```

这里并不知道为啥要为CSI Driver设置回调函数，先Mark一下🌟

### 启动各种东西

```go
// start the CPU manager in the clcm
err := e.clcm.StartCPUManager(e.GetActivePods, edgedutil.NewSourcesReady(e.isInitPodReady), e.statusManager, e.runtimeService)
if err != nil {
   klog.Errorf("Failed to start container manager, err: %v", err)
   return
}
e.logManager.Start()
e.runtimeClassManager.Start(utilwait.NeverStop)
klog.Infof("starting syncPod")
```

### 同步pod

在start的最后有这样一行代码：

```go
e.syncPod()
```

下面来看一下这个syncPod具体做了什么。

1. 睡眠10秒

   ```go
   time.Sleep(10 * time.Second)
   ```

2. 向metamanager发送信息，拿到正在退出的pods

   ```go
   //when starting, send msg to metamanager once to get existing pods
   info := model.NewMessage("").BuildRouter(e.Name(), e.Group(), e.namespace+"/"+model.ResourceTypePod,
      model.QueryOperation)
   beehiveContext.Send(metamanager.MetaManagerModuleName, *info)
   ```

接着是个大循环（分析请看注释）

```go
for {
   select {
   case <-beehiveContext.Done():
      // 收到全局的结束信号后，停止循环，结束edged进程
      klog.Warning("Sync pod stop")
      return
   default:
   }
   
   // 从context中拿到消息
   result, err := beehiveContext.Receive(e.Name())
   if err != nil {
      klog.Errorf("failed to get pod: %v", err)
      continue
   }
   
   // 解析消息
   _, resType, resID, err := util.ParseResourceEdge(result.GetResource(), result.GetOperation())
   if err != nil {
      klog.Errorf("failed to parse the Resource: %v", err)
      continue
   }
   // 拿到消息中的操作
   op := result.GetOperation()

   // 拿到消息中的内容
   content, err := result.GetContentData()
   if err != nil {
      klog.Errorf("get message content data failed: %v", err)
      continue
   }
   klog.Infof("result content is %s", result.Content)
   switch resType {
   case model.ResourceTypePod:
      // 如果是关于pod的信息，就进入这个分支
      if op == model.ResponseOperation && resID == "" && result.GetSource() == metamanager.MetaManagerModuleName {
         // 如果是来自metaManager的消息，就意味着里面是pod列表，于是利用这个列表添加pod，并刷新pod状态
         err := e.handlePodListFromMetaManager(content)
         if err != nil {
            klog.Errorf("handle podList failed: %v", err)
            continue
         }
         e.setInitPodReady(true)
      } else if op == model.ResponseOperation && resID == "" && result.GetSource() == EdgeController {
         // 如果是来自EdgeController的消息，也意味着里面是pod列表，于是利用这个列表添加pod，但是不会刷新pod状态
         err := e.handlePodListFromEdgeController(content)
         if err != nil {
            klog.Errorf("handle controllerPodList failed: %v", err)
            continue
         }
         e.setInitPodReady(true)
      } else {
         // 否则，就执行其他操作，包括了pod简单的增删改
         err := e.handlePod(op, content)
         if err != nil {
            klog.Errorf("handle pod failed: %v", err)
            continue
         }
      }
   case model.ResourceTypeConfigmap:
      // 如果是关于configmap信息，就进行configmap的crud
      if op != model.ResponseOperation {
         err := e.handleConfigMap(op, content)
         if err != nil {
            klog.Errorf("handle configMap failed: %v", err)
         }
      } else {
         klog.V(4).Infof("skip to handle configMap with type response")
         continue
      }
   case model.ResourceTypeSecret:
      // 如果是关于secret的信息，就进行secret的crud
      if op != model.ResponseOperation {
         err := e.handleSecret(op, content)
         if err != nil {
            klog.Errorf("handle secret failed: %v", err)
         }
      } else {
         klog.V(4).Infof("skip to handle secret with type response")
         continue
      }
   case constants.CSIResourceTypeVolume:
      // 如果是volume相关的信息，就进行volume的crud操作（利用CSI接口）
      klog.Infof("volume operation type: %s", op)
      res, err := e.handleVolume(op, content)
      if err != nil {
         klog.Errorf("handle volume failed: %v", err)
      } else {
         resp := result.NewRespByMessage(&result, res)
         // 并且把结果封装成消息，塞回context
         beehiveContext.SendResp(*resp)
      }
   default:
      klog.Errorf("resType is not pod or configmap or secret or volume: resType is %s", resType)
      continue
   }
}
```







