---
title: "深入MySQL的join查询"
date: Sat Apr 03 14:18:10 CST 2021
categories: ["DB类"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["DB类"]
draft: false
---

> MySQL的join有三种算法，NLJ, BNL, BKA(MySQL 5.6引入)

**Index Nested-Loop Join**，简称NLJ：

在被驱动表有索引时，会执行NLJ算法，当执行join语句时，

1. 先从驱动表中读出一行数据R；
2. 然后到被驱动表中进行查询；
3. 查出记录后跟R组成一行，作为结果集的一部分；
4. 重复执行上述操作，把驱动表中的数据都读完。

MySQL 5.6的时候引入了**Batched Key Access**(BKA)算法，对NLJ进行了优化，BKA的原理跟MRR类似，就是在查询被驱动表走索引之前，先对索引字段进行排序，按顺序走索引，减少磁盘IO。

BKA是先把驱动表的符合要求的数据行读进 join_buffer里面，排序后再拿进被驱动表查询。

---

**Block Nested-Loop Join**，简称BNL：

在被驱动表无索引时，会执行BNL算法，当执行join语句时，

1. 先把驱动表中满足条件的数据行都读入join_buffer中；
2. 然后扫描被驱动表，将被驱动表中的每一行数据都取出来，分别跟join_buffer中的无序数组的所有元素进行比较，满足条件的加入结果集。
3. 如果join_buffer放不下，那就分批将数据读如join_buffer。

BNL因为要在内存中将被驱动表的数据与join_buffer中的数据逐个判断，使得其效率非常低。

优化手段：

1. 直接给被驱动表加上合适的索引，走BKA算法；
2. 如果这个join操作并不经常使用，会使得上面建立的索引性价比比较低，因此建立一个临时表，将被驱动表写入临时表，提前过滤数据，然后加上索引，走BKA算法。
3. BNL算法对join_buffer做N次扫描的操作存在较大缺陷，一个很好的思路就是将驱动表中的数据以哈希表的形式存在join_buffer中，而不是无序数组。但MySQL官方一直没有做这个优化，于是我们可以在业务端来模拟，可以极大降低被驱动表无索引时的开销。


