---
title: "Golang Search Slice"
date: 2020年10月3日14:45:11
draft: false
tags:        [ "Go","Slice"]
categories :      [ "Go"]
---

## 题目

> 假设有一个超长的切片，切片的元素类型为int，切片中的元素为乱序排列。限时5秒，使用多个goroutine查找切片中是否存在给定值，
>在找到目标值或者超时后立刻结束所有goroutine的执行。比如切片为：[23, 32, 78, 43, 76, 65, 345, 762, …… 915, 86]，
>查找的目标值为345，如果切片中存在目标值程序输出:"Found it!"并且立即取消仍在执行查找任务的goroutine。如果在超时时间未
>找到目标值程序输出:"Timeout! Not Found"，同时立即取消仍在执行查找任务的goroutine。

首先题目里提到了在找到目标值或者超时后立刻结束所有`goroutine`的执行，完成这两个功能需要借助计时器、通道和`context`才行。
我能想到的第一点就是要用`context.WithCancel`创建一个上下文对象传递给每个执行任务的`goroutine`，外部在满足条件后（找到目标值或者已超时）
通过调用上下文的取消函数来通知所有`goroutine`停止工作。

### 解法

```
package main

import (
	"context"
	"fmt"
	"os"
	"time"
)

func main() {
    timer := time.NewTimer(time.Second * 5)
    data := []int{1, 2, 3, 10, 999, 8, 345, 7, 98, 33, 66, 77, 88, 68, 96}
    dataLen := len(data)
    size := 3
    target := 345
    ctx, cancel := context.WithCancel(context.Background())
    resultChan := make(chan bool)
    for i := 0; i < dataLen; i += size {
        end := i + size
        if end >= dataLen {
            end = dataLen - 1
        }
        go SearchTarget(ctx, data[i:end], target, resultChan)
    }
    select {
    case <-timer.C:
        fmt.Fprintln(os.Stderr, "Timeout! Not Found")
        cancel()
    case <-resultChan:
        fmt.Fprintf(os.Stdout, "Found it!\n")
        cancel()
    }
    
    time.Sleep(time.Second * 2)
}

func SearchTarget(ctx context.Context, data []int, target int, resultChan chan bool) {
    for _, v := range data {
        select {
        case <-ctx.Done():
            fmt.Fprintf(os.Stdout, "Task cancelded! \n")
            return
        default:
        }
        // 模拟一个耗时查找，这里只是比对值，真实开发中可以是其他操作
        fmt.Fprintf(os.Stdout, "v: %d \n", v)
        time.Sleep(time.Millisecond * 1500)
        if target == v {
            resultChan <- true
            return
        }
    }
}

```