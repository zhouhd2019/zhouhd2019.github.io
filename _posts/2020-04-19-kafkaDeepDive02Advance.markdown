---
layout:     post
title:      "Advance"
subtitle:   "kafkaDeepDive02"
date:       2020-04-19 15:16:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - kafka
    - mq
---

深入理解Kafka：核心设计与实践原理，四五章摘要

### 主题与分区
- Kafka会在log.dir或log.dirs配置的目录下创建相应的主题分区目录，命名方式是<topic>-<partition>
- 创建主题时会在ZooKeeper的/brokers/topics/目录下创建一个同名的实节点，记录该主题的分区副本分配方案
- 为了减少负载失衡的情况，Kafka引入优先副本的概念，即AR集合列表中的第一个副本。如果auto.leader.reblalance.enable打开(默认打开)，Kafka会定期检查非优先副本成为了leader的百分比，超出leader.imbalance.per.broker.pencentage的话，会执行选举，尽量让更多的优先副本称为leader副本。生产环境中不应打开自动再均衡，可以手动执行均衡

### 日志存储
- Log在物理上以文件夹的形式存储，LogSegment对应一个日志文件和两个索引文件(偏移量索引文件，时间戳索引文件)，以及可能的其它文件(例如事务索引文件)
- 向Log追加消息是顺序写入的，只有最后一个LogSegment才能写入。每个LogSegment都有一个baseOffset，表示第一条消息的offset，也作为文件名来标记LogSegment
- 偏移量索引文件用来建立消息偏移量到物理地址之间的映射关系，时间戳索引文件则根据指定的时间戳来查找对应偏移量信息。Kafka中的索引文件以稀疏索引的方式构造，不是每个消息都有对应索引，而是一定量消息的写入，才会增加索引项。索引项的偏移量是相对于所在文件baseOffset的，来减少所需存储空间
- Kafka的每个日志使用了ConcurrentSkipListMap跳表来保存各个LogSegment，每个LogSegment的baseOffset作为key，以此来快速定位到指定偏移量消息所在的LogSegment。找到所在LogSegment，再通过二分查找来找出消息之前的第一个索引项，获得文件偏移，从文件偏移开始按顺序查找。时间戳索引文件没有跳表来加速定位LogSegment