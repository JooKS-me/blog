---
title: "深入浅出IO模型（二）：NIO、select、poll、epoll"
date: Tue Mar 16 19:16:07 CST 2021
categories: ["操作系统"]
tags: ["操作系统"]
draft: false
---

BIO要产生多个线程的本质是recv的时候是阻塞的，必须要花费一个线程在上面。

![image-20210316173529617](https://img.jooks.cn/img/20210316173529.png)

上图是man recv产生的部分信息。说明了recv是可以设置成非阻塞模式的，即若发现连接套接字上没有数据，直接返回-1，不再等待。

## 解决方案一：朴素NIO

那么我们可以利用这一点，在程序中加上循环，不断去recv多个连接套接字检测（类似自旋锁？），一旦发现有数据就直接拿进来。

但是这种实现会有个问题，就是系统调用的次数太多了（不断recv）。这样虽然实现了单个线程接受所有请求，但是开销太大了。

## 解决方案二：select和poll多路复用器

基于上面的思想。Linux系统提供了另外两种类似的系统调用select和poll，直接将循环遍历的过程放在了内核中（先将要遍历的套接字传给内核）。一旦连接套接字有数据传过来就把这些套接字的文字描述符返回，recv只要查这几个返回的套接字(文字描述符数字，int类型)就行了。

这样虽然时间复杂还是O(N)，但减少了大量的系统调用，即减少了软中断次数，大大提高了运行效率。

> select和poll差不多，区别如下：
>
> select()  can  monitor  only  file  descriptors  numbers  that are less than FD_SETSIZE;
> poll(2) does not have this limitation.  See BUGS.
>
> select管理的文字描述符数量得小于FD_SETSIZE，而poll没有限制。

但是这样还有问题，因为每次调用前都要将套接字的句柄数组拷贝给内核，产生巨大的开销。

## 解决方案三（终极方案）：epoll多路复用

-  **epoll_create**(2)  creates a new epoll instance and returns a file descriptor referring to that instance
- Interest in particular file descriptors is then registered via **epoll_ctl**(2).  The set of file descriptors currently registered on an epoll instance is sometimes called  an **epoll set**.
- **epoll_wait**(2) waits for I/O events, blocking the calling thread if no events are currently available.

- **epoll_create**会在内核中开辟一个空间，来存放epoll对象（红黑树实现），这个epoll对象就是用来存放需要遍历的连接套接字句柄(fd)集合的。
- **epoll_ctl**将连接套接字句柄加进epoll空间
- 而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立**回调关系**，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫**ep_poll_callback**,它会将发生的事件添加到**rdlist双链表**中。
- 当调用**epoll_wait**就会从rdlist(就绪列表)里面拿fd，要是就绪队列为空则阻塞。

这样以来，巧妙地利用了linux的回调机制，当需要拿数据时只需epoll_wait即可，而epoll_wait的时间复杂度为O(1)。而且也通过维护epoll对象避免了select每次都需要拷贝句柄数组的缺点。


