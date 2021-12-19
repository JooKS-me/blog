---
title: "从JVM的角度看基本数据类型"
date: Mon Apr 26 09:24:44 CST 2021
categories: ["Java基础"]
tags: ["Java基础"]
draft: false
---

Java的8个基本类型如下图

![image-20210425093731211](https://img.jooks.cn/img/20210425093731.png)

## 基本类型的大小

对于局部变量来说，基本类型保存在 **Java虚拟机栈** 的 **栈帧** 中的 **局部变量区**。栈帧除了局部变量区，还有字节码的操作数栈。

局部变量区类似于数组，可以用正整数来索引。在局部变量区中，long、double需要两个数组来存储，其他基本类型、引用类型的值均占用一个数组单元。（32位Hotspot中，一个数组单元位为4字节；64位为8字节）

也就是说，boolean、byte、char、short这四种类型，在栈上占用的空间和int是一样的（引用类型也一样）。

> 因此通过AsmTools修改boolean的局部变量值为2不会出错，byte、char、short修改为超出本值域的值也不会出错。
>
> PS：见boolean小知识

但是在堆上的分配空间是与值域吻合的，boolean除外。

boolean字段在占1字节。

boolean数组是由byte数组来实现。

Hotspot虚拟机在存储时显式地进行掩码操作，只取最后一位的值存入boolean字段或数组。

## 基本类型的加载

jvm将堆中的boolean、byte、char、short加载到 **操作数栈** 上，然后将栈上的值当成int类型来运算。

这时，boolean(0-1)、char这两个无符号类型，变成int类型时，多出来的高位会被0填充。

byte、short这两个类型，加载时会随着符号扩展。如果为非负，高位用0填充；如果为负，高位用1填充。

## 类型转换

byte、short、int、long、float以及double的值域依次扩大，而且前面的值域被后面的值域包含，故前面的基本类型可以隐式转换成后面的类型。

## boolean小知识

局部变量boolean运算时，在字节码里面是这样的：

`if (boolean)` 会被编译成ifeq指令，判断局部变量boolean的值是否为0，若为0则跳过，否则继续执行。

`if (boolean == true)`会被编译成if_icmpne，先将1（true的表示形式）加载进操作数栈，然后判断boolean的值是否为1，若不为1则跳过。




