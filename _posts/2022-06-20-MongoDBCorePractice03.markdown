---
layout:     post
title:      "MongoDB基础知识"
subtitle:   "MongoDB核心原理与实践3"
date:       2022-06-20  23:06:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

MongoDB核心原理与实践摘录，第2篇深入理解MongoDB 7/8章

### 分片集群
- 分片集群由3部分组成：分片shard，mongos路由和config配置服务器。集群元信息在config上存储，mongos会从config上获取元信息
- 分片集合中的数据按chunk为单位存储，一个chunk默认大小为64MB
- balancer是一个后台进程，运行在config的Primary节点
- chunk迁移时，数据复制是通过文档复制来完成的，索引初始化会在数据复制之前进行
- config的元信息主要存放在几个数据库：changelog变更信息日志；chunks，每个chunk属于哪个集合/位于哪个分片/数据范围等等信息；collections集合信息，例如分片键；databases，数据库信息，例如主分片；setting配置信息，例如是否开启balancer/是否允许自动分割chunk

### GridFS
- GridFS将文件分成小块(255KB)，每个小块作为一条文档存到对应集合
- chunks集合存放文件块，files集合存放文件元数据信息
