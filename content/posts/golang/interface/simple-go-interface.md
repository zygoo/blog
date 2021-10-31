---
title: "golang中如何使用接口？"
date: 2021-10-29T10:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---
## 前言
刚开始的时候，看了100遍关于接口的基础知识，我还是不会用，还是不理解。

后面看了一些代码，已经大概可以理解接口的作用，但是我还是写不出优雅的代码。

对于OOP的思想完全是处于一种混沌的状态，go语言到底是如何表达这种思想的？接口的底层是怎么实现的？脑子里面有太多的疑问，所以把它们整理一下，希望有新的收获。

## 面向对象编程（OOP）
众所周知，面向对象编程（OOP）有三个特征：
* 封装，struct
* 继承，struct
* 多态，interface

在go语言中，封装和继承是通过struct来实现的，而多态则是通过接口（interface）来实现的，这篇blog重点来讲一讲interface的基础知识。

## 什么是接口？
其实接口的思想在计算机中很常见，那就是调用和实现解耦开来：
* 底层只管实现，不需要关心到底是谁来用
* 上层只管使用，不用管底层是怎么实现的，是谁实现的

比如说print一个hello world，最终的接口就是在显示器能看到这些字符。用户只关心用了一个print功能，而最终是哪个显示器，是哪个硬件，用户是不会关心的。而对于操作系统，它实现了打印功能，它也不会关心具体是谁来用了这个功能。

在go语言中，接口有两种含义：
* 方法的集合
* 数据类型

## 声明一个接口
```go
//声明一个接口
type Human interface{
  Say()
}
//定义两个类，这两个类分别实现了 Human 接口的 Say 方法
type women struct {
}
type man struct {
}
func (w *women) Say() {
    fmt.Println("I'm a women")
}
func(m *man) Say() {
    fmt.Println("I'm a man")
}
func main() {
    w := new(women)
    w.Say()
    m := new(man)
    m.Say()
}
```
如果一个具体类型实现了某个接口的所有方法，我们则称该具体类型实现了该接口。需要注意的是，必须是所有的方法。

## 接口类型
对于空接口来说，任何具体类型都实现了空接口，初学者很容易对空接口产生疑惑，举个例子：
```go
func Say(s interface{}) {
    // ...
}
```
思考一下，在Say函数内部，s属于什么类型？对于初学者来说，容易认为s属于任意类型。其实s属于接口类型。s不是任意类型，却可以转换为任意类型。

为什么呢？因为当我们往Say方法传入值的时候，go runtime会自动进行类型转换，将该值转换为接口类型的值。所有值在运行时都只会有一个类型，s的静态类型就是接口类型，即是interface{}。

对于像go这种静态类型的语言，类型只是编译时候的概念。那go是如何实现接口值动态转换为任意类型值的呢？

在go语言中，接口值有两部分组成，一个是指向该接口的具体类型的指针和另外一个指向该具体类型真实数据的指针。可以从runtime的源码查看interface的定义：
```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```
明白数据存储结构，我们可以避免一些坑，例如下面的代码是有错误的：
```go
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"pique", "zack", "paul"}
    PrintAll(names)
}
```
编译会报错：cannot use names (type []string) as type []interface {} in argument to PrintAll，因为printAll的入参是一个接口类型，我们不能把string类型的值直接传入，再传入之前需要进行转换，或者printAll内部函数实现类型断言。下面是正确的写法：
```go
func main() {
    names := []string{"pique", "zack", "paul"}
    vals := make([]interface{}, len(names))
    for i, v := range names {
        vals[i] = v
    }
    PrintAll(vals)
}
```
## 指针或值接收者的区别
指针接收者和值接受者的区别，是核心概念，我们再来理解一次。

我们都知道，在go语言中所有的数据都是值传递。在实现接口方法中，如果全部使用指针接收者，或者值接收者，会很好理解。那如果是两者兼存呢？
```go
package main

import "fmt"

type Human interface {
    Say()
}

type Man struct {
}

type Woman struct {
}

func (m Man) Say() {
    fmt.Println("I'm a man")
}

func (w *Woman) Say() {
    fmt.Println("I'm a woman")
}

func main() {
    humans := []Human{Man{}, Woman{}}
    for _, human := range humans {
        human.Say()
    }
}
```
上面的代码会报错：cannot use Woman literal (type Woman) as type Human in array or slice literal: Woman does not implement Human (Say method has pointer receiver)，提示Woman没有实现Human的接口。
这是因为Woman实现Human接口时，定义的是指针接收者。但是我们在main方法中传入的一个woman的结构体，并不是一个指针，因此报错了。如果我们将main函数略微改变一下：
```go
func main() {
    humans := []Human{&Man{}, &Woman{}}
    for _, human := range humans {
        human.Say()
    }
}
```
注意到在main方法中分别传入了Man和Woman的指针，编译照常通过了。Man明明实现的是Human的值接收者，并不是指针接收者，为什么没有报错？

原因就在于go语言中都是值传递，尽管传入的是Man指针，但是通过该指针我们可以找到其对应的值。go语言隐式地帮我们做了类型的转换，我们需要记住的就是：
* go语言中的指针类型可以获得其关联的任意值类型，但是反过来却不行
* 道理很简单，一个具体的值可能有无数个指针指向它，但是一个指针只会指向一个具体的值

## 参考
* [如何优雅的使用Go接口?](https://zhuanlan.zhihu.com/p/63219494)
* [GoLang：OOP(面向对象)？坑！](https://zhuanlan.zhihu.com/p/157786743)
