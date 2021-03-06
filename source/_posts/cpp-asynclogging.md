---
title: C++实现高性能日志框架
date: 2020-10-28 16:31:43
tags:
- c++
- async
- logging
categories:
- c++
copyright: true
---

在一个优秀的系统中，日志是不可或缺的组成部分。尤其是在服务端编程中，日志更是必不可少的，如果没有日志，我们将不得不依赖客户或支持团队，让他们描述在什么情况下发生了什么。随后在开发环境中重现问题，并进行各种调试，直至完全修复错误为止，然而这一般耗费很长时间。如果有了日志的帮助，可以从日志中快速的定位异常，并解决问题。
使用日志的好处并不止以上一点，还包括：
<!-- more -->
1. 采集和分析用户行为
2. 可用于数据统计及分析
3. 追踪程序执行的过程
4. 程序性能检测
5. 等等。

## 关于重复造轮子

在开源技术大爆发的时代，每一个领域中都有很多很好的解决方案。C++ 日志框架也很等多成熟的方案，如：Boost.Log、glog、Log4cpp、log4cplus、loguru、Easylogging++ 等等的优秀日志框架可以使用。
那么已经有了这么多成熟的方案下，还有必要去重新造轮子吗？没有必要。话虽如此，但是还是有必要去了解关于轮子的各个细节。在学习轮子的过程中，也能提高我们自己的编程能力和各种技术的理解。

## 高性能的日志库

在陈硕先生的《Linux多线程服务端编程》一书中提到，一个高性能的日志库应具备以下需求：

* 功能需求：
  1. 日志消息多种级别（level）。（必须）
  2. 日志消息可能有多个目的地（文件、socket、SMTP等）。（非必须）
  3. 日志消息格式可配置。（非必须）
  4. 可以设置运行时过滤器。（非必须）
* 性能需求：
  1. 每秒写几千上万条日志的时候没有明显的性能损失。
  2. 能应对一个进程产生大量日志数据的场景。
  3. 不阻塞正常的执行流程。
  4. 在多线程程序中，不造成争用。

在一个复杂的系统环境中，每一条日志应该对应不同的级别，比如 INFO、WARNING、ERROR 等，在开发环境中应增加 TRACE、DEBUG 等，这有助于开发人员快速定位问题和后续对日志数据进行统计以及分析。在不同的环境中调整不同级别的日志等级，还可减少日志的数目，如在生产环境中，可关闭掉 TRACE 和 DEBUG 日志的输出，只留下重要的日志。
性能要求是重中之重，一个程序的大部分资源应该用于处理业务，只有少部分资源来用做日志输出，所有只有日志库足够高效，程序员才敢在代码中输出足够多的诊断信息。

## 如何实现高性能的日志框架

我所设想的日志库应该满足以下需求：

1. 多线程程序并发写日志，即线程安全。
2. 一个进程的多线程只写到一个文件中。
3. 文件滚动，在日志文件满足特定需求（文件大小、特定时间等）后，需要将后续日志写入到新的日志文件中。
4. 只能占用程序少量资源。

为了满足以上需求，异步日志是必须的，因为在业务处理线程中直接往磁盘写数据的话，写操作可能会阻塞业务线程，这将导致程序性能下降。
一个解决方案是，将日志进行分离，专门启动一条线程用于写磁盘的操作，业务线程发来的数据插入到一个队列中（或者一个buffer块中），后端线程取出数据并写入磁盘，这样就不会影响到业务线程了。
那么使用队列还是双缓冲技术呢？
这里我采用双缓冲技术，因为使用队列的话，每次产生一条日志消息都需要通知(notify_one())后端线程。
双缓冲技术的基本思路是，使用两块 buffer，其中，buffer A 专门用于前端业务填充数据，后端线程负责将 buffer B 中的数据写入到文件中，当 buffer A 写满后，交换 A 和 B，让后端将 A 中的数据写入到文件中，而前端则往 B 中填数据。实际编写过程中，将不止用到两块 buffer，如果两块 buffer 都满了，并且后端线程还在进行磁盘 IO 操作的话，则需要临时申请 buffer 用来填充数据。当后端线程能正常处理数据后，将临时申请的 buffer 销毁，只保留两块 buffer。
这样做的好处是，创建日志消息时不需要等待磁盘文件操作，也避免了每条日志都需要触发后端日志线程。

如下图所示，异步日志的框架图：

![AsyncLogging.png](https://i.loli.net/2020/10/29/1HQT9A4Z2iranfV.png)

根据以上的大致思路，我实现了一个异步日志库 [momoko-log](https://github.com/Grizzly1127/momoko-log)，已托管到 github 上。

## momoko-log

momoko-log 是基于 C++11 开发的高性能异步日志框架，该框架只能用于 Linux 中，不支持跨平台。
momoko-log 日志库特点如下：

1. 每秒可写百万条数据到磁盘中。
2. 多线程程序并发写日志到一个日志文件中。
3. 文件滚动。
4. 日志多级别输出。
5. 接口简单(重载了多种数据类型的 `<<` 操作符)，使用方便。
6. 采用批处理方式记录日志。

momoko-log 日志输出格式固定，以避免频繁改动日志格式对后面日志数据分析加强难度。
日志格式如下：

```text
timestamp threadId logLevel file:line ->  content
```

使用方式如下：

```c++
#include "momoko-log/logger.h"

int main() {
    // set log level
    //SET_LOGLEVEL(momoko::Logger::INFO)

    // set to asynchronous logger
    LOG_SET_ASYNC(1)

    LOG_INFO << "info message, num." << 1;
    LOG_WARN << "warning message, num." << 2;
    LOG_ERROR << "error message, num." << 3;
    return 0;
}
```
