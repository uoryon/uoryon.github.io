---
layout: post
title:  "进程和线程"
date:   2017-02-25 08:31:19 +0800
categories: linux
---

# 进程和线程
其实我一直都还没仔细的学习过进程和线程，虽然资料很多，大多数教程都写了很详细的描述，还有一大堆相同点，不同点。可是看了还是云里雾里，理解不深刻，大概知道是什么个东西，但是真要谈起来还是有点问题。
这次就仔细深入的学习一下好了。

进程：跑起来的代码，有自己的内存空间。
进程间通信（IPC），有通过文件，管道，rpc 等的方式，父子进程的话，还可以用环境变量和文件描述符。我平时用得较少，一般都是 rpc，比较简单。就不深入讨论了。

线程：由一个程序开启。线程间共享内存，fd。交互机制比进程简单许多。而且线程机制可以理解为一个程序同时做多件事情，当计算机只有一个 cpu 核心的时候，可以有同时运行的幻觉，当有多个 cpu 核心的时候，就是真正的同时了。

简单描述了一下子。我们来看 linux 上的实现吧 arch/x86。
创建一个新进程，我们会调用 fork（clone等，这里用fork举例），
```
int sys_fork(struct pt_reg *regs) {
  return do_fork(SIGCHLD, regs->sp, regs, 0, NULL, NULL);
}
```
`struct pt_reg`保存着寄存器的信息，`sp`就是`stack register`了。
```
long do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        struct pt_regs *regs,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr)
{
  struct task_struct *p;
  ...
  p = copy_process(clone_flags, stack_start, regs, stack_size,
    child_tidptr, NULL, trace); //It copies the registers, and all the appropriate parts of the process environment (as per the clone flags)
  ...
  wake_up_new_task(p); // wake up a newly created task for the first time
  ...
}
```
大概过程既是这样，首先声明 一个`struct task_struct`，`copy_process` 将父进程的内存数据拷贝到子进程里去（会有 COW）。最后调用 `wake_up_new_task` 来唤醒交给 linux 调度器，视为首跑，运行程序。

线程，看看 `kernel_thread`的创建过程。
```
static void create_kthread(struct kthread_create_info *create)
{
  int pid;
  pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
}

/*
 * Create a kernel thread
 */
int kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
  struct pt_regs regs;

  memset(&regs, 0, sizeof(regs));

  regs.si = (unsigned long) fn;
  regs.di = (unsigned long) arg;

#ifdef CONFIG_X86_32
  regs.ds = __USER_DS;
  regs.es = __USER_DS;
  regs.fs = __KERNEL_PERCPU;
  regs.gs = __KERNEL_STACK_CANARY;
#else
  regs.ss = __KERNEL_DS;
#endif

  regs.orig_ax = -1;
  regs.ip = (unsigned long) kernel_thread_helper;
  regs.cs = __KERNEL_CS | get_kernel_rpl();
  regs.flags = X86_EFLAGS_IF | 0x2;

	/* Ok, create the new process.. */
  return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
}

```
一开始初始化一些寄存器信息，然后巴拉巴拉。最后的调用竟然是非常熟悉的函数，`do_fork`。看到 `CLONE_VM`
```
              If  CLONE_VM is set, the calling process and the child process run in the same memory space.  In particular, memory writes performed by the calling process or by
              the child process are also visible in the other process.  Moreover, any memory mapping or unmapping performed with mmap(2) or munmap(2) by the child  or  calling
              process also affects the other process.

              If  CLONE_VM is not set, the child process runs in a separate copy of the memory space of the calling process at the time of clone().  Memory writes or file map-
              pings/unmappings performed by one of the processes do not affect the other, as with fork(2).

```
这下清晰了。通过这样的方式，使得创建的线程可以访问同样的内存空间。那么其实，linux 对于线程和进程的处理都是一样的，每个进程每个线程都有一个 `struct task_struct`。那么都会有自己的 `pid`，进程和线程唯一的区别就是是否共享内存空间，对于 linux 底层调度器来说，是一视同仁的。

继续看一段经典的 `apue`代码。`pthread`稍微看了看，应该是对 `kernel_thread`的封装。不影响讨论。
```
#include "apue.h"
#include <pthread.h>

pthread_t ntid;

void
printids(const char *s)
{
        pid_t           pid;
        pthread_t       tid;

        pid = getpid();
        tid = pthread_self();
        printf("%s pid %u tid %u (0x%x)\n", s, (unsigned int)pid,
          (unsigned int)tid, (unsigned int)tid);
}

void *
thr_fn(void *arg)
{
        printids("new thread: ");
        return((void *)0);
}

int
main(void)
{
        int             err;

        err = pthread_create(&ntid, NULL, thr_fn, NULL);
        if (err != 0)
                err_quit("can't create thread: %s\n", strerror(err));
        printids("main thread:");
        sleep(1);
        exit(0);
}
```

输出是
```
[sh@kuki]~/Code/apue/c11% ./a.out
main thread: pid 2881 tid 3248428864 (0xc19f1740)
new thread:  pid 2881 tid 3240081152 (0xc11fb700)
```
这就奇怪了。不是说 pid 应该是不一样的吗？？
下面这个哥们说得比我好，可以一看。
[c - why use thread in linux print same pid? - Stack Overflow](http://stackoverflow.com/questions/18018419/why-use-thread-in-linux-print-same-pid)

恍然大悟，我们来验证一下，sleep多加一些。再 run 一次，去系统上看一看。
```
main thread: pid 3174 tid 913356608 (0x3670b740)
new thread:  pid 3174 tid 905008896 (0x35f15700)
```
用 ps 看一看
```
// ps -T -p 3174
  PID  SPID TTY          TIME CMD
 3174  3174 pts/0    00:00:00 a.out
 3174  3175 pts/0    00:00:00 a.out
```
确实有这个进程。在 SPID 那一列，不同发行版上 ps 实现可能有不同。
`stat` 看一看呢？
```
[root@kuki proc]# stat /proc/3174
  文件："/proc/3174"
  大小：0               块：0          IO 块：1024   目录
设备：3h/3d     Inode：20029       硬链接：9
权限：(0555/dr-xr-xr-x)  Uid：( 1000/   suhai)   Gid：( 1000/   suhai)
环境：unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
最近访问：2017-02-25 08:11:37.916000000 +0800
最近更改：2017-02-25 08:11:37.916000000 +0800
最近改动：2017-02-25 08:11:37.916000000 +0800
创建时间：-
[root@kuki proc]# stat /proc/3175
  文件："/proc/3175"
  大小：0               块：0          IO 块：1024   目录
设备：3h/3d     Inode：20050       硬链接：9
权限：(0555/dr-xr-xr-x)  Uid：( 1000/   suhai)   Gid：( 1000/   suhai)
环境：unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
最近访问：2017-02-25 08:11:57.160000000 +0800
最近更改：2017-02-25 08:11:57.160000000 +0800
最近改动：2017-02-25 08:11:57.160000000 +0800
创建时间：-

```
使用 kill 3109，可以把整个程序杀掉。

扩展阅读：
[multithreading - Difference between user-level and kernel-supported threads? - Stack Overflow](http://stackoverflow.com/a/15984127/2016779)
