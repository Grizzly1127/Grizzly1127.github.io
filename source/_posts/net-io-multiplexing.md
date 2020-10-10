---
title: select/poll/epoll之间的区别
date: 2020-09-25 15:53:27
tags:
- I/O多路复用
- select
- poll
- epoll
categories:
- net
---

前面介绍了 [《IO多路复用 - select》](../net-io-multiplexing-select)、[《IO多路复用 - poll》](../net-io-multiplexing-poll)和[《IO多路复用 - epoll》](../net-io-multiplexing-epoll)，那么这三者之间有什么优缺点呢？下面我们来看看

<!-- more -->

## 优缺点

### select

优点：

* select 遵循 POSIX 的规范，支持跨平台，具有良好的兼容性，可以在不同操作系统上使用 select 实现高性能服务器。

缺点：

* 单个进程能打开的最大连接数有限制，只能打开最大为 FD_SETSIZE 宏定义大小的连接数。
* 当有新的连接时，每次都需要将所有的 fd 集合从用户空间拷贝到内核空间，开销大。
* 当有新的 IO 事件发生时，每次都需要将所有的 fd 集合从内核空间拷贝到用户空间，开销大。
* 需要轮询遍历所有 fd 集合，才能知道哪些 fd 就绪，时间复杂度为 O(N)。

### poll

优点：

* 相较于 select，poll 是基于链表来存储 fd 集合的，所以并不会像 select 那样存在最大的连接数限制（受服务器资源限制）。

缺点：

* 非跨平台，只能在 Unix/Linux 操作系统上开发。
* 同 select 一样，都会有大量的 fd 集合被整体复制于用户空间和内核空间之间，开销大。
* IO 效率与 select 一样，都需要轮询遍历所有 fd 集合，才能知道哪些 fd 就绪，时间复杂度为 O(N)。

### epoll

优点：

* 理论上同样不存在最大连接数限制（受服务器资源限制）。
* 效率提升，只有活跃的 fd 才会主动调用 callback 回调函数，时间复杂度为 O(1)。
* 底层由红黑树来管理 fd 集合，通过就绪队列链表将就绪的 fds 拷贝到用户空间，而不需要将所有 fd 集合全部拷贝，省去了不必要的内存拷贝。

## 总结

epoll 对 select/poll 的缺点进行了改进。在 select/poll 中，进程只有在调用一定的方法后，内核才对所有监视的 fd 集合进行扫描，而 epoll 事先通过 epoll_ctl 来注册一个 fd，一旦某个 fd 就绪时，内核会采用 callback 的回调机制，迅速激活这个 fd，当进程调用 epoll_wait 时便能得到通知。
在高并发的场景下 epoll 一切都很好，是 Linux 目前大规模网络并发程序开发的首选模型。但是因为 epoll 底层的回调机制，在低并发、高活跃 fd 的场景下，select/poll 并不会比 epoll 差，可能性能会更高。
