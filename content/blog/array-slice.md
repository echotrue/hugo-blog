---
title: "Array And Slice"
date: 2020年10月21日11:49:53
draft: false
weight: 3
tags:        [ "Go","array","slice","数组","切片"]
categories :      [ "Go"]
---


### 数组

数组类型定义了`长度`和`元素`类型。数组的长度是固定的，长度是数组类型的一部分。数组不需要显式的初始化；数组的零值是可以直接使用的，数组元素会自动初始化为其对应类型的零值。
```
var arr [4]int      // 声明
fmt.Println(arr[1]) // 不需要显式的初始化
arr1 := [2]string{"Penn", "Teller"} // 数组的字面值
arr2 := [...]string{"Penn", "Teller"} // 编译器统计数组字面值中的元素数量
fmt.Println(arr1, len(arr1), cap(arr1))
fmt.Println(arr2, len(arr2), cap(arr2))
```
Go的数组是值语义。当一个数组变量被赋值或者被传递的时候，实际上会复制整个数组。 （为了避免复制数组，你可以传递一个指向数组的指针，但是数组指针并不是数组。）
```
func update(arr *[2]string)  {
    arr[1] = "axlrose"
}
func main() {
    arr1 := [2]string{"Penn", "Teller"}
    update(&arr1)
    fmt.Println(arr1)
}
```


### 切片的创建和初始化
Golang中切片有三种初始化方式：
- 通过下标的方式获得数组或者切片的一部分； 
- 使用字面量初始化新的切片；
- 使用关键字 make 创建切片：
```go
arr[0:3] or slice[0:3]

slice := []int{1, 2, 3, 4} // 通过字面量创建并初始化长度，容量都为4的切片
slice := []int{99: 0} // 通过字面量创建并初始化长度和容量都是100的切片

slice := make([]int, 3, 4) // make() 创建并初始化长度为3，容量为4的切片

var slice []int // 只创建切片，不初始化。值为nil，又称空切片，它的长度和容量都为0
```
> 需要注意的是使用下标初始化切片不会造成原始数组或者切片中数据的拷贝，它只会创建一个指向原始数组的切片值，所以修改新切片的数据也会修改原始切片。

### 零值切片，nil切片和空切片

- 零切片：切片元素的值均是元素类型所对应的的0值切片


```shell script
slice1 := make([]int, 10)
fmt.Println(slice1) //[0 0 0 0 0 0 0 0 0 0]

slice2 := make([]*int,10)
fmt.Println(slice2)  //[<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil>]
```


### 切片追加和扩容
在分配内存空间之前需要先确定新的切片容量，Go 语言根据切片的当前容量选择不同的策略进行扩容：

- 如果期望容量大于当前容量的两倍就会使用期望容量；
- 如果当前切片的长度小于 1024 就会将容量翻倍；
- 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；