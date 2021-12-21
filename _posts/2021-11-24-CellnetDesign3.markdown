---
layout:     post
title:      "cellnet分析3----Session&&SessionMgr"
subtitle:   "golang网络编程示例"
date:       2021-11-24  14:52:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - cellnet
    - golang
    - Server
---

- <https://github.com/davyxu/cellnet>
- 以cellnet为例，讲一下如何用golang实现一个服务器框架
- 本篇主要关注Session和SessionManager，同样是以tcp为例。最后看一下官方自带的聊天室例子


### SessionManager
- 每个peer基本都会带一个SessionManager，用于管理每个连接(会话)。例如tcpAcceptor创建时会新建一个CoreSessionManager
- CoreSessionManager使用sync.Map保存ID到会话的映射，提供了增加和删除等接口

### Session
- 连接和会话一一对应，对于tcpAcceptor来说，监听到新连接时，就会创建一个tcpSession来管理这个新的链接，tcpSession.Start会将自己加到SessionManager的会话Map里面
- tcpSession.Start还会开启两个协程，recvLoop和sendLoop
- recvLoop循环调用TCPMessageTransmitter.OnRecvMessage来完成读取，sendLoop类似

### Chat Example
- 首先是服务器代码，主要流程是创建一个事件队列，创建tcp.Acceptor并和事件队列联系起来，后续逻辑主要由proc.BindProcessorHandler指定的回调函数进行处理

```golang
	// 创建一个事件处理队列，整个服务器只有这一个队列处理事件，服务器属于单线程服务器
    queue := cellnet.NewEventQueue()
	// 创建一个tcp的侦听器，名称为server，连接地址为127.0.0.1:8801，所有连接将事件投递到queue队列,单线程的处理（收发封包过程是多线程）
	p := peer.NewGenericPeer("tcp.Acceptor", "server", "127.0.0.1:18801", queue)
	// 设定封包收发处理的模式为tcp的ltv(Length-Type-Value), Length为封包大小，Type为消息ID，Value为消息内容
	// 每一个连接收到的所有消息事件(cellnet.Event)都被派发到用户回调, 用户使用switch判断消息类型，并做出不同的处理
	proc.BindProcessorHandler(p, "tcp.ltv", func(ev cellnet.Event) {
		switch msg := ev.Message().(type) {
		// 有新的连接
		case *cellnet.SessionAccepted:
			log.Debugln("server accepted")
		// 有连接断开
		case *cellnet.SessionClosed:
			log.Debugln("session closed: ", ev.Session().ID())
		// 收到某个连接的ChatREQ消息
		case *proto.ChatREQ:
			// 准备回应的消息
			ack := proto.ChatACK{
				Content: msg.Content,       // 聊天内容
				Id:      ev.Session().ID(), // 使用会话ID作为发送内容的ID
			}
			// 在Peer上查询SessionAccessor接口，并遍历Peer上的所有连接，并发送回应消息（即广播消息）
			p.(cellnet.SessionAccessor).VisitSession(func(ses cellnet.Session) bool {
				ses.Send(&ack)
				return true
			})
		}
	})
	// 开始侦听
	p.Start()
	// 事件队列开始循环
	queue.StartLoop()
	// 阻塞等待事件队列结束退出( 在另外的goroutine调用queue.StopLoop() )
	queue.Wait()
```

- 客户端的代码和服务器的类似，不过要从命令行获取输入，然后发出去，收到聊天消息要打印出来

```golang
	// 创建一个事件处理队列，整个客户端只有这一个队列处理事件，客户端属于单线程模型
	queue := cellnet.NewEventQueue()
	// 创建一个tcp的连接器，名称为client，连接地址为127.0.0.1:8801，将事件投递到queue队列,单线程的处理（收发封包过程是多线程）
	p := peer.NewGenericPeer("tcp.Connector", "client", "127.0.0.1:18801", queue)
	// 设定封包收发处理的模式为tcp的ltv(Length-Type-Value), Length为封包大小，Type为消息ID，Value为消息内容
	// 并使用switch处理收到的消息
	proc.BindProcessorHandler(p, "tcp.ltv", func(ev cellnet.Event) {
		switch msg := ev.Message().(type) {
		case *cellnet.SessionConnected:
			log.Debugln("client connected")
		case *cellnet.SessionClosed:
			log.Debugln("client error")
		case *proto.ChatACK:
			log.Infof("sid%d say: %s", msg.Id, msg.Content)
		}
	})
	// 开始发起到服务器的连接
	p.Start()
	// 事件队列开始循环
	queue.StartLoop()
	log.Debugln("Ready to chat!")
	// 阻塞的从命令行获取聊天输入
	ReadConsole(func(str string) {
		p.(interface {
			Session() cellnet.Session
		}).Session().Send(&proto.ChatREQ{
			Content: str,
		})

	})
```
