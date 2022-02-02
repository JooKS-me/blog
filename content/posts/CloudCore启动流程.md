---
title: "CloudCore启动流程"
date: 2022-02-02T19:03:17+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeEdge"]
categories: ["云原生"]
draft: false
---

cloudcore的入口在 `~/cloud/cmd/cloudcore`

```go
func main() {
   command := app.NewCloudCoreCommand()
   logs.InitLogs()
   defer logs.FlushLogs()

   if err := command.Execute(); err != nil {
      os.Exit(1)
   }
}
```

这里主要是app.NewCloudCoreCommand()

NewCloudCoreCommand用了cobra框架来编写命令。

### 获取cloudcore配置

```go
opts := options.NewCloudCoreOptions()
```

我们可以从NewCloudCoreOptions方法看出，cloudcore默认会加载 `/etc/kubeedge/config/cloudcore.yaml`中的配置。而指定 `--config` 参数可以指定配置文件

```go
func NewCloudCoreOptions() *CloudCoreOptions {
   return &CloudCoreOptions{
      ConfigFile: path.Join(constants.DefaultConfigDir, "cloudcore.yaml"),
   }
}

func (o *CloudCoreOptions) Flags() (fss cliflag.NamedFlagSets) {
	fs := fss.FlagSet("global")
	fs.StringVar(&o.ConfigFile, "config", o.ConfigFile, "The path to the configuration file. Flags override values in this file.")
	return
}
```

接下来，我们看核心代码Run

### 防御

前面都是一些防御的代码，会对读取的配置进行校验。

```go
verflag.PrintAndExitIfRequested()
flag.PrintMinConfigAndExitIfRequested(v1alpha1.NewMinCloudCoreConfig())
flag.PrintDefaultConfigAndExitIfRequested(v1alpha1.NewDefaultCloudCoreConfig())
flag.PrintFlags(cmd.Flags())

if errs := opts.Validate(); len(errs) > 0 {
   klog.Exit(util.SpliceErrors(errs))
}

config, err := opts.Config()
if err != nil {
   klog.Exit(err)
}
if errs := validation.ValidateCloudCoreConfiguration(config); len(errs) > 0 {
   klog.Exit(util.SpliceErrors(errs.ToAggregate().Errors()))
}

if err := features.DefaultMutableFeatureGate.SetFromMap(config.FeatureGates); err != nil {
    klog.Exit(err)
}
```

### 初始化KubeEdge Client

```go
// To help debugging, immediately log version
klog.Infof("Version: %+v", version.Get())
client.InitKubeEdgeClient(config.KubeAPIConfig)
```

我们来详细看一下InitKubeEdgeClient方法。

```go
func InitKubeEdgeClient(config *cloudcoreConfig.KubeAPIConfig) {
   initOnce.Do(func() {
      kubeConfig, err := clientcmd.BuildConfigFromFlags(config.Master, config.KubeConfig)
      if err != nil {
         klog.Errorf("Failed to build config, err: %v", err)
         os.Exit(1)
      }
      kubeConfig.QPS = float32(config.QPS)
      kubeConfig.Burst = int(config.Burst)

      dynamicClient = dynamic.NewForConfigOrDie(kubeConfig)

      kubeConfig.ContentType = runtime.ContentTypeProtobuf
      kubeClient = kubernetes.NewForConfigOrDie(kubeConfig)

      crdKubeConfig := rest.CopyConfig(kubeConfig)
      crdKubeConfig.ContentType = runtime.ContentTypeJSON
      crdClient = crdClientset.NewForConfigOrDie(crdKubeConfig)
   })
}
```

其中initOnce#Do会确保里面的代码块只执行一次。

剩下的代码就是调用 `client-go` 的API，初始化了dynamicClient、kubeClient、crdClient。

### 确定TunnelPort

```go
// Negotiate TunnelPort for multi cloudcore instances
waitTime := rand.Int31n(10)
time.Sleep(time.Duration(waitTime) * time.Second)
tunnelport, err := NegotiateTunnelPort()
if err != nil {
   panic(err)
}
```

这里由于可能同时存在多个cloudcore，于是随机sleep了一个时间来错开NegotiateTunnelPort的执行时机。

接下来，我们详细看一下NegotiateTunnelPort方法，请看注释。

```go
func NegotiateTunnelPort() (*int, error) {
   kubeClient := client.GetKubeClient()
   // 创建一个叫kubeedge的namespace（如果没创建的话）
   err := httpserver.CreateNamespaceIfNeeded(kubeClient, constants.SystemNamespace)
   ...

   // 拿到namespace为kubeedge的叫tunnelPort的configMap
   tunnelPort, err := kubeClient.CoreV1().ConfigMaps(constants.SystemNamespace).Get(context.TODO(), modules.TunnelPort, metav1.GetOptions{})
   ...

   var record iptables.TunnelPortRecord
   if err == nil {
      recordStr, found := tunnelPort.Annotations[modules.TunnelPortRecordAnnotationKey]
      recordBytes := []byte(recordStr)
      if !found {
         return nil, errors.New("failed to get tunnel port record")
      }

     	// 将configMap解析成一个TunnelPortRecord对象
      if err := json.Unmarshal(recordBytes, &record); err != nil {
         return nil, err
      }

      // 如果当前ip已经存在了一个port，那么直接返回，如果不存在就继续。
      port, found := record.IPTunnelPort[localIP]
      if found {
         return &port, nil
      }

      // 协商出当前IP的port，主要策略是在当前已有的端口的最大值的基础上 + 1，最小是10351
      port = negotiatePort(record.Port)

      // 将这个协商好的port写回k8s的configMap中。
      ...

      return &port, nil
   }

   ...

   return nil, errors.New("failed to negotiate the tunnel port")
}
```

### 初始化globalInformersManager

```go
gis := informers.GetInformersManager()
```

这个globalInformersManager主要是对k8s原生的informers等资源的封装。

```go
func GetInformersManager() Manager {
   once.Do(func() {
      globalInformers = &informers{
         defaultResync:                0,
         keClient:                     client.GetKubeClient(),
         informers:                    make(map[string]cache.SharedIndexInformer),
         crdSharedInformerFactory:     crdinformers.NewSharedInformerFactory(client.GetCRDClient(), 0),
         k8sSharedInformerFactory:     k8sinformer.NewSharedInformerFactory(client.GetKubeClient(), 0),
         dynamicSharedInformerFactory: dynamicinformer.NewFilteredDynamicSharedInformerFactory(client.GetDynamicClient(), 0, v1.NamespaceAll, nil),
      }
   })
   return globalInformers
}
```

### 注册模块

```go
registerModules(config)
```

这里把cloudcore的所有组件都注册了，所谓的注册其实就是子组件的初始化。

```go
// registerModules register all the modules started in cloudcore
func registerModules(c *v1alpha1.CloudCoreConfig) {
   cloudhub.Register(c.Modules.CloudHub)
   edgecontroller.Register(c.Modules.EdgeController)
   devicecontroller.Register(c.Modules.DeviceController)
   synccontroller.Register(c.Modules.SyncController)
   cloudstream.Register(c.Modules.CloudStream, c.CommonConfig)
   router.Register(c.Modules.Router)
   dynamiccontroller.Register(c.Modules.DynamicController)
}
```

### 初始化context

```go
ctx := beehiveContext.GetContext()
```

这个地方的context是KubeEdge的另一个项目beehive中的，KubeEdge使用这个context来在各个组件之间传递信息。

### 初始化IptablesManager

```go
if config.Modules.IptablesManager == nil || config.Modules.IptablesManager.Enable && config.Modules.IptablesManager.Mode == v1alpha1.InternalMode {
   // By default, IptablesManager manages tunnel port related iptables rules
   // The internal mode will share the host network, forward to the stream port.
   streamPort := int(config.Modules.CloudStream.StreamPort)
   go iptables.NewIptablesManager(config.KubeAPIConfig, streamPort).Run(ctx)
}
```

这个IptablesManager回定义iptalbes规则，然后将kubectl logs的请求转发到边缘端，这样k8s的master节点就能查看到边端的日志信息了。

### 启动

```go
core.StartModules() //启动前面注册的模块
gis.Start(ctx.Done()) //ctx.Done()传入了一个chan，可以用于关闭informer
core.GracefulShutdown() //当收到结束信号时，做善后工作，结束运行。
```