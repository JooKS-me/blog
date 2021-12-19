---
title: "Java agent + ByteBuddy入门好文"
date: Mon May 10 18:33:02 CST 2021
categories: Java基础
tags: Java基础
draft: false
---

最近因为项目需要，来学一学这两个东西，发现了一些文章，这里mark一下。

Java agent：[https://zhuanlan.zhihu.com/p/147375268](https://zhuanlan.zhihu.com/p/147375268)

ByteBuddy：[https://zhuanlan.zhihu.com/p/151843984](https://zhuanlan.zhihu.com/p/151843984)

[https://blog.csdn.net/zjttlance/article/details/80684413](https://blog.csdn.net/zjttlance/article/details/80684413)

ByteBuddy官网Java Agent部分摘抄：
#### Creating Java agents

When an application grows bigger and becomes more modular, applying such a transformation at a specific program point is of course a cumbersome constraint to enforce. And there is indeed a better way to apply such class redefinitions *on demand*. Using a [Java agent](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html), it is possible to directly intercept any class loading activity that is conducted within a Java application. A Java agent is implemented as a simple *jar* file with an entry point that is specified in this jar file's manifest file as it is described under the linked resource. Using Byte Buddy, the implementation of such an agent is straight forward by using an `AgentBuilder`. Assuming that we previously defined a simple annotation named `ToString`, it would be trivial to implement `toString` methods for all annotated classes simply by implementing the Agent's `premain` method as follows:

```java
class ToStringAgent {
  public static void premain(String arguments, Instrumentation instrumentation) {
    new AgentBuilder.Default()
        .type(isAnnotatedWith(ToString.class))
        .transform(new AgentBuilder.Transformer() {
      @Override
      public DynamicType.Builder transform(DynamicType.Builder builder,
                                              TypeDescription typeDescription,
                                              ClassLoader classloader) {
        return builder.method(named("toString"))
                      .intercept(FixedValue.value("transformed"));
      }
    }).installOn(instrumentation);
  }
}
```

As a result of the applying the above `AgentBuilder.Transformer`, all `toString` methods of the annotated classes would now return `transformed`. We will learn all about Byte Buddy's `DynamicType.Builder` in the upcoming sections, do not worry about this class for now. The above code results of course in a trivial and meaningless application. Using this concept right, renders however a powerful tool for easily implementing aspect-oriented programing.

Note that it is also possible to instrument classes that were loaded by the bootstrap class loader when using an agent. However, this requires some preparation. First of all, the bootstrap class loader is represented by the `null` value which makes it impossible to load a class in this class loader using reflection. This is however sometimes necessary to load helper classes into the instrumented class's class loader to support the class's implementation. In order to load classes into the bootstrap class loader, Byte Buddy can create jar files and add these files to the bootstrap class loader's load path. To make this possible, it is however required to save these classes to disk. A folder for these classes can be specified using the `enableBootstrapInjection` command which also takes an instance of the `Instrumentation` interface in order to append the classes. Note that all user classes that are used by the instrumented class are also required to be put on the bootstrap search path which is possible using the `Instrumentation` interface.
