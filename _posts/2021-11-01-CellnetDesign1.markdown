---
layout:     post
title:      "cellnet分析1----Peer"
subtitle:   "golang网络编程示例"
date:       2021-11-01  20:52:00
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
- 本篇将关注Peer相关代码，主要是tcpAcceptor和tcpConnector


### 概述
- cellnet用Peer表示网络连接的一端，Acceptor和Connector是最常用的两种Peer
- 创建使用peer.NewGenericPeer函数，填入peer类型，自定义名字，地址和消息队列
- 调用Start就会开始工作
- 每种Peer单独一个go文件，init函数调用peer.RegisterPeerCreator将本类Peer的创建函数注册到peer模块的全局变量中，后续NewGenericPeer可以通过名字来创建
- tcpAcceptor的创建函数基本就是创建了一个CoreSessionManager来管理会话

### tcpAcceptor
- tcpAcceptor定义如下。
```golang
type tcpAcceptor struct {
	peer.SessionManager			// 会话管理器，后续展开
	peer.CorePeerProperty		// 保存名字、队列和地址
	peer.CoreContextSet			// 相当于一个Peer内的字典，方便逻辑存放需要的东西
	peer.CoreRunningTag			// 运行状态，可以检查当前是不是正在运行，还可以等待结束
	peer.CoreProcBundle			// 收发和处理消息的地方，后续展开
	peer.CoreTCPSocketOption	// TCP的一些选项，例如读写buffer大小、读写超时、nagle开关、最大包字节数
	peer.CoreCaptureIOPanic		// 是否捕获异常
	// 保存侦听器
	listener net.Listener 		// golang的net包提供的监听器
}
```
- 比较重要的是SessionManager、CoreProcBundle，后续展开
- 开始监听函数实现如下。
```golang
func (self *tcpAcceptor) Start() cellnet.Peer {
	// 之前可能启动过，等它结束了再启动。感觉报错更好
	self.WaitStopFinished()
	if self.IsRunning() {
		return self
	}
	ln, err := util.DetectPort(self.Address(), func(a *util.Address, port int) (interface{}, error) {
		return net.Listen("tcp", a.HostPortString(port))
	})
	// 省略错误检查
	self.listener = ln.(net.Listener)
	go self.accept()
	return self
}
```
- 异步接收连接accept实现如下。
```golang
func (self *tcpAcceptor) accept() {
	self.SetRunning(true)
	for {
		conn, err := self.listener.Accept()
		if self.IsStopping() {
			break
		}
		if err == nil {
			// 处理连接进入独立线程, 防止accept无法响应
			go self.onNewSession(conn)
		}else{
			// 如果错误是临时的，等一下
			if nerr, ok := err.(net.Error); ok && nerr.Temporary(){
				time.Sleep(time.Millisecond)
				continue
			}
			break
		}
	}
	self.SetRunning(false)
	self.EndStopping()
}
```
- 收到新连接时accept会启动新协程来处理，就是比较简单地创建一个新的会话，并往消息队列放消息
```golang
func (self *tcpAcceptor) onNewSession(conn net.Conn) {
	self.ApplySocketOption(conn)
	ses := newSession(conn, self, nil)
	ses.Start()
	self.ProcEvent(&cellnet.RecvMsgEvent{
		Ses: ses,
		Msg: &cellnet.SessionAccepted{},
	})
}
```
- 接收新连接时会顺便设置收发缓冲区大小，不过一般不需要设置，系统会自动增减缓冲区大小
- newSession就是创建一个新的session，调用它的Start，后续会展开讲Session和SessionManager

### tcpConnector
- 用于客户端连接服务器的逻辑，定义如下，前面几个和tcpAcceptor是一样的。tcpSession是客户端本身的会话
```golang
type tcpConnector struct {
	peer.SessionManager
	peer.CorePeerProperty
	peer.CoreContextSet
	peer.CoreRunningTag
	peer.CoreProcBundle
	peer.CoreTCPSocketOption
	peer.CoreCaptureIOPanic
	defaultSes *tcpSession
	tryConnTimes int // 尝试连接次数
	sesEndSignal sync.WaitGroup
	reconDur time.Duration
}
```
- tcpConnector主要是实现了连接和重连逻辑
```golang
func (self *tcpConnector) connect(address string) {
	self.SetRunning(true)
	for {
		self.tryConnTimes++
		// 尝试用Socket连接地址
		conn, err := net.Dial("tcp", address)
		self.defaultSes.setConn(conn)
		// 发生错误时退出
		if err != nil {
			// 没重连就退出
			if self.ReconnectDuration() == 0 || self.IsStopping() {
				self.ProcEvent(&cellnet.RecvMsgEvent{
					Ses: self.defaultSes,
					Msg: &cellnet.SessionConnectError{},
				})
				break
			}
			// 有重连就等待
			time.Sleep(self.ReconnectDuration())
			// 继续连接
			continue
		}
		self.sesEndSignal.Add(1)
		self.ApplySocketOption(conn)
		self.defaultSes.Start()
		self.tryConnTimes = 0
		self.ProcEvent(&cellnet.RecvMsgEvent{Ses: self.defaultSes, Msg: &cellnet.SessionConnected{}})
		// 连接成功消息丢给上层，这里进入等待，直到会话断开，需要退出或者重连
		self.sesEndSignal.Wait()
		self.defaultSes.setConn(nil)
		// 没重连就退出/主动退出
		if self.IsStopping() || self.ReconnectDuration() == 0 {
			break
		}
		// 有重连就等待
		time.Sleep(self.ReconnectDuration())
		// 继续连接
		continue
	}
	self.SetRunning(false)
	self.EndStopping()
}
```
