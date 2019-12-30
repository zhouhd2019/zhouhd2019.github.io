---
layout:     post
title:      "Redis11 framework"
subtitle:   "framework"
date:       2019-12-30 23:52:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - DataBase
    - Redis
---

本篇浏览一下redis的基本框架，主要是网络部分，这部分注释非常详细。先看看客户端对象，就是每个链接（包括用户和slave）在redis中是怎样表示的，略去不太重要或者目前没有提到的字段。应该是最后一部分了，像主从、sentinel和cluster这些暂时不会去看了。

返回给客户端的缓冲区有两个，buf和reply，前者是固定大小的数组，后者是一个链表。要返回的客户端的数据会先尝试放在buf，放不下了或者reply已经有数据的话，就会放在reply。所以真正发送数据给客户端的时候，要先发buf里面的，再发reply。

```c++
/* With multiplexing we need to take per-client state.
 * Clients are taken in a linked list. */
typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    int fd;                 /* Client socket. */
    redisDb *db;            /* Pointer to currently SELECTed DB. */

    sds querybuf;           /* Buffer we use to accumulate client queries. */
    size_t qb_pos;          /* The position we have read in querybuf. */

    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    struct redisCommand *cmd, *lastcmd;  /* Last command executed. */

    int reqtype;            /* Request protocol type: PROTO_REQ_* */
    int multibulklen;       /* Number of multi bulk arguments left to read. */
    long bulklen;           /* Length of bulk argument in multi bulk request. */

    list *reply;            /* List of reply objects to send to the client. */
    unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. */

    int flags;              /* Client flags: CLIENT_* macros. */

    multiState mstate;      /* MULTI/EXEC state */
    int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
    blockingState bpop;     /* blocking state */
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    listNode *client_list_node; /* list node in client list */

    /* Response buffer */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```

同样地，下面列出redisServer结构体部分字段，主要是网络相关的。全部的数量实在太多了，包含AOF/RDB/module/lua/master/slave/sentinel等等。

db指针指向数据库数组，并不止一个redisDb，默认16个，不过这其实是一个没有什么用的特性。commands指向包含redis支持的所有命令的字典，redis初始化时会将支持的命令放进commands字典，用于快速找到需要执行的命令，后面的各个命令则是常用命令，直接拿出来特别对待。el是事件循环结构体。bindaddr是绑定的所有IP地址，默认最多16个。clients是当前连接到服务器的所有客户端。

```c++
struct redisServer {
    /* General */
    redisDb *db;

    dict *commands;             /* Command table */
    /* Fast pointers to often looked up command */
    struct redisCommand *delCommand, *multiCommand, *lpushCommand,
                        *lpopCommand, *rpopCommand, *zpopminCommand,
                        *zpopmaxCommand, *sremCommand, *execCommand,
                        *expireCommand, *pexpireCommand, *xclaimCommand,
                        *xgroupCommand;

    aeEventLoop *el;

    /* Networking */
    char *bindaddr[CONFIG_BINDADDR_MAX]; /* Addresses we should bind to */
    int bindaddr_count;         /* Number of addresses in server.bindaddr[] */

    list *clients;              /* List of active clients */
    list *clients_to_close;     /* Clients to close asynchronously */
    list *clients_pending_write; /* There is to write or install handler. */
    list *slaves, *monitors;    /* List of slaves and MONITORs */
};
```

下面从main函数开始，看一下redis初始化到进入事件循环等待的过程。后续以删除指令为例子，看一下指令执行的过程。

main函数在server.c，主要有下面各个步骤：
- initServerConfig，初始化了全局的redisServer结构体
- moduleInitModulesSystem初始化模块系统
- 如果是sentinel_mode，执行initSentinelConfig和initSentinel
- 解析参数
- initServer()
- background，执行daemonize(fork+setsid+/dev/null)，创建pid文件（包含当前实例pid的文本文件，如果配置指定要创建，则非background也会创建）
- 非sentinel模式的初始化，例如载入磁盘数据和module；如果是sentinel，就执行sentinel初始化
- aeSetBeforeSleepProc(server.el,beforeSleep)
- aeSetAfterSleepProc(server.el,afterSleep)
- aeMain(server.el)
- aeDeleteEventLoop(server.el)
- return 0

重点在initServer和各个ae开头的函数中。

initServer之前会将redisServer对象属性初始化为0，initServer里面其中一部分工作就是创建各种所需对象，例如server.db就是redisDb数组，server.clients会被初始化为链表。网络部分在initServer初始化，listenToPort。有个比较特别的server.unixsocket，这是unix下为本机通信做的IPC，会直接用socket快一些，知道一下就好。initServer有一个重要步骤，就是初始化server.el，调用aeCreateEventLoop，创建事件循环对象。

```c++
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

注意aeEventLoop.setsize，这是fd数量上限，它的值是server.maxclients+CONFIG_FDSET_INCR，后者默认是128。aeCreateEventLoop重要一个步骤就是调用aeApiCreate，这是一个有多个定义的函数，根据使用不同的IO复用函数有不同的实现，主要是初始化各自不同的状态aeApiState，例如在epoll里面会调用epoll_create。

initServer注册了最基本的定时器函数，负责定时调用serverCron执行逻辑，第一次是下一毫秒马上执行，后面serverCron返回1000/server.hz，表示下次执行在1000/server.hz毫秒以后。serverCron里面还会对逻辑进行分块，每块都有各自的执行间隔，常见的有100和5000毫秒，像主动逐步rehash也是在serverCron里面执行。具体逻辑就不展开了。

各个tcp连接等等文件描述符的初始化也在initServer执行，调用aeCreateFileEvent创建fd。

```c++
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

主循环是aeMain，是常见的事件循环模式，先执行beforesleep，然后aeProcessEvents开始等待事件和处理事件。处理流程也很常见，执行aeApiPoll进入等待，超时时间设置为下个tick的时间，返回后执行aftersleep，再去处理返回的各个事件，最后处理时间事件。

上述就是redis的主循环，接下来会看看服务器接收客户端连接到处理一条命令的流程。

initServer里面会通过listenToPort，创建监听的socket，fd保存到server.ipfd，接下来马上会通过aeCreateFileEvent，创建对应的事件对象。

```c++
    /* Open the TCP listening socket for the user commands. */
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
        exit(1);

    //省略无关代码

    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```

这样在aeProcessEvents里面就可以得到客户端连接消息，调用accetpTcpHandler。以epoll为例。

```c++
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}

/* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
        numevents = aeApiPoll(eventLoop, tvp);

        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }

            // 省略，和上面类似调用回调函数
            processed++;
        }

```

accetpTcpHandler会间接调用accept接受客户端连接，然后调用createClient创建客户端对象，就是上文提到的client结构体。createClient的逻辑也很简单，一部分是socket相关，设置nonblock/nodelay/keepalive，并将fd加到事件监听；另一部分逻辑是初始化client各个字段。值得注意的是，一开始只需要监听可读事件，回调函数为readQueryFromClient。下面就是真正的消息处理流程了，客户端发送一条指令，服务器会调用readQueryFromClient。

readQueryFromClient先调用read读取数据，然后调用processInputBufferAndReplicate处理数据。如果不是master，简单调用processInputBuffer即可；否则master还要记录下processInputBuffer前后处理了多少数据，然后将已处理部分复制给slave。processInputBuffer将命令分为两种，inline和multibulk，前者是telnet直接交互的方式，空格分隔参数，后者是常见客户端的通信协议方式。简单来说就是两种不同方式来解析数据，调用processCommand来处理解析后的命令。忽略所有错误情况和主从的情况，简单来说，lookupCommand查找需要执行的命令，根据是不是multi模式，执行queueMultiCommand或者call。前者只是将命令保存在multiqueue即可，后者是真正的调用，不过实际关键的也只是c->cmd->proc(c)一行，因为命令已经解析好了，直接执行proc就可以了。