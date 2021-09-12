---
title: "C？Go？Cgo!"
date: 2021-09-04T10:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---
## cgo的非权威入门指南
### 为什么要用cgo?
三十年的风云激荡，互联网诞生了众多的编程语言，但是c语言永远是守护金字塔底层的夯土。所谓cgo, 就是要将c和go联合起来，建立一个生态，下面是对cgo价值的一个总结：
* 虽然golang号称打遍天下所有的软件细分领域，但这是有局限性的，不能解决所有的问题，有时候c语言可以帮助我们更高效地解决问题
* 通过cgo可以继承c/c++将近半个世纪积累，将其生态收纳进来

### 最简单的cgo程序
cgo的程序一般都比较复杂，下面我们来构造一个最简单的cgo程序，首先要忽略一下cgo的特性，同时展示出cgo程序和纯go程序的差别：
```go
// hello.go
package main

import "C"

func main() {
    println("hello cgo")
}
```
代码通过import "C"语句来启动cgo特性，虽然没有任何cgo的相关函数，但是go build命令会在编译和链接阶段启动cgo编译器，这已经是一个完整cgo程序。

### 基于c标准库函数输出字符串
```go
// hello.go
package main

//#include <stdio.h>
import "C"

func main() {
    C.puts(C.CString("Hello, World\n"))
}
```
* 我们不仅仅通过import "C"语句启动cgo特性，同时包含c语言的<stdio.h>头文件，然后通过cgo包的C.CString函数将Go语言字符串转换为c语言字符串，最后调用cgo包的C.puts函数向标准输出窗口打印转换后的c字符串
* 这段代码中，没有释放使用C.CString创建的c语言字符串，会导致内存泄漏。对于小程序来说，这是没有问题的，因为程序退出之后操作系统会自动回收程序的所有资源

### 使用自己的c函数

```go
// hello.go
package main

/*
#include <stdio.h>

static void SayHello(const char* s) {
    puts(s);
}
*/
import "C"

func main() {
    C.SayHello(C.CString("Hello, World\n"))
}
```
我们定义一个名叫SayHello的c函数来实现打印，然后从Go语言环境中调用这个SayHello函数。

## cgo的参数传递
当涉及到跨语言调用的时候，参数的定义和转换都是一门学问。我们不仅仅需要对两种语言都有所了解，而且要注意资源的申请和释放，类型的转换规则等细节。

### c语言中指针变量声明
在c语言中，变量的地址往往都是编译系统自动分配的，对于用户来说，我们是不知道某个变量的具体位置的。所以我们定义一个指针变量的o, 把普通变量a地址直接送给指针变量p, 大概就是p = &a这样的写法。
方法1：定义时直接进行初始化赋值
```cgo
unsigned char a;
unsigned char *p = &a;
```
方法2：定义后再进行赋值
```cgo
unsigned char a;
unsigned char *p;
p = &a;
```
### 字符串指针
在c语言不像golang, ruby，有特定的字符串类型，通常字符串是放在一个数组中：
```cgo
#include<stdio.h>
#include<string.h>
void main() {
     char str[] = {80,81,82,83} // str[] = "PQRS"
     int len = strLen(str), i;
     for i = 0; i < len; i++ {
        printf("%c",str[i])
     }
     printf("\n")
     char *p = str
     for i = 0; i < len; i++ {
         printf("%c", *(p++))
     }
     printf("\n")
     for i = 0; i < len; i++ {
         printf("%c", *(str+i))
     } 
}
PQRS
PQRS
PQRS
```
在上面的示例代码中，字符串数组也可以被认为是一个指针，除来字符串之外，c语言还支持使用一个指针指向字符串的方式来表示字符串。

用字符串数组表示字符串，非要弄个指针来表示字符串呢？它们最根本的区别就是存储的区域不一样：
* 字符数组存储在全局数据区(data,bss)或者栈区(stack)，而以指针形式表示的字符串却存在常量区(text)
* 全局数据区或者栈区的字符串，可以被读写
* 而常量区的字符串，只能被读取

### cgo的基本参数类型
golang和c的基本数值类型转换对照表如下：
| c语言类型              | cgo类型          | go语言类型             |
| ---------------------- | ---------------------- | ---------------------- |
| char                   | C.char                 | byte                   |
| singed char            | C.schar                | int8                   |
| unsigned char          | C.uchar                | uint8                  |
| short                  | C.short                | int16                  |
| unsigned short         | C.ushort               | uint16                 |
| int                    | C.int                  | int32                  |
| unsigned int           | C.uint                 | uint32                 |
| long                   | C.long                 | int32                  |
| unsigned long          | C.ulong                | uint32                 |
| long long int          | C.longlong             | int64                  |
| unsigned long long int | C.ulonglong            | uint64                 |
| float                  | C.float                | float32                |
| double                 | C.double               | float64                |
| size_t                | C.size_t              | uint                   |

注意c中的整型，比如int，在标准中是没有定义具体字长的，一般默认认为是4字节，对应cgo类型中C.int则明确定义了字长是4，但golang中的int字长则是8，因此对应的golang类型不是int而是int32。为了避免误用，c代码最好使用C99标准的数值类型，对应的转换关系如下：
| c语言类型    | cgo类型 | go语言类型    |
| ------------- | ------------- | ------------- |
| int8_t       | C.int8_t     | int8          |
| uint8_t      | C.uint8_t    | uint8         |
| int16_t      | C.int16_t    | int16         |
| uint16_t     | C.uint16_t   | uint16        |
| int32_t      | C.int32_t    | int32         |
| uint32_t     | C.uint32_t   | uint32        |
| int64_t      | C.int64_t    | int64         |
| uint64_t     | C.uint64_t   | uint64        |

### unsafe.Pointer
我们在学习golang的时候，应该听过这么一句话，go的指针不支持指针类型的转换，但这是为什么呢？

golang作为一门静态语言，所有的变量都必须是标准变量，也就是说不同类型之间不能进行赋值，计算等跨类型操作。那么指针也对应着相应的类型，也在compile的静态类型检查范围内。同时，静态类型语言，也被称为强类型，也就是一旦被定义，就不能被改变，下面我们看一个错误示范：

```go
package main

import "fmt"

func main() {
	num := 5
	numPointer := &num
	flnum := (*float32)(numPointer)
	fmt.Println(flnum)
}
// command-line-arguments
// cannot convert numPointer (type int*) to type *float
```
在示例中，我们创建了一个num变量，值为5，类型为int。取其指针地址后，试图强制转换为*float32, 结果就是报错。

为了解决这个问题，我们就需要用到unsafe.Pointer, 其实golang很多底层库的都使用到了unsafe.Pointer, 它表示任意类型且可寻址的指针值，可以在不同的指针类型之间进行转换，
但是这个库为什么叫unsafe.Pointer，简单来说，就是不推荐使用，因为它是不安全的。

### 切片传递
由于底层数据结构的差异，不能直接将golang的切片的指针传给c函数调用，而是需要将存储切片数据内部缓冲区的首地址以及切片长度取出来传递：

```go
package main

/*
#include <stdint.h>

static void fill_255(char* buf, int32_t len) {
	int32_t i;
	for (i = 0; i < len; i++) {
		buf[i] = 255;
	}
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	b := make([]byte, 5)
	fmt.Println(b) // [0 0 0 0 0]
	C.fill_255((*C.char)(unsafe.Pointer(&b[0])), C.int32_t(len(b)))
	fmt.Println(b) // [255 255 255 255 255]
}
```
### 字符串传递
golang的字符串和c中的字符串在底层的内存模型也是不一样的，golang字符串并没有使用'\0'终止符标识字符串的结束，因此直接将golang字符串底层数据指针传递给c函数是不行的。

有一种方案类似于切片的传递，将字符串数据指针和长度传递给c函数后，c函数自行申请一段内存拷贝字符串然后加上终止符后再使用。更好的方案是使用标准库提供的C.CString()将golang的字符串转换为c字符串，再传递给c函数调用：
```go
package main

/*
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

static char* cat(char* str1, char* str2) {
	static char buf[256];
	strcpy(buf, str1);
	strcat(buf, str2);
	return buf;
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	str1, str2 := "hello", " world"
	// golang string -> c string
	cstr1, cstr2 := C.CString(str1), C.CString(str2)
	defer C.free(unsafe.Pointer(cstr1)) // must call
	defer C.free(unsafe.Pointer(cstr2))
	cstr3 := C.cat(cstr1, cstr2)
	// c string -> golang string
	str3 := C.GoString(cstr3)
	fmt.Println(str3) // "hello world"
}
```
需要注意的是，C.CString()返回的c字符串是在堆上新创建的并且不受gc的管理，使用完后需要自行调用C.free()释放，否则会造成内存泄露，而且这种内存泄露用pprof也定位不出来。

## 参考

* [chai大佬的go语言圣经](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-01-hello-cgo.html)
* [golang cgo 使用总结](http://litang.me/post/golang-cgo/)
* [Cgo中如何将 []byte 转换为 *char](https://segmentfault.com/q/1010000009515544)
* [CGO类型及常见类型转换](https://linkscue.com/2018/11/12/2018-11-12-cgo-types-covertion/)
* [slice内存模型](https://blog.csdn.net/tennysonsky/article/details/78789878)
* [有点不安全却又一亮的 Go unsafe.Pointer](https://www.jianshu.com/p/fb25dce48317)
* [C语言基础——字符串指针（指向字符串的指针）](https://blog.csdn.net/u013812502/article/details/81196367)
* [C语言指针变量的声明](http://c.biancheng.net/cpp/html/1925.html)


