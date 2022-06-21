---
layout:     post
title:      "MongoDB基础知识"
subtitle:   "MongoDB核心原理与实践4"
date:       2022-06-21  23:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

MongoDB核心原理与实践摘录，第3篇MongoDB运维管理和第4篇MongoDB应用实践

### 运维管理
- mongotop用于监控活跃集合消耗在读写操作上的时间。只能连接具体的mongod实例，对于分片集群，只能连接具体分片的某个节点
- mongostat用来监控数据库实例的各种操作，包括mongod和mongos，具体操作包括crud等每秒执行次数、内存使用情况、等待队列长度、客户端数量、网络情况
- db.stat()用于显示数据库信息，主要是文档、索引的数量和空间等
- db.serverStatus()某个mongod或者mongos实例的统计信息，例如连接数、内存、cache和锁相关信息

### 应用实践
- 过期实践索引字段的expireAfterSeconds设置为0，则文档会在字段指定的时间过期
- mongostat监控不了主节点的索引过期删除操作，因为该操作只在主节点进行，直接调用内部接口，不是实际的删除操作，而从节点从oplog得到的是删除操作
- mongoDB默认线程模型是一个连接创建一个线程，如果连接数量不稳定，则高峰时需要创建大量线程，也需要后续的销毁操作。而adaptive线程模型是动态的线程池模型
- cachesize不能调得太大，否则容易出现脏数据积压，刷盘时写入压力过大
- 增加evict线程数量，调小evict线程工作阈值，可以平缓写入压力
- 写入较多时，可以调小checkpoint，以此减少checkpoint间脏数据，缓解IO压力
- mongodb3.6后readconcern默认配置majority，禁用readconcern可以有效减少读取耗时，也能减少节点内存占用，因为readconcern需要节点维护多个snapshot
- 创建索引时会额外申请一定内存，默认最多500M
