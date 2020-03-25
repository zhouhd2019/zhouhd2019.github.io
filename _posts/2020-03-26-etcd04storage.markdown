---
layout:     post
title:      "storage
subtitle:   "etcd storage"
date:       2020-03-26 00:11:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - etcd
    - raft
---

对应第6章storage。

### etcd v2 storage
- v2完全基于内存，需要持久化时会序列化成json存盘。数据以树的形式保存，支持watch一个子树。限制在于：EventHistory使用环形队列实现，有长度限制；watcher出现阻塞时会被直接删除，客户端需要重新连接。上述两个限制导致客户端不能依赖watcher机制来实现完整数据同步，因为可能丢失。另外，key过期时间每个key单独设置。
- watcher机制实现在watcherHub，包含：watchers映射，节点名映射到watcher实例列表；EventHistory保存最近修改的Event实例。添加watcher时会先到EventHistory查看有没有所需数据
- store是v2存储实现结构体，主要成员有：Root树形结构的根节点；ttlKeyHeap过期时间的最小堆，内含节点到下标的索引用于快速修改

### etcd v3 storage
- v3 store分为两部分：backend默认使用BoltDB；内存索引KeyIndex
- etcd在BoltDB中存储的Key是revision，Value是etcd自定义的键值对组合，键值对的每个版本都会被保存到BoltDB。revision由两部分组成，main revision每次事务递增1，sub revision同一个事务中每次操作都会递增1
- backend.run()会按照指定的时间间隔，定时提交批量读写数据，然后立即开始一个新的批量读写事务
- 由于事务中有bucket到kv对的缓存，以此减少到数据库查询的次数，所以即使是只读事务，也需要锁
- 客户端使用key查询，而BoltDB存储的是版本号到kv对的映射，所以需要内存索引来关联Key到版本信息。实现上使用BTree维护Key到KeyIndex的映射，通过KeyIndex查找对应revision信息，最后去backend获得数据
- keyIndex主要包含：key原始key值；modified最后一次修改的revision；generations每个generation代表当前key一次从创建到删除的历史
- 收到快照会调用store.restore()来恢复数据，先应用到BoltDB，再将每个键值对写入通道，由另一个goroutine读取通道来完成内存索引的恢复
- watchableStore结构体包含synced和unsynced两个watcherGroup实例，synced里面的watcher都已经同步完毕，unsynced里面的由一个专门的goroutine来进行同步。还有一个goroutine用来发送watchableStore.victims中累积的Event事件
- watcherGroup主要有三部分：watchers记录所有watcher实例，keyWatchers记录监听单个key的watcher实例，ranges记录范围监听的watcher
- 读写事务存在更新的话，会在End()触发相应watcher实例
- etcd会用一个后台goroutine来定时检查有没有过期的lease，如果过期数量过多会进行限流，目前检查会对leaseMap进行遍历