---
layout:     post
title:      "Data System Advance"
subtitle:   "Designing Data-Intensive Application 02"
date:       2020-06-09 23:45:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - DataBase
---

数据密集型应用系统设计摘要，第二部分

### 数据复制
- 链式复制，节点组成一个链，只在头部节点写入，写入确认和读取在尾部节点
- 主从复制实现方法：基于语句实现，对调用了非确定性函数的语句、使用了自增列的语句和有副作用（触发器/存储过程等等）的语句不太适用；基于预写日志，和存储引擎紧密耦合，难以滚动升级；基于行的逻辑日志复制；基于触发器的复制
- 单调读一致性比强一致性弱，比最终一致性强。它保证某个用户在不同节点多次读取同一个数据，不会出现回滚现象。最简单的实现方法是确保每个用户总是读取固定的同一副本（不同用户可以从不同副本读取），例如基于用户ID哈希
- 前缀一致读，是指多个写请求被读取时，读取这些内容也会按照写入的顺序进行。分布式数据库不同分区独立运行，没有全局写入顺序，所以会导致用户看到没有顺序的部分新值和旧值。简单处理方法是有因果顺序关系的写入都交给同一个分区
- 多主架构主要用于多个数据中心，每个数据中心有一个主节点。常见拓扑结构有环形、星形、全连接。全连接的复制最快，但容易出现顺序问题，违反前缀一致性

##### 无主节点系统
- 客户端需要将写请求发送到多个节点，根据版本号来确定哪个是新值。读请求则需要quorum
- 失效节点重新上线，可以通过客户端判定并更新，也可以通过后台进程比较节点间的差异来更新
- 大规模集群可以增加一些额外备用节点，如果写入失败，会将写请求发送给这些备用节点，再次判断写入是否成功时把备用节点也考虑进去，等网络恢复时备用节点会将数据传回失效节点。这种方法叫sloppy quorum，可以提高写入可用性


### 数据分区
- 分区容易出现不均匀，某些节点比其它分区节点承担更多的数据量或者查询负载，这被称为倾斜
- Cassandra在范围分区和哈希分区之间折中，主键可以由多个列组成，可以部分列用于哈希分区，其它的用范围分区，这样如果哈希部分确定了，就可以对其它列执行高效的区间查询
- 分区二级索引可以实现成各个分区自行维护局部索引，这样请求需要发送到所有分区；也可以建立全局索引，不过全局索引也进行分区，每个分区存放一部分
- 动态再平衡策略：固定数量，创建远超实际节点数的分区数，为每个节点分配多个分区；动态分区，根据分区数据量等参数动态拆分合并分区；按节点比例分区，每个节点具有固定数量分区，增加节点数，分区数也会同步增加


### 事务
- 读提交，防止脏读脏写，只能看到已成功提交的数据，只会覆盖已成功提交的数据。一般用行级锁防止脏写
- 快照级别隔离与可重复读，用锁实现防止脏写，用MVCC实现读取，和RC不一样的是，RC只需要保存两个版本，snapshot和RR需要多个版本
- 串行化，防止所有可能的竞争条件，结果和串行执行结果一致。一般直接用单线程执行所有事务
- 两阶段加锁，2PL，多个事务可以同时读取同一对象，但只要出现写操作，就必须加锁来独占对象
- 谓词锁，不属于某个特定对象，而是作用于满足提交的所有对象，实现中一般会扩大条件，实现成索引区间锁，例如InnoDB的范围锁
- 可串行化快照隔离，可能发生冲突时继续执行，提交时数据库再去检查是否发生冲突


### 一致性与共识
- 线性化基本思想是让一个系统看起来只有一个数据副本，而且所有操作都是原子的。成功提交的写请求，能马上被其它所有客户端看到
- 可线性化(Linearizability)是读写单个值的最新保证；可串行化(Serializability)是事务的隔离属性，确保结果与串行执行结果一致。注意，可串行化的快照隔离不是线性化的
- 主从复制部分支持可线性化，例如主节点异常下线或者从节点以为自己是主节点，会违法可线性化；共识算法支持可线性化；多主复制不可线性化；无主复制，特殊情况下不可线性化
- CAP，一致性/可用性/分区容错性。实际上分区是一种故障，无法避免，所以可选的是可用性或者一致性（线性化）
- 线性化是全序，因果一致只要求偏序（部分有序）
- 2PC，两阶段提交，提交事务分为准备和提交两阶段，由协调者负责，无法处理协调者崩溃的情况
- 服务发现不需要共识，读取到旧值不一定有问题，可以加入一些只读的缓存副本，不参与投票，来简单提高可用性
