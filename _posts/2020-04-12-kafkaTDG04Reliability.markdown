---
layout:     post
title:      "Reliability"
subtitle:   "kafkaTDG04"
date:       2020-04-12 22:59:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - kafka
    - mq
---

kafka The Definitive Guide(权威指南)摘要，对应第6章。后面章节不再摘要

### 复制
- 跟随者副本需要满足以下条件才会被认为是同步的：近期向zookeeper发送过心跳；近期从首领获取过最新消息
- unclean.leader.election.enable，是否允许非同步副本称为首领
- min.insync.replicas，最少同步副本的数量达到这个参数，写入才可能成功

### 可靠生产者
- acks=all，配合min.insync.replicas，才能很好保证消息成功记录
- 生产者遇到可重试错误（如首领不可用）时，应该保持重试，但kafka(0.10.0)无法保证消息只保存依次，只能应用程序自己加入唯一标识符，或者做到消息幂等。对于不可重试错误（如单条消息太大），也要进行适当处理

### 可靠消费者
- kafka一般能够保证消息至少被消费一次，因为消费过的消息需要提交偏移量
- 减少重复消费，可以手动提交偏移量，处理再均衡的情况，自行记录处理过的消息，必要时自行保存出错消息并进行重试
  