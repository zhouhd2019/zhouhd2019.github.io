---
layout:     post
title:      "cellnet分析0----EventQueue"
subtitle:   "golang网络编程示例"
date:       2021-10-28  20:52:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - cellnet
    - golang
    - Server
---

<https://github.com/davyxu/cellnet>
以cellnet为例，讲一下如何用golang实现一个服务器框架。本篇先讲消息队列


### 概述和官方文档
- cellnet用消息队列来处理收到的消息和不需要马上处理的操作，建议每个网络处理线程一个消息队列，减少竞争
- 创建和开启队列的官方例子如下：
```golang
queue := cellnet.NewEventQueue()
// 启动队列
queue.StartLoop()
// 这里添加队列使用代码
// 等待队列结束, 调用queue.StopLoop(0)将退出阻塞
queue.Wait()
```
- 投递回调代码如下：
```golang
queue.Post(func() {fmt.Println("hello")})
```

### EventQueue
- 典型的暴露接口，隐藏实现的做法
```golang
// 事件队列
type EventQueue interface {
	// 事件队列开始工作
	StartLoop() EventQueue
	// 停止事件队列
	StopLoop() EventQueue
	// 等待退出
	Wait()
	// 投递事件, 通过队列到达消费者端
	Post(callback func())
	// 是否捕获异常
	EnableCapturePanic(v bool)
	// 获取事件数量
	Count() int
}

type CapturePanicNotifyFunc func(interface{}, EventQueue)

type eventQueue struct {
	*Pipe
	endSignal sync.WaitGroup
	capturePanic bool
	onPanic CapturePanicNotifyFunc
}
```
- 创建队列实现如下，主要是创建了Pipe来存放消息，并且指定了崩溃时的处理函数
```golang
// 创建默认长度的队列
func NewEventQueue() EventQueue {
	return &eventQueue{
		Pipe: NewPipe(),
		// 默认的崩溃捕获打印
		onPanic: func(raw interface{}, queue EventQueue) {
			fmt.Printf("%s: %v \n%s\n", time.Now().Format("2006-01-02 15:04:05"), raw, string(debug.Stack()))
			debug.PrintStack()
		},
	}
}
```
- 开启了崩溃捕获，并且指定了处理函数后，每次处理消息时，如果发生崩溃就会调用处理函数
```golang
// 保护调用用户函数
func (self *eventQueue) protectedCall(callback func()) {
	if self.capturePanic {
		defer func() {
			if err := recover(); err != nil {
				self.onPanic(err, self)
			}
		}()
	}
	callback()
}
```

### StartLoop
- 最关键的函数是循环StartLoop，通过Pick不断地从队列Pipe中取出消息，并进行处理
```golang
// 开启事件循环
func (self *eventQueue) StartLoop() EventQueue {
	self.endSignal.Add(1)
	go func() {
		var writeList []interface{}
		for {
			writeList = writeList[0:0]
			exit := self.Pick(&writeList)
			// 遍历要发送的数据
			for _, msg := range writeList {
				switch t := msg.(type) {
				case func():
					self.protectedCall(t)
				case nil:
					break
				default:
					log.Printf("unexpected type %T", t)
				}
			}
			if exit {
				break
			}
		}
		self.endSignal.Done()
	}()
	return self
}
```
- 停止循环就是简单地放一个nil，从StartLoop也可以看出来
```golang
// 停止事件循环
func (self *eventQueue) StopLoop() EventQueue {
	self.Add(nil)
	return self
}
```
- 线程调用StartLoop后，可以调用Wait等待循环结束
```golang
// 等待退出消息
func (self *eventQueue) Wait() {
	self.endSignal.Wait()
}
```

### 主动投递
- 逻辑代码可以主动投递一个函数到消息队列，例如别的线程要让主线程执行某段逻辑
```golang
// 在会话对应的Peer上的事件队列中执行callback，如果没有队列，则马上执行
func SessionQueuedCall(ses Session, callback func()) {
	if ses == nil {
		return
	}
	q := ses.Peer().(interface {
		Queue() EventQueue
	}).Queue()
	QueuedCall(q, callback)
}

// 有队列时队列调用，无队列时直接调用
func QueuedCall(queue EventQueue, callback func()) {
	if queue == nil {
		callback()
	} else {
		queue.Post(callback)
	}
}

// 派发事件处理回调到队列中
func (self *eventQueue) Post(callback func()) {
	if callback == nil {
		return
	}
	self.Add(callback)
}
```

### Pipe
- 简单的队列实现，支持唤醒
```golang
// 不限制大小，添加不发生阻塞，接收阻塞等待
type Pipe struct {
	list      []interface{}
	listGuard sync.Mutex
	listCond  *sync.Cond
}
```
- 可以通过Pick取出数据，没有数据则阻塞，等待添加时通知
```golang
// 如果没有数据，发生阻塞
func (self *Pipe) Pick(retList *[]interface{}) (exit bool) {
	self.listGuard.Lock()
	for len(self.list) == 0 {
		self.listCond.Wait()
	}
	// 复制出队列
	for _, data := range self.list {
		if data == nil {
			exit = true
			break
		} else {
			*retList = append(*retList, data)
		}
	}
	self.list = self.list[0:0]
	self.listGuard.Unlock()
	return
}
```
- 添加时注意需要先锁上，以此支持别的线程投递消息
```golang
// 添加时不会发送阻塞
func (self *Pipe) Add(msg interface{}) {
	self.listGuard.Lock()
	self.list = append(self.list, msg)
	self.listGuard.Unlock()

	self.listCond.Signal()
}
```
