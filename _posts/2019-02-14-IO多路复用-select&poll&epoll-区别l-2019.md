---
layout:     post
title:      "IO多路复用 select/poll/epoll 区别, 2019"
subtitle:   "简介"
date:       2019-02-14 16:02:00
author:     "Ruer"
header-img: "img/bg/hello_world.jpg"
catalog: true
tags:
    - Linux
---

IO 多路复用是一种同步 IO 模型，一个线程监听多个 IO 事件，当有 IO 事件就绪时，就会通知线程去执行相应的读写操作，没有就绪事件时，就会阻塞交出 CPU。多路是指网络链接，复用指的是复用同一线程。

## select 原理

fd_set 数据结构定义如下，可以看出 fd_set 是一个整型数组，用于保存 socket 文件描述符。

```C++
typedef long int __fd_mask;

/* fd_set for select and pselect.  */
typedef struct
{
    #ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
    # define __FDS_BITS(set) ((set)->fds_bits)
    #else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
    # define __FDS_BITS(set) ((set)->__fds_bits)
    #endif
} fd_set;
```

执行过程:

![1](/img/Linux/网络编程/select原理.png)

流程：
1. 用户线程调用 select，将 fd_set 从用户空间拷贝到内核空间。
2. 内核在内核空间对 fd_set 遍历一遍，检查是否有就绪的 socket 描述符，如果没有的话，就会进入休眠，直到有就绪的 socket 描述符。
3. 内核返回 select 的结果给用户线程，即就绪的文件描述符数量。
4. 用户拿到就绪文件描述符数量后，再次对 fd_set 进行遍历，找出就绪的文件描述符。
5. 用户线程对就绪的文件描述符进行读写操作。

优点：
* 所有平台都支持，良好的跨平台性。
缺点：
* 每次调用 select，都需要将 fd_set 从用户空间拷贝到内核空间，当 fd 很多时，这个开销很大。
* 最大连接数（支持的最大文件描述符数量）有限制，一般为1024。
* 每次有活跃的 socket 描述符时，都需要遍历一次 fd_set，造成大量的时间开销，时间复杂度是O(n)。
* 将 fd_set 从用户空间拷贝到内核空间，内核空间也需要对 fd_set 遍历一遍。

## poll 原理

数据结构定义如下，链表存储。

```C++
/* Data structure describing a polling request.  */
struct pollfd
{
    int fd;			/* File descriptor to poll.  */
    short int events;		/* Types of events poller cares about.  */
    short int revents;		/* Types of events that actually occurred.  */
};
```

与 select 的异同点：

* 相同点：
    * 内核线程都需要遍历文件描述符，并且当内核返回就绪的文件描述符数量后，还需要遍历一次找出就绪的文件描述符。
    * 需要将文件描述符数组或链表从用户空间拷贝到内核空间。
    * 性能开销会随文件描述符的数量而线性增大。
* 不同点：
    * select 存储的数据结构是文件描述符数组，poll 采用链表。
    * select 有最大连接数限制，poll 没有最大限制，因为 poll 采用链表存储。

执行过程（基本与 select 类型）：
* 用户线程调用 poll 系统调用，并将文件描述符链表拷贝到内核空间。
* 内核对文件描述符遍历一遍，如果没有就绪的描述符，则内核开始休眠，直到有就绪的文件描述符。
* 返回给用户线程就绪的文件描述符数量。
* 用户线程再遍历一次文件描述符链表，找出就绪的文件描述符。
* 用户线程对就绪的文件描述符进行读写操作。

## epoll 原理

核心代码：

```C++
#include <sys/epoll.h>

// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 红黑树用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    ...
    /*红黑树的根节点，这颗树存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表存储所有就绪的文件描述符*/
    struct list_head rdlist;
    ...
};

// API
int epoll_create(int size); // 内核中间加一个 eventpoll 对象，把所有需要监听的 socket 都放到 eventpoll 对象中
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // epoll_ctl 负责把 socket 增加、删除到内核红黑树
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);// epoll_wait 检测双链表中是否有就绪的文件描述符，如果有，则返回
```

核心点：
* epoll_create 创建 eventpoll 对象（红黑树，双链表）。
* 一棵红黑树，存储监听的所有文件描述符，并且通过 epoll_ctl 将文件描述符添加、删除到红黑树。
* 一个双链表，存储就绪的文件描述符列表，epoll_wait 调用时，检测此链表中是否有数据，有的话直接返回。
* 所有添加到 eventpoll 中的事件都与设备驱动程序建立回调关系。

缺点：
* 只能工作在 linux 下

优点：
* 时间复杂度为 O(1)，当有事件就绪时，epoll_wait 只需要检测就绪链表中有没有数据，如果有的话就直接返回。
* 不需要从用户空间到内核空间频繁拷贝文件描述符集合，使用了内存映射(mmap)技术。
* 当有就绪事件发生时采用回调的形式通知用户线程。

epoll 的水平触发（LT）和边缘触发（ET）的区别：
* LT 模式：只要文件描述符还有数据可读，每次 epoll_wait 就会返回它的事件（只要有数据就触发）。
* ET 模式：只有数据流到来的时候才触发，不管缓冲区是否还有数据（只有数据流到来才会触发）。

## 三者的区别

|                             | select                                              | poll                                              | epoll                                                                     |
| :-------------------------- | :-------------------------------------------------- | :------------------------------------------------ | :------------------------------------------------------------------------- |
| 底层数据结构                 | 数组存储文件描述符                                    | 链表存储文件描述符                                 | 红黑树存储监控的文件描述符，双链表存储就绪的文件描述符                          |
| 如何从 fd 数据中获取就绪的 fd | 遍历 fd_set                                          | 遍历链表                                          | 回调                                                                       |
| 时间复杂度                   | 获得就绪的文件描述符需要遍历 fd 数组，O(n)             | 获得就绪的文件描述符需要遍历 fd链表，O(n)            | 当有就绪事件时，系统注册的回调函数就会被调用，将就绪的 fd 放入到就绪链表中。O(1) |
| fd 数据拷贝                  | 每次调用 select，需要将 fd 数据从用户空间拷贝到内核空间 | 每次调用 poll，需要将 fd 数据从用户空间拷贝到内核空间 | 使用内存映射(mmap)，不需要从用户空间频繁拷贝 fd 数据到内核空间                 |
| 最大连接数                   | 有限制，一般为1024                                    | 无限制                                            | 无限制                                                                     |

## 应用场景

1. 连接数较少并且都很活跃,用 select 和 poll 效率更高。
2. 连接数较多并且都不很活跃,使用 epoll 效率更高。