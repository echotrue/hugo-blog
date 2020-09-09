---
title: "Golang channel"
date: 2020年9月8日14:51:55
draft: false
tags:        [ "Go"]
categories :      [ "Go" ]
---

### 概念

&emsp;&emsp;从字面上看，`channel`的意思大概就是管道的意思。`channel`是一种`goroutine`用以接收或发送消息的安全的消息队列，`channel`就像两个`goroutine`之间的导管，来实现各种资源的同步。在官方`Effective go`文档中有一句非常著名的话可以说明`channel`在使用`Golang`进行并发编程的时候扮演了极为重要的角色

> *Do not communicate by sharing memory; instead, share memory by communicating.*



### Channel类型

```
readOnlyCh  := make(<-chan int)//表示一个元素类型为T的单向接收通道类型。 编译器不允许向此类型的值中发送数据。
writeOnlyCh := make(chan<- int)//表示一个元素类型为T的单向发送通道类型。 编译器不允许从此类型的值中接收数据
readWriteCh := make(chan int)//表示一个元素类型为T的双向通道类型。 编译器允许从此类型的值中接收和向此类型的值中发送数据
```



### 阻塞

&emsp;&emsp;根据`Channel`缓冲区的大小，我们又可以将`Channel`分为`Unbuffered channels`与`Buffered channels`。其中，`Unbuffered channels`的缓冲区大小为0，这种`channel`的接收者会阻塞直至接收到消息，发送者会阻塞直至接收者接收到消息，这种机制可以用于两个`goroutine`进行状态同步。`Buffered channels`拥有缓冲区，当缓冲区已满时，发送者会阻塞；当缓冲区为空时，接收者会阻塞。引用[The Nature Of Channels In Go](https://www.ardanlabs.com/blog/2014/02/the-nature-of-channels-in-go.html)中的两张图片来说明两种`channel`的特性。

- `Unbuffered Channels`

<img src="C:\Users\dy101\Desktop\1.png" alt="avatar" style="zoom: 33%;" />







- `Buffered Channels`

<img src="C:\Users\dy101\Desktop\2.png" alt="avatar" style="zoom: 80%;" />



### <a id="code1">基本用法</a>

```
ch := make(chan string)

go func() {
    ch <- "hello"
}()

msg := <-ch
fmt.Println(msg)
```

&emsp;&emsp;以上代码，利用不带缓冲的`channel`双向阻塞的特性。主`goroutine`(就是main函数)会阻塞直到接收到子`goroutine`向`ch`中写入的值。所以保证了`hello`一定会输出。



### 利用`Channel`实现并发的同步

&emsp;&emsp;[基本用法](#code1)中的代码片段使用阻塞接收的方式，实现了主`goroutine`等待子`goroutine`完成。最终达到了两个`goroutine`的同步。使用`WaitGroup`同样能达到多个`goroutine`的同步，尤其是需要等待多个协程的情况下，`WaitGroup`会是更好的选择。

```
func worker(i int, wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(time.Second)
	fmt.Printf("worker %d stared\n", i)
}
func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}
	wg.Wait()
}
```



执行结果：

```
worker 1 stared
worker 5 stared
worker 4 stared
worker 3 stared
worker 2 stared
```



### Channel 选择器

&emsp;&emsp;`select`语句主要用在从多个读或者写`channel`的操作中进行选择。`select`语句会一直阻塞直到，有至少一个读或者写`channel`操作就绪。如果同时有多个操作准备就绪，`select`语句会随机选择其中一个执行。`select`语法类似`switch`，每个`case`相当于一个通道操作。

```
c1 := make(chan string)
c2 := make(chan string)

go func() {
    time.Sleep(1 * time.Second)
    c1 <- "one"
}()
go func() {
    time.Sleep(1 * time.Second)
    c2 <- "two"
}()

for i := 0; i < 2; i++ {
    select {
        case msg1 := <-c1:
        fmt.Printf("接到消息：%s\n", msg1)
        case msg2 := <-c2:
        fmt.Printf("接到消息：%s\n", msg2)
    }
}
```



以上代码会输出：接到消息one，接到消息two。

### Channel遍历

`for...range`可以用来遍历通道，它会反复从通道接收数据直到通道关闭。

```
queue := make(chan string, 2)
queue <- "one"
queue <- "two"
close(queue)

for elem := range queue {
    fmt.Println(elem)
}
```



### Channel 的关闭

&emsp;&emsp;内置函数`close()`可以用来关闭`channel`,`close()`函数只能关闭可读写或者只写的通道。通道的关闭通常应该遵循一定的原则：由生产者（发送者）来关闭，保证不关闭已关闭的通道(或向已关闭的通道发送值)。

```
ch := make(chan string) //可以关闭的通道
ch := make(chan<- string) //可以关闭的通道
ch := make(<-chan string) //不能关闭的通道
```



1、关闭 一个通道意味着不能再向这个通道发送值了。 该特性可以向通道的接收方传达工作已经完成的信息。

```
msg := make(chan string)
done := make(chan bool)
go func() {
    for {
        select {
            case m, ok := <-msg:
            if ok {
                fmt.Printf(m)
            } else {
                fmt.Println("All message has received.")
                done <- true
                return
            }
        }
    }
}()

for i := 1; i < 4; i++ {
    msg <- fmt.Sprintf("Message %d\n", i)
}
close(msg)
<-done
```

2、向一个已经关闭的`channel`发送数据会`panic`

3、从一个已经关闭的通道中读数据，依然可以读到数据。读到的内容是通道元素类型所对应的的零值。（例如：int类型channel读到的是0）。

```
ch := make(chan int)
dataCh := make(chan string)

go func() {
    dataCh <- "str one"
    dataCh <- "str two"

    close(dataCh)
}()

go func() {
    for {
        time.Sleep(time.Millisecond * 500)
        select {
            case str := <-dataCh:
            fmt.Println("-->",str)
        }
    }
}()

<-ch

//输出：
--> str one
--> str two
--> 
--> 
```

4、当发送者关闭通道后，通道接收器可以通过向接收表达式分配第二个参数来判断通道是否关闭。`c,ok := <-ch`，如果没有更多的值要接受且通道已经关闭，`ok`为`false`

```
ch := make(chan int)
dataCh := make(chan string)

go func() {
    dataCh <- "str one"
    dataCh <- "str two"

    close(dataCh)
}()

go func() {
    for {
        time.Sleep(time.Millisecond * 500)
        select {
            case str, ok := <-dataCh:
            if ok {
                fmt.Println("-->", str)
            } else {
                fmt.Println("通道已关闭")
                ch <- 1
                return
            }
        }
    }
}()
<-ch
```



### channel的基本操作和注意事项

channel存在`3种状态`：

1. nil，未初始化的状态，只进行了声明，或者手动赋值为`nil`
2. active，正常的channel，可读或者可写
3. closed，已关闭，**千万不要误认为关闭channel后，channel的值是nil**

channel可进行`3种操作`：

1. 读
2. 写
3. 关闭

把这3种操作和3种channel状态可以组合出`9种情况`：

| 操作      | nil的channel | 正常channel | 已关闭channel |
| :-------- | :----------- | :---------- | :------------ |
| <- ch     | 阻塞         | 成功或阻塞  | 读到零值      |
| ch <-     | 阻塞         | 成功或阻塞  | panic         |
| close(ch) | panic        | 成功        | panic         |
