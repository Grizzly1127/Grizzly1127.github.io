---
title: IO多路复用 - select
date: 2020-09-24 10:52:36
tags:
- I/O多路复用
- select
categories:
- net
---

在服务器的编程中经常会需要构造高性能的 I/O 模型，最常用的就是多路复用 I/O 模型。
多路复用的本质是同步非阻塞 I/O，多路复用的优势并不是单个连接处理的更快，而是在于能处理更多的链接。
<!-- more -->
在服务端的网络 I/O 编程过程中，需要同时处理多个客户端的数据时，可以利用多线程或者 I/O 多路复用技术进行处理。
在[《I/O模型浅析》](../net-io-model)中已经简单介绍过多路复用I/O，接下来介绍 select、poll 和 epoll。
这三者的源码在 Linux kernel 源码中，想要查看源码需要下载，两种下载方式：

1. 官方链接： <https://www.kernel.org/>
2. 如果不能科学上网，下载速度将会很慢，可以使用上海交大的源下载：
<http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/>

![kernel.png](https://i.loli.net/2020/10/29/PFJYs8yI6VMlaBv.png)
因为我使用的系统是 `CentOS release 6.8 (Final)`，其中 linux 内核版本为 `2.6.32`，所以我下载了 `linux-2.6.32.9.tar.gz`。
其它版本可自行选择下载。

---

## select介绍

### select相关函数

通过 `man select` 命令可以查看 select 的函数原型如下：

```c
// 需要包含的头文件
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

具体参数详解：

* nfds: 整数值，指集合中所有文件描述符的范围，即所有文件描述符的最大值加1。
* readfds: 指向 fd_set 结构的指针，文件描述符集合，检查该组文件描述符的可读性。
* writefds: 指向 fd_set 结构的指针，文件描述符集合，检查该组文件描述符的可写性。
* exceptfds: 指向 fd_set 结构的指针，文件描述符集合，检查该组文件描述符的异常条件。
* timeout: 设定 select 的超时时间，其中：
  * 值为NULL，则将 select() 设置为阻塞状态，当监视的文件描述符集合中的某一个描述符发生变化才会返回结果并向下执行。
  * 值等于0，则将 select() 置为非阻塞状态，执行 select() 后立即返回，无论文件描述符是否发生变化。
  * 值大于0，则将select()函数的超时时间设为这个值，在超时时间内阻塞，超时后返回结果。

函数返回：

* 正数：表示发生变化的文件描述符数量。
* 0：select 超时。
* -1：发生错误，将所有文件描述符集合清0，并通过 errno 输出错误详情。

以下是与 select() 函数相关的几个宏：

```c
#include <sys/select.h>
int FD_ZERO(int fd, fd_set *fdset);   //一个 fd_set类型变量的所有位都设为 0
int FD_CLR(int fd, fd_set *fdset);  //清除某个位时可以使用
int FD_SET(int fd, fd_set *fdset);   //设置变量的某个位置位
int FD_ISSET(int fd, fd_set *fdset); //测试某个位是否被置位
```

### select使用

使用流程图如下：
![select.png](https://i.loli.net/2020/10/29/7RFfJjZzT3sKSWh.png)

实现简单的服务代码：

```c++
#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <netinet/in.h>
#include <sys/time.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    int server_sockfd, client_sockfd;
    int server_len, client_len;
    struct sockaddr_in server_address;
    struct sockaddr_in client_address;
    int result;
    fd_set readfds, testfds;
    server_sockfd = socket(AF_INET, SOCK_STREAM, 0);//建立服务器端socket
    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = htonl(INADDR_ANY);
    server_address.sin_port = htons(8888);
    server_len = sizeof(server_address);
    bind(server_sockfd, (struct sockaddr *)&server_address, server_len);
    listen(server_sockfd, 5); //监听队列最多容纳5个
    FD_ZERO(&readfds);
    FD_SET(server_sockfd, &readfds);//将服务器端socket加入到集合中
    while(1)
    {
        char ch;
        int fd;
        int nread;
        testfds = readfds;//将需要监视的描述符集copy到select查询队列中，select会对其修改，所以一定要分开使用变量
        printf("server waiting\n");

        /*无限期阻塞，并测试文件描述符变动 */
        result = select(FD_SETSIZE, &testfds, (fd_set *)0,(fd_set *)0, (struct timeval *) 0); //FD_SETSIZE：系统默认的最大文件描述符
        if(result < 1)
        {
            perror("server5");
            exit(1);
        }

        /*扫描所有的文件描述符*/
        for(fd = 0; fd < FD_SETSIZE; fd++)
        {
            /*找到相关文件描述符*/
            if(FD_ISSET(fd,&testfds))
            {
                /*判断是否为服务器套接字，是则表示为客户请求连接。*/
                if(fd == server_sockfd)
                {
                    client_len = sizeof(client_address);
                    client_sockfd = accept(server_sockfd, (struct sockaddr *)&client_address, (socklen_t *)&client_len);
                    FD_SET(client_sockfd, &readfds);//将客户端socket加入到集合中
                    printf("adding client on fd %d\n", client_sockfd);
                }
                /*客户端socket中有数据请求时*/
                else
                {
                    ioctl(fd, FIONREAD, &nread);//取得数据量交给nread

                    /*客户数据请求完毕，关闭套接字，从集合中清除相应描述符 */
                    if(nread == 0)
                    {
                        close(fd);
                        FD_CLR(fd, &readfds); //去掉关闭的fd
                        printf("removing client on fd %d\n", fd);
                    }
                    /*处理客户数据请求*/
                    else
                    {
                        read(fd, &ch, 1);
                        sleep(5);
                        printf("serving client on fd %d\n", fd);
                        ch++;
                        write(fd, &ch, 1);
                    }
                }
            }
        }
    }
    return 0;
}
```

### select实现原理

select 的源码在 `fs/select.c` 文件中。
Linux 中的系统函数调用入口都是由宏 `SYSCALL_DEFINEx` 定义的，其中 `x` 为函数参数个数。在 `select.c` 中可以找到 select() 的系统调用宏定义：

```c
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
        fd_set __user *, exp, struct timeval __user *, tvp)
{
    struct timespec end_time, *to = NULL;
    struct timeval tv;
    int ret;

    // 是否设置超时时间
    if (tvp) {
        if (copy_from_user(&tv, tvp, sizeof(tv))) // 从用户空间拷贝数据到内核空间
            return -EFAULT;

        to = &end_time;
        if (poll_select_set_timeout(to,
                tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),
                (tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))
            return -EINVAL;
    }

    ret = core_sys_select(n, inp, outp, exp, to); // 主要函数
    ret = poll_select_copy_remaining(&end_time, tvp, 1, ret); // 如果有超时值, 并拷贝离超时时刻还剩的时间到用户空间的timeval中

    return ret;
}
```

主要处理函数在 core_sys_select() 中：

```c
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
               fd_set __user *exp, struct timespec *end_time)
{
    fd_set_bits fds; // fd_set_bits结构体中定义的全是指针，这些指针是用来指向描述符集合的
    void *bits;
    int ret, max_fds;
    unsigned int size;
    struct fdtable *fdt;
    /* Allocate small arguments on the stack to save memory and be faster */
    long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

    ret = -EINVAL;
    if (n < 0)
        goto out_nofds;

    /* max_fds can increase, so grab it once to avoid race */
    rcu_read_lock();
    fdt = files_fdtable(current->files); // 获取当前进程的文件描述符表
    max_fds = fdt->max_fds;
    rcu_read_unlock();
    if (n > max_fds) // 如果传入的n大于当前进程最大的文件描述符，给予修正
        n = max_fds;

    /*
     * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
     * since we used fdset we need to allocate memory in units of
     * long-words.
     */
    size = FDS_BYTES(n);
    // 以一个文件描述符占一bit来计算，传递进来的这些fd_set需要用掉多少个字
    bits = stack_fds;
    if (size > sizeof(stack_fds) / 6) {
        // 除6，为什么？因为每个文件描述符需要6个bitmaps
        /* Not enough space in on-stack array; must use kmalloc */
        ret = -ENOMEM;
        bits = kmalloc(6 * size, GFP_KERNEL); // stack中分配的太小，直接kmalloc
        if (!bits)
            goto out_nofds;
    }
    // 这里就可以明显看出struct fd_set_bits结构体的用处了。
    fds.in      = bits;
    fds.out     = bits +   size;
    fds.ex      = bits + 2*size;
    fds.res_in  = bits + 3*size;
    fds.res_out = bits + 4*size;
    fds.res_ex  = bits + 5*size;

    // get_fd_set仅仅调用copy_from_user从用户空间拷贝了fd_se
    if ((ret = get_fd_set(n, inp, fds.in)) ||
        (ret = get_fd_set(n, outp, fds.out)) ||
        (ret = get_fd_set(n, exp, fds.ex)))
        goto out;
    zero_fd_set(n, fds.res_in); // 对这些存放返回状态的字段清0
    zero_fd_set(n, fds.res_out);
    zero_fd_set(n, fds.res_ex);

    ret = do_select(n, &fds, end_time); // 关键函数，完成主要的工作

    if (ret < 0) // 有错误
        goto out;
    if (!ret) { // 超时返回，无设备就绪
        ret = -ERESTARTNOHAND;
        if (signal_pending(current))
            goto out;
        ret = 0;
    }

    // 把结果集,拷贝回用户空间
    if (set_fd_set(n, inp, fds.res_in) ||
        set_fd_set(n, outp, fds.res_out) ||
        set_fd_set(n, exp, fds.res_ex))
        ret = -EFAULT;

out:
    if (bits != stack_fds)
        kfree(bits); // 如果有申请空间，那么释放fds对应的空间
out_nofds:
    return ret; // 返回就绪的文件描述符的个数
}
```

接下来是主角 `do_select()` 登场：

```c
int do_select(int n, fd_set_bits *fds, struct timespec *end_time)
{
    ktime_t expire, *to = NULL;
    struct poll_wqueues table;
    poll_table *wait;
    int retval, i, timed_out = 0;
    unsigned long slack = 0;

    rcu_read_lock();
    // 根据已经设置好的fd bitmap检查用户打开的fd, 要求对应fd必须打开, 并且返回最大的fd
    retval = max_select_fd(n, fds);
    rcu_read_unlock();

    if (retval < 0)
        return retval;
    n = retval;

    // 一些重要的初始化:
    // poll_wqueues.poll_table.qproc函数指针初始化，该函数是驱动程序中poll函数实现中必须要调用的poll_wait()中使用的函数
    poll_initwait(&table);
    wait = &table.pt;
    if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
        wait = NULL;
        timed_out = 1;  // 如果系统调用带进来的超时时间为0，那么设置timed_out = 1，表示不阻塞，直接返回。
    }

    if (end_time && !timed_out)
        slack = estimate_accuracy(end_time); // 超时时间转换

    retval = 0;
    for (;;) {
        unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;

        inp = fds->in; outp = fds->out; exp = fds->ex;
        rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

        // 所有n个fd的循环
        for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
            unsigned long in, out, ex, all_bits, bit = 1, mask, j;
            unsigned long res_in = 0, res_out = 0, res_ex = 0;
            const struct file_operations *f_op = NULL;
            struct file *file = NULL;

            // 先取出当前循环周期中的32个文件描述符对应的bitmaps
            in = *inp++; out = *outp++; ex = *exp++;
            all_bits = in | out | ex;  // 组合一下，有的fd可能只监测读，或者写，或者err，或者同时都监测
            if (all_bits == 0) { // 这32个描述符没有任何状态被监测，就跳入下一个32个fd的循环中
                i += __NFDBITS; //每32个文件描述符一个循环，正好一个long型数
                continue;
            }

            // 本次32个fd的循环中有需要监测的状态存在
            for (j = 0; j < __NFDBITS; ++j, ++i, bit <<= 1) { // 初始bit = 1
                int fput_needed;
                if (i >= n) // i用来检测是否超出了最大待监测的fd
                    break;
                if (!(bit & all_bits))
                    continue; // bit每次循环后左移一位的作用在这里，用来跳过没有状态监测的fd
                file = fget_light(i, &fput_needed); // 得到file结构指针，并增加引用计数字段f_count
                if (file) { // 如果file存在
                    f_op = file->f_op;
                    mask = DEFAULT_POLLMASK;
                    if (f_op && f_op->poll) {
                        wait_key_set(wait, in, out, bit); // 设置当前fd待监测的事件掩码
                        mask = (*f_op->poll)(file, wait);
                         /* 调用驱动程序中的poll函数，以evdev驱动中的evdev_poll()为例该函数会调用函数poll_wait(file, &evdev->wait, wait)，
                         继续调用__pollwait()回调来分配一个poll_table_entry结构体，该结构体有一个内嵌的等待队列项，
                         设置好wake时调用的回调函数后将其添加到驱动程序中的等待队列头中。*/
                    }
                    fput_light(file, fput_needed);
                    // 释放file结构指针，实际就是减小他的一个引用计数字段f_count。
                    // mask是每一个fop->poll()程序返回的设备状态掩码。
                    if ((mask & POLLIN_SET) && (in & bit)) {
                        res_in |= bit; // fd对应的设备可读
                        retval++;
                        wait = NULL; // 后续有用，避免重复执行__pollwait()
                    }
                    if ((mask & POLLOUT_SET) && (out & bit)) {
                        res_out |= bit; // fd对应的设备可写
                        retval++;
                        wait = NULL;
                    }
                    if ((mask & POLLEX_SET) && (ex & bit)) {
                        res_ex |= bit;
                        retval++;
                        wait = NULL;
                    }
                }
            }
            // 根据poll的结果写回到输出位图里,返回给上级函数
            if (res_in)
                *rinp = res_in;
            if (res_out)
                *routp = res_out;
            if (res_ex)
                *rexp = res_ex;
             // 这里的目的纯粹是为了增加一个抢占点。在支持抢占式调度的内核中（定义了CONFIG_PREEMPT），cond_resched是空操作。
            cond_resched();
        }
        wait = NULL; // 后续有用，避免重复执行__pollwait()
        if (retval || timed_out || signal_pending(current))
            break;
        if (table.error) {
            retval = table.error;
            break;
        }
        // 跳出这个大循环的条件有: 有设备就绪或有异常(retval!=0), 超时(timed_out = 1), 或者有中止信号出现
        /*
         * If this is the first loop and we have a timeout
         * given, then we convert to ktime_t and set the to
         * pointer to the expiry value.
         */
        if (end_time && !to) {
            expire = timespec_to_ktime(*end_time);
            to = &expire;
        }
        // 第一次循环中，当前用户进程从这里进入休眠，上面传下来的超时时间只是为了用在睡眠超时这里而已超时，poll_schedule_timeout()返回0；被唤醒时返回-EINTR
        if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
                       to, slack))
            timed_out = 1; /* 超时后，将其设置成1，方便后面退出循环返回到上层 */
    }
    // 清理各个驱动程序的等待队列头，同时释放掉所有空出来的page页(poll_table_entry)
    poll_freewait(&table);

    return retval; // 返回就绪的文件描述符的个数
}
```

select 使用 bitmap 的方式来传递文件描述符集合，所以就会有最大长度限制，在 Linux 平台下 select 限制文件描述符只能有 `1024` 个，如果需要超过 1024，就需要修改内核代码并重新编译。
select 使用 bitmap 的方式来回传就绪的文件描述符集合，调用者需要循环遍历每一个位判断是否就绪。当文件描述符很多，但是空闲的文件描述符大大多于就绪的文件描述符的时候，效率就很低了，所以一般不建议修改文件描述符的限制数量。
在 `include/linux/posix_types.h` 中可以看到宏定义：

```c
#undef __FD_SETSIZE
#define __FD_SETSIZE    1024
```
