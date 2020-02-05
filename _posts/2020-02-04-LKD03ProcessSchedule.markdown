---
layout:     post
title:      "LDK Process Schedule"
subtitle:   "进程调度"
date:       2020-02-04 21:29:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - Linux
    - Linux Kernel
---

对应LKD3第4章进程调度。

- 每个进程的进程描述符task_struct都有一个sched_entity成员变量，用来存放进程调度相关信息。sched_entity->vruntime存放进程的虚拟运行时间，CFS调度算法会选择vruntime最小的进程来作为下一个运行的进程。
- CFS调度器使用红黑树存储可运行进程，键值是vruntime。所以理论上是找树最左边的叶子节点，不过这个节点会缓存在rb_leftmost。
- 进程调度的入口在schedule()，它会调用pick_next_task()，根据优先级从高到低检查每个调度类返回首个可执行进程。有一个优化，如果所有可运行进程的调度类都是CFS，那么直接让CFS决定就可以了。
- 休眠进程会处于TASK_INTERRUPTIBLE或者TASK_UNINTERRUPTIBLE。进入休眠时进程会被从红黑树移到等待队列。唤醒时如果被唤醒的进程优先级比当前运行进程高，会设置need_resched标志，表明需要重新调度。注意每个进程都有一个need_resched，因为访问进程描述符的数值比访问全局变量快。
- 用户抢占发生在系统调用或者中断处理程序返回用户态的时候。
- 每个进程的thread_info有一个preempt_count表示当前有没有持有锁（非抢占区域），如果不为0则不可以进行内核抢占
- 内核抢占会发生在：中断处理程序正在执行，且返回内核空间之前；内核代码再一次可抢占；内核任务直接调用schedule()；内核任务阻塞。