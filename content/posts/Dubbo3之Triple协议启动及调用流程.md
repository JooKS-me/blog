---
title: "Dubbo3 Triple协议启动及调用流程"
date: 2022-05-20T00:23:50+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["微服务","Dubbo","RPC","中间件"]
categories: ["中间件"]
draft: false
---

> 我源码分析的文章写得比较烂QAQ
>
> 所以这篇文章只能叫源码阅读过程。

## 调试环境准备

下载 Dubbo 的代码

```shell
git clone https://github.com/apache/dubbo
```

构建代码（如果是 arm 架构机器需要带上 `os.detected.classifier=osx-x86_64` ）

```shell
mvn clean install -DskipTests -Dos.detected.classifier=osx-x86_64
```

构建完成后，Dubbo Compiler 会根据 proto 文件生成代码，位于相应模块的 build 目录下，如下图。

![image-20220518171928845](https://img.jooks.cn/img/202205181719876.png)

右键java目录将生成的代码标记为Ganerated Sources Root。

接着运行zookeeper，在2181端口。

本文将以 `dubbo-demo-triple` 的 `org.apache.dubbo.demo.provider.ApiProvider` 和 `org.apache.dubbo.demo.consumer.ApiConsumer` 为入口，分析 Triple 协议的调用流程。

## 生产者服务启动流程

 `org.apache.dubbo.demo.provider.ApiProvider` 的 main 方法中调用了 `DubboBootstrap#start`，开启了生产者服务。

而bootstrap的start方法主要是调用了 `DefaultApplicationDeployer#start` 。我们接下来主要看这个方法（请看注释）。

```java
/**
 * Start the bootstrap
 *
 * @return
 */
@Override
public Future start() {
    synchronized (startLock) {
        if (isStopping() || isStopped() || isFailed()) {
            throw new IllegalStateException(getIdentifier() + " is stopping or stopped, can not start again");
        }

        try {
            // 检查开启标志
            ...

            // 如果已经开启过了就直接返回
            if (isStarted() && !hasPendingModule) {
                return CompletableFuture.completedFuture(false);
            }

            // pending -> starting : first start app
            // started -> starting : re-start app
            // 设置开启标志
            onStarting();

            // 初始化
            initialize();
              
          	// * 开启逻辑
            doStart();
        } catch (Throwable e) {
            onFailed(getIdentifier() + " start failure", e);
            throw e;
        }

        return startFuture;
    }
}
```

这里最重要的是 doStart 方法。

doStart -> startModules -> moduleModel.getDeployer().start();

```java
// DefaultModuleDeployer#start

@Override
public synchronized Future start() throws IllegalStateException {
    if (isStopping() || isStopped() || isFailed()) {
        throw new IllegalStateException(getIdentifier() + " is stopping or stopped, can not start again");
    }

    try {
        if (isStarting() || isStarted()) {
            return startFuture;
        }

        // 标记开始
        onModuleStarting();

        // 初始化
        applicationDeployer.initialize();
        initialize();

        // 1. 暴露服务（消费者不会调用）
        exportServices();

        // prepare application instance
        // exclude internal module to avoid wait itself
        // 先去开启子模块
        if (moduleModel != moduleModel.getApplicationModel().getInternalModule()) {
            applicationDeployer.prepareInternalModule();
        }

        // 这个地方生产者不会调用
        referServices();

        // 设置已经开启
        ...
    } catch (Throwable e) {
        onModuleFailed(getIdentifier() + " start failed: " + e, e);
        throw e;
    }
    return startFuture;
}
```

1. 暴露服务的代码如下所示。

   ```java
   private void exportServiceInternal(ServiceConfigBase sc) {
       ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
       if (!serviceConfig.isRefreshed()) {
           // 刷新配置
           serviceConfig.refresh();
       }
       if (sc.isExported()) {
           return;
       }
       if (exportAsync || sc.shouldExportAsync()) {
           // 异步暴露
           ExecutorService executor = executorRepository.getServiceExportExecutor();
           CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
               try {
                   if (!sc.isExported()) {
                       sc.export();
                       exportedServices.add(sc);
                   }
               } catch (Throwable t) {
                   logger.error(getIdentifier() + " export async catch error : " + t.getMessage(), t);
               }
           }, executor);
   
           asyncExportingFutures.add(future);
       } else {
           // 同步暴露
           if (!sc.isExported()) {
               // * 这个地方会把服务信息写到注册中心
               sc.export();
               exportedServices.add(sc);
           }
       }
   }
   ```

   上面 * 的地方有点复杂。。。核心调用链路如下：

   ServiceConfig#export  -> ServiceConfig#doExporter -> ServiceConfig#doExportUrls -> ServiceConfig#doExportUrlsFor1Protocol -> ServiceConfig#exportUrl -> ServiceConfig#exportRemote -> ServiceConfig#doExportUrl，这个地方是我觉得服务者暴露最最最最重要的一个方法。

   吐槽一下，这个地方的调用链路，太绕了吧 QAQ... 迷路了无数次

   ```java
   private void doExportUrl(URL url, boolean withMetaData) {
       Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
       if (withMetaData) {
           invoker = new DelegateProviderMetaDataInvoker(invoker, this);
       }
       Exporter<?> exporter = protocolSPI.export(invoker);
       exporters.add(exporter);
   }
   ```

   这里的doExportUrl，首先利用 proxyFactory 得到一个 invoker 对象，再通过 DelegateProviderMetaDataInvoker 包装一下（不重要），然后利用 Protocol 类对象的 export 方法导出 invoker（重点）。

   - TripleProtocol#export -> PortUnificationExchanger#bind -> PortUnificationServer#bind -> PortUnificationServer#doOpen

     ```java
     protected void doOpen() {
         bootstrap = new ServerBootstrap();
     
         bossGroup = NettyEventLoopFactory.eventLoopGroup(1, EVENT_LOOP_BOSS_POOL_NAME);
         workerGroup = NettyEventLoopFactory.eventLoopGroup(
             getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
             EVENT_LOOP_WORKER_POOL_NAME);
     
         final boolean enableSsl = getUrl().getParameter(SSL_ENABLED_KEY, false);
         final SslContext sslContext;
         if (enableSsl) {
             sslContext = SslContexts.buildServerSslContext(url);
         } else {
             sslContext = null;
         }
         bootstrap.group(bossGroup, workerGroup)
             .channel(NettyEventLoopFactory.serverSocketChannelClass())
             .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
             .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
             .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) throws Exception {
                     // Do not add idle state handler here, because it should be added in the protocol handler.
                     final ChannelPipeline p = ch.pipeline();
                     final PortUnificationServerHandler puHandler;
                     puHandler = new PortUnificationServerHandler(url, sslContext, true, protocols,
                         channels);
                     p.addLast("negotiation-protocol", puHandler);
                 }
             });
         // bind
     
         String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
         int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
         if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
             bindIp = ANYHOST_VALUE;
         }
         InetSocketAddress bindAddress = new InetSocketAddress(bindIp, bindPort);
         ChannelFuture channelFuture = bootstrap.bind(bindAddress);
         channelFuture.syncUninterruptibly();
         channel = channelFuture.channel();
     }
     ```

     这里调用了netty的相关api，建立tcp监听，而netty服务端最重要的是各种handler，这里就是 `PortUnificationServerHandler`

   - 这个handler主要看detect方法。

   - ```java
     protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
         throws Exception {
         // Will use the first five bytes to detect a protocol.
         if (in.readableBytes() < 5) {
             return;
         }
     
         if (isSsl(in)) {
             enableSsl(ctx);
         } else {
             for (final WireProtocol protocol : protocols) {
                 in.markReaderIndex();
                 final ProtocolDetector.Result result = protocol.detector().detect(ctx, in);
                 in.resetReaderIndex();
                 switch (result) {
                     case UNRECOGNIZED:
                         continue;
                     case RECOGNIZED:
                         protocol.configServerPipeline(url, ctx.pipeline(), sslCtx);
                         ctx.pipeline().remove(this);
                     case NEED_MORE_DATA:
                         return;
                     default:
                         return;
                 }
             }
             byte[] preface = new byte[in.readableBytes()];
             in.readBytes(preface);
             Set<String> supported = url.getApplicationModel()
                 .getExtensionLoader(WireProtocol.class)
                 .getSupportedExtensions();
             LOGGER.error(String.format("Can not recognize protocol from downstream=%s . "
                     + "preface=%s protocols=%s", ctx.channel().remoteAddress(),
                 Bytes.bytes2hex(preface),
                 supported));
     
             // Unknown protocol; discard everything and close the connection.
             in.clear();
             ctx.close();
         }
     }
     ```

     经过ssl、长度等校验后，进入 `protocol.configServerPipeline(url, ctx.pipeline(), sslCtx);`

     ```java
     public void configServerPipeline(URL url, ChannelPipeline pipeline, SslContext sslContext) {
         final List<HeaderFilter> headFilters;
         if (filtersLoader != null) {
             headFilters = filtersLoader.getActivateExtension(url,
                 HEADER_FILTER_KEY);
         } else {
             headFilters = Collections.emptyList();
         }
         final Http2FrameCodec codec = Http2FrameCodecBuilder.forServer()
             .gracefulShutdownTimeoutMillis(10000)
             .initialSettings(new Http2Settings().headerTableSize(
                     config.getInt(H2_SETTINGS_HEADER_TABLE_SIZE_KEY, DEFAULT_SETTING_HEADER_LIST_SIZE))
                 .maxConcurrentStreams(
                     config.getInt(H2_SETTINGS_MAX_CONCURRENT_STREAMS_KEY, Integer.MAX_VALUE))
                 .initialWindowSize(
                     config.getInt(H2_SETTINGS_INITIAL_WINDOW_SIZE_KEY, DEFAULT_WINDOW_INIT_SIZE))
                 .maxFrameSize(config.getInt(H2_SETTINGS_MAX_FRAME_SIZE_KEY, DEFAULT_MAX_FRAME_SIZE))
                 .maxHeaderListSize(config.getInt(H2_SETTINGS_MAX_HEADER_LIST_SIZE_KEY,
                     DEFAULT_MAX_HEADER_LIST_SIZE)))
             .frameLogger(SERVER_LOGGER)
             .build();
         final Http2MultiplexHandler handler = new Http2MultiplexHandler(
             new ChannelInitializer<Channel>() {
                 @Override
                 protected void initChannel(Channel ch) {
                     final ChannelPipeline p = ch.pipeline();
                     p.addLast(new TripleCommandOutBoundHandler());
                     p.addLast(new TripleHttp2FrameServerHandler(frameworkModel, lookupExecutor(url),
                         headFilters));
                 }
             });
         pipeline.addLast(codec, new TripleServerConnectionHandler(), handler,
             new TripleTailHandler());
     }
     ```

     这里加入了几个handler，分别是：EpollServerSocketChannel/NioServerSocketChannel、Http2FrameCodec、TripleServerConnectionHandler、Http2MultiplexHandler (包含TripleCommandOutBoundHandler、TripleHttp2FrameServerHandler)、TripleTailHandler

     - Http2FrameCodec（Netty自带）

       它的主要作用是将HTTP/2中的frames和Http2Frame对象进行映射。Http2Frame是netty中对应所有http2 frame的封装，这样就可以在后续的handler中专注于处理Http2Frame对象即可，从而摆脱了http2协议的各种细节，可以减少使用者的工作量。[1]

       在后续的handler中，我们可以看到各种 Frame ，应该就是经过这个handler的处理

     - TripleServerConnectionHandler

       这个handler没什么特别，就是对 Http2PingFrame、Http2GoAwayFrame 等特殊 Frame 做处理。

     - Http2MultiplexHandler

       这个handler加载了TripleCommandOutBoundHandler、TripleHttp2FrameServerHandler

       TripleHttp2FrameServerHandler 继承了 ChannelDuplexHandler，是 ChannelInboundHandler 和 ChannelOutboundHandler 的混合。下面是它的 channelRead 方法。

       ```java
       public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
           if (msg instanceof Http2HeadersFrame) {
               onHeadersRead(ctx, (Http2HeadersFrame) msg);
           } else if (msg instanceof Http2DataFrame) {
               onDataRead(ctx, (Http2DataFrame) msg);
           } else if (msg instanceof ReferenceCounted) {
               // ignored
               ReferenceCountUtil.release(msg);
           }
       }
       ```

       会分别对http2 header 帧和 数据帧做处理。

       ```java
       public void onHeadersRead(ChannelHandlerContext ctx, Http2HeadersFrame msg) throws Exception {
           TripleServerStream tripleServerStream = new TripleServerStream(ctx.channel(),
               frameworkModel, executor,
               pathResolver, acceptEncoding, filters);
           ctx.channel().attr(SERVER_STREAM_KEY).set(tripleServerStream);
           tripleServerStream.transportObserver.onHeader(msg.headers(), msg.isEndStream());
       }
       ```

       ```java
       public void onDataRead(ChannelHandlerContext ctx, Http2DataFrame msg) throws Exception {
           final TripleServerStream tripleServerStream = ctx.channel().attr(SERVER_STREAM_KEY)
               .get();
           tripleServerStream.transportObserver.onData(msg.content(), msg.isEndStream());
       }
       ```

       onHeadersRead 新建了一个 `TripleServerStream` 对象，然后将其放进上下文中。

       onDataRead 将给定的数据添加到 deframer 并尝试传递给 listener。这个地方的 listener 比较复杂，后面再看，总之就是把消息吃进来了。（*最后面会再来详细看）

     - TripleTailHandler

       处理未处理的消息以避免内存泄漏和 netty 的 unhandled exception

## 消费者服务引用

消费者启动的核心是 `referenceConfig#get()`。

```java
@Override
public T get() {
    if (destroyed) {
        throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
    }

    if (ref == null) {
        // ensure start module, compatible with old api usage
        getScopeModel().getDeployer().start();

        synchronized (this) {
            if (ref == null) {
                init();
            }
        }
    }

    return ref;
}
```

这里主要是调用了 `init` 方法进行初始化。

主要逻辑：init -> createProxy -> ReferenceConfig#createInvokerForRemote -> TripleProtocol#refer

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    ExecutorService streamExecutor = getOrCreateStreamExecutor(
        url.getOrDefaultApplicationModel());
    TripleInvoker<T> invoker = new TripleInvoker<>(type, url, acceptEncodings,
        connectionManager, invokers, streamExecutor);
    invokers.add(invoker);
    return invoker;
}
```

这里将远程服务抽象成了一个invoker对象

接下来的主要逻辑：TripleInvoker的构造方法 -> connectionManager.connect(url)

```java
@Override
public Connection connect(URL url) {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    return connections.compute(url.getAddress(), (address, conn) -> {
        if (conn == null) {
            final Connection created = new Connection(url);
            created.getClosePromise().addListener(future -> connections.remove(address, created));
            return created;
        } else {
            conn.retain();
            return conn;
        }
    });
}
```

在 new Connection(url) 这一行，调用了 Connection 的构造方法

```java
public Connection(URL url) {
    url = ExecutorUtil.setThreadName(url, "DubboClientHandler");
    url = url.addParameterIfAbsent(THREADPOOL_KEY, DEFAULT_CLIENT_THREADPOOL);
    this.url = url;
    this.protocol = ExtensionLoader.getExtensionLoader(WireProtocol.class)
        .getExtension(url.getProtocol());
    this.connectTimeout = url.getPositiveParameter(Constants.CONNECT_TIMEOUT_KEY,
        Constants.DEFAULT_CONNECT_TIMEOUT);
    this.remote = getConnectAddress();
    this.bootstrap = create();
}
```

这里的 create 代码如下。

```java
private Bootstrap create() {
        final Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(NettyEventLoopFactory.NIO_EVENT_LOOP_GROUP.get())
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .remoteAddress(remote)
            .channel(socketChannelClass());

        final ConnectionHandler connectionHandler = new ConnectionHandler(this);
        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectTimeout);
        bootstrap.handler(new ChannelInitializer<SocketChannel>() {

            @Override
            protected void initChannel(SocketChannel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                SslContext sslContext = null;
                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    pipeline.addLast("negotiation", new SslClientTlsHandler(url));
                }

                //.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                // TODO support IDLE
//                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
                pipeline.addLast(connectionHandler);
                protocol.configClientPipeline(url, pipeline, sslContext);
                // TODO support Socks5
            }
        });
        return bootstrap;
    }
```

这里设置了 netty 客户端。

这里添加的 handler 有：EpollSocketChannel /。NioSocketChannel、SslClientTlsHandler、ConnectionHandler、Http2FrameCodec、Http2MultiplexHandler、TripleTailHandler，后面三个是 `protocol.configClientPipeline(url, pipeline, sslContext)` 添加的。

- SslClientTlsHandler

  这个 handler 就是在握手的时候判断有没有出错，然后进行响应处理。

- ConnectionHandler

  主要用来处理连接相关的，比如断连了就尝试重连。

- Http2FrameCodec

  这个跟服务端的一样，用来将HTTP/2中的frames和Http2Frame对象进行映射。

这样其实就是把 bootstrap 层层包装，包装进 invoker

回到 ReferenceConfig#init -> ReferenceConfig#createProxy。

在 refer 完成后，接着根据 invoker 创建代理对象

```java
// create service proxy
return (T) proxyFactory.getProxy(invoker, ProtocolUtils.isGeneric(generic));
```

这里经过尝试会发现进入 AbstractProxyFactory#getProxy。另外这里的 invoker 大概长下面这样。

![image-20220519214226513](https://img.jooks.cn/img/202205192142592.png)

```java
@Override
public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    // when compiling with native image, ensure that the order of the interfaces remains unchanged
    LinkedHashSet<Class<?>> interfaces = new LinkedHashSet<>();
    ClassLoader classLoader = getClassLoader(invoker);

    String config = invoker.getUrl().getParameter(INTERFACES);
    if (StringUtils.isNotEmpty(config)) {
        String[] types = COMMA_SPLIT_PATTERN.split(config);
        for (String type : types) {
            try {
                interfaces.add(ReflectUtils.forName(classLoader, type));
            } catch (Throwable e) {
                // ignore
            }

        }
    }

    Class<?> realInterfaceClass = null;
    if (generic) {
        try {
            // find the real interface from url
            String realInterface = invoker.getUrl().getParameter(Constants.INTERFACE);
            realInterfaceClass = ReflectUtils.forName(classLoader, realInterface);
            interfaces.add(realInterfaceClass);
        } catch (Throwable e) {
            // ignore
        }

        if (GenericService.class.equals(invoker.getInterface()) || !GenericService.class.isAssignableFrom(invoker.getInterface())) {
            interfaces.add(com.alibaba.dubbo.rpc.service.GenericService.class);
        }
    }

    interfaces.add(invoker.getInterface());
    interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));

    try {
        return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
    } catch (Throwable t) {
        if (generic) {
            if (realInterfaceClass != null) {
                interfaces.remove(realInterfaceClass);
            }
            interfaces.remove(invoker.getInterface());

            logger.error("Error occur when creating proxy. Invoker is in generic mode. Trying to create proxy without real interface class.", t);
            return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
        } else {
            throw t;
        }
    }
}
```

这个地方收集到的 interfaces 如下图。第 2、3 个是 interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES)) 加进来的。

（打断点第一次拦截到）

![image-20220519214840214](https://img.jooks.cn/img/202205192148273.png)

（第二次拦截到）

![image-20220519215149175](https://img.jooks.cn/img/202205192151205.png)

最终调用 getProxy 方法，这里有 javassist 和 jdk 两种实现。

![image-20220519215101835](https://img.jooks.cn/img/202205192151868.png)

我还不清楚什么时机用哪个，这里默认是 javassist ，如果 javassist 失败了会尝试 jdk 代理。

JavassistProxyFactory#getProxy -> Proxy#getProxy

```java
public static Proxy getProxy(Class<?>... ics) {
    if (ics.length > MAX_PROXY_COUNT) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // ClassLoader from App Interface should support load some class from Dubbo
    ClassLoader cl = ics[0].getClassLoader();
    ProtectionDomain domain = ics[0].getProtectionDomain();

    // use interface class name list as key.
    String key = buildInterfacesKey(cl, ics);

    // get cache by class loader.
    final Map<String, Proxy> cache;
    synchronized (PROXY_CACHE_MAP) {
        cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new ConcurrentHashMap<>());
    }

    Proxy proxy = cache.get(key);
    if (proxy == null) {
        synchronized (ics[0]) {
            proxy = cache.get(key);
            if (proxy == null) {
                // create Proxy class.
                proxy = new Proxy(buildProxyClass(cl, ics, domain));
                cache.put(key, proxy);
            }
        }
    }
    return proxy;
}
```

这里的 buildProxyClass 方法可以看见用到了很多 javassist 的 API 做字节码增强，手动实现代理类 （orz，好强好强）

代理类构造完成后，利用反射 API 构造实例，然后返回。

## 客户端调用流程

> 现在问题来了，我们在客户端调用实例，为什么会调用远程服务呢？
>
> 肯定是代理类上动了手脚！那么动了什么手脚呢？

其实，我对这里 javassist 实现代理不太懂，但是感觉跟 jdk 代理应该是差不多的，二者都有一个 InvokerInvocationHandler，对 invoker 进行了封装。然后这个 InvokerInvocationHandler 继承了 jdk 的原生类 InvocationHandler。

当使用代理类时就会调用 InvocationHandler 的 invoker 方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 如果调用了 Object 类的方法则直接调用
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    if (parameterTypes.length == 0) {
        if ("toString".equals(methodName)) {
            return invoker.toString();
        } else if ("$destroy".equals(methodName)) {
            invoker.destroy();
            return null;
        } else if ("hashCode".equals(methodName)) {
            return invoker.hashCode();
        }
    } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
        return invoker.equals(args[0]);
    }
  
    // 如果不是一些常见方法的话，就是远程调用了
    // 根据调用信息构建一个 RpcInvocation 对象
    RpcInvocation rpcInvocation = new RpcInvocation(serviceModel, method.getName(), invoker.getInterface().getName(), protocolServiceKey, method.getParameterTypes(), args);

    if (serviceModel instanceof ConsumerModel) {
        rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);
        rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));
    }
    
    // 执行远程调用逻辑
    return InvocationUtil.invoke(invoker, rpcInvocation);
}
```

InvocationUtil.invoke(invoker, rpcInvocation) 里面还会统计一些 metrics 信息。

InvocationUtil#invoke -> MigrationInvoker#invoke -> MockClusterInvoker#invoke -> AbstractCluster$ClusterFilterInvoker#invoke -> ... -> TripleInvoker#doInvoke

中间经过了非常非常复杂的调用链路，在 TripleInvoker 调用前，会先经过 filter 链，然后各种 Invoker。（发现这条链路的方法是自下而上，预判会到这个位置，然后在 TripleInvoker#doInvoke 打个断点，然后就可以看到整个调用链路了。这样比起自上而下会简单很多。）

```java
@Override
protected Result doInvoke(final Invocation invocation) {
    if (!connection.isAvailable()) {
        CompletableFuture<AppResponse> future = new CompletableFuture<>();
        RpcException exception = TriRpcStatus.UNAVAILABLE.withDescription(
                String.format("upstream %s is unavailable", getUrl().getAddress()))
            .asException();
        future.completeExceptionally(exception);
        return new AsyncRpcResult(future, invocation);
    }

    ConsumerModel consumerModel = (ConsumerModel) (invocation.getServiceModel() != null
        ? invocation.getServiceModel() : getUrl().getServiceModel());
    ServiceDescriptor serviceDescriptor = consumerModel.getServiceModel();
    final MethodDescriptor methodDescriptor = serviceDescriptor.getMethod(
        invocation.getMethodName(),
        invocation.getParameterTypes());
    ClientCall call = new TripleClientCall(connection, streamExecutor,
        getUrl().getOrDefaultFrameworkModel());

    AsyncRpcResult result;
    try {
        switch (methodDescriptor.getRpcType()) {
            case UNARY:
                result = invokeUnary(methodDescriptor, invocation, call);
                break;
            case SERVER_STREAM:
                result = invokeServerStream(methodDescriptor, invocation, call);
                break;
            case CLIENT_STREAM:
            case BI_STREAM:
                result = invokeBiOrClientStream(methodDescriptor, invocation, call);
                break;
            default:
                throw new IllegalStateException("Can not reach here");
        }
        return result;
    } catch (Throwable t) {
        final TriRpcStatus status = TriRpcStatus.INTERNAL.withCause(t)
            .withDescription("Call aborted cause client exception");
        RpcException e = status.asException();
        try {
            call.cancelByLocal(e);
        } catch (Throwable t1) {
            LOGGER.error("Cancel triple request failed", t1);
        }
        CompletableFuture<AppResponse> future = new CompletableFuture<>();
        future.completeExceptionally(e);
        return new AsyncRpcResult(future, invocation);
    }
}
```

```java
AsyncRpcResult invokeUnary(MethodDescriptor methodDescriptor, Invocation invocation,
    ClientCall call) {
    ExecutorService callbackExecutor = getCallbackExecutor(getUrl(), invocation);
    int timeout = calculateTimeout(invocation, invocation.getMethodName());
    invocation.setAttachment(TIMEOUT_KEY, timeout);
    final AsyncRpcResult result;
    DeadlineFuture future = DeadlineFuture.newFuture(getUrl().getPath(),
        methodDescriptor.getMethodName(), getUrl().getAddress(), timeout, callbackExecutor);

    RequestMetadata request = createRequest(methodDescriptor, invocation, timeout);

    final Object pureArgument;
    if (methodDescriptor.getParameterClasses().length == 2
        && methodDescriptor.getParameterClasses()[1].isAssignableFrom(
        StreamObserver.class)) {
        StreamObserver<Object> observer = (StreamObserver<Object>) invocation.getArguments()[1];
        future.whenComplete((r, t) -> {
            if (t != null) {
                observer.onError(t);
                return;
            }
            if (r.hasException()) {
                observer.onError(r.getException());
                return;
            }
            observer.onNext(r.getValue());
            observer.onCompleted();
        });
        pureArgument = invocation.getArguments()[0];
        result = new AsyncRpcResult(CompletableFuture.completedFuture(new AppResponse()),
            invocation);
    } else {
        if (methodDescriptor instanceof StubMethodDescriptor) {
            pureArgument = invocation.getArguments()[0];
        } else {
            pureArgument = invocation.getArguments();
        }
        result = new AsyncRpcResult(future, invocation);
        result.setExecutor(callbackExecutor);
        FutureContext.getContext().setCompatibleFuture(future);
    }
    ClientCall.Listener callListener = new UnaryClientCallListener(future);

    final StreamObserver<Object> requestObserver = call.start(request, callListener);
    requestObserver.onNext(pureArgument);
    requestObserver.onCompleted();
    return result;
}
```

即使这里是 Unary 调用方式，也会涉及到 StreamObserver，通过 requestObserver.onNext 来发送数据，然后异步返回。

如果是同步调用，那么会在 AbstractInvoker#waitForResultIfSync 中根据等待返回（有超时时间）。

接下来探究一下 StreamObserver 是什么东西。

## StreamObserver 探究

```java
ClientCall.Listener callListener = new UnaryClientCallListener(future);

final StreamObserver<Object> requestObserver = call.start(request, callListener);
requestObserver.onNext(pureArgument);
requestObserver.onCompleted();
return result;
```

这是 triple 协议远程调用 (unary) 的最后面的一段代码。

这里的 call#start 返回了一个 StreamObserver 对象，我们重点关注一下。

```java
@Override
public StreamObserver<Object> start(RequestMetadata metadata,
    ClientCall.Listener responseListener) {
    this.requestMetadata = metadata;
    this.listener = responseListener;
    this.stream = new TripleClientStream(frameworkModel, executor, connection.getChannel(),
        this);
    return new ClientCallToObserverAdapter<>(this);
}
```

这里返回了一个 ClientCallToObserverAdapter，ClientCallToObserverAdapter 的继承关系图如下。

![image-20220519235220736](https://img.jooks.cn/img/202205192352800.png)

那么后续调用 onNext 方法时，就是调用 ClientCallToObserverAdapter 实现的 onNext 方法。

```java
@Override
public void onNext(Object data) {
    if (terminated) {
        throw new IllegalStateException(
            "Stream observer has been terminated, no more data is allowed");
    }
    call.sendMessage(data);
}
```

接下来看一看 TripleClientCall#sendMessage

```java
@Override
public void sendMessage(Object message) {
    if (canceled) {
        throw new IllegalStateException("Call already canceled");
    }
    if (!headerSent) {
        headerSent = true;
        stream.sendHeader(requestMetadata.toHeaders());
    }
    final byte[] data;
    try {
        data = requestMetadata.packableMethod.packRequest(message);
        int compressed =
            Identity.MESSAGE_ENCODING.equals(requestMetadata.compressor.getMessageEncoding())
                ? 0 : 1;
        final byte[] compress = requestMetadata.compressor.compress(data);
        stream.sendMessage(compress, compressed, false)
            .addListener(f -> {
                if (!f.isSuccess()) {
                    cancelByLocal(f.cause());
                }
            });
    } catch (Throwable t) {
        LOGGER.error(String.format("Serialize triple request failed, service=%s method=%s",
            requestMetadata.service,
            requestMetadata.method), t);
        cancelByLocal(t);
        listener.onClose(TriRpcStatus.INTERNAL.withDescription("Serialize request failed")
            .withCause(t), null);
    }
}
// stream listener end
```

首先，如果还没发送 header，就先发送 header。

然后，这里将 message 转化成了一个 byte 数组，然后进行压缩处理。

然后调用一个 TripleClientStream 对象的 sendMessage 方法发送数据。

```java
@Override
public ChannelFuture sendMessage(byte[] message, int compressFlag, boolean eos) {
    final DataQueueCommand cmd = DataQueueCommand.createGrpcCommand(message, false,
        compressFlag);
    return this.writeQueue.enqueue(cmd)
        .addListener(future -> {
                if (!future.isSuccess()) {
                    cancelByLocal(
                        TriRpcStatus.INTERNAL.withDescription("Client write message failed")
                            .withCause(future.cause())
                    );
                    transportException(future.cause());
                }
            }
        );
}
```

这里先将数据封装成一个 DataQueueCommand 对象，然后放入 writeQueue 队列。

然后这个队列里面的任务会异步执行，因为在 enqueue 方法的最后会执行 scheduleFlush 方法，执行 flush 操作。

```java
public void scheduleFlush() {
    if (scheduled.compareAndSet(false, true)) {
        channel.eventLoop().execute(this::flush);
    }
}

private void flush() {
    try {
        QueuedCommand cmd;
        int i = 0;
        boolean flushedOnce = false;
        while ((cmd = queue.poll()) != null) {
            // 见下面 QueuedCommand#run 方法
            cmd.run(channel);
            i++;
            if (i == DEQUE_CHUNK_SIZE) {
                i = 0;
                channel.flush();
                flushedOnce = true;
            }
        }
        if (i != 0 || !flushedOnce) {
            channel.flush();
        }
    } finally {
        scheduled.set(false);
        if (!queue.isEmpty()) {
            scheduleFlush();
        }
    }
}
```

```java
// QueuedCommand#run
public void run(Channel channel) {
    if (channel.isActive()) {
        channel.write(this, promise);
    } else {
        promise.trySuccess();
    }
}
```

往 channel 中写数据后，flush channel。

## 流调用原理探究（包含反压实现）

Dubbo3 的 Triple 协议有三种调用模式：unary、server_stream (服务端流)、bi_stream (双向流) 。有了上面的基础后，看 Stream RPC 这块就比较简单了。

```java
AsyncRpcResult invokeServerStream(MethodDescriptor methodDescriptor, Invocation invocation,
    ClientCall call) {
    RequestMetadata request = createRequest(methodDescriptor, invocation, null);
    // 多了一个 responseObserver
    StreamObserver<Object> responseObserver = (StreamObserver<Object>) invocation.getArguments()[1];
    final StreamObserver<Object> requestObserver = streamCall(call, request, responseObserver);
    requestObserver.onNext(invocation.getArguments()[0]);
    requestObserver.onCompleted();
    return new AsyncRpcResult(CompletableFuture.completedFuture(new AppResponse()), invocation);
}
```

这里跟上面 unary 方式不同之处是：从第二个参数取了 responseObserver。这样，在客户端实现一个 StreamObserver，然后作为第二个参数传给方法即可对服务端流进行消费。比如官网给的demo：

```java
delegate.sayHelloServerStream("server stream", new StreamObserver<String>() {
    @Override
    public void onNext(String data) {
        System.out.println(data);
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onCompleted() {
        System.out.println("onCompleted");
    }
});
```

然后生产者就可以发消息。

```java
@Override
public void greetServerStream(String request, StreamObserver<String> response) {
    for (int i = 0; i < 10; i++) {
        response.onNext("hello," + request);
    }
    response.onCompleted();
}
```

所以，我们发现流调用跟普通调用的区别在使用了 streamCall。

```java
StreamObserver<Object> streamCall(ClientCall call,
    RequestMetadata metadata,
    StreamObserver<Object> responseObserver) {
    if (responseObserver instanceof CancelableStreamObserver) {
        final CancellationContext context = new CancellationContext();
        ((CancelableStreamObserver<Object>) responseObserver).setCancellationContext(context);
        context.addListener(context1 -> call.cancelByLocal(new IllegalStateException("Canceled by app")));
    }
    ObserverToClientCallListenerAdapter listener = new ObserverToClientCallListenerAdapter(
        responseObserver);
    return call.start(metadata, listener);
}
```

 在 streamCall 中，将消费者用户定义的 responseObserver (StreamObserver) 封装成了 ObserverToClientCallListenerAdapter（与 unray 的不同之处），然后调用 TripleClientCall 的 start 方法。

```java
@Override
public StreamObserver<Object> start(RequestMetadata metadata,
    ClientCall.Listener responseListener) {
    this.requestMetadata = metadata;
    this.listener = responseListener;
    this.stream = new TripleClientStream(frameworkModel, executor, connection.getChannel(),
        this);
    return new ClientCallToObserverAdapter<>(this);
}
```

接下来看一下 TripleClientStream 的构造方法。

```java
public TripleClientStream(FrameworkModel frameworkModel,
    Executor executor,
    Channel parent,
    ClientStream.Listener listener) {
    super(executor, frameworkModel);
    this.parent = parent;
    this.listener = listener;
    this.writeQueue = createWriteQueue(parent);
}
```

这里有个很重要的方法就是 createWriteQueue。

```java
private WriteQueue createWriteQueue(Channel parent) {
    final Http2StreamChannelBootstrap bootstrap = new Http2StreamChannelBootstrap(parent);
    final Future<Http2StreamChannel> future = bootstrap.open().syncUninterruptibly();
    if (!future.isSuccess()) {
        throw new IllegalStateException("Create remote stream failed. channel:" + parent);
    }
    final Http2StreamChannel channel = future.getNow();
    channel.pipeline()
        .addLast(new TripleCommandOutBoundHandler())
        .addLast(new TripleHttp2ClientResponseHandler(createTransportListener()));
    channel.closeFuture()
        .addListener(f -> transportException(f.cause()));
    return new WriteQueue(channel);
}
```

在这里给 channel 加上了 TripleHttp2ClientResponseHandler

```java
@Override
protected void channelRead0(ChannelHandlerContext ctx, Http2StreamFrame msg) throws Exception {
    if (msg instanceof Http2HeadersFrame) {
        final Http2HeadersFrame headers = (Http2HeadersFrame) msg;
        transportListener.onHeader(headers.headers(), headers.isEndStream());
    } else if (msg instanceof Http2DataFrame) {
        final Http2DataFrame data = (Http2DataFrame) msg;
        transportListener.onData(data.content(), data.isEndStream());
    } else {
        super.channelRead(ctx, msg);
    }
}
```

这是 TripleHttp2ClientResponseHandler#channelRead0，对于 Http2HeadersFrame 和 Http2DataFrame 进行不同的处理。

- Header

  ```java
  @Override
  public void onHeader(Http2Headers headers, boolean endStream) {
      executor.execute(() -> {
          if (endStream) {
              if (!halfClosed) {
                  writeQueue.enqueue(CancelQueueCommand.createCommand(Http2Error.CANCEL),
                      true);
              }
              onTrailersReceived(headers);
          } else {
              onHeaderReceived(headers);
          }
      });
  }
  ```

  ```java
  void onHeaderReceived(Http2Headers headers) {
      if (transportError != null) {
          transportError.appendDescription("headers:" + headers);
          return;
      }
      if (headerReceived) {
          transportError = TriRpcStatus.INTERNAL.withDescription("Received headers twice");
          return;
      }
      Integer httpStatus =
          headers.status() == null ? null : Integer.parseInt(headers.status().toString());
  
      if (httpStatus != null && Integer.parseInt(httpStatus.toString()) > 100
          && httpStatus < 200) {
          // ignored
          return;
      }
      headerReceived = true;
      transportError = validateHeaderStatus(headers);
  
      // todo support full payload compressor
      CharSequence messageEncoding = headers.get(TripleHeaderEnum.GRPC_ENCODING.getHeader());
      if (null != messageEncoding) {
          String compressorStr = messageEncoding.toString();
          if (!Identity.IDENTITY.getMessageEncoding().equals(compressorStr)) {
              DeCompressor compressor = DeCompressor.getCompressor(frameworkModel,
                  compressorStr);
              if (null == compressor) {
                  throw TriRpcStatus.UNIMPLEMENTED.withDescription(String.format(
                      "Grpc-encoding '%s' is not supported",
                      compressorStr)).asException();
              } else {
                  decompressor = compressor;
              }
          }
      }
      TriDecoder.Listener listener = new TriDecoder.Listener() {
          @Override
          public void onRawMessage(byte[] data) {
              TripleClientStream.this.listener.onMessage(data);
          }
  
          public void close() {
              finishProcess(statusFromTrailers(trailers), trailers);
          }
      };
      deframer = new TriDecoder(decompressor, listener);
      TripleClientStream.this.listener.onStart();
  }
  ```

  总之就是进行了一系列初始化工作，然后定义了一个 TriDecoder.Listener，当 listener 的 onRawMessage 方法被调用时，会调用 TripleClientStream.this.listener.onMessage(data)。

  最后调用了 TripleClientStream.this.listener.onStart()。

  **这个地方的 TripleClientStream.this.listener 就是 stream rpc 与 unary 的不同之处。** 

  ![image-20220520132943647](https://img.jooks.cn/img/202205201329733.png)

  unary 传进来的 listener 是 UnaryClientCallListener，两种 stream rpc 传进来的是 ObserverToClientCallListenerAdapter（对消费者定义的 StreamObserver 的封装）。

  ```java
  // ObserverToClientCallListenerAdapter#onStart
  @Override
  public void onStart(ClientCall call) {
      this.call = call;
      if (call.isAutoRequest()) {
          call.request(1);
      }
  }
  ```

  ```java
  // UnaryClientCallListener#onStart
  @Override
  public void onStart(ClientCall call) {
      future.addTimeoutListener(
          () -> call.cancelByLocal(new IllegalStateException("client timeout")));
      call.request(2);
  }
  ```

  这个地方 call#request 会去调用 netty 的 CompositeByteBuf（零拷贝相关） 去接收 n 个消息。具体的代码在 TriDecoder#deliver 中。

  ```java
  private void deliver() {
      // We can have reentrancy here when using a direct executor, triggered by calls to
      // request more messages. This is safe as we simply loop until pendingDelivers = 0
      if (inDelivery) {
          return;
      }
      inDelivery = true;
      try {
          // Process the uncompressed bytes.
          while (pendingDeliveries > 0 && hasEnoughBytes()) {
              switch (state) {
                  case HEADER:
                      processHeader();
                      break;
                  case PAYLOAD:
                      // Read the body and deliver the message.
                      processBody();
  
                      // Since we've delivered a message, decrement the number of pending
                      // deliveries remaining.
                      pendingDeliveries--;
                      break;
                  default:
                      throw new AssertionError("Invalid state: " + state);
              }
          }
          if (closing) {
              if (!closed) {
                  closed = true;
                  accumulate.clear();
                  accumulate.release();
                  listener.close();
              }
          }
      } finally {
          inDelivery = false;
      }
  }
  ```

- Data

  当收到消息时会进入onData

  ```java
  @Override
  public void onData(ByteBuf data, boolean endStream) {
      executor.execute(() -> {
          if (transportError != null) {
              transportError.appendDescription(
                  "Data:" + data.toString(StandardCharsets.UTF_8));
              ReferenceCountUtil.release(data);
              if (transportError.description.length() > 512 || endStream) {
                  handleH2TransportError(transportError);
              }
              return;
          }
          if (!headerReceived) {
              handleH2TransportError(TriRpcStatus.INTERNAL.withDescription(
                  "headers not received before payload"));
              return;
          }
          deframer.deframe(data);
      });
  }
  ```

  这里调用了 deframer.deframe(data) 。**这里的 accumulate 是 CompositeByteBuf 类型，用来处理零拷贝。**这里的 addComponent 方法就是用来动态增加 ByteBuf 。 

  ```java
  @Override
  public void deframe(ByteBuf data) {
      if (closing || closed) {
          // ignored
          return;
      }
      accumulate.addComponent(true, data);
      deliver();
  }
  ```

总结为下图（可能有点乱），如果是流的方式，则会不断拉数据。pendingDeliveries 会在 1 - 2 之间不断跳动（不会变成 0），最总流结束后，调用 TriDecoder#close 结束调用。

这里其实跟反压很像。（这个是不是就是dubbo实现的反压？）

![image-20220521180303274](https://img.jooks.cn/img/202205211803308.png)

unary 其实也类似，不过在 onData 时不会增加 pendingDeliveries。

bi-stream 也类似，与 serverStream 不同的是返回的是一个 StreamObserver，然后用户可以直接通过 onNext 向生产者发送数据。

>  客户端是这样发的，但是服务端是怎么接收的呢？

这个问题是前面遗留的，主要逻辑在生产者的 TripleHttp2FrameServerHandler 中。

惯例看 channelRead 方法

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof Http2HeadersFrame) {
        onHeadersRead(ctx, (Http2HeadersFrame) msg);
    } else if (msg instanceof Http2DataFrame) {
        onDataRead(ctx, (Http2DataFrame) msg);
    } else if (msg instanceof ReferenceCounted) {
        // ignored
        ReferenceCountUtil.release(msg);
    }
}
```

```java
public void onDataRead(ChannelHandlerContext ctx, Http2DataFrame msg) throws Exception {
    final TripleServerStream tripleServerStream = ctx.channel().attr(SERVER_STREAM_KEY)
        .get();
    tripleServerStream.transportObserver.onData(msg.content(), msg.isEndStream());
}

public void onHeadersRead(ChannelHandlerContext ctx, Http2HeadersFrame msg) throws Exception {
    TripleServerStream tripleServerStream = new TripleServerStream(ctx.channel(),
        frameworkModel, executor,
        pathResolver, acceptEncoding, filters);
    ctx.channel().attr(SERVER_STREAM_KEY).set(tripleServerStream);
    tripleServerStream.transportObserver.onHeader(msg.headers(), msg.isEndStream());
}
```

发现其实 server 端接收跟 client 端接收是差不多的，都是利用 TriDecoder 的相关方法实现消息的接收。

另外，如果我们使用 stub 来调用，那么会调用自动生成的 DubboXXXXTriple 类中的方法，然后调用 StubInvocationUtil -> InvocationUtil 实现远程调用。

## 总结

由于 Triple协议 是建立在 http2 的基础上，因此天然具备了 stream 和 全双工 的能力。

通过Triple协议的Streaming RPC方式，会在consumer跟provider之间建立多条用户态的长连接，Stream。同一个TCP连接之上能同时存在多个Stream，其中每条Stream都有StreamId进行标识，对于一条Stream上的数据包会以顺序方式读写。

## 参考

[1] https://blog.csdn.net/superfjj/article/details/121507952

