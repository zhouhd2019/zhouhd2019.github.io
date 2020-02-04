---
layout:     post
title:      "LDK Process Mangement"
subtitle:   "进程管理"
date:       2020-02-04 20:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - Linux
    - Linux Kernel
---

对应LKD3第三章进程管理。

- 进程就是处于执行期的程序，包括可执行程序代码、数据、文件、信号、一个或多个内存映射的内存地址空间及执行线程等等。每个线程，也就是执行线程，都拥有一个独立的程序计数器、进程栈和一组进程寄存器。Linux里的线程只是一种特殊的进程
- 进程退出后被设置为僵死状态，直到它的父进程调用wait()或者waitpid()为止。
- 内核把进程的列表存放在叫做任务队列的双向循环链表中。链表项是类型为task_struct、称为进程描述符的结构，包含一个具体进程的所有信息。
- 进程描述符task_struct可以通过thread_info->task访问到，而thread_info在进程内核栈的顶部。
- task_struct->state表示当前进程的状态，可以是下列5种之一：TASK_RUNNING（可执行、正在执行或者在运行队列中等待执行）、TASK_INTERRUPTIBLE（可中断，进程正在睡眠等待某些条件的达成）、TASK_UNINTERRUPTIBLE（不可中断，正在处理重要事务，不处理信号）、\_\_TASK_TRACED（正被其它进程跟踪）、\_\_TASK_STOPPED（停止）。
- fork()通过拷贝当前进程创建一个子进程，区别仅仅在于PID、PPID、某些资源和统计量。exec()负责读取可执行文件并将其载入地址空间开始运行。
- fork()、vfork()和\_\_clone()库函数根据各自需要的参数标志去调用clone()，然后clone()调用do_fork()来完成大部分创建工作。
- vfork()和fork()基本一致，只是不拷贝父进程的**页表项**，调用后父进程被阻塞直到子进程退出或者执行exec()。
- 线程的创建和进程创建类似，只是调用clone()时传递一些参数标志来指明需要共享的资源，CLONE_VM共享地址空间，CLONE_FS共享文件系统信息，CLONE_FILES共享打开的文件，CLONE_SIGHAND共享信号处理函数及被打断的信号。
- 进程退出都会调用do_exit()，会进行一系列销毁操作，并将task_struct->exit_state设置为EXIT_ZOMBIE，然后调用schedule()切换到新的进程。此时老进程不可运行且处于ZOMBIE退出状态，只剩下内核栈、thread_info和task_struct，这些信息存在的意义就是让父进程知道而已，等到父进程wait以后，这些信息也会被通过release_task()释放。
- 如果父进程比子进程先退出，do_exit()会给当前进程找一个新的父进程，现在所在线程组内的其他进程找，如果没有就只能是init进程