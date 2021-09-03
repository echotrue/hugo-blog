---
title: "One Nano Request"
date: 2021-08-31T16:57:22+08:00
weight: 2
draft: false
---

## Websocket request

一般由网关节点向外部暴露`n.ClientAddr`地址并监听，接收客户端发送的请求。以`websocket`请求为例。`listenAndServeWS`方法负责监听`n.ClientAddr`并接收`ws`请求。接收到请求后交给节点下的`LocalHandler`的`handleWs()`方法，`handleWs()`会对连接对象封装成`wsConn`类型，结构如下：
```Go
type wsConn struct {  
	conn *websocket.Conn  
	typ int // message type  
	reader io.Reader  
}
```

然后开辟一个新的`goroutine`（`LocalHandler.handle()`方法）从连接中持续读取数据。

### LocalHandler.handle()
实例化一个`Agent`对象，相当于`User Agent`。它对应一个用户代理，用来存储原始连接信息。
```Go
agent struct {  
	// regular agent member 
	session *session.Session // session会话
	conn net.Conn // low-level conn fd 
	lastMid uint64 // last message id
	state int32 // current agent state
	chDie chan struct{} // wait for close  
	chSend chan pendingMessage // push message queue  
	lastAt int64 // last heartbeat unix time stamp  
	decoder *codec.Decoder // binary decoder  
	pipeline pipeline.Pipeline

	rpcHandler rpcHandler  
	srv reflect.Value // cached session reflect.Value  
}
```

保存当前会话到当前节点。然后开辟一个`goroutine`监听`channel`消息。监听的消息类型如下：
- `<-ticker.C`类型。每隔30秒检测上一次心跳包的时间距离现在是否超过60秒，如果没超过则发送心跳包响应数据`hbd`到`chWrite`通道。反之，则`return`跳出当前`for...select`.进而执行`defer`延迟函数：
```Go
defer func() {  
	ticker.Stop()  //停止心跳包检测定时器
	close(a.chSend)  //关闭推送消息队列
	close(chWrite)  
	a.Close()  //关闭当前User Agent
	if env.Debug {  
		log.Println(fmt.Sprintf("Session write goroutine exit, SessionID=%d, UID=%d", a.session.ID(), a.session.UID()))  
	}
}()
```

- `<-chWrite`类型。主要接收心跳包响应数据，并响应数据到客户端。
- `<-a.chSend`类型。处理推送数据
- `<-a.chDie`类型。
- `<-env.Die`类型。应用退出通知


接着，循环从当前连接中读取网络数据包，并由`LocalHandler.processPacket()`方法根据网络数据包的类型处理消息。网络数据包有以下类型：
- Handshake = 0x01 表示握手：请求（客户端）<===>握手响应（服务端）
- HandshakeAck = 0x02 表示从客户端到服务端的握手确认
- Heartbeat = 0x03 心跳数据包
- Data = 0x04 通用数据包
- Kick = 0x05 disconnect message from server

其中通用数据包是`message.Message`类型的。`message`的类型包含：Request/Notify/Response/Push。分别代表请求/通知/响应/推送。

`Handshake`和`HandshakeAck`类型分别为设置当前`UserAgent`的状态为`statusHandshake`和`statusWorking`.`Data`类型数据会解码数据包中的`Data`字段并交给`processMessage()`处理。然后更新`agent.lastAt`为当前时间。

循环读取消息失败时，会执行defer延迟函数。依次通知其他节点当前连接关闭。然后关闭当前连接。

### LocalHandler.processMessage()

判断`message.Route`是否是本地服务列表中。如果是本地服务进入`localProcess()` 反之进入`remoteProcess()`。

先看`localProcess()`
