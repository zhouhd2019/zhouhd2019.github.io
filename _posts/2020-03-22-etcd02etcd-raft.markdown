---
layout:     post
title:      "etcd-raft"
subtitle:   "Raft协议etcd实现"
date:       2020-03-22 22:28:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - etcd
    - raft
---

对应第3章etcd-raft。

### raft结构体
- checkQuorum机制，如果Leader发现没有收到超过半数的心跳响应，就会主动切换成Follower
- preVote机制，Follower发起选举前先向其它节点发送消息，询问是否愿意参与选举，有超过半数的节点响应选举，才会发起新一轮选举
- raft.step，当前节点收到消息的处理函数，不同状态下会指向不同的函数
- etcd-raft中，使用MemoryStorage来保存当前节点接收到的Entry，主要包含pb.SnapShot/pb.HardState/ents数组；另外用unstable来保存Leader接收到的客户端Entry记录，或者Follower刚接收到的Leader Entry记录/SnapShot。当Entry记录被写入Storage(MemoryStorage)后，unstable就会清除对应Entry记录
- raftLog结构体核心字段有四个：storage，MemoryStorage实例；unstable，unstable实例；committed，已提交Entry最大索引值；applied，已应用最大索引值
- Follower选举计时器超时，先通过tickElection()创建MsgHup消息，并让raft.Step处理，raft.Step会将节点切换成PreCandidate状态，然后创建PreVote消息，将它追加到raft.msgs字段，等待上层将消息发出
- Follower接收Entry有三种状态，SnapShot正在发送快照/Probe等待消息被响应才能继续发送/Replicate正常发送
- MsgCheckQuorum，Leader选举定时器超时就会给自己发消息，去检查节点连接状态，会通过通信记录和日志记录来检查
- 只有Leader能处理MsgProp客户端写入请求，Candidate会忽略，Follower会转发
- MsgReadIndex是只读请求消息。在ReadOnlySafe模式下，如果是Leader来处理，则会广播带已提交位置（即只读消息的索引）的心跳消息，来确认自己是Leader，等当前committed索引大于等于ReaderIndex创建时的committed，就可以给客户端返回结果；如果是Follower来处理，则会先向Leader请求，Leader正常情况下会返回MsgReadIndexResp消息，收到后Follower才能处理客户端请求。在ReadOnlyLeasedBased模式下，Leader会直接返回结果，不确认自己是不是Leader；Follower还是需要先和Leader确认
- Leader收到MsgTransferLeader会进行手动转移Leader，先让目标Follower日志更新到最新，然后向它发送MsgTimeoutNow，让目标Follower选举定时器超时从而发起选举

### node结构体（Node接口实现）
- Node接口在raft结构体基础上做一层封装，对外提供简单API接口，node是Node的实现
- Node需要给上层返回Ready结构体实例，它内嵌了SoftState和HardState，SoftState封装了Leader节点ID和自身角色，HardState封装了当前任期/投票给的节点/raftLog已提交索引。除了两个State，还有：Entries，给上层unstable的记录，让上层将它们保存到Storage；CommittedEntries，已提交待应用的记录；SnapShot，待持久化的快照数据；Messages，待发给其它节点的消息；ReadStates，当前节点等待处理的只读请求
- node结构体接收上层消息的chan主要有propc/recvc/confc/advancec/tickc，给上层发送的主要有confstatec返回ConfState/readyc/status
- node.run()会处理结构体中封装的所有chan，是主要的逻辑函数