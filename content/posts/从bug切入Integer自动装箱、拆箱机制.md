---
title: "从bug切入Integer自动装箱、拆箱机制"
date: Mon Mar 08 00:02:17 CST 2021
categories: ["Java基础"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["Java基础"]
draft: false
---

今天在刷力扣的时候发现自己写的跟官方题解几乎一模一样，就是变量名啥的不一样，但是就是错了。自闭了半个小时后发现原来是一个地方我写的是Integer类型，而题解写的是int类型。百度后发现，Integer类型在数据-128～127范围内是可以直接用==的，而到了范围外则不行！

好家伙，卡我半个小时！于是我直接点开了Integer源码。（以下代码都是基于JDK1.8）

## 自动装箱是怎么一回事？

比如下面就用到了自动装箱

```java
public class IntegerTest {
    public static void main(String[] args) {
        Integer i = 10;
        i = 1;
        i = 2;
    }
}
```

其实在直接给i赋值时，使用了Integer.valueOf()

```java
//比如
i = Integer.valueOf(10);
i = Integer.valueOf(1);
i = Integer.valueOf(2);
```

那么valueOf是啥呢，源码如下：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

可以看到这里先根据IntegerCache对i的大小做了判断，那么IntegerCache是啥嘞？源码如下：

```java
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

IntegerCache是Integer的一个内部类，它有一个static代码块，这个代码块初始化了cache数组，默认情况下，cache数组范围是[-128, 127]，我们也可用通过设置JVM参数`-XX:AutoBoxCacheMax=<size>`来修改cache的范围，从源码来看，这个size至少得是127。

也就是说，如果重新给Integer对象引用赋值，值在范围内，则直接指向cache数组中的某个对象；否则，new一个Integer对象。

这样会导致如下现象。

```java
public class IntegerTest {
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        System.out.println(a == b);  //true
        Integer c = 1000;
        Integer d = 1000;
        System.out.println(c == d);  //false
    }
}
```

## 那么自动拆箱又是怎么回事？

自动拆箱其实是使用了intValue方法，源码如下：

```java
private final int value;

public int intValue() {
    return value;
}
```

也就是说，当碰到`int == Integer`的时候，Integer类型会先调用intValue方法，拿出int类型的value，然后才能进行运算。

## 下面再看看equals吧

```java
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

上面这段是Integer的equals源码，突然发现！只要equals括号里面不是Integer类型，直接返回false。

于是我设计了下面这个测试代码。

```java
public class IntegerTest {
    public static void main(String[] args) {
        Integer i = new Integer(10);
        int j = 10;
        System.out.println(i.equals(j));  //true
        System.out.println(i.equals((long) j));  //false
    }
}
```

输出的第一行是true，因为int类型的j会直接`new Integer(j)`然后变成Integer对象。

输出的第二行是false，正如源码所写的那样。


