---
layout:     post
title:      "Redis09 stream"
subtitle:   "stream"
date:       2019-12-03 20:40:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - DataBase
    - Redis
---

前两节看了下listpack和rax，这两个东西在stream里面都用到了，接下来就是stream消息队列。默认消息ID由当前毫秒时间和序列号组成。stream主要由存放消息和消费组的rax组成。消费组主要存了待确认消息和组内消费者。消费组的主要成员是待确认消息。

```c++
/* Stream item ID: a 128 bit number composed of a milliseconds time and
 * a sequence counter. IDs generated in the same millisecond (or in a past
 * millisecond if the clock jumped backward) will use the millisecond time
 * of the latest generated ID and an incremented sequence. */
typedef struct streamID {
    uint64_t ms;        /* Unix time in milliseconds. */
    uint64_t seq;       /* Sequence number. */
} streamID;

typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. */
    uint64_t length;        /* Number of elements inside this stream. */
    streamID last_id;       /* Zero if there are yet no items. */
    rax *cgroups;           /* Consumer groups dictionary: name -> streamCG */
} stream;

/* Consumer group. */
typedef struct streamCG {
    streamID last_id;       /* Last delivered (not acknowledged) ID for this
                               group. Consumers that will just ask for more
                               messages will served with IDs > than this. */
    rax *pel;               /* Pending entries list. This is a radix tree that
                               has every message delivered to consumers (without
                               the NOACK option) that was yet not acknowledged
                               as processed. The key of the radix tree is the
                               ID as a 64 bit big endian number, while the
                               associated value is a streamNACK structure.*/
    rax *consumers;         /* A radix tree representing the consumers by name
                               and their associated representation in the form
                               of streamConsumer structures. */
} streamCG;

/* A specific consumer in a consumer group.  */
typedef struct streamConsumer {
    mstime_t seen_time;         /* Last time this consumer was active. */
    sds name;                   /* Consumer name. This is how the consumer
                                   will be identified in the consumer group
                                   protocol. Case sensitive. */
    rax *pel;                   /* Consumer specific pending entries list: all
                                   the pending messages delivered to this
                                   consumer not yet acknowledged. Keys are
                                   big endian message IDs, while values are
                                   the same streamNACK structure referenced
                                   in the "pel" of the conumser group structure
                                   itself, so the value is shared. */
} streamConsumer;
```

streamAppendItem向stream插入一条消息。每个listpack都由一条主消息开始，保存了field-value对，如果后续消息的field和master的一样，那么就可以不保存field。注意并不是每条消息都一个listpack，主消息id对应着listpack，而后续其它消息直接插入未满的listpack，直到这个listpack超过配置大小或者entry数量上限，才会新建rax node和listpack。所以每条消息有多个listpack entry，每个listpack会保存多条消息。

streamTrimByLength用来批量删减消息，值得注意的有两个地方，如果当前listpack长度小于等于要删除的长度，会删除整个listpack。另一个地方是函数参数approx，表示要求的长度可以不完全准确，这样就只会整个listpack删除。

streamIteratorRemoveEntry删除单个消息，如果当前listpack只有一个消息，可以直接删除整个，否则只能将这个消息标记为删除，当然长度等等信息都要更新。

streamReplyWithRange回复客户端需要的多条消息，读一个消息，就往回复缓冲写一个消息，默认情况下还要增加或者更新未确认消息对象。如果客户端要求的是已经读取过的待确认消息，就会转而调用streamReplyWithRangeFromConsumerPEL。

xaddCommand就是xadd的实现。首先是一堆的检查和命令行参数解析，然后streamTypeLookupWriteOrCreate找到或创建指定stream，接着streamAppendItem插入消息。剩下就是一些回复客户端之类的行为：1.触发key被修改事件，实际上就是看看key在不在watch列表，在的话就要通知对应客户端；2.notifyKeyspaceEvent，观察者模式的实现，发布key变化的事件；3.指定了最大长度，就检查是否需要删掉老消息；4.如果有在等待这个key的客户端，就将key放入db->ready_keys，后面redis会通知那些正在等待的客户端有新key到来。

xreadCommand处理了xread和xreadgroup，比较特别的地方在于支持阻塞，blockForKeys可以让对应的client对象暂停处理请求，直到处理完当前的xread/xreadgroup。
