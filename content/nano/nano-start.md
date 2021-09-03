---
title: "Nano Start"
date: 2021-08-31T16:54:27+08:00
weight: 1
draft: false
---
## 基础概念
一个`nano`应用程序由各个不同粒度的单元组成，这其中涉及到一些基础概念。例如：`node`,`cluster`,`component`,`service`,`handler`等等。这里简单介绍各个单元的概念，并概述他们之间的关系。

### Cluster

`cluster`表示一个`nano`的集群，这个集群包含一组节点，每一个节点提供一组不同的服务，所有从客户端发送的服务请求首先会被发送到`gate`网关，然后被发送到响应的节点
```Go
type cluster struct {  
	// If cluster is not large enough, use slice is OK 
	currentNode *Node  //当前节点对象
	rpcClient *rpcClient  // rpc客户端用于和其他节点通信

	mu sync.RWMutex  
	members []*Member  //所有的节点成员，member包含了节点地址，节点下的服务，是否是主节点
}
```


`Node`是`nano`集群中的一个节点，该节点提供一组服务，所有这些服务都会注册到集群。消息将会被转发到提供各自服务的节点。
```Go
type Node struct {  
 Options // 节点配置项  
 ServiceAddr string // 当前节点的服务地址，主要用于RPC通信 
  
 cluster *cluster  //集群对象
 handler *LocalHandler //节点本地处理程序
 server *grpc.Server  //grpc 服务器
 rpcClient *rpcClient  //rpc 客户端用于和其他节点通信
  
 mu sync.RWMutex  
 sessions map[int64]*session.Session  //会话列表
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

## Start Node
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

### install service and handler

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

### initNode

#### init gRPC and register service

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

因此，Master主服务器节点上注册了`MemberServer`和`MasterServer`两组服务。而非Master节点服务器上只注册了`MemberServer`一组服务。详见[Node实现的服务列表](#node实现的服务列表)和[Cluster实现的服务列表](#cluster实现的服务列表)

### Initialize all components
依次执行当前节点下所有组件的`Init()`和`AfterInit()`两个生命周期函数

### Listen client address
如果设置了`node.ClientAddr`地址，则开始监听并接收请求。`node.ClientAddr`是暴露给外部供外部访问的地址。


### Node实现的服务列表
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


### Cluster实现的服务列表
```Go
type MasterServer interface {  
	Register(context.Context, *RegisterRequest) (*RegisterResponse, error)  
	Unregister(context.Context, *UnregisterRequest) (*UnregisterResponse, error)  
}
```
