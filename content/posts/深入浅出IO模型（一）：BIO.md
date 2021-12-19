---
title: "深入浅出IO模型（一）：BIO"
date: Tue Mar 16 17:13:33 CST 2021
categories: ["操作系统"]
tags: ["操作系统"]
draft: false
---

## 预备阶段

首先，我们编写一个简单的socket程序

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;

public class jooks {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(8090); //主动套接字 => 监听套接字
        System.out.println("new a socket on port:8090");
        while (true) {
            Socket client = socket.accept(); //监听套接字 => 已连接套接字
            System.out.println("client\t" + client.getPort());
            new Thread(() -> {
                try {
                    InputStream is = client.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(is));
                    while (true) {
                        System.out.println(reader.readLine());  //recv
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

通过`strace -ff -o ./ooxx java jooks`抓取运行这个java程序产生的系统调用

然后可以在当前文件目录下发现很多文件

![image-20210316155752746](https://img.jooks.cn/img/20210316155752.png)

这些ooxx.*就代表一个个线程被记录的系统调用。（启动jvm就会产生很多线程）

这时候通过`grep 'port:8090' ./*`查找我们java程序中写的输出。

![image-20210316155829654](https://img.jooks.cn/img/20210316155829.png)

发现在1865中。打开ooxx.1865文件发现如下内容。

![image-20210316160023508](https://img.jooks.cn/img/20210316160023.png)

通过`jps`命令查看java进程的id号。

![image-20210316160232909](https://img.jooks.cn/img/20210316160232.png)

然后进入/proc/1864/fd

![image-20210316160335298](https://img.jooks.cn/img/20210316160335.png)

发现这里有如下文件，0-5分别对应了ooxx文件中的数字(文件描述符)。其中最后两个是socket。

## 开始通信

这时通过`nc 127.0.0.1 8090`与java进程产生通信

![image-20210316155055049](https://img.jooks.cn/img/20210316155055.png)

![image-20210316160511732](https://img.jooks.cn/img/20210316160511.png)

这时候，多了一个ooxx文件

![image-20210316160615389](https://img.jooks.cn/img/20210316160615.png)

等会儿再进去，先去看看ooxx.1865线程发生了什么变化

![image-20210316160957473](https://img.jooks.cn/img/20210316160957.png)

可以看到这里accept了127.0.0.1:42556的连接。同时，/proc/1864/fd也发生了变化

![image-20210316161211043](https://img.jooks.cn/img/20210316161211.png)

发现这里多了一个叫6的socket，这就是ooxx.1865中看到的accpet返回的。

我们再看看新产生的ooxx.1906。

![image-20210316162006373](https://img.jooks.cn/img/20210316162006.png)

发现这里有个recvfrom，然后还write了我们nc发送的数据，这时候nc再发一行试试。

![image-20210316162117056](https://img.jooks.cn/img/20210316162117.png)

至此实操结束。

## 做个小总结

刚刚发生了什么，如下图所示。

创建主动套接字4

![image-20210316163120763](https://img.jooks.cn/img/20210316163120.png)

根据4创建监听套接字5

![image-20210316163208802](https://img.jooks.cn/img/20210316163208.png)

绑定8090端口、设置监听

![image-20210316163645889](https://img.jooks.cn/img/20210316163645.png)

调用poll() ===========>  客户端发送连接，服务端accpet()，然后返回已连接套接字6

![image-20210316163726827](https://img.jooks.cn/img/20210316163726.png)

从6，recv（阻塞式）

![image-20210316163846640](https://img.jooks.cn/img/20210316163846.png)

---

画个图

![image-20210316170726816](https://img.jooks.cn/img/20210316170726.png)

可以发现，这种模式处理多个客户端请求时，是产生多个线程来应付的，产生的系统调用也很多，不是一个好办法。

---
以上就是bio模型
