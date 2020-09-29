---
title: IO多路复用 - poll
date: 2020-09-25 10:52:36
tags:
- I/O多路复用
- poll
categories:
- net
---

上一篇[《IO多路复用 - select》](../net-io-multiplexing-select)介绍了 select。接下来我们来解析一下 poll。
<!-- more -->

## poll介绍

### poll相关函数

通过 `man poll` 命令可以查看 poll 的函数原型如下：

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

具体参数详解：

* fds: 指向 pollfd 结构体数组的指针。
* nfds: 表示 fds 结构体数组的长度。
* timeout: 设定 poll 的超时时间，单位为毫秒，其中：
  * 值为 -1 ：poll 永远阻塞。
  * 值为 0 ：poll 立即返回。
  * 值为正数： 设定超时时间。

函数返回：

* 返回值大于 0 ：表示poll由于监听的文件描述符就绪返回，并且返回结果就是就绪的文件描述符的个数。
* 返回值等于 0 ：poll 超时。
* 返回值小于 0 ：发生错误。

pollfd 结构体：

```c
struct pollfd {
    int fd;        // 文件描述符
    short events;  // 等待的事件（POLLIN/POLLOUT/POLLERR）
    short revents; // 实际发生的事件，不需赋值
};
```

events & revents 取值如下：

|事件|描述|是否可作为输入(events)|是否可作为输出(revents)|
|---|---|:---:|:---:|
|POLLIN|数据可读（包括普通数据&优先数据）|√|√|
|POLLOUT|数据可写（普通数据&优先数据）|√|√|
|POLLRDNORM|普通数据可读|√|√|
|POLLRDBAND|优先级带数据可读（linux不支持）|√|√|
|POLLPRI|高优先级数据可读，比如TCP带外数据|√|√|
|POLLWRNORM|普通数据可写|√|√|
|POLLWRBAND|优先级带数据可写|√|√|
|POLLRDHUP|TCP连接被对端关闭，或者关闭了写操作，由GNU引入|√|√|
|POPPHUP|挂起|×|√|
|POLLERR|错误|×|√|
|POLLNVAL|文件描述符没有打开|×|√|

**注意：**
pollfd 结构体中的 events 由用户来设置，告诉内核我们关注的事件，而 revents 是返回时内核设置的，以说明文件描述符发生了什么事件。

### poll使用

使用流程图如下：
![poll](poll.png)

实现简单的服务代码：

```c++
#include <cstdio>
#include <sys/types.h>
#include <sys/socket.h>
#include <poll.h>
#include <netdb.h>
#include <netinet/in.h>
#include <errno.h>
#include <unistd.h>
#include <iostream>

#define BUF_SIZE 10240
#define CLIENT_SIZE 1024

int main() {
    int ret = 0;
    int server_sockfd, client_sockfd;
    struct sockaddr_in server_address;
    struct sockaddr_in client_address;
    int client_len = 0;
    struct pollfd clients[CLIENT_SIZE];
    int conn_count = 0; // 当前客户端连接数

    server_sockfd = socket(AF_INET, SOCK_STREAM, 0);//建立服务器端socket
    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = htonl(INADDR_ANY);
    server_address.sin_port = htons(8888);
    bind(server_sockfd, (struct sockaddr *)&server_address, sizeof(server_address));
    listen(server_sockfd, 5); //监听队列最多容纳5个

    clients[0].fd = server_sockfd;
    clients[0].events = POLLIN | POLLERR;

    // 初始化所有客户端
    for (int i = 1; i < CLIENT_SIZE; ++i) {
        clients[i].fd = -1;
    }

    printf("server start\n");
    while(1) {
        ret = poll(clients, conn_count + 1, -1);
        if (ret == -1) {
            printf("server error\n");
            exit(1);
        }
        if (ret == 0) {
            printf("timeout\n");
            continue;
        }

        for (int i = 0; i < conn_count + 1; ++i) {
            // 客户端关闭或者其他错误
            if ((clients[i].revents & POLLRDHUP) || (clients[i].revents & POLLERR)) {
                int fd = clients[i].fd;
                clients[i] = clients[conn_count];
                --i;
                --conn_count;
                close(fd);
            }
            // 新的客户端连接
            else if ((clients[i].fd == server_sockfd) && (clients[i].revents & POLLIN)) {
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

                ++conn_count;
                clients[conn_count].fd = client_sockfd;
                clients[conn_count].events = POLLIN | POLLERR;
                clients[conn_count].revents = 0;
            }
            // 有可读数据
            else if (clients[i].revents & POLLIN) {
                char buf[BUF_SIZE] = {0};
                int len = recv(clients[i].fd, buf, BUF_SIZE - 1, 0);
                if (len > 0) {
                    printf("%s\n", buf);
                } else if (len == 0) {
                    printf("client %d exit\n", clients[i].fd);
                } else {
                    fprintf(stderr, "recv: %d\n", errno);
                    exit(1);
                }
            }
        }
    }
    return 0;
}
```

### poll实现原理

poll 的源码也在 `fs/select.c` 文件中。
poll() 的系统调用宏定义：

```c
SYSCALL_DEFINE3(poll, struct pollfd __user *, ufds, unsigned int, nfds,
        long, timeout_msecs)
{
    struct timespec end_time, *to = NULL;
    int ret;

    if (timeout_msecs >= 0) {
        to = &end_time;
        // 设定超时时间
        poll_select_set_timeout(to, timeout_msecs / MSEC_PER_SEC,
            NSEC_PER_MSEC * (timeout_msecs % MSEC_PER_SEC));
    }

    // 主要函数
    ret = do_sys_poll(ufds, nfds, to);

    if (ret == -EINTR) {
        struct restart_block *restart_block;

        restart_block = &current_thread_info()->restart_block;
        restart_block->fn = do_restart_poll;
        restart_block->poll.ufds = ufds;
        restart_block->poll.nfds = nfds;

        if (timeout_msecs >= 0) {
            restart_block->poll.tv_sec = end_time.tv_sec;
            restart_block->poll.tv_nsec = end_time.tv_nsec;
            restart_block->poll.has_timeout = 1;
        } else
            restart_block->poll.has_timeout = 0;

        ret = -ERESTART_RESTARTBLOCK;
    }
    return ret;
}
```

主要处理函数在 do_sys_poll() 中：

```c
int do_sys_poll(struct pollfd __user *ufds, unsigned int nfds,
        struct timespec *end_time)
{
    struct poll_wqueues table;
    int err = -EFAULT, fdcount, len, size;
    /* Allocate small arguments on the stack to save memory and be
       faster - use long to make sure the buffer is aligned properly
       on 64 bit archs to avoid unaligned access */
    /* 为了加快处理速度和提高系统性能，这里优先定义好一个大小为POLL_STACK_ALLOC的栈空间，
       该栈空间转换为poll_list结构体，以存储需要被检测的socket描述符 */
    long stack_pps[POLL_STACK_ALLOC/sizeof(long)];
    struct poll_list *const head = (struct poll_list *)stack_pps;
    struct poll_list *walk = head;
    unsigned long todo = nfds;

    if (nfds > current->signal->rlim[RLIMIT_NOFILE].rlim_cur)
        return -EINVAL;

    // 通过计算得到前面分配的栈空间能存储多少个pollfd结构
    len = min_t(unsigned int, nfds, N_STACK_PPS);
    for (;;) {
        walk->next = NULL;
        walk->len = len;
        if (!len)
            break;

        // 从用户态空间复制len个pollfd拷贝到内核空间中
        if (copy_from_user(walk->entries, ufds + nfds-todo,
                    sizeof(struct pollfd) * walk->len))
            goto out_fds;

        todo -= walk->len;
        if (!todo)
            break;

        /* POLLFD_PER_PAGE表示一页能存储多少个pollfd，可以计算出来，一页是4K，
           而pollfd的大小为8个字节，也就是一页能存储512个pollfd。
           如果在分配一页内存之后，还不够nfds使用，则继续下一个循环进行分配 */
        len = min(todo, POLLFD_PER_PAGE);
        size = sizeof(struct poll_list) + sizeof(struct pollfd) * len;
        walk = walk->next = kmalloc(size, GFP_KERNEL);
        if (!walk) {
            err = -ENOMEM;
            goto out_fds;
        }
    }

    poll_initwait(&table);
    // 最重要的处理部分
    fdcount = do_poll(nfds, head, &table, end_time);
    poll_freewait(&table);

    // 将链表上的所有pollfd中revents状态写入到用户空间
    for (walk = head; walk; walk = walk->next) {
        struct pollfd *fds = walk->entries;
        int j;

        for (j = 0; j < walk->len; j++, ufds++)
            if (__put_user(fds[j].revents, &ufds->revents))
                goto out_fds;
    }

    err = fdcount;
out_fds:
    // 之前调用kmalloc分配的内存现在进行释放
    walk = head->next;
    while (walk) {
        struct poll_list *pos = walk;
        walk = walk->next;
        kfree(pos);
    }

    return err;
}
```

接下来是重中之重的 do_poll() 函数：

```c
static int do_poll(unsigned int nfds,  struct poll_list *list,
           struct poll_wqueues *wait, struct timespec *end_time)
{
    poll_table* pt = &wait->pt;
    ktime_t expire, *to = NULL;
    int timed_out = 0, count = 0;
    unsigned long slack = 0;

    /* Optimise the no-wait case */
    if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
        pt = NULL;
        timed_out = 1;
    }

    if (end_time && !timed_out)
        slack = estimate_accuracy(end_time);

    for (;;) {
        struct poll_list *walk;

        // 对所有的struct pollfd循环，以调用do_pollfd函数。
        for (walk = list; walk != NULL; walk = walk->next) {
            struct pollfd * pfd, * pfd_end;

            pfd = walk->entries;
            pfd_end = pfd + walk->len;
            for (; pfd != pfd_end; pfd++) {
                /*
                 * Fish for events. If we found one, record it
                 * and kill the poll_table, so we don't
                 * needlessly register any other waiters after
                 * this. They'll get immediately deregistered
                 * when we break out and return.
                 */
                // 调用 do_pollfd() 以检查socket文件描述符的状态变化，如果有变化，则count加1
                if (do_pollfd(pfd, pt)) {
                    count++;
                    pt = NULL;
                }
            }
        }
        /*
         * All waiters have already been registered, so don't provide
         * a poll_table to them on the next loop iteration.
         */
        pt = NULL;
        if (!count) {
            count = wait->error;
            /* 检查是否有需要处理的信号，这里的意思是就算是poll调用进入到sys_poll系统调用之后，
             * 也可以接收外部信号，从而退出当前系统调用（因为我们知道一般的系统调用都不会被中断的，
             * 所以系统调用一般都尽量很快的返回）
             */
            if (signal_pending(current))
                count = -EINTR;
        }

        // for循环退出的条件：如果有文件描述符发生变化，则退出，或者超时退出
        if (count || timed_out)
            break;

        /*
         * If this is the first loop and we have a timeout
         * given, then we convert to ktime_t and set the to
         * pointer to the expiry value.
         */
        if (end_time && !to) {
            expire = timespec_to_ktime(*end_time);
            to = &expire;
        }

        if (!poll_schedule_timeout(wait, TASK_INTERRUPTIBLE, to, slack))
            timed_out = 1;
    }
    return count;
}
```
