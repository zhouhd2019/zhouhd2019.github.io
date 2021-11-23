---
layout:     post
title:      "cellnet分析2----Processor"
subtitle:   "golang网络编程示例"
date:       2021-11-03  20:52:00
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
- 本篇以tcp.ltv为例看看消息处理的相关代码


### 概述
- cellnet使用Processor处理消息的收发过程。常见的设置消息处理函数代码如下。
```golang
proc.BindProcessorHandler(p, "tcp.ltv", func(ev cellnet.Event) {
	switch msg := ev.Message().(type) {
	// 有新的连接
	case *cellnet.SessionAccepted:
		log.Debugln("server accepted")
	}

})
```
- BindProcessorHandler根据procName(上述代码是tcp.ltv)创建对应的processor，并将peer和消息处理函数传入processor
- processor注册是会传入创建函数，tcp.ltv的注册代码如下，指定了peer(bundle)的三个组成transmitter/hooker/callback
```golang
proc.RegisterProcessor("tcp.ltv", func(bundle proc.ProcessorBundle, userCallback cellnet.EventCallback, args ...interface{}) {

	bundle.SetTransmitter(new(TCPMessageTransmitter))
	bundle.SetHooker(new(MsgHooker))
	bundle.SetCallback(proc.NewQueuedEventCallback(userCallback))

})
```
- 接口ProcessorBundle要求实现SetTransmitter/SetHooker/SetCallback，peer一般用CoreProcBundle，实现就是简单地设置字段
```golang
type CoreProcBundle struct {
	transmit cellnet.MessageTransmitter
	hooker   cellnet.EventHooker
	callback cellnet.EventCallback
}
```
- transmitter/hooker/callback接口如下：
```golang
// 消息收发器
type MessageTransmitter interface {
	// 接收消息
	OnRecvMessage(ses Session) (msg interface{}, err error)
	// 发送消息
	OnSendMessage(ses Session, msg interface{}) error
}
// 处理钩子(参数输入, 返回输出, 不给MessageProccessor处理时，可以将Event设置为nil)
type EventHooker interface {
	// 入站(接收)的事件处理
	OnInboundEvent(input Event) (output Event)
	// 出站(发送)的事件处理
	OnOutboundEvent(input Event) (output Event)
}
// 用户端处理
type EventCallback func(ev Event)
```

### TCPMessageTransmitter
- tcp.ltv用的transmitter是TCPMessageTransmitter。接收消息主要是调用了util.RecvLTVPacket，每个包分为长度、消息类型和内容三部分，然后调用DecodeMessage根据消息类型，找到消息定义元数据，根据元数据解析消息内容，后续再看消息定义相关内容
```golang
// 接收Length-Type-Value格式的封包流程
func RecvLTVPacket(reader io.Reader, maxPacketSize int) (msg interface{}, err error) {
	// Size为uint16，占2字节
	var sizeBuffer = make([]byte, bodySize)
	// 持续读取Size直到读到为止
	_, err = io.ReadFull(reader, sizeBuffer)
	// 用小端格式读取Size
	size := binary.LittleEndian.Uint16(sizeBuffer)
	// 分配包体大小
	body := make([]byte, size)
	// 读取包体数据
	_, err = io.ReadFull(reader, body)
	msgid := binary.LittleEndian.Uint16(body)
	msgData := body[msgIDSize:]
	// 将字节数组和消息ID用户解出消息
	msg, _, err = codec.DecodeMessage(int(msgid), msgData)
	return
}
```
- 发送消息调用了util.SendLTVPacket
```golang
// 发送Length-Type-Value格式的封包流程
func SendLTVPacket(writer io.Writer, ctx cellnet.ContextSet, data interface{}) error {
	var (
		msgData []byte
		msgID   int
		meta    *cellnet.MessageMeta
	)
	switch m := data.(type) {
	case *cellnet.RawPacket: // 发裸包
		msgData = m.MsgData
		msgID = m.MsgID
	default: // 发普通编码包
		var err error
		// 将用户数据转换为字节数组和消息ID
		msgData, meta, err = codec.EncodeMessage(data, ctx)
		if err != nil {
			return err
		}
		msgID = meta.ID
	}
	pkt := make([]byte, bodySize+msgIDSize+len(msgData))
	// Length
	binary.LittleEndian.PutUint16(pkt, uint16(msgIDSize+len(msgData)))
	// Type
	binary.LittleEndian.PutUint16(pkt[bodySize:], uint16(msgID))
	// Value
	copy(pkt[bodySize+msgIDSize:], msgData)
	// 将数据写入Socket
	err := WriteFull(writer, pkt)
	// Codec中使用内存池时的释放位置
	if meta != nil {
		codec.FreeCodecResource(meta.Codec, msgData, ctx)
	}
	return err
}
```

### MsgHooker
- tcp.ltv的hooker是MsgHooker，消息入站实现如下。简单来说就是先让rpc尝试解析这个消息，不是rpc的就让relay来处理。
rpc和relay具体处理后续再展开看看
```golang
func (self MsgHooker) OnInboundEvent(inputEvent cellnet.Event) (outputEvent cellnet.Event) {
	var handled bool
	var err error
	inputEvent, handled, err = rpc.ResolveInboundEvent(inputEvent)
	if !handled {
		inputEvent, handled, err = relay.ResoleveInboundEvent(inputEvent)
		if !handled {
			msglog.WriteRecvLogger(log, "tcp", inputEvent.Session(), inputEvent.Message())
		}
	}
	return inputEvent
}
```
- MsgHooker消息出站实现如下，其实这个时候消息已经打好包了，这里只是简单地打个log
```golang
func (self MsgHooker) OnOutboundEvent(inputEvent cellnet.Event) (outputEvent cellnet.Event) {
	handled, err := rpc.ResolveOutboundEvent(inputEvent)
	if !handled {
		handled, err = relay.ResolveOutboundEvent(inputEvent)
		if !handled {
			msglog.WriteSendLogger(log, "tcp", inputEvent.Session(), inputEvent.Message())
		}
	}
	return inputEvent
}
```

### EventCallback
- tcp.ltv会将回调函数包装一下，代码如下。简单来说就是收到消息时将它放入消息队列，等待后续处理
```golang
// 让EventCallback保证放在ses的队列里，而不是并发的
func NewQueuedEventCallback(callback cellnet.EventCallback) cellnet.EventCallback {
	return func(ev cellnet.Event) {
		if callback != nil {
			cellnet.SessionQueuedCall(ev.Session(), func() {
				callback(ev)
			})
		}
	}
}
```

### 收发消息处理
- 上述讲的是tcp.ltv这种Processor的transmitter/hooker/eventcallback，这三部分怎么用的，实现在CoreProcBundle
```golang
func (self *CoreProcBundle) ReadMessage(ses cellnet.Session) (msg interface{}, err error) {
	if self.transmit != nil {
		return self.transmit.OnRecvMessage(ses)
	}
	return nil, notHandled
}
func (self *CoreProcBundle) SendMessage(ev cellnet.Event) {
	if self.hooker != nil {
		ev = self.hooker.OnOutboundEvent(ev)
	}
	if self.transmit != nil && ev != nil {
		self.transmit.OnSendMessage(ev.Session(), ev.Message())
	}
}
func (self *CoreProcBundle) ProcEvent(ev cellnet.Event) {
	if self.hooker != nil {
		ev = self.hooker.OnInboundEvent(ev)
	}
	if self.callback != nil && ev != nil {
		self.callback(ev)
	}
}
```
- 从上面代码可以看到，ReadMessage通过transmitter从session中接收数据，如果是tcp就是从tcp连接中读取网络数据
- SendMessage先让hooker处理要发送的event，然后通过transmitter发送
- ProcEvent用于投递消息，例如ReadMessage一般最后会用ProcEvent将收到的消息投递到消息队列。投递时先让hooker处理，例如前述的MsgHooker会尝试用tcp和relay解析消息，然后调用eventcallback来将消息真正投递到消息队列
