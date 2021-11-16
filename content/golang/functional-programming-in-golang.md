---
title: "Functional Programming in Golang"
date: 2021-11-16T16:22:06+08:00
draft: false
tags:        [ "函数","闭包","函数式选项","高阶函数"]
categories :      [ "Go"]
---

### Higher-order function

`Higher-order function`又称为高阶函数.高阶函数至少支持以下特性之一:
- 将一个或多个函数作为参数（即过程参数）
- 返回函数作为其结果

以下为一个使用高阶函数的限流器Demo:
```Go
type Limiter interface {  
	Limit(key string) bool  
}  

type ServerInterceptor func(ctx context.Context) (interface{}, error)  

func Interceptor(limiter Limiter) ServerInterceptor {  
	return func(ctx context.Context) (interface{}, error) {  
		if limiter.Limit("k") {  
			return nil, errors.New("请求过于频繁")  
		} 
		return "ok", nil  
	}  
}
```

### Functional Options Pattern

`Functional Options Pattern`又称函数式选项模式.由于Golang中不支持参数默认值,所以针对一些函数的可选参数没有合适的处理方式.我们可以通过这种方式来构造结构体对象.

```Go
type Person struct {  
	Name string  
	Age uint8  
	Gender uint8  
}  
  
type Option func(*Person)  
  
func SetName(n string) Option {  
	return func(o *Person) {  
		o.Name = n  
	}  
}  
  
func SetAge(a uint8) Option {  
	return func(o *Person) {  
		o.Age = a  
	}  
}  
  
func NewPerson(opts ...Option) Person {  
	opt := Person{}  
	for _, o := range opts {  
		o(&opt)  
	} 
	return opt  
}
```

### Closure Function

在Golang中函数被看作是第一类值,这就意味着函数像变量一样有类型,有值.
- 函数变量的零值是nil. 这意味着它可以和nil进行比较.但两个函数变量之间不能比较
- 调用nil的函数变量会导致报错

```Go
type f func()

func main() {
	var a f
	if a == nil {
		fmt.Println("空函数")
	} else {
		a()
	}
}
```

#### 什么是闭包
闭包是匿名函数与匿名函数所引用环境的组合。匿名函数有动态创建的特性，该特性使得匿名函数不用通过参数传递的方式，就可以直接引用外部的变量。这就类似于常规函数直接使用全局变量一样.
```Go
func incr() func() int {
	var x int
	return func() int {
		x++
		return x
	}
}

func main() {
	i := incr()
	fmt.Println(i())
	fmt.Println(i())
	fmt.Println(i())
}
```
通过`i:=incr()`把`incr`函数的返回值(`func() int 类型的函数`)赋值给`i`,`i`就是一个闭包.`i`中有着指向`x`地址的指针.所以调用`i()`会修改x的值.因此,以上代码会输出1,2,3

```Go
func main() {
	x := 1
	func() {
		fmt.Println(&x)
	}()
	fmt.Println(&x)
}
```
以上会输出`0xc00000a0b8,0xc00000a0b8`.这也再次证明了闭包中保存着外部变量的地址


#### 循环中的闭包
```Go
func main() {
	var dummy [3]int
	var f func()
	for i := 0; i < len(dummy); i++ {
		f = func() {
			println(i)
		}
	}
	f() // 3
}
```
以上代码输出3 . 由于闭包f可以访问`i`的引用,而`i`实际加到3的时候由于不满足`i < len(dumy)`而结束循环 . 因此调用`f()`的时候通过解`i`的引用得到的值是3. 如果使用`for range` 则不会输出3

```Go
func main() {
	var dummy [3]int
	var f func()
	for i := range dummy{
		f = func() {
			println(i)
		}
	}
	f() // 3
}
```

另外一个例子:
```Go
func main() {
	var funcSlice []func()
	for i := 0; i < 3; i++ {
		funcSlice = append(funcSlice, func() {
			println(i)
		})
	}
	for j := 0; j < 3; j++ {
		funcSlice[j]()
	}
}
```

通常为了避免以上问题,可以声明一个新的局部变量`j:=i`,且把之后对`i`的操作改为对`j`的操作改为对`j`的.也可以通过声明新的匿名函数,并将i作为参数传递进去.由于`Golang`函数参数是按值传递,所以每个闭包函数可以独立引用传进来的参数


### 匿名函数小知识点

#### 匿名函数的调用
函数体后面的`()`表示直接执行当前函数
```Go
i := 1
f := func() {
	i += 2
}
f()
fmt.Println(i)
```
以上代码等同于
```Go
i := 1
func() {
	i += 2
}()
fmt.Println(i)
```

#### 匿名函数传参
将匿名函数赋值给变量使用时
```Go
i := 1
f := func(n int) {
	fmt.Println(n)
}
f(i)
```

直接执行匿名函数时,这里的m,n为形参 . 2,2为实参
```Go
i := 1
func(m, n int) {
	i = i + m + n
}(2, 2)
fmt.Println(i)
```