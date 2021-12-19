---
title: "Apache Shenyu源码阅读计划（三）Dubbo请求"
date: Tue Jul 27 23:41:19 CST 2021
categories: ["中间件"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["中间件"]
draft: false
---

## 运行项目

首先准备环境，因为这个dubbo的example用的注册中心是zookeeper，那我们首先打开zk。（注意这里zk是给dubbo用的，不是给shenyu用的）

然后开启`admin`、`bootstrap`、`examples.apache.dubbo.service.TestApacheDubboApplication`

启动dubbo的example的时候，发现console咕噜咕噜地输出日志。仔细一看发现是`RegisterUtils#doRegister`打的日志，虎躯一震，因为这个类所在的模块叫`shenyu-register-client-http`。但是仔细想想好像没什么不对，这个应该是shenyu通过HTTP往admin上面写配置，看一下下面的变量值应该就明白了。

![image-20210727221322185](https://img.jooks.cn/img/20210727221322.png)

这里暂时放过，后面再看。

## 发起请求

先到`AlibabaDubboPlugin#doExecute`打一个断点，然后随便找一个接口用postman去访问，比如：`http://localhost:9195/dubbo/findById`（post请求，带上json请求体）。

> 为什么这里是`AlibabaDubboPlugin`而不是Apache的呢？明明我们开的是Apache的example啊！这个不清楚，这里打个标记。

然后来看一看`AlibabaDubboPlugin#doExecute`里面的具体内容。

```java
@Override
protected Mono<Void> doExecute(final ServerWebExchange exchange, final ShenyuPluginChain chain, final SelectorData selector, final RuleData rule) {
    // 获取请求体
    String param = exchange.getAttribute(Constants.PARAM_TRANSFORM);
    // 获取shenyu上下文，里面包括了rpc类型、http请求方法、路径、rpc方法所在的类等信息，具体可以看下面的图
    ShenyuContext shenyuContext = exchange.getAttribute(Constants.CONTEXT);
    assert shenyuContext != null;
    // 获取元数据，这个应该跟admin里面配置的元数据是一个东西
    MetaData metaData = exchange.getAttribute(Constants.META_DATA);
    // 如果metaData有问题就进去（为空/服务方法为空/服务类不存在），然后直接报个错。
    if (!checkMetaData(metaData)) {
        ...
    }
    // 如果请求体有问题，也是直接报个错。
    if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(param)) {
        ...
    }
    // ！这里应该就是去调用dubbo的服务了，等下仔细看。
    Object result = alibabaDubboProxyService.genericInvoker(param, metaData);
    // 整理一下调用的结果
    if (Objects.nonNull(result)) {
        exchange.getAttributes().put(Constants.RPC_RESULT, result);
    } else {
        exchange.getAttributes().put(Constants.RPC_RESULT, Constants.DUBBO_RPC_RESULT_EMPTY);
    }
    exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
    return chain.execute(exchange);
}
```

下面这个图是shenyu上下文的内容。

![image-20210727223326643](https://img.jooks.cn/img/20210727223326.png)

下面这个图是元数据里面的内容。

![image-20210727224148533](https://img.jooks.cn/img/20210727224148.png)

然后来仔细看一看dubbo服务是怎么被调用的，也就是`AlibabaDubboProxyService#genericInvoker`。

```java
public Object genericInvoker(final String body, final MetaData metaData) throws ShenyuException {
    // 1. 从缓存中拿到对应的dubbo服务的ReferenceConfig
    ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
    // 如果拿到的reference是空的，或者里面没有具体服务的方法就进去
    if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
        ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
        // 重新按照元数据构建出reference
        reference = ApplicationConfigCache.getInstance().initRef(metaData);
    }
    // 根据reference拿到具体服务的句柄，这是dubbo的内容
    GenericService genericService = reference.get();
    try {
        Pair<String[], Object[]> pair;
        if (StringUtils.isBlank(metaData.getParameterTypes()) || ParamCheckUtils.dubboBodyIsEmpty(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            // 2. 如果有请求体，那就构建出pair
            pair = bodyParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        // 传入参数，调用远程服务，并返回
        return genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
    } catch (GenericException e) {
        log.error("dubbo invoker have exception", e);
        throw new ShenyuException(e.getExceptionMessage());
    }
}
```

1. **从缓存中拿到对应的dubbo服务的ReferenceConfig**

   这里的缓存是guava的LoadingCache，本质上是一个ConcurrentMap，然后key是rpc的调用路径，value是dubbo服务的引用。

   ```java
   private final LoadingCache<String, ReferenceConfig<GenericService>> cache = CacheBuilder.newBuilder()
               .maximumSize(maxCount)
               .removalListener(notification -> {
                   ReferenceConfig config = (ReferenceConfig<GenericService>) notification.getValue();
                   if (config != null) {
                       try {
                           Class cz = config.getClass();
                           Field field = cz.getDeclaredField("ref");
                           field.setAccessible(true);
                           // After the configuration change, Dubbo destroys the instance, but does not empty it. If it is not handled,
                           // it will get NULL when reinitializing and cause a NULL pointer problem.
                           field.set(config, null);
                       } catch (NoSuchFieldException | IllegalAccessException e) {
                           log.error("modify ref have exception", e);
                       }
                   }
               })
               .build(new CacheLoader<String, ReferenceConfig<GenericService>>() {
                   @Override
                   public ReferenceConfig<GenericService> load(final String key) {
                       return new ReferenceConfig<>();
                   }
               });
   ```

2. **如果有请求体，那就构建出pair**

   我们这里来看一看具体是怎么构建的。`BodyParamResolveService`是一个接口，它是这样描述的`This service is used to construct the parameters required for the dubbo and sofa generalization.`

   那么我们就来看看它在dubbo的具体实现。

   ```java
   public class DubboBodyParamResolveServiceImpl implements BodyParamResolveService {
   
       @Override
       public Pair<String[], Object[]> buildParameter(final String body, final String parameterTypes) {
           return BodyParamUtils.buildParameters(body, parameterTypes);
       }
   }
   ```

   非常简单的一个类，里面调用了`BodyParamUtils#buildParameters`，继续追踪，这就是一个比较复杂的方法了。

   ```java
   public static Pair<String[], Object[]> buildParameters(final String body, final String parameterTypes) {
       List<String> paramNameList = new ArrayList<>();
       List<String> paramTypeList = new ArrayList<>();
   
       // 是否'{'开头 and '}'结尾的话那就进去
       if (isNameMapping(parameterTypes)) {
           // 解析paramTypes
           Map<String, String> paramNameMap = GsonUtils.getInstance().toObjectMap(parameterTypes, String.class);
           paramNameList.addAll(paramNameMap.keySet());
           paramTypeList.addAll(paramNameMap.values());
       } else {
           // 解析body
           Map<String, Object> paramMap = GsonUtils.getInstance().toObjectMap(body);
           paramNameList.addAll(paramMap.keySet());
           paramTypeList.addAll(Arrays.asList(StringUtils.split(parameterTypes, ",")));
       }
   
       // 如果请求体只有一种类型，且不是java开头或者[Ljava开头，那就按照单个参数的方式解析
       if (paramTypeList.size() == 1 && !isBaseType(paramTypeList.get(0))) {
           return buildSingleParameter(body, parameterTypes);
       }
       // 按照多个参数的方式解析，实际上跟单个参数解析是一样的，上面单独拿出去是因为返回Pair不一样
       Map<String, Object> paramMap = GsonUtils.getInstance().toObjectMap(body);
       Object[] objects = paramNameList.stream().map(key -> {
           Object obj = paramMap.get(key);
           if (obj instanceof JsonObject) {
               return GsonUtils.getInstance().convertToMap(obj.toString());
           } else if (obj instanceof JsonArray) {
               return GsonUtils.getInstance().fromList(obj.toString(), Object.class);
           } else {
               return obj;
           }
       }).toArray();
       String[] paramTypes = paramTypeList.toArray(new String[0]);
       return new ImmutablePair<>(paramTypes, objects);
   }
   ```

这样，一个dubbo请求的整个过程就打通了。再来看一下`shenyu-plugin-alibaba-dubbo`模块还有哪些类没有看。

![image-20210727231623460](https://img.jooks.cn/img/20210727231623.png)

额，AlibabaDubboPluginDataHandler还没看。

```java
public class AlibabaDubboPluginDataHandler implements PluginDataHandler {

    @Override
    public void handlerPlugin(final PluginData pluginData) {
        if (null != pluginData && pluginData.getEnabled()) {
            DubboRegisterConfig dubboRegisterConfig = GsonUtils.getInstance().fromJson(pluginData.getConfig(), DubboRegisterConfig.class);
            DubboRegisterConfig exist = Singleton.INST.get(DubboRegisterConfig.class);
            if (Objects.isNull(dubboRegisterConfig)) {
                return;
            }
            if (Objects.isNull(exist) || !dubboRegisterConfig.equals(exist)) {
                // If it is null, initialize it
                ApplicationConfigCache.getInstance().init(dubboRegisterConfig);
                ApplicationConfigCache.getInstance().invalidateAll();
            }
            Singleton.INST.single(DubboRegisterConfig.class, dubboRegisterConfig);
        }
    }

    @Override
    public String pluginNamed() {
        return PluginEnum.DUBBO.getName();
    }
}
```

这里就是获取插件的配置，然后对前面提到的ApplicationConfigCache进行初始化。

还有AlibabaDubboMetaDataSubscriber。

```java
public class AlibabaDubboMetaDataSubscriber implements MetaDataSubscriber {

    private static final ConcurrentMap<String, MetaData> META_DATA = Maps.newConcurrentMap();

    @Override
    public void onSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
            MetaData exist = META_DATA.get(metaData.getPath());
            if (Objects.isNull(exist) || Objects.isNull(ApplicationConfigCache.getInstance().get(metaData.getPath()))) {
                // The first initialization
                ApplicationConfigCache.getInstance().initRef(metaData);
            } else {
                // There are updates, which only support the update of four properties of serviceName rpcExt parameterTypes methodName,
                // because these four properties will affect the call of Dubbo;
                if (!Objects.equals(metaData.getServiceName(), exist.getServiceName())
                        || !Objects.equals(metaData.getRpcExt(), exist.getRpcExt())
                        || !Objects.equals(metaData.getParameterTypes(), exist.getParameterTypes())
                        || !Objects.equals(metaData.getMethodName(), exist.getMethodName())) {
                    ApplicationConfigCache.getInstance().build(metaData);
                }
            }
            META_DATA.put(metaData.getPath(), metaData);
        }
    }

    @Override
    public void unSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            META_DATA.remove(metaData.getPath());
        }
    }
}
```

这个东西还不太明白机制，但是看起来应该是订阅metadata的变化，然后同步更新到ApplicationConfigCache。

Over

最后再来解决前面提到的问题：

> 为什么这里是`AlibabaDubboPlugin`而不是Apache的呢？明明我们开的是Apache的example啊！

因为在Bootstrap的pom文件里面，ApacheDubboPlugin的依赖是被注释掉的，我们只需要注释AlibabaDubboPlugin的依赖，然后开启ApacheDubboPlugin的依赖就行了。

![image-20210727233907098](https://img.jooks.cn/img/20210727233907.png)


