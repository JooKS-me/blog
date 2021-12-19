---
title: "Apache Shenyu源码阅读计划（二）HTTP请求梳理"
date: Sat Jun 12 23:14:53 CST 2021
categories: ["中间件"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["中间件"]
draft: false
---

就利用上一篇文章的例子，打上断点调试。

发一个请求，结果出现如下日志：

```java
2021-06-12 16:15:16 [shenyu-netty-kqueue-3] INFO  org.apache.shenyu.plugin.base.AbstractShenyuPlugin - context_path selector success match , selector name :/context-path/http
2021-06-12 16:15:16 [shenyu-netty-kqueue-3] INFO  org.apache.shenyu.plugin.base.AbstractShenyuPlugin - context_path rule success match , rule name :/context-path/http
2021-06-12 16:15:16 [shenyu-netty-kqueue-3] INFO  org.apache.shenyu.plugin.base.AbstractShenyuPlugin - divide selector success match , selector name :/http
2021-06-12 16:15:16 [shenyu-netty-kqueue-3] INFO  org.apache.shenyu.plugin.base.AbstractShenyuPlugin - divide rule success match , rule name :/http/order/save
2021-06-12 16:15:16 [shenyu-netty-kqueue-3] INFO  org.apache.shenyu.plugin.httpclient.WebClientPlugin - The request urlPath is http://192.168.1.108:8189/order/save, retryTimes is 0
```

发现了比较重要的`AbstractShenyuPlugin`，于是进入`AbstractShenyuPlugin`，发现了一个`execute()`方法，注释说这个方法是用来处理web请求的。

于是给execute打断点，再发一个请求。

```java
public Mono<Void> execute(final ServerWebExchange exchange, final ShenyuPluginChain chain) {
    // 获取插件名字
  	String pluginName = named();
  	// 根据插件名获取PluginData
    PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
    // 判断当前插件是否可用，不可用则跳过
  	if (pluginData != null && pluginData.getEnabled()) {
      	// 获取SelectorData集合
        final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
      	// 若没有Selector，则直接跳过这个插件
        if (CollectionUtils.isEmpty(selectors)) {
            return handleSelectorIfNull(pluginName, exchange, chain);
        }
      	// 根据exchange从selectors中选出SelectorData，exchange内容示例见下方截图
        SelectorData selectorData = matchSelector(exchange, selectors);
        if (Objects.isNull(selectorData)) {
            return handleSelectorIfNull(pluginName, exchange, chain);
        }
      	// 打日志
        selectorLog(selectorData, pluginName);
      	// 获取规则列表
        List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
        if (CollectionUtils.isEmpty(rules)) {
            return handleRuleIfNull(pluginName, exchange, chain);
        }
        RuleData rule;
      	// TOLOOK
        if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
            //get last
            rule = rules.get(rules.size() - 1);
        } else {
            rule = matchRule(exchange, rules);
        }
        if (Objects.isNull(rule)) {
            return handleRuleIfNull(pluginName, exchange, chain);
        }
      	// 打日志
        ruleLog(rule, pluginName);
      	// 传入exhcange、selector信息、rule信息，具体处理请求。
        return doExecute(exchange, chain, selectorData, rule);
    }
    return chain.execute(exchange);
}
```

exchange内容如下图所示，目前还不太清楚是干啥的，暂时放一放。

![image-20210612171226551](https://img.jooks.cn/img/20210612171226.png)

接下来到了ContextPathPlugin的`doExchange()`方法，具体来看一看。

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final ShenyuPluginChain chain, final SelectorData selector, final RuleData rule) {
  	// 从exchange中取出ShenyuContext，TOLOOK
    ShenyuContext shenyuContext = exchange.getAttribute(Constants.CONTEXT);
    assert shenyuContext != null;
  	// 获取ContextMappingHandle（Context mapping thread handle）
    ContextMappingHandle contextMappingHandle = ContextPathRuleHandleCache.getInstance().obtainHandle(CacheKeyUtils.INST.getKey(rule));
    if (Objects.isNull(contextMappingHandle) || StringUtils.isBlank(contextMappingHandle.getContextPath())) {
        log.error("context path rule configuration is null ：{}", rule);
        return chain.execute(exchange);
    }
    // 构建context路径和真实url
    buildContextPath(shenyuContext, contextMappingHandle);
    return chain.execute(exchange);
}
```

`buildContextPath()`方法具体内容如下：

```java
private void buildContextPath(final ShenyuContext context, final ContextMappingHandle handle) {
    String realURI = "";
    if (StringUtils.isNoneBlank(handle.getContextPath())) {
        context.setContextPath(handle.getContextPath());
        context.setModule(handle.getContextPath());
      	// 原来的contextPath是'/http/order/save'，经过下面这一步就变成了'/order/save'，变成了realURI
        realURI = context.getPath().substring(handle.getContextPath().length());
    } else {
        if (StringUtils.isNoneBlank(handle.getAddPrefix())) {
            realURI = handle.getAddPrefix() + context.getPath();
        }
    }
    context.setRealUrl(realURI);
    if (StringUtils.isNoneBlank(handle.getRealUrl())) {
        log.info("context path replaced old :{} , real:{}", context.getRealUrl(), handle.getRealUrl());
        context.setRealUrl(handle.getRealUrl());
    }
}
```

这个方法执行完之后，继续回到插件调用链。。。（这里的context保存在了exchange的attributes中，猜测后面会统一执行插件？）

然后跟上面类似，到了DividePlugin的doExcute()方法。

> 后面再仔细看DividePlugin插件的doExcute()

然后接着就不知道看哪里了。于是在`ShenYuWebHandler.execute()`这里打一个断点，因为这里是目前所知的最早执行的程序。通过函数调用栈发现前一个执行的是`ShenYuWebHandler.handle()`

```java
public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
    Mono<Void> execute = new DefaultShenyuPluginChain(plugins).execute(exchange);
    if (scheduled) {
        return execute.subscribeOn(scheduler);
    }
    return execute;
}
```

可以发现这里新建了一个`DefaultShenyuPluginChain`，且把插件集合给放进构造方法。

> 以下都是spring的内容。

然后在通过函数调用栈往前看，发现是`DefaultWebFilterChain.filter()`方法。

```java
public Mono<Void> filter(ServerWebExchange exchange) {
   return Mono.defer(() ->
         this.currentFilter != null && this.chain != null ?
               invokeFilter(this.currentFilter, this.chain, exchange) :
               this.handler.handle(exchange));
}
```

再往上是`FilteringWebHandler.handle()`方法。

再往上是`WebHandlerDecorator.handle()`。

再往上是`ExceptionHandlingWebHandler.handle()`。

再往上是`HttpWebHandlerAdapter.handle()`。

```java
public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
   if (this.forwardedHeaderTransformer != null) {
      request = this.forwardedHeaderTransformer.apply(request);
   }
   // 这里应该是根据request和response生成exchange
   ServerWebExchange exchange = createExchange(request, response);

   LogFormatUtils.traceDebug(logger, traceOn ->
         exchange.getLogPrefix() + formatRequest(exchange.getRequest()) +
               (traceOn ? ", headers=" + formatHeaders(exchange.getRequest().getHeaders()) : ""));

   return getDelegate().handle(exchange)
         .doOnSuccess(aVoid -> logResponse(exchange))
         .onErrorResume(ex -> handleUnresolvedError(exchange, ex))
         .then(Mono.defer(response::setComplete));
}
```

再往上是`ReactiveWebServerApplicationContext.handle()`

```java
public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
   return this.handler.handle(request, response);
}
```

再往上是`ReactorHttpHandlerAdapter.apply()`

再往上是`HttpServerHandle.onStateChange()`

再往上是`TcpServerBind.ChildObserver.onStateChange()`

再往上是`HttpServerOperations.onInboundNext()`，这里开始进入Netty的内容（WebFlux是基于Netty实现的）

```java
protected void onInboundNext(ChannelHandlerContext ctx, Object msg) {
   if (msg instanceof HttpRequest) {
      try {
         listener().onStateChange(this, HttpServerState.REQUEST_RECEIVED);
      }
      catch (Exception e) {
         onInboundError(e);
         ReferenceCountUtil.release(msg);
         return;
      }
      if (msg instanceof FullHttpRequest) {
         super.onInboundNext(ctx, msg);
         if (isHttp2()) {
            onInboundComplete();
         }
      }
      return;
   }
   if (msg instanceof HttpContent) {
      if (msg != LastHttpContent.EMPTY_LAST_CONTENT) {
         super.onInboundNext(ctx, msg);
      }
      if (msg instanceof LastHttpContent) {
         onInboundComplete();
      }
   }
   else {
      super.onInboundNext(ctx, msg);
   }
}
```

算了，还是看不懂（爬。。。）

再往上走全是netty的东西，over。

梳理一下就是（搬运）：

- HttpServerOperations : 明显的netty的请求接收的地方，请求入口

- TcpServerBind

- HttpServerHandle

- ReactorHttpHandlerAdapter ：生成response和request

- ReactiveWebServerApplicationContext

- HttpWebHandlerAdapter ：exchange 的生成

- ExceptionHandlingWebHandler

- WebHandlerDecorator

- FilteringWebHandler

- DefaultWebFilterChain

- SoulWebHandler ：plugins调用链

- DividePlugin ：plugin具体处理

> 下面再来看一下之前没仔细看的DividePlugin.doExcute()

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final ShenyuPluginChain chain, final SelectorData selector, final RuleData rule) {
  	// 取得shenyuContext
    ShenyuContext shenyuContext = exchange.getAttribute(Constants.CONTEXT);
    assert shenyuContext != null;
  	// 根据rule生成ruleHandle
    DivideRuleHandle ruleHandle = UpstreamCacheManager.getInstance().obtainHandle(CacheKeyUtils.INST.getKey(rule));
    long headerSize = 0;
  	// 下面计算header的字节总长
    for (List<String> multiHeader : exchange.getRequest().getHeaders().values()) {
        for (String value : multiHeader) {
            headerSize += value.getBytes(StandardCharsets.UTF_8).length;
        }
    }
    if (headerSize > ruleHandle.getHeaderMaxSize()) {
        log.error("request header is too large");
        Object error = ShenyuResultWrap.error(ShenyuResultEnum.REQUEST_HEADER_TOO_LARGE.getCode(), ShenyuResultEnum.REQUEST_HEADER_TOO_LARGE.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    if (exchange.getRequest().getHeaders().getContentLength() > ruleHandle.getRequestMaxSize()) {
        log.error("request entity is too large");
        Object error = ShenyuResultWrap.error(ShenyuResultEnum.REQUEST_ENTITY_TOO_LARGE.getCode(), ShenyuResultEnum.REQUEST_ENTITY_TOO_LARGE.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
  	// 获取上游api列表
    List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
    if (CollectionUtils.isEmpty(upstreamList)) {
        log.error("divide upstream configuration error： {}", rule.toString());
        Object error = ShenyuResultWrap.error(ShenyuResultEnum.CANNOT_FIND_URL.getCode(), ShenyuResultEnum.CANNOT_FIND_URL.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
    DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
    if (Objects.isNull(divideUpstream)) {
        log.error("divide has no upstream");
        Object error = ShenyuResultWrap.error(ShenyuResultEnum.CANNOT_FIND_URL.getCode(), ShenyuResultEnum.CANNOT_FIND_URL.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    // set the http url
    String domain = buildDomain(divideUpstream);
    String realURL = buildRealURL(domain, shenyuContext, exchange);
    exchange.getAttributes().put(Constants.HTTP_URL, realURL);
    // set the http timeout
    exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
    exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
    return chain.execute(exchange);
}
```

然后我想知道到底是哪里发了请求QAQ，继续往下看。

哦对，之前日志打印的最后一行是：`INFO  org.apache.shenyu.plugin.httpclient.WebClientPlugin - The request urlPath is http://192.168.1.108:8189/order/save, retryTimes is 0`

所以直接去看WebClientPlugin

```java
public Mono<Void> execute(final ServerWebExchange exchange, final ShenyuPluginChain chain) {
    final ShenyuContext shenyuContext = exchange.getAttribute(Constants.CONTEXT);
    assert shenyuContext != null;
    String urlPath = exchange.getAttribute(Constants.HTTP_URL);
    if (StringUtils.isEmpty(urlPath)) {
        Object error = ShenyuResultWrap.error(ShenyuResultEnum.CANNOT_FIND_URL.getCode(), ShenyuResultEnum.CANNOT_FIND_URL.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
    int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
    log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
    // 判断是哪种http请求，比如GET/POST/...
  	HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
    WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
    // 重点来了
  	return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
}
```

继续跟进`handleRequestBody()`方法。

```java
private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                     final ServerWebExchange exchange,
                                     final long timeout,
                                     final int retryTimes,
                                     final ShenyuPluginChain chain) {
    return requestBodySpec.headers(httpHeaders -> {
        httpHeaders.addAll(exchange.getRequest().getHeaders());
        httpHeaders.remove(HttpHeaders.HOST);
    })
            .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
            .exchange()
            .doOnError(e -> log.error(e.getMessage(), e))
            .timeout(Duration.ofMillis(timeout))
            .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                .retryMax(retryTimes)
                .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
            .flatMap(e -> doNext(e, exchange, chain));

}
```

这里就是向真实的uri发起了请求。

除此之外，shenyu-plugin-httpclient模块下还有一个WebClientResponsePlugin这个类

![image-20210612223935554](https://img.jooks.cn/img/20210612223935.png)

这个类据说是（TO CONFIRM）用来向客户端返回response的，下面看看它的execute方法。

```java
public Mono<Void> execute(final ServerWebExchange exchange, final ShenyuPluginChain chain) {
    return chain.execute(exchange).then(Mono.defer(() -> {
        ServerHttpResponse response = exchange.getResponse();
        ClientResponse clientResponse = exchange.getAttribute(Constants.CLIENT_RESPONSE_ATTR);
        if (Objects.isNull(clientResponse)
                || response.getStatusCode() == HttpStatus.BAD_GATEWAY
                || response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR) {
            Object error = ShenyuResultWrap.error(ShenyuResultEnum.SERVICE_RESULT_ERROR.getCode(), ShenyuResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (response.getStatusCode() == HttpStatus.GATEWAY_TIMEOUT) {
            Object error = ShenyuResultWrap.error(ShenyuResultEnum.SERVICE_TIMEOUT.getCode(), ShenyuResultEnum.SERVICE_TIMEOUT.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        response.setStatusCode(clientResponse.statusCode());
        response.getCookies().putAll(clientResponse.cookies());
        response.getHeaders().putAll(clientResponse.headers().asHttpHeaders());
        return response.writeWith(clientResponse.body(BodyExtractors.toDataBuffers())).doOnCancel(() -> clean(exchange));
    }));
}
```

## 总结

一路下来，其实有很多细节没有深入探讨。

但是梳理了一下HTTP从打到shenyu网关，再到shenyu去请求上游API，再把结果返回回去的整个流程，发现ShenYu比较核心的部分其实是shenyu-plugin模块。（感觉）

另外发现了官网上一篇对divide插件的解读，里面提到了主机探活机制，这是我没发现的地方：https://dromara.org/zh/blog/soul_source_learning_16_divide_sxj/
