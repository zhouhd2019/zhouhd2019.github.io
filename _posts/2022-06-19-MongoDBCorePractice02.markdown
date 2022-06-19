---
layout:     post
title:      "MongoDB基础知识"
subtitle:   "MongoDB核心原理与实践2"
date:       2022-06-19  22:22:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

MongoDB核心原理与实践摘录，第2篇深入理解MongoDB 5/6章

### WiredTiger(WT)存储引擎
- WT一般用B-Tree表示数据，磁盘上的leaf page包含page header、block header和真正的key/value数据
- 内存中的B-Tree结构包含3种类型page，root page/internal page/leaf page
- 内存leaf page维护了多种数据，比较重要的有：
    - WT_ROW数组，保存从磁盘读取的key/value
    - WT_PAGE_MODIFY.WT_UPDATE链表，每次修改都会在这里插入一个元素，如果一个文档被多次修改，则会将修改记录以链表的形式存放
    - WT_PAGE_MODIFY.WT_INSERT_HEAD链表，插入文档会在这里插入记录，以跳表形式存储，实际数据用内含的WT_UPDATE存储
    - WT_PAGE_LOOKASIDE，reconcile时还有读操作访问被修改的数据，则这部分数据会被单独保存到lookaside表(WiredTigerLAS.wt文件)，后续读page时需要同时读lookaside表
- page状态包括：
```C++
#define WT_REF_DISK 0 /* Page is on disk */
#define WT_REF_DELETED 1 /* Page is on disk, but deleted */
#define WT_REF_LIMBO 2 /* Page is in cache without history */
#define WT_REF_LOCKED 3 /* Page locked for exclusive access */
#define WT_REF_LOOKASIDE 4 /* Page is on disk with lookaside */
#define WT_REF_MEM 5 /* Page is in cache and valid */
#define WT_REF_READING 6 /* Page being read */
#define WT_REF_SPLIT 7 /* Parent page split (WT_REF dead) */
```
- 默认60S或者Journal文件大小达到2GB执行一次checkpoint，数据文件关闭时也会执行checkpoint
- Journal是一种WAL事务日志，记录修改的动作：50ms定期落盘，提交写动作时设置j:true强制同步落盘，日志文件达到100M重新开一个也会落盘

### 复制集
- 当节点的votes配置属性值为1且节点状态state正常才可以投票，正常是指PRIAMRY/SECONDARY/STARTUP2正在初始化/RECOVERING/ARBITER/ROLLBACK
- 最多只能有7个投票节点，votes和priority都要配置为0
- 触发复制集选举：新节点添加到复制集；初始化复制集；重新配置复制集；Primary节点发生故障
- 节点基于oplog进行数据同步，local.oplog.rs是一个在local数据库下固定大小的集合
- 一次update涉及多条记录，每条被修改的记录会生成一条日志记录保存到oplog
- 默认w=1，Primary节点写入完成就返回
- 默认j=false，如果true，每个节点journal写入磁盘才返回
- 配置节点可以指定tags标签，客户端构造MongoClient时传入readPreferenceTags可以使得从节点读倾向在tags合适的节点上进行
- maxStalenessSeconds表示Secondary节点最新写操作与Primary节点最新写操作之间允许的最大间隔，可以用来在读取时排除同步得比较慢的Secondary
- readConcern实现基于snapshot，对于majority，当某个snapshot的写入都同步到大部分节点，后续读就可以在该snapshot上进行，主从节点会不停交换写入信息
- readConcern设置为majority可以保证数据不会回滚，只需要到某个节点进行读取，以该节点所知的同步状态为准，不需要到所有节点上读
