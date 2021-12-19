---
title: "ShardingSphere-Agent插件机制梳理"
date: Thu May 20 23:17:48 CST 2021
categories: ["中间件"]
tags: ["中间件"]
draft: false
---

> 首先fork ss，然后git clone，然后git remote add upstream xxxx，然后git pull upstream master保持本地代码与主仓库一致，然后在项目的根目录下执行`mvn clean install`，然后添加虚拟机参数`-javaagent:xxx`，即可正常使用ss支持的可观察性相关插件。

ShardingSphereAgent类是javaagent参数指定的入口，里面写了premain方法，也就是在proxy启动前运行的。

```java
public static void premain(final String arguments, final Instrumentation instrumentation) throws IOException {
    // 1.加载配置文件，默认是在/conf/agent.yaml
    AgentConfiguration agentConfiguration = AgentConfigurationLoader.load();
    // 2.将agent配置放入注册中心
    AgentConfigurationRegistry.INSTANCE.put(agentConfiguration);
    // 3.创建PluginLoader并加载所有插件
    PluginLoader pluginLoader = createPluginLoader();
    // 4.安装AgentBuilder
    setUpAgentBuilder(instrumentation, pluginLoader);
    // 5.安装PluginBootService
    setupPluginBootService(agentConfiguration.getPlugins());
}
```

大致流程如下图（大图可见：[https://img.jooks.cn/img/20210515110232.jpg](https://img.jooks.cn/img/20210515110232.jpg)）：

![ss-agent机制梳理](https://img.jooks.cn/img/20210515110232.jpg)

## 1.加载配置文件

这里跟上一篇文章加载配置文件类似：[https://www.jooks.cn/article/55](https://www.jooks.cn/article/55)

## 2.将agent配置放入注册中心

这里的AgentConfigurationRegistry使用了单例模式（这种实现第一次见。。。喵啊），里面其实是一个ConcurrentHashMap，然后put把agent配置放入了map。

## 3.创建PluginLoader并加载所有插件

```java
private static PluginLoader createPluginLoader() throws IOException {
    PluginLoader result = PluginLoader.getInstance();
    result.loadAllPlugins();
    return result;
}
```

可以发现这里的PluginLoader也是单例模式（但是实现跟AgentConfigurationRegistry不一样）。

且PluginLoader继承自ClassLoader类。

然后重点看一看`result.loadAllPlugins();`

```java
public void loadAllPlugins() throws IOException {
    // 3.1 从/plugins目录下读取jar文件，包装成File类
    File[] jarFiles = AgentPathBuilder.getPluginPath().listFiles(file -> file.getName().endsWith(".jar"));
    if (null == jarFiles) {
        return;
    }
    // 3.2 创建pointMap
    Map<String, PluginInterceptorPoint> pointMap = Maps.newHashMap();
    // 3.3 将配置文件中指定不加载的插件放入一个集合。
    Set<String> ignoredPluginNames = AgentConfigurationRegistry.INSTANCE.get(AgentConfiguration.class).getIgnoredPluginNames();
    // 3.4 将File逐个包装成PluginJar，然后添加到jars列表中。
    try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
        for (File each : jarFiles) {
            outputStream.reset();
            JarFile jar = new JarFile(each, true);
            jars.add(new PluginJar(jar, each));
            log.info("Loaded jar {}.", each.getName());
        }
    }
    // 3.5 通过SPI机制加载各个PluginDefinitionService，并把PluginInterceptorPoint放入pointMap
    loadPluginDefinitionServices(ignoredPluginNames, pointMap);
    // 这里build出来的map是真正immutable的，往里面put东西会报错
    interceptorPointMap = ImmutableMap.<String, PluginInterceptorPoint>builder().putAll(pointMap).build();
}
```

1. 从/plugins目录下获取插件的jar文件，包装成File。当前plugins目录内容如下。

   这个目录是shardingsphere-agent-distribution模块编译后产生的，为什么会在这里出现这些jar包呢？

   因为agent-distribution模块下有一个`shardingsphere-agent-binary-distribution.xml`文件，这里面把agent-plugin模块下各个子模块编译好的jar包引入到这里。

![image-20210514130658781](https://img.jooks.cn/img/20210514130658.png)

2. 创建pointMap

   这里PluginInterceptorPoint类，其内部类Builder可以通过agent机制替换构造方法、普通方法、静态方法。

3. 将配置文件中指定不加载的插件放入一个集合。

4. 将File逐个包装成PluginJar，然后添加到jars列表中。

5. 通过SPI机制加载各个PluginDefinitionService

```java
private void loadPluginDefinitionServices(final Set<String> ignoredPluginNames, final Map<String, PluginInterceptorPoint> pointMap) {
    PluginServiceLoader.newServiceInstances(PluginDefinitionService.class)
            .stream()
            .filter(each -> ignoredPluginNames.isEmpty() || !ignoredPluginNames.contains(each.getType()))
            .forEach(each -> buildPluginInterceptorPointMap(each, pointMap));
}
```

这里的`PluginServiceLoader.newServiceInstances(PluginDefinitionService.class)`用到了Java的SPI机制，会去resources/META-INF/services目录下找到接口的全限定名`org.apache.shardingsphere.agent.spi.definition.PluginDefinitionService`命名的文件，这里以tracing模块的jaeger插件为例，文件内容如下图，里面指出了Service接口的具体实现类。

![image-20210514161851882](https://img.jooks.cn/img/20210514161851.png)

这样就会把`PluginDefinitionService`接口的具体实现类`JaegerPluginDefinitionService`实例化返回。

![image-20210514162116853](https://img.jooks.cn/img/20210514162116.png)

然后再通过filter方法把配置文件中指定ignore的PluginDefinitionService过滤掉，最后将剩下的服务安装并加入到pointMap中，详细过程如下。

```java
private void buildPluginInterceptorPointMap(final PluginDefinitionService pluginDefinitionService, final Map<String, PluginInterceptorPoint> pointMap) {
    AbstractPluginDefinitionService definitionService = (AbstractPluginDefinitionService) pluginDefinitionService;
    // install详细见下面，返回一个PluginInterceptorPoint集合
    definitionService.install().forEach(each -> {
        String target = each.getClassNameOfTarget();
        // 如果对一个目标类定义了多个interceptor，那就把这个类的points都汇总到一起
        if (pointMap.containsKey(target)) {
            PluginInterceptorPoint pluginInterceptorPoint = pointMap.get(target);
            pluginInterceptorPoint.getConstructorPoints().addAll(each.getConstructorPoints());
            pluginInterceptorPoint.getInstanceMethodPoints().addAll(each.getInstanceMethodPoints());
            pluginInterceptorPoint.getClassStaticMethodPoints().addAll(each.getClassStaticMethodPoints());
        } else {
            pointMap.put(target, each);
        }
    });
}
```

这里的install方法代码如下，

```java
public final Collection<PluginInterceptorPoint> install() {
    defineInterceptors();
    return interceptorPointMap.values().stream().map(PluginInterceptorPoint.Builder::install).collect(Collectors.toList());
}
```

return 语句是把interceptorPointMap给转化为一个List。

上面的defineInterceptors()方法则是一个抽象方法，由具体的PluginDefinitionService类实现，下面以JaegerPluginDefinitionService为例。

```java
@Override
public void defineInterceptors() {
    defineInterceptor(COMMAND_EXECUTOR_TASK_ENHANCE_CLASS)
            .aroundInstanceMethod(ElementMatchers.named(COMMAND_EXECUTOR_METHOD_NAME))
            .implement(COMMAND_EXECUTOR_TASK_ADVICE_CLASS)
            .build();
    defineInterceptor(SQL_PARSER_ENGINE_ENHANCE_CLASS)
            .aroundInstanceMethod(ElementMatchers.named(SQL_PARSER_ENGINE_METHOD_NAME))
            .implement(SQL_PARSER_ENGINE_ADVICE_CLASS)
            .build();
    defineInterceptor(JDBC_EXECUTOR_CALLBACK_ENGINE_ENHANCE_CLASS)
            .aroundInstanceMethod(
                    ElementMatchers.named(JDBC_EXECUTOR_METHOD_NAME)
                            .and(ElementMatchers.takesArgument(0, ElementMatchers.named(JDBC_EXECUTOR_UNIT_ENGINE_ENHANCE_CLASS)))
            )
            .implement(JDBC_EXECUTOR_CALLBACK_ADVICE_CLASS)
            .build();
}
```

这个地方给三个原始方法添加了环绕处理（对方法执行前后进行了具体操作，这里还不懂Jaeger的API，暂时放一放），并把目标类名加入到了interceptorPointMap。

到了这里，所有应该加载的插件的PluginInterceptorPoint都放进了pluginLoader实例的interceptorPointMap中。

## 4.setUpAgentBuilder

```java
private static void setUpAgentBuilder(final Instrumentation instrumentation, final PluginLoader pluginLoader) {
    AgentBuilder agentBuilder = new AgentBuilder.Default().with(new ByteBuddy().with(TypeValidation.ENABLED))
            .ignore(ElementMatchers.isSynthetic()).or(ElementMatchers.nameStartsWith("org.apache.shardingsphere.agent."));
    agentBuilder.type(pluginLoader.typeMatcher())
            .transform(new ShardingSphereTransformer(pluginLoader)).with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION).with(new LoggingListener()).installOn(instrumentation);
}
```

这里就是先创建一个忽略 `Synthetic` 和 `以org.apache.shardingsphere.agent.开头的类` 的AgentBuilder实例。

然后把agentBuilder实例作用到之前pluginLoader实例里的interceptorPointMap中包含的类名，然后绑定Transform实例，选择RETRANSFORMATION策略，绑定监听器，绑定instrumentation。

**可以发现这里最重要的是new ShardingSphereTransformer(pluginLoader)，等下再看。**

## 5.setupPluginBootService

```java
private static void setupPluginBootService(final Map<String, PluginConfiguration> pluginConfigurationMap) {
    // 1. 启动所有插件的service
    PluginBootServiceManager.startAllServices(pluginConfigurationMap);
    // 2. 添加jvm关闭钩子，让jvm在关闭前先把service正常关闭
    Runtime.getRuntime().addShutdownHook(new Thread(PluginBootServiceManager::closeAllServices));
}
```

1. 先是启动所有插件的service，代码如下，

   ```java
   public static void startAllServices(final Map<String, PluginConfiguration> pluginConfigurationMap) {
       Set<String> ignoredPluginNames = AgentConfigurationRegistry.INSTANCE.get(AgentConfiguration.class).getIgnoredPluginNames();
       for (Map.Entry<String, PluginConfiguration> entry: pluginConfigurationMap.entrySet()) {
           AgentTypedSPIRegistry.getRegisteredServiceOptional(PluginBootService.class, entry.getKey()).ifPresent(pluginBootService -> {
               try {
                   if (!ignoredPluginNames.isEmpty() && ignoredPluginNames.contains(pluginBootService.getType())) {
                       return;
                   }
                   pluginBootService.start(entry.getValue());
                   // CHECKSTYLE:OFF
               } catch (final Throwable ex) {
                   // CHECKSTYLE:ON
                   log.error("Failed to start service.", ex);
               }
           });
       }
   }
   ```

   这里会调用没有被ignore的插件的pluginBootService的start方法。

2. 添加jvm关闭钩子，让jvm在关闭前先把service正常关闭，代码如下，

   ```java
   public static void closeAllServices() {
       AgentTypedSPIRegistry.getAllRegisteredService(PluginBootService.class).forEach(each -> {
           try {
               each.close();
               // CHECKSTYLE:OFF
           } catch (final Throwable ex) {
               // CHECKSTYLE:ON
               log.error("Failed to close service.", ex);
           }
       });
   }
   ```

1和2中的start和close都由具体的XxPluginBootService实现。

## ShardingSphereTransformer

在setUpAgentBuilder方法里面设置了agent的transformer和listener，这里来详细看看。

#### ShardingSphereTransformer

```java
@Override
public Builder<?> transform(final Builder<?> builder, final TypeDescription typeDescription, final ClassLoader classLoader, final JavaModule module) {
    if (pluginLoader.containsType(typeDescription)) {
        Builder<?> result = builder;
        result = result.defineField(EXTRA_DATA, Object.class, Opcodes.ACC_PRIVATE | Opcodes.ACC_VOLATILE)
                .implement(AdviceTargetObject.class)
                .intercept(FieldAccessor.ofField(EXTRA_DATA));
        PluginInterceptorPoint pluginInterceptorPoint = pluginLoader.loadPluginInterceptorPoint(typeDescription);
        result = interceptorConstructorPoint(typeDescription, pluginInterceptorPoint.getConstructorPoints(), result);
        result = interceptorClassStaticMethodPoint(typeDescription, pluginInterceptorPoint.getClassStaticMethodPoints(), result);
        result = interceptorInstanceMethodPoint(typeDescription, pluginInterceptorPoint.getInstanceMethodPoints(), result);
        return result;
    }
    return builder;
}
```

这里就是设置拦截点。










