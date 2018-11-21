---
layout: post
title:  "Systemtap 简单聊聊"
date:   2017-05-04 22:46:19 +0800
categories: linux
---
Systemtap 允许开发者写一些简单的脚本来观察活动着的 linux 系统。能够检查一些复杂的系统以及函数问题。
<!--more-->

一般默认是不会装 systemtap 的。可以通过下面这一条来安装
```
yum install systemtap
```
安装好了之后，就可以有一个命令 `stap`。试试下面这个简单的脚本`hello-world.stp`。
```
probe begin
{
  print ("Hello World.")
  exit ()
}
```

在命令行中运行
```
# stap hello-world.stp
Hello World.
```

非常的简单。

`probe` 可以放在下面这些地方：
*begin* The startup of the systemtap session.
*end* The end of the systemtap session.
*kernel.function("sys_open")*The entry to the function named sys_open in the kernel.
*syscall.close.return*	The return from the close system call.
*module("ext3").statement(0xdeadbeef)*The addressed instruction in the ext3 filesystem driver.
*timer.ms(200)*A timer that fires every 200 milliseconds.
*timer.profile*A timer that fires periodically on every CPU.
*perf.hw.cache_misses*A particular number of CPU cache misses have occurred.
*procfs("status").read*A process trying to read a synthetic file.
*process("a.out").statement(“\*@main.c:200”) *Line 200 of the a.out program.

这完全像一个 `debugger`。只要你能确定是哪里都能放一个`probe`。举些例子：
```
probe kernel.function("*@net/socket.c") { }
probe kernel.function("*@net/socket.c").return { }
```

放了`probe`呢。当然是想知道可以输出什么有用的信息了。如下：
*tid()*The id of the current thread.
*pid()*The process (task group) id of the current thread.
*uid()*The id of the current user.
*execname()*The name of the current process.
*cpu()*The current cpu number.
*gettimeofday_s()*Number of seconds since epoch.
*get_cycles()*Snapshot of hardware cycle counter.
*pp()*A string describing the probe point being currently handled.
*ppfunc()*If known, the the function name in which this probe was placed.
*$$vars*If available, a pretty-printed listing of all local variables in scope.
*print_backtrace()*If possible, print a kernel backtrace.
*print_ubacktrace()*If possible, print a user-space backtrace.

systemtap 还支持一些基础的控制语句，例如 if-else-if，比较等。支持一些函数，嵌入 C 的代码。功能非常强大，希望以后可以带来一些实战吧。
