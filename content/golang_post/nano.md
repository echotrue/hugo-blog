---
title: "Nano"
date: 2021-08-14T16:19:29+08:00
draft: false
---

`cluster`表示一个`nano`的集群，这个集群包含一组节点，每一个节点提供一组不同的服务，所有从客户端发送的服务请求首先会被发送到`gate`网关，然后被发送到响应的节点
```Go
type cluster struct {  
	// If cluster is not large enough, use slice is OK 
	currentNode *Node  
	rpcClient *rpcClient  

	mu sync.RWMutex  
	members []*Member  
}
```


`Node`是`nano`集群中的一个节点，该节点提供一组服务，所有这些服务都会注册到集群。消息将会被转发到提供各自服务的节点。
```Go
type Node struct {  
 Options // 节点配置项  
 ServiceAddr string // 当前节点的服务地址，主要用于RPC通信 
  
 cluster *cluster  
 handler *LocalHandler  
 server *grpc.Server  
 rpcClient *rpcClient  
  
 mu sync.RWMutex  
 sessions map[int64]*session.Session  
}
```

`Node.Options`

```Go
type Options struct {  
	Pipeline pipeline.Pipeline  
	IsMaster bool  
	AdvertiseAddr string  
	RetryInterval time.Duration  
	ClientAddr string  
	Components *component.Components  
	Label string  
	IsWebsocket bool  
	TSLCertificate string  
	TSLKey string  
}
```

#### Start Node
- 初始化配置
设置集群的`Options`选项。其中`Options.Components`类型定义了一个组件切片`comps`，这个切片的类型是`CompWithOptions`，其中`Comp`是组件，`Opts`是该组件的属性。
```Go
// Options.Components

type Components struct {  
	comps []CompWithOptions  
}

type CompWithOptions struct {  
	Comp Component  
	Opts []Option  
}    
```

而这些`Comp`组件则是实现了各种逻辑的单个服务。并且这些组件必须实现`Component`接口。

- 初始化节点属性
初始化`Node`的属性：`session`用来保存客户端会话,`cluster`是nano集群,`handler`存储了本地及远程服务.结构如下：

```Go
type cluster struct {  
	// If cluster is not large enough, use slice is OK currentNode *Node  
	rpcClient *rpcClient  

	mu sync.RWMutex  
	members []*Member  
}

type LocalHandler struct {  
	localServices map[string]*component.Service // all registered service  
	localHandlers map[string]*component.Handler // all handler method  

	mu sync.RWMutex  
	remoteServices map[string][]*clusterpb.MemberInfo  

	pipeline pipeline.Pipeline  
	currentNode *Node  
}
```

接下来将上面组件切片中组件依次注册到节点的handle中。这里的组件被定义为`Service`类型，组件中的方法被定义为`Handler`类型。`Service`实现了一个特定服务，该服务下的方法将在对应事件触发是被调用。

```Go
type Service struct {
	Name string // 服务名称 
	Type reflect.Type // 接收者类型  
	Receiver reflect.Value // 服务下方法的接收者 
	Handlers map[string]*Handler // 注册的方法
	SchedName string // name of scheduler variable in session data  
	Options options // 选项  
}

type Handler struct {  
	Receiver reflect.Value // receiver of method
	Method reflect.Method // method stub  
	Type reflect.Type // low-level type of method  
	IsRawArg bool // whether the data need to serialize  
}

```
注册服务及服务下func部分代码：
```Go
func (h *LocalHandler) register(comp component.Component, opts []component.Option) error {  
	 s := component.NewService(comp, opts)  

	 if _, ok := h.localServices[s.Name]; ok {  
	 return fmt.Errorf("handler: service already defined: %s", s.Name)  
	 }  
	 if err := s.ExtractHandler(); err != nil {  
	 return err  
	 }  

	 // register all localHandlers  
	 h.localServices[s.Name] = s  
	 for name, handler := range s.Handlers {  
	 n := fmt.Sprintf("%s.%s", s.Name, name)  
	 log.Println("Register local handler", n)  
	 h.localHandlers[n] = handler  
	 }  
	 return nil  
}
```
根据组件名初始化一个`component.Service`对象作为`value`，以组件的名称作为`key`，赋值给`localHandler.localServices`.
`ExtractHandler`检测组件及组件下方法是否导出。并将组件下的方法注册到`component.Service.Handlers`.接下来再将这些组件方法注册到`localHandler.localHandlers`.

`cache()`定义了握手响应数据和心跳包数据

## initNode

### init gRPC and register service

`n.server = grpc.NewServer()`创建一个`gRPC`服务器。此时该服务器上没有注册任何服务，也没有开始接收请求。

`clusterpb.RegisterMemberServer(n.server, n)`向`gRPC`注册`MemberServer`服务。`MemberServer`服务包含了一些列`API`,而节点`n`实现了`MemberServer`接口。因此所有启动的节点均提供这些`API`服务。

```Go
type MemberServer interface {  
	HandleRequest(context.Context, *RequestMessage) (*MemberHandleResponse, error)  
	HandleNotify(context.Context, *NotifyMessage) (*MemberHandleResponse, error)  
	HandlePush(context.Context, *PushMessage) (*MemberHandleResponse, error)  
	HandleResponse(context.Context, *ResponseMessage) (*MemberHandleResponse, error)  
	NewMember(context.Context, *NewMemberRequest) (*NewMemberResponse, error)  
	DelMember(context.Context, *DelMemberRequest) (*DelMemberResponse, error)  
	SessionClosed(context.Context, *SessionClosedRequest) (*SessionClosedResponse, error)  
	CloseSession(context.Context, *CloseSessionRequest) (*CloseSessionResponse, error)  
}
```

开辟一个`goroutine`开始监听`n.ServiceAddr`并开始接收`rpc`请求。

`n.rpcClient = newRPCClient()`初始化当前节点的`rpc`客户端对象。

`clusterpb.RegisterMemberServer(n.server, n)`，将当前Node实现了`MemberServer`的服务注册到`gRPC`服务器。

- 如果当前是主服务器`Master`,`clusterpb.RegisterMasterServer(n.server, n.cluster)`向服务器注册服务实现。`n.cluster`实现了`Master`接口的`Register`和`Unregister`两个API。当其他节点启动时会将其所有的服务通过`Register`注册到Master主节点。这里的`member`描述集群的一个成员，即节点。将节点作为成员添加到集群，设置集群`rpcClient`.

- 如果当前不是主服务器节点，根据主服务器Master的RPC地址`node.AdvertiseAddr`创建连接池，并获取连接。然后创建gRPC客户端。将当前节点成员信息通过主服务器的`Register`服务发送到集群的主服务器`Master`.主服务器节点的`cluster.Register()`接收注册请求,然后将新加入的节点成员信息通过`NewMember`事件发送到各个非Master节点，并将新加的节点成员信息添加到集群的成员列表，同时将新加节点成员中的所有服务添加到集群的`handle`中。并响应此时集群中所有服务给新节点。当前节点拿到集群中所有其他成员信息后，首先将这些远程的服务添加到当前节点`handler`的远程服务列表中。然后将这些远程节点成员添加到当前节点的集群的成员列表中

因此，Master主服务器节点上注册了`MemberServer`和`MasterServer`两组服务。而非Master节点服务器上只注册了`MemberServer`一组服务。

## Initialize all components
依次执行当前节点下所有组件的`Init()`和`AfterInit()`两个生命周期函数

## Listen client address
如果设置了`node.ClientAddr`地址，则开始监听并接收请求。`node.ClientAddr`是暴露给外部供外部访问的地址。


## Node实现的服务列表
```Go
type MemberServer interface {  
	HandleRequest(context.Context, *RequestMessage) (*MemberHandleResponse, error)  
	HandleNotify(context.Context, *NotifyMessage) (*MemberHandleResponse, error)  
	HandlePush(context.Context, *PushMessage) (*MemberHandleResponse, error)  
	HandleResponse(context.Context, *ResponseMessage) (*MemberHandleResponse, error)  
	NewMember(context.Context, *NewMemberRequest) (*NewMemberResponse, error)  
	DelMember(context.Context, *DelMemberRequest) (*DelMemberResponse, error)  
	SessionClosed(context.Context, *SessionClosedRequest) (*SessionClosedResponse, error)  
	CloseSession(context.Context, *CloseSessionRequest) (*CloseSessionResponse, error)  
}
```


## Cluster实现的服务列表
```Go
type MasterServer interface {  
	Register(context.Context, *RegisterRequest) (*RegisterResponse, error)  
	Unregister(context.Context, *UnregisterRequest) (*UnregisterResponse, error)  
}
```


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

`Handshake`和`HandshakeAck`类型分别为设置当前`UserAgent`的状态为`statusHandshake`和`statusWorking`.`Data`类型数据会解码数据包中的`Data`字段并交给`processMessage()`处理。然后更新`agent.lastAt`为当前时间。

### LocalHandler.processMessage()
`message`的类型包含：Request/Notify/Response/Push。分别代表请求/通知/响应/推送。

判断`message.Route`是否是本地服务列表中。如果是本地服务进入`localProcess()` 反之进入`remoteProcess()`。

先看`localProcess()`
