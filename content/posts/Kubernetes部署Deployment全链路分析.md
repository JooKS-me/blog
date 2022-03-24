---
title: "Kubernetes部署Deployment全链路分析"
date: 2022-03-23T00:23:50+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","Kubernetes"]
categories: ["云原生"]
draft: false
---

当我们编写好了一个deployment.yaml文件，然后执行`kubectl apply -f deployment.yaml`的时候，会发生什么呢？这篇文章将对deployment对象的创建全流程进行分析。

## kubectl

ctl里面其实就是用client去调apiserver，创建一个deployment对象。

## apiserver

这个东西好复杂，先留着以后再填。

## Deployment Controller

DeploymentController的入口在 `cmd/kube-controller-manager/app/apps.go#startDeploymentController`。

```go
func startDeploymentController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
   dc, err := deployment.NewDeploymentController(
      controllerContext.InformerFactory.Apps().V1().Deployments(),
      controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
      controllerContext.InformerFactory.Core().V1().Pods(),
      controllerContext.ClientBuilder.ClientOrDie("deployment-controller"),
   )
   if err != nil {
      return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
   }
   go dc.Run(ctx, int(controllerContext.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs))
   return nil, true, nil
}
```

这里首先创建了一个DeploymentController，然后运行Run方法。

在 NewDeploymentController 方法中，主要是做一些函数、属性的赋值。

重点看Run方法。

```go
// Run begins watching and syncing.
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
   defer utilruntime.HandleCrash()
   defer dc.queue.ShutDown()

   klog.InfoS("Starting controller", "controller", "deployment")
   defer klog.InfoS("Shutting down controller", "controller", "deployment")

   if !cache.WaitForNamedCacheSync("deployment", ctx.Done(), dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
      return
   }

   for i := 0; i < workers; i++ {
      go wait.UntilWithContext(ctx, dc.worker, time.Second)
   }

   <-ctx.Done()
}
```

正如注释写的那样，Run方法开始监听和同步。

```go
for i := 0; i < workers; i++ {
    go wait.UntilWithContext(ctx, dc.worker, time.Second)
}
```

```go
// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (dc *DeploymentController) worker(ctx context.Context) {
	for dc.processNextWorkItem(ctx) {
	}
}
```

这里的work函数的作用是，不断地循环，从事件队列中取出事件，然后对事件进行处理，最后将其标记为完成。

```go
func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
   key, quit := dc.queue.Get()
   if quit {
      return false
   }
   defer dc.queue.Done(key)

   err := dc.syncHandler(ctx, key.(string))
   dc.handleErr(err, key)

   return true
}
```

显然，这里最关键的就是 syncHandler 函数，也就是对事件的具体处理。

发现了一篇文章，写的非常好，所以我不写了（摆烂）

> https://juejin.cn/post/6844903971430072328

