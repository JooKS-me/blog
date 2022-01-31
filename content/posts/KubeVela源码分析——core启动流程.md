---
title: "KubeVela源码分析——core启动流程"
date: 2022-01-30T20:23:56+08:00
categories: ["云原生"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["云原生","Kubernetes","KubeVela"]
draft: false
---

> 按照之前的习惯，从启动的地方开始看源码，所以这篇文章将分析kubevela-core的启动流程。

主要看 [kubevala/cmd/core/main.go](https://github.com/oam-dev/kubevela/blob/master/cmd/core/main.go)

### 定义变量

```go
var metricsAddr, logFilePath, leaderElectionNamespace string
var enableLeaderElection, logDebug bool
...
var retryPeriod time.Duration
var enableClusterGateway bool
```

### 定义flag

```go
flag.BoolVar(&useWebhook, "use-webhook", false, "Enable Admission Webhook")
flag.StringVar(&certDir, "webhook-cert-dir", "/k8s-webhook-server/serving-certs", "Admission webhook cert/key dir.")
...
flag.BoolVar(&controllerArgs.EnableCompatibility, "enable-asi-compatibility", false, "enable compatibility for asi")
standardcontroller.AddOptimizeFlags()

flag.Parse()
// setup logging
klog.InitFlags(nil)
if logDebug {
	_ = flag.Set("v", strconv.Itoa(int(commonconfig.LogDebug)))
}
```

这个地方flag的作用可以参考：https://www.jianshu.com/p/a5c11a590567

### 开启pprof服务

```go
if pprofAddr != "" {
   // Start pprof server if enabled
   mux := http.NewServeMux()
   mux.HandleFunc("/debug/pprof/", pprof.Index)
   mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
   mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
   mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
   mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
   pprofServer := http.Server{
      Addr:    pprofAddr,
      Handler: mux,
   }
   klog.InfoS("Starting debug HTTP server", "addr", pprofServer.Addr)

   go func() {
      go func() {
         ctx := context.Background()
         <-ctx.Done()

         ctx, cancelFunc := context.WithTimeout(context.Background(), 60*time.Minute)
         defer cancelFunc()

         if err := pprofServer.Shutdown(ctx); err != nil {
            klog.Error(err, "Failed to shutdown debug HTTP server")
         }
      }()

      if err := pprofServer.ListenAndServe(); !errors.Is(http.ErrServerClosed, err) {
         klog.Error(err, "Failed to start debug HTTP server")
         panic(err)
      }
   }()
}
```

pprof的作用就是性能分析，具体使用请看：https://www.jianshu.com/p/f4690622930d

### 输出kubevela的信息

```go
klog.InfoS("KubeVela information", "version", version.VelaVersion, "revision", version.GitRevision)
klog.InfoS("Disable capabilities", "name", disableCaps)
klog.InfoS("Vela-Core init", "definition namespace", oam.SystemDefinitonNamespace)
```

### 初始化multicluster

```go
// wrapper the round tripper by multi cluster rewriter
if enableClusterGateway {
   if _, err := multicluster.Initialize(restConfig, true); err != nil {
      klog.ErrorS(err, "failed to enable multicluster")
      os.Exit(1)
   }
}
```

这个地方涉及到了ClusterGateway，我们下一篇文章再深入分析。

### 生成一个controller-runtime-manager

```go
mgr, err := ctrl.NewManager(restConfig, ctrl.Options{
   Scheme:                     scheme,
   MetricsBindAddress:         metricsAddr,
   LeaderElection:             enableLeaderElection,
   LeaderElectionNamespace:    leaderElectionNamespace,
   LeaderElectionID:           kubevelaName,
   Port:                       webhookPort,
   CertDir:                    certDir,
   HealthProbeBindAddress:     healthAddr,
   LeaderElectionResourceLock: leaderElectionResourceLock,
   LeaseDuration:              &leaseDuration,
   RenewDeadline:              &renewDeadline,
   RetryPeriod:                &retryPeriod,
   // SyncPeriod is configured with default value, aka. 10h. First, controller-runtime does not
   // recommend use it as a time trigger, instead, it is expected to work for failure tolerance
   // of controller-runtime. Additionally, set this value will affect not only application
   // controller but also all other controllers like definition controller. Therefore, for
   // functionalities like state-keep, they should be invented in other ways.
   NewClient: ctrlClient.DefaultNewControllerClient,
})
```

其中，ctrl#NewManager这个方法的注释写道：

```
// NewManager returns a new Manager for creating Controllers.
```

因此，我们猜测这个manager可以用来创建控制器~

### 健康检查

```go
if err := registerHealthChecks(mgr); err != nil {
   klog.ErrorS(err, "Unable to register ready/health checks")
   os.Exit(1)
}
```

我们来看一下registerHealthChecks这个函数

```go
// registerHealthChecks is used to create readiness&liveness probes
func registerHealthChecks(mgr ctrl.Manager) error {
   klog.Info("Create readiness/health check")
   if err := mgr.AddReadyzCheck("ping", healthz.Ping); err != nil {
      return err
   }
   // TODO: change the health check to be different from readiness check
   if err := mgr.AddHealthzCheck("ping", healthz.Ping); err != nil {
      return err
   }
   return nil
}
```

可以看到，这个地方分别对readiness和health做了ping检查。

这里有个TODO，意思是需要health的检查还需要完善🤔。

### 检查DisabledCapabilities

```go
if err := utils.CheckDisabledCapabilities(disableCaps); err != nil {
   klog.ErrorS(err, "Unable to get enabled capabilities")
   os.Exit(1)
}
```

进入CheckDisabledCapabilities，

```go
// CheckDisabledCapabilities checks whether the disabled capability controllers are valid
func CheckDisabledCapabilities(disableCaps string) error {
   switch disableCaps {
   case common.DisableNoneCaps:
      return nil
   case common.DisableAllCaps:
      return nil
   default:
      for _, c := range strings.Split(disableCaps, ",") {
         if !allBuiltinCapabilities.Contains(c) {
            return fmt.Errorf("%s in disable caps list is not built-in", c)
         }
      }
      return nil
   }
}
```

从上面的代码可以发现，disableCaps的值只能是 `""` 或者 `"all"` 。

至于这个disableCaps是干啥用的🤔，现在从注释能看出这个跟disabled capability controllers有关。不过可以从git的记录看出来，这个是 [#687](https://github.com/oam-dev/kubevela/pull/687) 这个pr加的，我们后面有需要再看吧。

### 设置ApplyMode

```go
switch strings.ToLower(applyOnceOnly) {
case "", "false", string(oamcontroller.ApplyOnceOnlyOff):
   controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOff
   klog.Info("ApplyOnceOnly is disabled")
case "true", string(oamcontroller.ApplyOnceOnlyOn):
   controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOn
   klog.Info("ApplyOnceOnly is enabled, that means workload or trait only apply once if no spec change even they are changed by others")
case string(oamcontroller.ApplyOnceOnlyForce):
   controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyForce
   klog.Info("ApplyOnceOnlyForce is enabled, that means workload or trait only apply once if no spec change even they are changed or deleted by others")
default:
   klog.ErrorS(fmt.Errorf("invalid apply-once-only value: %s", applyOnceOnly),
      "Unable to setup the vela core controller",
      "apply-once-only", "on/off/force, by default it's off")
   os.Exit(1)
}
```

从日志里面可以看出来ApplyMode的作用。

### 设置DiscoveryMapper

```go
dm, err := discoverymapper.New(mgr.GetConfig())
if err != nil {
   klog.ErrorS(err, "Failed to create CRD discovery client")
   os.Exit(1)
}
controllerArgs.DiscoveryMapper = dm
```

那么，Discovery是啥🤔

这玩意儿跟k8s的服务发现有关，DiscoveryClient可以用来拿到k8s里的资源信息，更多可以参考：https://www.jianshu.com/p/d5bc743a48a5

### 设置PackageDiscover

```go
pd, err := packages.NewPackageDiscover(mgr.GetConfig())
if err != nil {
   klog.Error(err, "Failed to create CRD discovery for CUE package client")
   if !packages.IsCUEParseErr(err) {
      os.Exit(1)
   }
}
controllerArgs.PackageDiscover = pd
```

这个PackageDiscover应该是用来从k8s集群中加载CUE包的。

### 安装WebHook

```go
if useWebhook {
   klog.InfoS("Enable webhook", "server port", strconv.Itoa(webhookPort))
   oamwebhook.Register(mgr, controllerArgs)
   if err := waitWebhookSecretVolume(certDir, waitSecretTimeout, waitSecretInterval); err != nil {
      klog.ErrorS(err, "Unable to get webhook secret")
      os.Exit(1)
   }
}
```

### 安装controller

```go
if err = oamv1alpha2.Setup(mgr, controllerArgs); err != nil {
   klog.ErrorS(err, "Unable to setup the oam controller")
   os.Exit(1)
}

if err = standardcontroller.Setup(mgr, disableCaps, controllerArgs); err != nil {
   klog.ErrorS(err, "Unable to setup the vela core controller")
   os.Exit(1)
}
```

这里controller的安装写的好妙啊，比如下面这段代码：

```go
for _, setup := range []func(ctrl.Manager, controller.Args) error{
   application.Setup, traitdefinition.Setup, componentdefinition.Setup, policydefinition.Setup, workflowstepdefinition.Setup,
   applicationconfiguration.Setup,
} {
   if err := setup(mgr, args); err != nil {
      return err
   }
}
```

还能这样@_@!

### 设置环境变量STORAGE_DRIVER

```go
if driver := os.Getenv(system.StorageDriverEnv); len(driver) == 0 {
   // first use system environment,
   err := os.Setenv(system.StorageDriverEnv, storageDriver)
   if err != nil {
      klog.ErrorS(err, "Unable to setup the vela core controller")
      os.Exit(1)
   }
}
klog.InfoS("Use storage driver", "storageDriver", os.Getenv(system.StorageDriverEnv))
```

### 启动controller manager

```go
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
   klog.ErrorS(err, "Failed to run manager")
   os.Exit(1)
}
```

### 总结

KubeVela注释、日志很多，源码看起来很舒服。