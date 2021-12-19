---
title: "记一次服务端Java程序崩溃"
date: Fri Feb 19 12:29:53 CST 2021
categories: ["Linux"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["Linux"]
draft: false
---

昨天在服务器上挂了PhantomJS之后，博客的后端API突然访问不到了，服务器也突然变得很卡，过了一段时间，Java进程直接没了。于是我再`java -jar`运行，又可以了。但是第二天起来又发现蹦了。下面是腾讯云后台的内存监控截图，可以很形象地说明情况。

![image-20210219121503498](https://image-1301164990.cos.ap-shanghai.myqcloud.com/img/20210219121503.png)

我看了一下Spring Boot的输出日志，发现并没有异常。加上我的服务器只有1G小内存，于是我怀疑是被系统自动杀掉了。后来了解到原来是Linux内核的OOM Killer机制。



这里不贴内核代码，简单说一下。**OOM机制** 会监控那些占用内存过大，尤其是瞬间很快消耗大量内存的进程，为了防止内存耗尽内核会把该进程杀掉。而这个杀掉是有另外规则的，比如系统内核进程虽然占用很大但不能杀掉，对吧。它首先会看 **oom_score** ，按这个评分优先杀高的，默认是0。然后再看内存占用大小，优先杀内存占用大的。



怎么看有没有触发OOM机制呢，可以看一看/var/log/messages这个日志文件，会有详细说明。



## 如何调整？

直接说我的history吧。

我把java的进程的oom_score降低至-15：`echo -15 > /proc/28462/oom_score_adj`

再把不重要的php进程升高：`echo 1  > /proc/17863/oom_score_adj`

这里发现php有三个进程，我直接写了一个shell脚本。


