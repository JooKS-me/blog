---
title: "EdgeCore启动流程"
date: 2022-02-03T00:23:01+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","KubeEdge"]
categories: ["云原生"]
draft: false
---

EdgeCore的启动流程跟CloudCore十分类似，所以这篇文章会比较短。

配置文件的默认路径是 `/etc/kubeedge/config/edgecore.yaml`，然后也可以通过 `--config` 指定。

主要看一下RUN

刚开始也是校验读入的配置，然后就跟CloudCore不太一样了。

### 检查环境

```go
// Check the running environment by default
checkEnv := os.Getenv("CHECK_EDGECORE_ENVIRONMENT")
// Force skip check if enable metaserver
if config.Modules.MetaManager.MetaServer.Enable {
   checkEnv = "false"
}
if checkEnv != "false" {
   // Check running environment before run edge core
   if err := environmentCheck(); err != nil {
      klog.Exit(fmt.Errorf("failed to check the running environment: %v", err))
   }
}
```

上面这段代码是判断是否需要检查环境，如果enable了metaserver则不需要检查。

我们来看一下environmentCheck具体是检查了什么。

```go
// environmentCheck check the environment before edgecore start
// if Check failed,  return errors
func environmentCheck() error {
   processes, err := ps.Processes()
   if err != nil {
      return err
   }
  
   for _, process := range processes {
      switch process.Executable() {
      case "kubelet": // if kubelet is running, return error
         return errors.New("kubelet should not running on edge node when running edgecore")
      case "kube-proxy": // if kube-proxy is running, return error
         return errors.New("kube-proxy should not running on edge node when running edgecore")
      }
   }
   return nil
}
```

我们可以得知，当系统中运行着 `kubelet` 和 `kube-proxy` 进程时，就会启动错误。

### 获取Node IP

```go
// Get edge node local ip only when the custiomInterfaceName has been set.
// Defaults to the local IP from the default interface by the default config
if config.Modules.Edged.CustomInterfaceName != "" {
   ip, err := netutil.ChooseBindAddressForInterface(config.Modules.Edged.CustomInterfaceName)
   if err != nil {
      klog.Errorf("Failed to get IP address by custom interface %s, err: %v", config.Modules.Edged.CustomInterfaceName, err)
      os.Exit(1)
   }
   config.Modules.Edged.NodeIP = ip.String()
   klog.Infof("Get IP address by custom interface successfully, %s: %s", config.Modules.Edged.CustomInterfaceName, config.Modules.Edged.NodeIP)
} else {
   if net.ParseIP(config.Modules.Edged.NodeIP) != nil {
      klog.Infof("Use node IP address from config: %s", config.Modules.Edged.NodeIP)
   } else if config.Modules.Edged.NodeIP != "" {
      klog.Errorf("invalid node IP address specified: %s", config.Modules.Edged.NodeIP)
      os.Exit(1)
   } else {
      nodeIP, err := util.GetLocalIP(util.GetHostname())
      if err != nil {
         klog.Errorf("Failed to get Local IP address: %v", err)
         os.Exit(1)
      }
      config.Modules.Edged.NodeIP = nodeIP
      klog.Infof("Get node local IP address successfully: %s", nodeIP)
   }
}
```

如果Edged的配置中有CustomInterfaceName，那么就使用指定的接口来获取IP；

否则，先尝试从 Edged 的 NodeIP 读取；如果 Edged.NodeIP 没有配置，就先获取 hostname，然后根据hostname获取ip。

> 把这段if else写在RUN里面似乎有点不美观，感觉应该封装成一个方法。。

### 注册、启动

```go
registerModules(config)
// start all modules
core.Run()
```

这里跟CloudCore没什么差别。

