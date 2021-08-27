---
title: "Panic and Recover"
date: 2019-08-09T15:55:02+08:00
draft: false
tags:        [ "Go","panic"]
categories :      [ "Go","panic"]
---



### panic

&emsp;&emsp; `Golang` 中常见的错误处理方式是返回error给调用者。通常error使用场景是发生了逻辑错误。但是，如果是无法恢复的错误，可以选择使用panic。panic可以主动触发。也可以被动触发（例如：数组越界）。

&emsp;&emsp;panic会停掉当前正在执行的程序，与os.Exit(-1)不同的是：panic会有序的撤退，它会先处理完当前goroutine已经defer的任务，然后再退出整个程序。

```
func main() {
    var user = os.Getenv("USER_")
    go func() {
        defer func() {
            fmt.Println("defer 1")
        }()
        if user == ""{
            panic("should set user env.")
        }
    }()
    time.Sleep(1*time.Second)
    fmt.Println("get result")
}
```

&emsp;&emsp;上述代码输出：

```
defer 1
panic: should set user env.

goroutine 19 [running]:
main.main.func1(0x0, 0x0)
	D:/gopath/src/race_condition/index.go:16 +0x86
created by main.main
	D:/gopath/src/race_condition/index.go:11 +0x59

Process finished with exit code 2
```

说明panic坚守了自己的原则：**执行且只执行当前goroutine的defer，defer的特点是LIFO，即后进先出。如果有多个defer的时候，会倒序执行**



### recover

&emsp;&emsp;有时候不希望因为panic导致整个进程终止，因此需要像其他语言捕获异常。在Golang中可以通过在当前goroutine的defer中使用recover来捕获panic。recover只在defer的函数中有效，如果不是在defer上下文中调用，recover会直接返回nil