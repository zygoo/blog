---
title: "golang中的OOP思想"
date: 2021-11-01T08:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---

## 前言
OOP是一个非常宏观的话题，我们今天只是希望可以有一些简单的总结，通过与其他语言的比较，看看golang中是如何表达OOP思想。

## c++的OOP: class类和interface接口
c++， java的OOP(object orientation programming，面向对象编程)实现，一个必不可少的条件就是class或者interface接口的数据结构，这种数据结构拥有了OOP最核心的三个特性：
* 封装，由类内的可见性来保障，语法关键字有public，protected，private
* 继承，也就是我们常说的父类和子类，也可以被成为基类（Base Class）和派生类（Derived Class）。C++的底层有重载，重写甚至虚函数等语法来制定或者说约束整套继承的规则
* 多态，是父类/基类或者说是接口类，用子类/派生类进行实例化之后呈现出子类/派生类的行为特征，在c++中，底层是通过虚函数的语法机制来做到的

c++中只有类的概念，语法关键字是class或者struct，这两者实际作用是一致的：
```go
/* 
* struct结构，默认为public
*/
struct Person {
    // 类成员/类属性
    std::string name;
    int age;
    // 类函数/类方法
    Person(std::string _name, int _age) : name(_name), age(_age) {};
    void Show() { 
        printf("name=%s, age=%d\n", name.c_str(), age); 
    };
};

/*
* class结构，默认为private
 */
class Animal {
// 类成员/类属性
     std::string name;
     int age;
public:
// 类函数/类方法
     Animal(std::string _name, int _age) : name(_name), age(_age) {};
     void Show() {
        printf("name=%s, age=%d\n", name.c_str(), age);
     }
};
```
python其实也只有类的概念，语法的关键字是class：
```go
class Animal:
	def __init__(self, name, _age):
		self.name = _name
		self.age = _age
	def show(self):
		print("name=%s, age=%d\n", name.c_str(), age)
```
而java则同时有类和接口的概念，语法关键字分别为class和interface:
```go
/* 
* class 类结构
*/
public class Animal {
	string name
	int age
	
	public Animal(String _name, int _age) {
		name = _name
		age = _age
    }
    
    public void Show() {
        System.out.println("name=" + name + ", age=" + age);
    }
}

/*
 * interface 接口结构
 */
public interface Animal {
	public void Show(); 
	public void Eat();
}
```

对golang来说，虽然它有关键字struct和interface，但是和c++的struct语法不完全相同：
* golang支持在struct内声明"类成员"而不支持"类方法"
* golang支持在struct外部声明自己的类方法

## Golang怎么做OOP?

习惯了c++, java和python用语法关键字"class"修饰类名，在类内定义类成员和类方法。外部可以通过实例化类的对象"."或者"->"获得它的成员或者方法。那golang是怎么做到的类的数据结构的呢？

### 封装
golang中没有public, protected和private语法关键字，它是通过大小写字母来控制可见性的:
* 如果常量，变量，类型，接口，结构，函数等名称是大写字母则代表能被其他包访问，其作用相当于public
* 以非大写开头则不能被其他包访问，其作用相当于private, 当然，在同一个包内是可以访问的

### 继承
golang中没有像":", "implements", "extends"等关键字，但是也可以做类似继承的功能。具体的做法如下，"子类"的字段嵌入"父类"即可, 代码如下:
```go
type Base struct{
	one string
}

type Derived struct{
	Base
}
```
实际上，上述的做法并不是真正的继承，而是匿名组合，本质还是一种组合的形式。只不过在调用的时候，可以直接通过实例化变量访问到父类的成员变量或者成员方法。

### 多态
golang的多态依靠interface实现的，interface是一种duck type，被具体的实例类型实例化之后就可以表现出实例类型的行为特征，以下代码为：
```go
package main

import "fmt"

type Animal interface {
	Show()
}

type Cat struct {
	name string
	age int
}

func(this *Cat) Show() {
	fmt.Printf("Cat: name=%s, age=%d\n", this.name, this.age)
}

type Dog struct {
	name string
	age int
}

func(this *Dog)Show() {
	fmt.Printf("Cat: name=%s, age=%d\n", this.name, this.age)
}

func main() {
	var a1, a2 Animal
	a1 = &Cat{
		name: "zy",
		age: 2,
    }
    
    a2 = &Dog{
    	name: "hmz",
    	age: 1,
    }
    a1.Show()
    a2.Show()
}
```

## 匿名组合中的静态绑定的问题
```go
package main
import "fmt"

type BaseBird struct {
    age int
}

func (this *BaseBird) Call()  {
    this.Add()
}
func (this *BaseBird)Add()  {
    fmt.Printf("before add: age=%d\n", this.age)
    this.age = this.age + 1
    fmt.Printf("after add: age=%d\n", this.age)
}

type DerivedBird struct {
    BaseBird
}
func (this *DerivedBird) Add()  {
    fmt.Printf("before add: age=%d\n", this.age)
    this.age = this.age + 2
    fmt.Printf("after add: age=%d\n", this.age)
}

func main() {
    var b1 BaseBird
    var b2 DerivedBird

    b1 = BaseBird{age: 1}
    b1.Cal()

    b2 = DerivedBird{BaseBird{1}}
    b2.Cal()
}

//before add: age=1
//after add: age=2

//before add: age=1
//after add: age=2
```
我们来尝试理解一下这个运行结果，其实也很好理解。在golang中，所谓"继承"的做法，实际上匿名组合。golang的组合就是静态绑定，或者说golang所有的struct方法都是静态绑定的。在上述的例子中，"父类"BaseBird的方法Call()调用的本方法Add()。
虽然在所谓"子类"DerivedBird中实现了Add()，但是对于父类的call来说，在编译时期，就已经确定了他访问的就是自己的Add()。

那为什么C++中可以做到通过this指针访问到子类的方法呢？那就是利用虚函数或者说是虚函数表，要知道，即使在C++的继承中，如果被调用的函数没有被virtual语法修饰为虚函数的话，最终访问的也还是父类的方法，如下面的例子：

```go
class BaseBird {
public:
    int age;
    BaseBird(int _age) : age(_age) {};
    ~BaseBird() = default;
    void Cal() { this->Add(); };
    // virtual void Add() { // 被调用的方法是否为虚函数，结果完全不一样
    void Add() {
        printf("before add, age=%d\n", age);
        age += 1;
        printf("after add, age=%d\n", age);
    };
};
class DerivedBird : public BaseBird {
public:
    DerivedBird(int _age) : BaseBird(_age) {};
    ~DerivedBird() = default;
    void Add() {
        printf("before add, age=%d\n", age);
        age += 2;
        printf("after add, age=%d\n", age);
    };
};

int main()
{
    DerivedBird d(1);
    d.Cal();
    BaseBird b(1);
    b.Cal();
    return 0;
}
```
如果Add()被声明为虚函数，那么结果是：
```go
before add age=1
afer add age=3

before add age=1
after add age=2
```

## 解决静态绑定的问题
假设我们的真实业务场景中，确实需要这种设计：公共逻辑（父类）中有一部分需要执行到不同的具体类逻辑，我们看看实现的方法：
```go
package main

import "fmt"

type Bird interface {
    Add()
}
func Cal(bird Bird)  {
    bird.Add()
}

type BaseBird struct {
    age int
}
func (this *BaseBird)Add()  {
    fmt.Printf("before add: age=%d\n", this.age)
    this.age = this.age + 1
    fmt.Printf("after add: age=%d\n", this.age)
}

type DerivedBird struct {
    BaseBird
}
func (this *DerivedBird) Add()  {
    fmt.Printf("before add: age=%d\n", this.age)
    this.age = this.age + 2
    fmt.Printf("after add: age=%d\n", this.age)
}

func main() {
    var b1, b2 Bird
    b1 = &BaseBird{age:1}
    b2 = &DerivedBird{BaseBird{age:1}}
    Cal(b1)
    Cal(b2)
}
```
运行得到的结果：
```go
before add: age=1
after add: age=2

before add: age=1
after add: age=3
```
使用golang做OOP，需要抛弃c++和java的思维体系，使用新的语法+思维方式来开发。

## 参考
* [GoLang：OOP(面向对象)？坑！](https://zhuanlan.zhihu.com/p/157786743)





