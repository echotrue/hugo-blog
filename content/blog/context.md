---
title: "Context"
date: 2020年10月3日15:13:54
draft: false
weight: 1
tags:        [ "Go","context","上下文"]
categories :      [ "Go"]
---

### 引言
`Golang`的`Context`包是专门用来简化对于处理单个请求的多个`goroutine`之间与请求域的数据，取消信号，截止时间等相关操作。
一个实际的例子是：
>在`Go`服务器程序中，每个请求都会有一个`goroutine`去处理。然而，处理程序可能还需要创建额外的`goroutine`去访问其他资源，比如：数据库，
>RPC服务等。由于这些`goroutine`都是在处理同一个请求，所以他们往往需要访问一些共享的资源，比如：用户身份信息，认证token
>，请求截止时间等。当请求超时或者被取消后，所有的`goroutine`都应该马上退出并且释放相关的资源。这种情况也需要用`Context`来为我们来取消掉所有
>的`goroutine`

### Context定义
`context`的主要数据结构是一种嵌套的结构或者说是单向的继承关系的结构，比如最初的context是一个小盒子，里面装了一些数据，
之后从这个context继承下来的children就像在原本的context中又套上了一个盒子，然后里面装着一些自己的数据。或者说context是一种分层的结构，
根据使用场景的不同，每一层context都具备有一些不同的特性，这种层级式的组织也使得context易于扩展，职责清晰。

`context`包的核心是`interface Context` ,声明如下：
```
type Context interface {

Deadline() (deadline time.Time, ok bool)

Done() <-chan struct{}

Err() error

Value(key interface{}) interface{}

}
```
`Context`定义很简单，一共四个方法：
1. Deadline方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；
第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。
2. Done方法返回一个只读的chan，类型为struct{}，我们在goroutine中，如果该方法返回的chan可以读取，
则意味着parent context已经发起了取消请求，我们通过Done方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。之后，Err 方法会返回一个错误，告知为什么 Context 被取消。
3. Err方法返回取消的错误原因，因为什么Context被取消。
4. Value方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。

### Context的实现方法
`Context` 虽然是个接口，但是并不需要使用方实现，`golang`内置的`context` 包，已经帮我们实现了2个方法，一般在代码中，
开始上下文的时候都是以这两个作为最顶层的`parent context`，然后再衍生出子`context`。这些 `Context` 对象形成一棵树：
当一个 `Context` 对象被取消时，继承自它的所有 `Context` 都会被取消。两个实现如下：
```
var (
    background = new(emptyCtx)
    todo = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```
1. 一个是Background，主要用于main函数、初始化以及测试代码中，作为Context这个树结构的最顶层的Context，也就是根Context，它不能被取消。
2. 一个是TODO，如果我们不知道该使用什么Context的时候，可以使用这个，但是实际应用中，暂时还没有使用过这个TODO。
3. 他们两个本质上都是emptyCtx结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。

```
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    
    return
}

func (*emptyCtx) Done() <-chan struct{} {

    return nil
}

func (*emptyCtx) Err() error {

    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {

    return nil
}

```

### Context的继承
有了如上的根Context，那么是如何衍生更多的子Context的呢？这就要靠context包为我们提供的With系列的函数了。
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```

通过这些函数，就创建了一颗Context树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。
1. WithCancel函数，传递一个父Context作为参数，返回子Context，以及一个取消函数用来取消Context。
2. WithDeadline函数，和WithCancel差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消Context，当然我们也可以不等到这个时候，可以提前通过取消函数进行取消。
3. WithTimeout和WithDeadline基本上一样，这个表示是超时自动取消，是多少时间后自动取消Context的意思。
4. WithValue函数和取消Context无关，它是为了生成一个绑定了一个键值对数据的Context，这个绑定的数据可以通过Context.Value方法访问到，这是我们实际用经常要用到的技巧，一般我们想要通过上下文来传递数据时，可以通过这个方法，如我们需要tarce追踪系统调用栈的时候。

### Context使用技巧和原则

- 不要把Context放在结构体中，要以参数的方式传递，parent Context一般为Background
- 应该要把Context作为第一个参数传递给入口请求和出口请求链路上的每一个函数，放在第一位，变量名建议都统一，如ctx。
- 给一个函数方法传递Context的时候，不要传递nil，否则在tarce追踪的时候，就会断了连接
- Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
- Context是线程安全的，可以放心的在多个goroutine中传递
- 可以把一个 Context 对象传递给任意个数的 gorotuine，对它执行 取消 操作时，所有 goroutine 都会接收到取消信号。

### Context使用示例

#### 请求链路传值
```
func func1(ctx context.Context) {
    ctx = context.WithValue(ctx, "k1", "v1")
    func2(ctx)
}

func func2(ctx context.Context) {
    fmt.Println(ctx.Value("k1").(string))
}
func main() {
    ctx := context.Background()
    func1(ctx)
}
```
我们在`func1`通过`WithValue(parent Context, key, val interface{}) Context`，赋值k1为v1，
在其下层函数`func2`通过`ctx.Value(key interface{}) interface{}`获取k1的值，比较简单。这里有个疑问，如果我是在`func2`里赋值，
在`func1`里面能够拿到这个值吗？答案是不能，`context`只能自上而下携带值，这个是要注意的一点。

#### 取消耗时操作，及时释放资源

##### 主动取消
```
func func1(ctx context.Context, wg *sync.WaitGroup) error {
    defer wg.Done()
    respC := make(chan int)
    // 处理逻辑
    go func() {
        time.Sleep(time.Second * 5)
        respC <- 10
    }()
    // 取消机制
    select {
    case <-ctx.Done():
        fmt.Println("cancel.")
        return errors.New("cancel")
    case r := <-respC:
        fmt.Println(r)
        return nil
    }
}

func main() {
    wg := new(sync.WaitGroup)
    ctx, cancel := context.WithCancel(context.Background())
    wg.Add(1)
    go func1(ctx, wg)
    time.Sleep(time.Second * 2)
    cancel()
    wg.Wait()
}
```

##### 超时取消

