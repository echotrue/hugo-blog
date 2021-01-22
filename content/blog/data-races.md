---
title: "Golang Data Races"
date: 2020年10月3日11:12:05
draft: false
tags:        [ "Go","并发","数据竞争","并发安全","not concurrency-safety","concurrency-safety"]
categories :      [ "Go"]
---

### 关于Golang并发安全

1. [谈谈go语言编程的并发安全](http://yanyiwu.com/work/2015/02/07/golang-concurrency-safety.html)
2. [Benign Data Races: What Could Possibly Go Wrong?](https://software.intel.com/content/www/us/en/develop/blogs/benign-data-races-what-could-possibly-go-wrong.html)

### 什么是数据竞争

```
package main

import (
	"fmt"
)

func main() {
	var i int
	go func() {
		i = 5
	}()
	fmt.Println(i)
}
```
先通过以上程序来了解什么是数据竞争。首先声明一个变量i，默认值为0。然后开启一个单独的`goroutine`来设置`i`的值。
同时，在不知道开启的`goroutine`是否已经执行完成的情况下打印i的值。所以，当前正在发生两个操作：
- 变量`i`的值正在被设置为5
- 打印i的值

所以，最后程序打印出来的值可能是0或者5。这就叫数据竞争，i的值根据以上两个操作哪一个先完成而不同。

### 检测数据竞争
`Golang`有一个内置的数据竞争检测器，只需要在使用`Go`命令行工具的时候添加`-race`标志。例如：让我们尝试用`-race`标志来
运行我们刚刚编写的程序：
```
$ go run -race main.go
0
==================
WARNING: DATA RACE
Write at 0x00c000122068 by goroutine 7:
  main.main.func1()
      F:/go_project/api-service/test/core/main.go:10 +0x3f


Previous read at 0x00c000122068 by main goroutine:
  main.main()
      F:/go_project/api-service/test/core/main.go:12 +0x8f


Goroutine 7 (running) created at:
  main.main()
      F:/go_project/api-service/test/core/main.go:9 +0x81
==================
Found 1 data race(s)
exit status 66


```
0是打印结果，第一部分告诉我们在子`goroutine`中尝试写入的位置，第二部分告诉我们在主`goroutine`中，同时有一个读的操作。
第三部分描述了导致数据竞争的`goroutine`是在哪里创建。
除了`go run`名另外，`go build`和`go test`命令也支持使用`-race`标志。这个会使编译器创建的应用程序能够记录所有运行期
间对共享变量访问，并且会记录下每一个读或者写共享变量的goroutine的身份信息。

竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件，并不能证明之后不会发生数据竞争。由于需要额外
的记录，因此构建时加了竞争检测的程序跑起来会慢一些，且需要更大的内存，即使是这样，这些代价对于很多生产环境的工作来说还是可
以接受的。对于一些偶发的竞争条件来说，使用附带竞争检查器的应用程序可以节省很多花在Debug上的时间。

### 数据竞争解决方案
Go提供了很多解决它的选择。所有这些解决方案的思路都是确保在我们写入变量时阻止对该变量的访问。一般常用的解决数据竞争的方案有：
使用WaitGroup锁，使用通道阻塞以及使用Mutex锁，下面我们一个个来看他们的用法并比较一下这几种方案的不同点。

### 使用WaitGroup
```
func main() {
    var i int
    var wg sync.WaitGroup
    wg.Add(1) // 通知程序有一个需要等待完成的任务
    go func() {
        i = 5
        wg.Done() // 告诉程序任务已经执行完成
    }()
    wg.Wait() // 阻塞当前程序直到等待的任务执行完成
    fmt.Println(i)
}
```

### 使用通道
```
func main() {
    var i int
    wait := make(chan struct{})
    go func() {
        i = 5
        wait <- struct{}{}
    }()
    <-wait
    fmt.Println(i)
}
```

### 使用Mutex
以上两种解决方案都是在确定的读取和写入顺序的情况下来保证数据的一致性。当程序读取和写入的先后顺序不固定的时候，以上方案便不能满足我们。
这种情况下我们应该考虑使用`Mutex`互斥锁。使用互斥锁，可以保证读取和写入操作不能同时发生。

```
type SafeNumber struct {
    i int
    m sync.Mutex
}

func (sn *SafeNumber) Set(n int) {
    sn.m.Lock()
    defer sn.m.Unlock()
    sn.i = n
}

func (sn *SafeNumber) Get() int {
    sn.m.Lock()
    defer sn.m.Unlock()
    return sn.i
}
func main() {
    sn := new(SafeNumber)
    go func() {
        sn.Set(5)
    }()
    fmt.Println(sn.Get())
}
```


