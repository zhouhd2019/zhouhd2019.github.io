---
layout:     post
title:      "etcd server&client"
subtitle:   "server&client"
date:       2020-03-26 23:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - etcd
    - raft
---

对应第7章etcd-server，第8章etcd-client

### raftNode结构体
- raftNode结构体主要包括：ticker逻辑时钟；raftStorage，和raftLog.storage字段指向的MemoryStorage为同一实例；etcdserver.Storage，负责持久化，包括WAL和Snapshotter；transport和其它节点交互
- raftNode.start()方法会启动一个goroutine，与底层etcd-raft交互，主要是处理Ready

### EtcdServer
- EtcdServer结构体实现了etcdserver.Server，是etcd最核心的结构，包含了之前提到了绝大部分结构体：etcdserver.raftNode，和底层etcd-raft通信；membership.RaftCluster，记录当前集群节点信息；store，v2版本存储；snapshotter读写快照；be，v3版本的后端存储backend；kv，v3版本的存储
- EtcdServer执行NewServer()方法进行一系列初始化：初始化raftNode；启动Transport实例，注册Handler实例用于集群通信；启动当前节点对外服务，启动多个goroutine执行不同方法，例如处理Ready、定期清理持久化文件，publish()将当前节点信息发送到其它集群，linearizableReadLoop
- v3的应用消息接口除了普通的处理，还支持限流和权限控制

### etcd client
- Client v3定义了6类服务：Auth/Cluster/KV/Lease/Maintenance/Watch，服务器实现就在EtcdServer，会在启动时调用Etcd.server()方法注册各种gRPC服务的注册