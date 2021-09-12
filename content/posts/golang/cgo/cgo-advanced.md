---
title: "Cgo的进阶使用"
date: 2021-09-06T10:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---

cgo在使用c/c++语言的时候一般有三种方式：
* 直接使用源码
* 动态链接库
* 静态链接库

直接使用源码就是在import "c"之前的注释部分包含c代码，或者在当前的包中包含c/c++源文件。
链接静态库和动态库的方式比较类似，都是通过LDFLAGS中指定要链接的库。在项目中，动态链接库用得要更多一些，
所以我们重点来介绍一下如何去使用动态链接库，以及其中会遇到的坑。

## LDFLAGS和CFLAGS

```go
package controller

/*
*cgo LDFLAGS: -L ${SRCDIR}/bupt/msg -Wl,-rpath ${SRCDIR}/bupt/msg -lactive
*cgo CFLAGS: -I ${SRCDIR}/bupt/msg
#include <stdint.h>
#include <stdlib.h>
#include <active_server.h>
 */
import "C"
import "unsafe"

func main() {
	outSize := (*C.uint)(unsafe.Pointer(&make([]byte,4)[0]))
	if C.active_msg_handler(outSize) != 0 {
		panic("error")
    }
}
```
先看一下LDFLAGS中参数使用，${SRCDIR}指的是当前文件夹所在的路径，${SRCDIR}/bupt/msg指的是动态链接库所在的路径。
项目中的动态链接库文件名为active.so, 所以很明显-lactive的意义就是去链接active.so这个动态链接库。

-L和-Wl, -rpath后面都跟了同样的参数，这有什么区别呢？-L这个指令用于指定动态链接库的目录。编译时指定目录，
只是在程序链接成可执行文件时使用，但是在运行的时候，并不会去指定的目录寻找链接库，这时候就要使用-Wl,-rpath指令。
这块涉及到链接器和动态链接器的区别，其中奥义，还需要细细品味，就留下一个TODO项吧。

CFLAGS的使用方法很简单，就是加载头文件的所有路径，然后将头文件include进来。接下来我们就可以使用c库中的接口方法了，传入参数后，
处理返回的结果，便大功告成。

## 相对路径和绝对路径

从上文的例子中，我们都使用了绝对路径，那可以使用相对路径么？在CFLAGS中，是可以使用相对路径的。
但是因为一些历史遗留原因，LDFLAGS不支持相对路径，我们必须为链接库指定路径。

## 一个项目中同时链接多个动态库

在这种场景下，需要注意的是，每个动态链接库的命名都应该是不一样的。做项目的时候，多个动态链接库命名都是一样的，导致运行时出现了问题。

## 访问c中的struct
cgo中传入参数的时候，我们经常需要访问struct，将结构体指针作为参数传递。使用的方式为：C.Struct_<struct的名字>，注意要记得释放内存。
```go
cData := (*C.struct__log_export_header__)(C.Bytes(data)) // data作为入参
defer C.Free(unsafe.Pointer(cData))
```

## cgo和c++
在前面的例子中，都是混编了c的代码，那c++怎么办呢？特别是暴露的接口函数，其入参是c++风格的时候，使用第三方组件swig的成本太高。下面分享一个小技巧：

```cgo
// gdal.h
#include<iostream>
#include<string>
int buptAdal(std::string img_path, float input_gsd, std::string out_put_json)
int buptSeedling(std::string img_path, std::string tiff_path, std::string out_json_path, float *progress)
```
在这个头文件的基础上，增加一层wrap，便可以支持cgo调用时候传入c风格的参数：
```cgo
// cwarp.h
#ifdef _CWRAP_H__
#define _CWRAP_H__

#ifdef _cplusplus
extern "C"
#endif
int WrapbuptAdal(const char* img_path, float input_gsd, const char* out_put_json)
int WrapbuptSeedling(const char* img_path, const char* tiff_path, const char* out_json_path, float *progress)
```

```cgo
// cwrap.cpp
#include "gdal.h"
extern "C" {
   #include "cwrap.h"
}
int WrapbuptAdal(const char* img_path, float input_gsd, const char* out_put_json) {
    std::string src = img_path
    std::string dst = out_put_json
    return buptGdal(src, input_gsd, dst)
}

int WrapbuptSeedling(const char* img_path, const char* out_json_path, float *progress) {
    std::string src = img_path
    std::string dst = out_put_json
    return WrapbuptSeedling(src, ht_path, dst, progress)
}
```

## 参考
* [CGO简明教程](https://juejin.im/entry/5b5089e3e51d45198565aaa2)
* [CGO官方指南](https://golang.org/cmd/cgo/#hdr-Go_references_to_C)
* [cgo使用需要注意的一些问题](https://www.bandari.net/blog/24)
* [还是chai大佬的go语言圣经](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-10-link.html)

