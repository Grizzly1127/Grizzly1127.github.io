---
title: IO多路复用 - epoll
date: 2020-09-25 15:47:20
tags:
- I/O多路复用
- epoll
categories:
- net
---

前面讲了 select 和 poll 的原理，接下来我们学习它们改进后的增强版本：epoll。
<!-- more -->

## epoll介绍

### epoll相关函数

在 `man` 手册中，可以查到 epoll 的函数原型如下：

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

具体参数详解：

* epoll_create : 创建 epoll 句柄。
  * size : 并没有什么用，但是必须要大于 0，否则创建 epoll 句柄失败。
* epoll_ctl : epoll 事件注册函数。
  * epfd : 由 epoll_create 创建的 epoll 句柄。
  * op : fd 的操作类型：
    * EPOLL_CTL_ADD : 注册新的 fd 到 epfd 中。
    * EPOLL_CTL_MOD : 修改已注册的 fd 的监听事件。
    * EPOLL_CTL_DEL : 从 epfd 中删除一个 fd。
  * fd : 监听的文件描述符。
  * event : 要监听的事件，事件属性可以查看下表。
* epoll_wait : 等待 epoll 事件的产生。
  * epfd : 由 epoll_create 创建的 epoll 句柄。
  * events : 内核得到的就绪事件集合。
  * maxevents : 内核events的大小。
  * timeout : 设定超时时间：
    * 0 : 立即返回。
    * -1 : 阻塞。
    * 正数 : 设定超时时间。

epoll_event 结构如下：

```c
typedef union epoll_data {
    void        *ptr;
    int          fd;
    __uint32_t   u32;
    __uint64_t   u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t   events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

event 中指定的位掩码的值如下：

|事件|描述|是否可作为 epoll_ctl() 的输入|是否可作为 epoll_wait() 的输出|
|---|---|:---:|:---:|
|EPOLLIN|表示对应的文件描述符可读|√|√|
|EPOLLOUT|表示对应的文件描述符可写|√|√|
|EPOLLRDHUP|Linux 2.6.17 版本后添加的事件，表示对端断开连接|√|√|
|EPOLLPRI|表示对应的文件描述符有紧急数据可读（外带数据）|√|√|
|EPOLLERR|表示对应的文件描述符发生错误|×|√|
|EPOLLHUP|表示对应的文件描述符被挂断|×|√|
|EPOLLET|采用边缘触发事件通知|√|×|
|EPOLLONESHOT|在完成事件通知之后禁用检查|√|×|

**工作模式：**
epoll 除了提供 select/poll 那种 IO 事件的水平触发（level-triggered）外，还提供了边缘触发模式（edge-triggered），这就是的用户空间程序有可能缓存 IO 状态，减少 `epoll_wait/epoll_pwait` 的调用，提高应用程序效率。

* 水平触发（LT） : 默认工作模式，即当 epoll_wait 检测到某描述符事件就绪并通知应用程序时，应用程序可以不立即处理该事件。下次调用 epoll_wait 时，会再次通知此事件。
* 边缘触发（ET） : 当 epoll_wait 检测到某描述符事件就绪并通知应用程序时，应用程序必须立即处理该事件。如果不处理，下次调用 epoll_wait 时，不会再次通知此事件。（直到你做了某些操作导致该描述符变成未就绪状态了，也就是说边缘触发只在状态由未就绪变为就绪时只通知一次）。

**epoll 工作在 ET 模式的时候，必须使用非阻塞套接字，以避免由于一个文件句柄的阻塞读/写操作把处理多个文件描述符的任务饿死。**

## epoll使用

使用流程图如下：
![epoll](epoll.png)

实现简单的服务代码：

```c++
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>
#include <stdio.h>
#include <iostream>

#define MAX_EVENTS 500
#define BUF_SIZE 10240

static void set_nonblocking(int fd) {
    int opts = fcntl(fd, F_GETFL);
    if (opts < 0) {
        printf("fcntl(F_GETFL)\n");
        exit(1);
    }
    opts |= O_NONBLOCK;
    if (fcntl(fd, F_SETFL, opts) < 0) {
        printf("fcntl(F_SETFL)\n");
        exit(1);
    }
}

static void add_event(int epollfd, int fd, int event) {
    struct epoll_event ev;
    ev.events = event;
    ev.data.fd = epollfd;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &ev);
}

static void delete_event(int epollfd, int fd, int event) {
    struct epoll_event ev;
    ev.events = event;
    ev.data.fd = epollfd;
    epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, &ev);
}

static void modify_event(int epollfd, int fd, int event) {
    struct epoll_event ev;
    ev.events = event;
    ev.data.fd = epollfd;
    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &ev);
}


int main() {
    int server_sockfd, client_sockfd;
    struct sockaddr_in server_address;
    struct sockaddr_in client_address;
    int client_len = 0;

    server_sockfd = socket(AF_INET, SOCK_STREAM, 0);//建立服务器端socket
    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = htonl(INADDR_ANY);
    server_address.sin_port = htons(8888);
    bind(server_sockfd, (struct sockaddr *)&server_address, sizeof(server_address));
    listen(server_sockfd, 5); //监听队列最多容纳5个

    int epollfd = epoll_create(1);     //这个参数已经被忽略，但是仍然要大于0
    if (epollfd < 0) {
        printf("Create epoll error!\n");
        return -1;
    }

    add_event(epollfd, server_sockfd, EPOLLIN|EPOLLET); // 设置为边缘触发模式

    struct epoll_event events[MAX_EVENTS];
    while (1) {
        // 获取已经准备好的描述符事件
        int count = epoll_wait(epollfd, events, MAX_EVENTS, -1);
        int fd;
        for (int i = 0; i < count; ++i) {
            fd = events[i].data.fd;
            // 新的客户端连接
            if ((fd == server_sockfd) &&(events[i].events & EPOLLIN)) {
                client_len = sizeof(client_address);
                // 接受新的客户端连接
                client_sockfd = accept(server_sockfd, (struct sockaddr *)&client_address, (socklen_t *)&client_len);
                if (client_sockfd == -1) {
                    if (errno == EINTR) {
                        continue;
                    } else {
                        printf("accept error\n");
                        exit(1);
                    }
                }
                // 边缘模式必须要设置为非阻塞模式fd
                set_nonblocking(fd);
            }
            // 有可读事件
            else if (events[i].events & EPOLLIN) {
                char buf[BUF_SIZE] = {0};
                int nread;
                int len = 0;
                // 边缘触发模式需要讲数据读取完，否则下次不会再通知
                while ((nread = read(fd, buf + len, BUFSIZ - 1)) > 0) {
                    len += nread;
                }
                if (len > 0) {
                    printf("%s\n", buf);
                } else if (len == 0) {
                    printf("client %d exit\n", fd);
                    close(fd);
                    delete_event(epollfd, fd, EPOLLIN);
                } else {
                    fprintf(stderr, "recv: %d\n", errno);
                    close(fd);
                    delete_event(epollfd, fd, EPOLLIN);
                }
            }
        }
    }
    return 0;
}
```

### epoll实现原理

epoll 的源码也在 `fs/eventpoll.c` 文件中。

#### epoll_create()

系统调用宏定义：

```c
SYSCALL_DEFINE1(epoll_create, int, size)
{
    if (size <= 0) // 此处已经遗弃 size 参数，只要 size 大于 0 即可
        return -EINVAL;

    // 主要调用函数
    return sys_epoll_create1(0);
}
```

用户调用 epoll_create 时，检查完 size 参数后，直接调用了 sys_epoll_create1() 函数来完成主要的工作。
sys_epoll_create1() 如下：

```c
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
    int error;
    struct eventpoll *ep = NULL;

    /* Check the EPOLL_* constant for consistency.  */
    // 检查 EPOLL_* 常量的一致性
    BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

    if (flags & ~EPOLL_CLOEXEC)
        return -EINVAL;
    /*
     * 为 ep 分配内存并初始化，存储在file结构的private_data成员中。
     * private_data成员用来存储文件描述符真正对应的对象。例如
     * 如果文件描述符是一个套接字的话，其对应的file实例的private_data
     * 成员存储的就是一个socket实例。
     */
    error = ep_alloc(&ep);
    if (error < 0)
        return error;
    /*
     * 创建eventpoll文件，这个文件的file_operations为eventpoll_fops，
     * 私有的数据为eventpoll实例
     */
    error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep,
                 flags & O_CLOEXEC);
    if (error < 0)
        ep_free(ep);

    return error;
}
```

其中，有一个很重要的结构体 `eventpoll`，其结构如下：

```c
struct eventpoll {
    spinlock_t lock; // 自旋锁
    struct mutex mtx;

    wait_queue_head_t wq; // sys_epoll_wait()使用的等待队列
    wait_queue_head_t poll_wait; // file->poll()使用的等待队列

    /*
     * 文件描述符就绪列表，用户调用 epoll_wait 的时候，将 rdllist 中的 epitem 出列，
     * 将触发的事件拷贝到用户空间，之后判断 epitem 是否需要重新添加回 rdllist。
     */
    struct list_head rdllist;

    /*
     * 红黑树根节点，用于存储监听的文件描述符。一个 fd 被添加（EPOLL_CTL_ADD）到 epoll 中之后，
     * 内核会为它生成对应的 epitem 结构对象，epitem 被添加到 rbr 中。
     */
    struct rb_root rbr; // 红黑树，用于存储监听的文件描述符

    /*
     * 单链表，就绪事件拷贝到用户空间时，将所有 epitem 链接起来
     */
    struct epitem *ovflist;

    /* 创建 epoll 的用户 */
    struct user_struct *user;
};
```

从上面的结构体中可以看出，epoll 底层两个重要的数据结构是 `红黑树` 和 `单链表的就绪队列`，其中红黑树用于管理所有监听的 fd，就绪队列用于将产生了事件的 fd 传回给用户。

那么，创建 epoll 完成后，就需要对文件描述符进行监听、删除、修改等操作，这将会使用到 `epoll_ctr()` 函数。

#### epoll_ctr()

系统调用宏定义：

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    int error;
    struct file *file, *tfile;
    struct eventpoll *ep;
    struct epitem *epi;
    struct epoll_event epds;

    error = -EFAULT;

    // 检查是否需要从用户空间拷贝 event 参数，如果需要拷贝则调用 copy_from_user 来拷贝
    if (ep_op_has_event(op) &&
        copy_from_user(&epds, event, sizeof(struct epoll_event)))
        goto error_return;

    // 在 epoll_create1 创建了 file 实例来存储 eventpoll 并生成了 epfd，此时通过 epfd 来获取
    error = -EBADF;
    file = fget(epfd);
    if (!file)
        goto error_return;

    // 获取要操作的文件描述符对应的 file 实例
    tfile = fget(fd);
    if (!tfile)
        goto error_fput;

    // 检查对应的文件是否支持 poll
    error = -EPERM;
    if (!tfile->f_op || !tfile->f_op->poll)
        goto error_tgt_fput;


    // 检查 fd 对应的文件是否是一个 eventpoll 文件
    error = -EINVAL;
    if (file == tfile || !is_file_epoll(file))
        goto error_tgt_fput;


    // 前面说过，eventpoll 是存储在 private_data 当中的，所以此时取出
    ep = file->private_data;

    mutex_lock(&ep->mtx);

    // 在 eventpoll 中存储文件描述符的红黑树中查找 fd 对应的 epitem 实例
    epi = ep_find(ep, tfile, fd);

    error = -EINVAL;
    switch (op) {
    // 添加
    case EPOLL_CTL_ADD:
        // 如果要添加的 fd 不存在，则调用 ep_insert 插入到红黑树中
        if (!epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_insert(ep, &epds, tfile, fd);
        } else // 如果存在则返回 EEXIST 错误
            error = -EEXIST;
        break;
    // 删除
    case EPOLL_CTL_DEL:
        // 如果要删除的 fd 存在，则调用 ep_remove 从红黑树中删除
        if (epi)
            error = ep_remove(ep, epi);
        else // 如果不存在则返回 ENOENT 错误
            error = -ENOENT;
        break;
    // 修改
    case EPOLL_CTL_MOD:
        // 如果要修改的 fd 存在，则调用 ep_modify 修改红黑树中对应 fd 的感兴趣的事件
        if (epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_modify(ep, epi, &epds);
        } else
            error = -ENOENT;
        break;
    }
    mutex_unlock(&ep->mtx);

error_tgt_fput:
    fput(tfile);
error_fput:
    fput(file);
error_return:

    return error;
}
```

epoll_ctl() 函数支持三种操作：添加、删除和修改，这些操作是基于存储文件描述符的红黑树上的，都是对红黑树进行相应的操作。
接下来我们看看最重要的函数 `epoll_wait()`。

#### epoll_wait()

系统调用宏定义：

```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
        int, maxevents, int, timeout)
{
    int error;
    struct file *file;
    struct eventpoll *ep;

    // 检查 maxevents 参数是否合法
    if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
        return -EINVAL;

    // 验证用户空间传入的 events 指向的内存是否可写
    if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {
        error = -EFAULT;
        goto error_return;
    }

    // 获取 epoll_create 中创建的文件
    error = -EBADF;
    file = fget(epfd);
    if (!file)
        goto error_return;

    // 检查 fd 对应的文件是否是一个 eventpoll 文件
    error = -EINVAL;
    if (!is_file_epoll(file))
        goto error_fput;

    // 从 private_data 中获取 eventpoll
    ep = file->private_data;

    // 主要函数
    error = ep_poll(ep, events, maxevents, timeout);

error_fput:
    fput(file);
error_return:

    return error;
}
```

epoll_wait 的主要工作在 ep_poll() 函数中完成：

```c

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
            int maxevents, long timeout)
{
    int res, eavail;
    unsigned long flags;
    long jtimeout;
    wait_queue_t wait;

    // timeout 是以毫秒为单位
    jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?
        MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;

retry:
    spin_lock_irqsave(&ep->lock, flags);

    res = 0;
    if (list_empty(&ep->rdllist)) {
        /*
         * We don't have any available event to return to the caller.
         * We need to sleep here, and we will be wake up by
         * ep_poll_callback() when events will become available.
         */
        init_waitqueue_entry(&wait, current);
        wait.flags |= WQ_FLAG_EXCLUSIVE;
        // 将当前进程加入到 eventpoll 的等待队列中，等待文件状态就绪或者超时或者被信号中断
        __add_wait_queue(&ep->wq, &wait);

        for (;;) {
            /*
             * We don't want to sleep if the ep_poll_callback() sends us
             * a wakeup in between. That's why we set the task state
             * to TASK_INTERRUPTIBLE before doing the checks.
             */
            set_current_state(TASK_INTERRUPTIBLE);
            // 如果就绪队列不为空，也就是说已经有文件的状态就绪或者超时，则退出循环
            if (!list_empty(&ep->rdllist) || !jtimeout)
                break;
            // 如果当前进程收到信号，则退出循环，返回 EINTR
            if (signal_pending(current)) {
                res = -EINTR;
                break;
            }

            spin_unlock_irqrestore(&ep->lock, flags);
            /*
             * 等待 ep_poll_callback() 将当前进程唤醒或者超时，返回值是剩余的时间。
             * 从这里开始进程会进入睡眠状态，让出处理器，直到某些文件的状态就绪或者超时。
             * 当文件状态就绪时，eventpoll 的回调函数 ep_poll_callback() 会唤醒
             * 在 ep->wq 指向的等待队列中的进程。
             * （ep_poll_callback() 回调函数在 ep_insert() 时注册）
             */
            jtimeout = schedule_timeout(jtimeout);
            spin_lock_irqsave(&ep->lock, flags);
        }
        __remove_wait_queue(&ep->wq, &wait);

        set_current_state(TASK_RUNNING);
    }

    /*
     * ep->ovflist 链表存储的向用户传递事件时暂存就绪的文件。
     * 所以不管是就绪队列 ep->rdllist 不为空，或者 ep->ovflist 不等于
     * EP_UNACTIVE_PTR，都有可能现在已经有文件的状态就绪。
     * ep->ovflist 不等于 EP_UNACTIVE_PTR 有两种情况，一种是 NULL，此时
     * 可能正在向用户传递事件，不一定就有文件状态就绪，
     * 一种情况时不为 NULL，此时可以肯定有文件状态就绪，
     * 参见 ep_send_events()。
     */
    eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;

    spin_unlock_irqrestore(&ep->lock, flags);

    /*
     * 如果没有被信号中断，并且有事件就绪，
     * 但是没有获取到事件(有可能被其他进程获取到了)，
     * 并且没有超时，则跳转到retry标签处，重新等待
     * 文件状态就绪。
     */
    if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && jtimeout)
        goto retry;

    // 返回获取到的事件的个数或者错误码
    return res;
}
```

ep_poll() 如果有事件发生，则调用 `ep_send_events()/ep_scan_ready_list()` 将发生的事件拷贝到用户空间中。

```c
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    struct ep_send_events_data esed;

    esed.maxevents = maxevents;
    esed.events = events;

    return ep_scan_ready_list(ep, ep_send_events_proc, &esed);
}

static int ep_scan_ready_list(struct eventpoll *ep,
                  int (*sproc)(struct eventpoll *,
                       struct list_head *, void *),
                  void *priv)
{
    int error, pwake = 0;
    unsigned long flags;
    struct epitem *epi, *nepi;
    LIST_HEAD(txlist);

    /*
     * We need to lock this because we could be hit by
     * eventpoll_release_file() and epoll_ctl().
     */
    mutex_lock(&ep->mtx);

    spin_lock_irqsave(&ep->lock, flags);
    // 将就绪队列中就绪的文件链表暂存在临时表头 txlist 中，并且初始化就绪队列。
    list_splice_init(&ep->rdllist, &txlist);
    /*  
     * 将 ovflist 置为 NULL，表示此时正在向用户空间传递
     * 事件。如果此时有文件状态就绪，不会放在
     * 就绪队列中，而是放在 ovflist 链表中。
     */
    ep->ovflist = NULL;
    spin_unlock_irqrestore(&ep->lock, flags);

    // 调用 sproc 指向的函数 ep_send_events_proc() 将就绪队列中的事件存入用户传入的内存中。
    error = (*sproc)(ep, &txlist, priv);

    spin_lock_irqsave(&ep->lock, flags);
    /*
     * 在调用 ep_send_events_proc() 的过程中，可能有文件状态就绪，这些事件
     * 会暂存在 ovflist 链表中，所以这里要将 ovflist 中的事件移到就绪队列中
     */
    for (nepi = ep->ovflist; (epi = nepi) != NULL;
        nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
        if (!ep_is_linked(&epi->rdllink))
            list_add_tail(&epi->rdllink, &ep->rdllist);
    }

    // 重新初始化 ovflist，表示传递事件已完成
    ep->ovflist = EP_UNACTIVE_PTR;

    /*
     * 如果 ep_send_events_proc() 中处理出错或者某些文件的触发方式设置为 LT，
     * txlist 中可能还有事件，需要将这些就绪的事件重新添加到 eventpoll 的就绪度列中。
     */
    list_splice(&txlist, &ep->rdllist);

    if (!list_empty(&ep->rdllist)) {
        if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
        if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }
    spin_unlock_irqrestore(&ep->lock, flags);

    mutex_unlock(&ep->mtx);

    // 对轮询等待队列执行安全唤醒
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);

    return error;
}


static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
                   void *priv)
{
    struct ep_send_events_data *esed = priv;
    int eventcnt;
    unsigned int revents;
    struct epitem *epi;
    struct epoll_event __user *uevent;


    for (eventcnt = 0, uevent = esed->events;
         !list_empty(head) && eventcnt < esed->maxevents;) {
        epi = list_first_entry(head, struct epitem, rdllink);

        list_del_init(&epi->rdllink);
        /*
         * 调用文件的poll函数有两个作用，一是在文件的唤醒
         * 队列上注册回调函数，二是返回文件当前的事件状
         * 态，如果第二个参数为NULL，则只是查看文件当前
         * 状态。
         */
        revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &
            epi->event.events;

        if (revents) {
            // 向用户空间内存传值，如果失败，将当前的 epitem 实例重新放回到链表中。
            if (__put_user(revents, &uevent->events) ||
                __put_user(epi->event.data, &uevent->data)) {
                list_add(&epi->rdllink, head);
                // 如果此时已经获取了部分事件，则返回已经获取的事件个数，否则返回 EFAULT
                return eventcnt ? eventcnt : -EFAULT;
            }
            eventcnt++;
            uevent++;
            if (epi->event.events & EPOLLONESHOT)
                epi->event.events &= EP_PRIVATE_BITS;
            /*
             * 如果是水平触发模式，需要将当前的 epitem 实例添加回链表中，
             * 下次读取事件时会再次上报
             */
            else if (!(epi->event.events & EPOLLET)) {

                list_add_tail(&epi->rdllink, &ep->rdllist);
            }
        }
    }

    return eventcnt;
}
```
