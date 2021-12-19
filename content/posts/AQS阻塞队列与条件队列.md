---
title: "AQS阻塞队列与条件队列"
date: Wed Mar 03 21:22:33 CST 2021
categories: ["Java基础"]
tags: ["Java基础"]
draft: false
---

深入AQS之后发现AQS是有两个队列的，分别是阻塞队列和条件队列，这里我画了一个示意图。

![image-20210303211454144](https://img.jooks.cn/img/20210303211454.png)

## 阻塞队列

The wait queue is a variant of a "CLH" (Craig, Landin, and Hagersten) lock queue. AQS类的源码给出了超级详细的解说。。。（从没见过这么多注释的源码）

AQS类的head和tail表示队首和队尾的节点。

#### 以独占方式获取资源为例：

以ReentrantLock为例(调用acquire方法)，在AQS内部通过CAS把state从0设置为1，然后设置当前锁的持有者为当前线程；后续，若其他线程请求这个锁时，发现持有者不是自己，**就把这个线程(包装成节点)放入AQS阻塞队列然后挂起，**然后调用LockSupport.park(this)将这个线程挂起来。

后面使用release方法释放锁，会调用LockSupport.unpark(thread)，而这里的thread是阻塞队列的head。

共享方式也是类似，通常是CAS失败然后送进阻塞队列。

## 条件队列

调用条件对象的await方法会把当前线程给放进条件队列中（当然，得先获取锁），然后释放锁；

之后调用条件对象的signal或signalAll方法，会把当前线程从条件队列移动到AQS的阻塞队列里面。
