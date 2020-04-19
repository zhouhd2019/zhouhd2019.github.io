---
layout:     post
title:      "Deep"
subtitle:   "kafkaDeepDive03"
date:       2020-04-19 20:30:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - kafka
    - mq
---

深入理解Kafka：核心设计与实践原理，6-8章摘要，后续章节不再摘要

### 深入服务端
- 协议请求头4个字段，api_key请求名/api_version版本/correlation_id请求ID/client_id，其中correlation_id用来标记本次请求，服务器返回的对应响应也会返回这个correlation_id，这样客户端就知道对应请求
- Kafka集群中有一个broker会被选举为控制器，负责管理整个集群中所有分区和副本的状态。控制器需要监听分区、主题、broker相关的变化，更新集群元信息
- 非控制器broker基本只会监听ZooKeeper的/Controller节点，其它节点主要由控制器来监听和管理

### 深入客户端
- Kafka提供的消费者分区分配策略有三种，RangeAssignor/RoundRobinAssignor/StickyAssignor。前两者都不会考虑多个主题的均衡，只会考虑自身主题。StickyAssignor会尽量保持均匀，而且会尽量保持分区重分配前后的分配相同
- 消费者之间的分区分配需要管理和协同。Kafka将全部消费组分成多个子集，每个消费组的子集在服务端对应一个GroupCoordinator对其进行管理，消费者客户端中的ConsumerCoordinator负责和GroupCoordinator交互，两者之间最重要的工作就是进行消费者再均衡
- 当有消费者加入消费组，会有以下几个阶段：FIND_COORDINATOR，新消费者确定所属消费组对应的GroupCoordinator，找到它所在broker，并和broker建立连接；JOIN_GROUP，加入消费组，选举消费组leader和分区分配策略，前者基本是随机，后者投票制；SYNC_GROUP，同步分配方案；HEARTBEAT，开始工作，保持心跳
- enable.idempotence设置为true，可以开启幂等功能。部分参数最好使用默认，retries默认Integer.MAX_VALUE，max.in.flight.requests.per.connection不能大于5，acks必须是-1或者all。Kafka引入producer id和sequence number，来实现幂等。broker会为每个<PID, 分区>维护一个序列号SN_OLD，下一个接收到的消息必须是SN_OLD+1
- 事务可以保证对多个分区写入操作的原子性，应用程序必须提供唯一的transactionalId
- 幂等和事务主要保证的是生产端的，消费端不可能保证，例如事务写入了多个分区，这些分区属于不同消费者，这样消费者就不可能保证事务原子性
- 消费端有参数isolation.level，默认read_uncommitted，如果改成read_committed，消费者将不能读到未提交的事务修改的值

### 可靠性探究
- 当ISR集合中的一个follower副本落后leader的时间超过replica.lag.time.max.ms，会被移到OSR。不用消息数量作为标准，是因为应用场景的不同，消息数量和生产速度会有很大的差异
- leader epoch代表leader的任期，每变更一次就加一。用来保证broker重启后消息不会发生丢失和不一致。每个副本会记录epoch对应的LogStartOffset。broker发生重启，不会马上用HW来截断日志，而是向leader请求自己最后消息epoch对应的LEO，来确定需不需要截断
