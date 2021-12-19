---
title: "对于操作系统CAS的深入思考"
date: Sat Apr 10 22:11:51 CST 2021
categories: 操作系统
tags: 操作系统
draft: false
---

也许会存在这样一种情况，进程/线程A和进程/线程B要对一个值value进行修改，A要将value修改为2，B要将value修改为3，但是这种修改是互斥的，也就是说A和B一次只能有一个修改成功。

这种情况就显然是使用CAS（Compare And Swap）了，但是CAS在操作系统里面是通过原语实现的，cas过程中是不可中断的，具有原子性，这个没有问题。

但是原子性可不代表线程安全，因此就产生了疑问，会不会存在A和B同时通过了验证的情况呢，如果是这样那么CAS就没有**线程安全**这一说了。具体描述可见下图。

![时刻图](https://img.jooks.cn/img/20210410214427.png)

实际上，上图的情况只有多CPU的环境才能出现，因为单CPU时若要从A切换到B是不允许的，因为A正在执行原语（执行了关中断指令）。

而对于多线程环境，操作系统会把总线锁住，或者是锁住缓存行，使得进程B在compare的时候阻塞住。

![](https://img.jooks.cn/img/20210410221054.png)

上图来自[https://blog.csdn.net/houxinlin_csdn/article/details/108511052](https://blog.csdn.net/houxinlin_csdn/article/details/108511052)

