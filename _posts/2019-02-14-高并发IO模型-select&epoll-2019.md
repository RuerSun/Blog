---
layout:     post
title:      "高并发I/O模型 select/epoll, 2019"
subtitle:   "简介"
date:       2019-02-14 16:02:00
author:     "Ruer"
header-img: "img/bg/hello_world.jpg"
catalog: true
tags:
    - Linux
---

## 为什么需要 select 和 epoll

![1](/img/Linux/网络编程/为什么需要select和epoll.png)

参考如下代码分析

![2](/img/Linux/网络编程/监听方式比较.png)

## 对 CPU 的影响

操作系统支持多任务，把进程分为“运行”和“等待”等几种状态。运行状态是进程获得 CPU 使用权，等待状态是阻塞状态，不占用 CPU。操作系统会分时执行各个运行状态的进程，由于速度很快，看上去就像是同时执行多个任务。

下图中的计算机中运行着 A、B、C 三个进程，其中进程 A 执行着上述基础网络程序，一开始，这3个进程都被操作系统的工作队列所引用，处于运行状态，分时执行。

![3](/img/Linux/网络编程/工作队列.png)

进程 A 执行到创建 socket 的语句时，操作系统会创建一个由文件系统管理的 socket 对象（如下图）。这个 socket 对象包含了发送缓冲区、接收缓冲区、等待队列等成员。等待队列是个非常重要的结构，它指向所有需要等待该 socket 事件的进程。

当程序执行到 select/epoll 时，操作系统会将进程 A 从工作队列移动到该 socket 的等待队列中（如下图）。工作队列只剩下 B 和 C 进程，不会执行进程 A 的程序。所以进程 A 被阻塞，不会往下执行代码。

![4](/img/Linux/网络编程/等待队列.png)

当 socket 接收到数据后，操作系统将该 socket 等待队列上的进程重新放回到工作队列，该进程变成运行状态，继续执行代码。

![5](/img/Linux/网络编程/唤醒进程.png)

## select 原理

![6](/img/Linux/网络编程/select原理.png)

1. 将所有 fd 通过 select 传递给内核。
2. 某个 socket 收到数据后，中断程序唤醒进程 A，如上图。
3. 程序需遍历 socket 列表，从而得到就绪的 socket。
4. socket 数据收取完毕后，重复1~3流程。

![7](/img/Linux/网络编程/select缺陷.png)

1. select 的最大只能监视1024个 socket。每次调用 select 都需要将进程加入到所有监视 socket 的等待队列，每次唤醒都需要从每个队列中移除，因此内核需要遍历2次 socket 列表，存在 CPU 和内存开销。试想 socket 列表有100万个？

2. 应用层需要再遍历1次 socket 列表。进程被唤醒后，程序并不知道哪些 socket 收到数据，只能再遍历一次。

## epoll 原理

![8](/img/Linux/网络编程/epoll原理.png)

1. 通过 epoll_ctl 预先将 socket 动态传递给内核。
2. 执行 epoll_wait，无需再将 fd 传递给内核。
3. 某个 socket 收到数据后，中断程序唤醒进程 A。
4. socket 数据收取完毕后，重复2~3流程。

select 低效的第1个原因是将“维护就绪队列”和“阻塞进程”两个步骤合二为一。如下图所示，每次调用 select 都需要这两步操作，然而大多数应用场景中，需要监视的 socket 相对固定，并不需要每次都修改。epoll 将这两个操作分开，先用 epoll_ctl 维护等待队列，再调用 epoll_wait 阻塞进程。显而易见的，效率就能得到提升。

![9](/img/Linux/网络编程/epoll功能分离.png)

select 低效的第2个原因在于程序不知道哪些 socket 收到数据，只能一个个遍历。如果内核维护一个“就绪列表”，引用收到数据的 socket，就能避免遍历。如左图所示，计算机共有3个 socket，收到数据的 sock2 和 sock3 被 rdlist（就绪列表）所引用。当进程被唤醒后，只要获取 rdlist 的内容，就能够知道哪些 socket 收到数据。

![10](/img/Linux/网络编程/epoll就绪列表1.png)

![11](/img/Linux/网络编程/epoll就绪列表2.png)

## 小结

![12](/img/Linux/网络编程/高并发IO小结.png)