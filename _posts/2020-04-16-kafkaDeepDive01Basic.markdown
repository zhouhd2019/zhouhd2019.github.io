---
layout:     post
title:      "Basic"
subtitle:   "kafkaDeepDive01"
date:       2020-04-19 00:49:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - kafka
    - mq
---

深入理解Kafka：核心设计与实践原理，前三章摘要。由于已经有TDG的摘要，所以一些基础不再列出。

### 初识Kafka
- leader副本负责维护和跟踪ISR集合中所有follower的滞后情况。当follower副本落后太多或失效时，leader副本会把它从ISR剔除。如果OSR中有follower副本追上leader副本，那么leader副本会把它从OSR集合移到ISR

### 生产者
- 消息send()到broker之前会经过拦截器、序列化器和分区器
- 如果key不为null，默认分区器会对key进行哈希，根据哈希值计算分区号，所有分区都可能被选中；如果key为null且有可用分区时，那么只会选中可用分区
- 拦截器有三个回调，第一个onSend()用于消息序列化和计算分区前做一些工作，例如过滤、修改、统计消息；第二个onAcknowledgement()会在消息被应答前或者消息发送失败时响应，优先于用户设定的callback；第三个close()可以用来进行关闭拦截器时的清理
- 拦截器可以有多个，形成拦截链，如果前面失败了，后面会从上一个成功的接着执行
- 生产者由两个线程协调进行，分别为主线程和Sender线程。主线程创建消息，然后通过可能的拦截器、序列化器和分区器，之后缓存到消息收集器RecordAccumulator。消息收集器用来缓存消息，便于Sender线程批量发送。消息收集器为每个分区都维护了一个双端队列，队列内放的是ProducerBatch，每个ProducerBatch可以包含多个ProducerRecord
- Sender从RecordAccumulator获取被缓存的ProducerBatch，将ProducerBatch转换成Request，并确定消息发向的broker。发送前还会将Request保存到InFlightRequests，表示消息已发出但还没收到回应。没有收到回应的消息太多，就会停止发送消息
- InFlightRequests中记录最少的队列对应的broker，会被认为是负载最小的leastLoadedNode。元数据请求等消息会发给leaseLoadedNode
- 需要保证消息顺序时，max.in.flight.requests.per.connection要设置成1

### 消费者
- subscribe()可以订阅指定的一个或多个分区，也可以通过正则表达式订阅
- KafkaProducer是线程安全的，但KafkaConsumer不是线程安全的，需要每个线程有自己的KafkaConsumer，或者只有一个KafkaConsumer来读消息，消息的具体处理交给线程池，但这样容易有偏移提交的问题
