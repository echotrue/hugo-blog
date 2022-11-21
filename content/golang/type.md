---
title: "Type"
date: 2022年10月31日14:40:30
draft: false
---

处理接口值时,变量的"动态类型"很重要.动态类型定义如下([源](http://golang.org/ref/spec#Types)):

> 变量的静态类型(或仅仅类型)是其声明定义的类型.**接口类型的变量也有一个不同的动态类型,它是运行时存储在变量中的值的实际类型.**动态类型可能在执行期间有所不同,但始终可分配给接口变量的静态类型.对于非接口类型,动态类型始终是静态类型.

考虑这个例子:

```
var someValue interface{} = 2
```

静态类型`someValue`是`interface{}`动态类型,`int`并且可能在未来很好地改变.例:

```
var someValue interface{} = 2

someValue = "foo"
```

在上面的示例中,动态类型从`someValue`更改`int`为`string`.



Slice，Map，函数三种引用类型以及含有以上三种类型的结构体和数组不能直接用==比较，只能用reflect.deepEqual进行比较。

channel可以用==比较，且只有两个通道是由同一个 make 创建才相等;

接口可以用==比较，且只有两个接口具有相同的动态类型和动态值两者才相等；并且当 interface 与非 interface 比较时，会将非interface 转换成 interface，然后再按照 两个 interface 比较 的规则进行比较；接口的动态类型和动态值都为nil，接口才是nil。

结构体和数组作为复合类型，能否比较以其内部的元素是否能比较决定，且数组要求长度相同。

空结构体不可相互比较：

若逃逸到堆上，空结构体则默认分配的是 runtime.zerobase 变量，是专门用于分配到堆上的 0 字节基础地址。因此两个空结构体都是 runtime.zerobase，一比较当然就是 true 了。

若没有发生逃逸，也就分配到栈上，在 Go 编译器的代码优化阶段，会对其进行优化，直接返回 false。并没有比较的意义了。
————————————————
版权声明：本文为CSDN博主「傅里叶、」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_34562093/article/details/120981451