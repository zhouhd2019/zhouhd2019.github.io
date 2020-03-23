---
layout:     post
title:      "network WAL&SnapShot"
subtitle:   "etcd网络 WAL日志与快照"
date:       2020-03-23 23:59:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - etcd
    - raft
---

对应第4章网络层，第5章WAL日志与快照。

### etcd-rafthttp
- 每个节点之间会建立两种消息通道：Stream维护HTTP长连接，主要负责传输数据量较小、发送频繁的数据；Pipiline通道负责传输数据量大频率低的数据，例如MsgSnap，发送完成就断开
- 每个stream通道有两个goroutine，一个用于建立HTTP连接，读取数据并将数据反序列化为Message，传递给etcd-raft；另一个goroutine会读取etcd-raft返回的消息，序列化并写入stream
- rafthttp.Transporter接口定义了发送消息和发送快照函数，创建Handler处理HTTP请求，以及几个增删改Peer的函数。rafthttp.Transport实现了rafthttp.Transporter，重要成员包括：Raft指向etcd-raft实例，streamRt和pipilineRt用于处理对应请求
- 每个peer包含三类读写chan：pipiline的msgc用于发送snapshot，streamWriter的msgc用于发送消息，streamReader的recvc和proc接收普通消息和日志消息

### WAL日志
- 回顾一下处理一条Entry的大致流程：etcd-raft模块将刚收到的Entry保存到raftLog.unstable；etcd-raft模块将Entry封装到Ready实例，返回给上层；如果是Leader，会马上发送Entry给其它节点；上层将Ready中的Entry记录到WAL，然后进行持久化操作，最后通知etcd-raft模块进行处理；etcd-raft将Entry从unstalbe移动到storage；当Entry被复制到半数以上节点，会被认为已提交，被封装到Ready；上层会将Ready中的该Entry应用到状态机
- WAL日志有4种：metadata，entry，state(当前集群信息HardState)，crc，snapshot(相关信息，没有真正数据)
- WAL结构体的filePipiline会负责预先创建并分配空间给临时文件，当需要写新日志时直接重命名就可以
- 日志文件名x-y前面是递增序列后面是首条日志索引值
- 追加日志WAL.Save()先写入entry，然后写入state，最后sync()刷新到磁盘，刷新时先尝试调用encoder.flush()，不行再用系统提供的接口。encoder每写满一页就会刷新到磁盘。日志文件达到配置大小就会使用临时文件，将后续日志写到新文件，保证每个文件不会太大。

### SnapShotter
- etcd定期创建快照，以此减少恢复节点时的时间。恢复时先读取快照，再根据快照最后的日志读取WAL日志