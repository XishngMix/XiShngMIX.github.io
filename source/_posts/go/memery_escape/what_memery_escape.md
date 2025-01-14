---
title: Golang 内存逃逸
date: 2022-10-3 00:17
tags:
- 基本知识
- 底层原理
categories:
- 基本知识
- 内存逃逸
---

## 什么是内存逃逸？

在程序中，每个函数块都会有自己的内存区域用来存自己的局部变量（内存占用少）、返回地址、返回值之类的数据，这一块内存区域有特定的结构和寻址方式，寻址起来十分迅速，开销很少。这一块内存地址称为栈。栈是线程级别的，大小在创建的时候已经确定，当变量太大的时候，会"逃逸"到堆上，这种现象称为内存逃逸。简单来说，局部变量通过堆分配和回收，就叫内存逃逸。

## 有什么危害？

堆是一块没有特定结构，也没有固定大小的内存区域，可以根据需要进行调整。全局变量，内存占用较大的局部变量，函数调用结束后不能立刻回收的局部变量都会存在堆里面。变量在堆上的分配和回收都比在栈上开销大的多。对于 go 这种带 GC 的语言来说，会增加 gc 压力，同时也容易造成内存碎片。

## 如何判断程序是否发生了内存逃逸？

build时添加`-gcflags=-m`

选项可分析内存逃逸情况,比如输出`./main.go:3:6: moved to heap: x`

表示局部变量x逃逸到了堆上。

## 逃逸场景有哪些？
- **指针逃逸：**向 `channel` 发送指针数据。因为在编译时，不知道channel中的数据会被哪个 goroutine 接收，因此编译器没法知道变量什么时候才会被释放，因此只能放入堆中。

```go
package main

func main() {
    ch := make(chan int, 1)
    x := 5
    ch <- x    // x不发生逃逸，因为只是复制的值
    ch1 := make(chan *int, 1)
    y := 5
    py := &y
    ch1 <- py  // y逃逸，因为y地址传入了chan中，编译时无法确定什么时候会被接收，所以也无法在函数返回后回收y
}
```
- **栈空间不足逃逸：**当栈空间不足以存放当前对象时或无法判断当前切片长度时会将对象分配到堆中。

```go
package main

func main() {
    s := make([]int, 10000, 10000)
    for index, _ := range s {
        s[index] = index
    }
}
```
- **动态类型逃逸：**在 interface 类型上调用方法时会把interface变量使用堆分配， 因为方法的真正实现只能在运行时知道。

```go
package main

type foo interface {
    fooFunc()
    }

type foo1 struct{}

func (f1 foo1) fooFunc() {}

func main() {
    var f foo
    f = foo1{}
    f.fooFunc()   // 调用方法时，f发生逃逸，因为方法是动态分配的
}
```
- **闭包引用对象逃逸：**局部变量在函数调用结束后还被其他地方使用，比如函数返回局部变量指针或闭包中引用包外的值。因为变量的生命周期可能会超过函数周期，因此只能放入堆中。

```go
package main

func Foo () func (){
    x := 5			// x发生逃逸，因为在Foo调用完成后，被闭包函数用到，还不能回收，只能放到堆上存放
    return func () {
        x += 1
        }
        }

func main() {
    inner := Foo()
    inner()
}
```
## 如何避免内存逃逸？

- 对于小型的数据，使用传值而不是传指针，避免内存逃逸。
- 避免使用长度不固定的slice切片，在编译期无法确定切片长度，只能将切片使用堆分配。
- interface调用方法会发生内存逃逸，在热点代码片段，谨慎使用。