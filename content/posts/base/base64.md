---
title: "base64编码后出现换行符？"
date: 2021-09-07T10:18:06+08:00
draft: false
toc: true
images:
tags:
- os
---

在开发的过程中，遇到一个很神奇的问题，就是对一段二进制文件进行base64 encode之后，竟然出现了换行符。趁着这个机会，对base64做一下总结。

## base64基本原理
Base64的加密方式是将三个8位的字节转化为四个6位的字节，不足8位的高位补0，3 * 8 = 4 * 6 ，所以经过base6加密的字符串大约比要比未加密的字符串要大三分之一。

因为Base64有效位只有六位，所以最大能表示的字符就为2的6次方64。大小写的字母26*2 + 10个数字 + 两个特殊符号 + /, 一共64个字符。

举个例子，我们对"ace"进行加密：
```cgo
ace转化为二进制为：‭01100001‬ ‭01100011‬ ‭01100101‬
转化为base64的四字节六位：011000 01‬‭0110 0011‬01 100101‬
那因为计算机是一字节八位的存数，所以高位补00后变为：00011000 0001‬‭0110 000011‬01 00100101‬
转化为十进制：24 22 13 37
```
通过查表，我们得到最终结果：YWNl

## 在base64结尾加等号

例子中为了方便演示我只取了三个字节的字符串，实际中会存在字节数量不是3倍数的情况。将原ASCII码(每个字符8位)转换为转换为base编码字符，会存在两种情况：
1. 剩余一个ASCII码，余数为1，这时候添加两个等号，因为一个8位的字符至少可以变成两个6位的base64字符，可以补满4个base64字节
2. 当余数为2时，只需要添加一个等号，因为两8位的字符至少可以变成三个6位的base64字符

## url encode and decode
标准的Base64并不适合直接放在URL里传输，因为URL编码器会把标准Base64中的“/”和“+”字符变为形如“%XX”的形式，而这些“%”号在存入数据库时还需要再进行转换，因为ANSI SQL中已将“%”号用作通配符。

为解决此问题，可采用一种用于URL的改进Base64编码，
1. 在标准Base64中的“+”和“/”分别改成了“-”和“_”
2. 在末尾去掉填充的=号，去掉=号的encode称为URL-Safe-Base64, 如果没有去掉"="， 就只是URL-Base64而已
3. golang是强类型语言，URL-Safe-Base64和URL-Base64的使用是严格区分的。比如像ruby这种弱类型语言，其底层库会兼容这两种URL-encode的方式

```go
package main

import "encoding/base64"

func Base64UrlSafeEncode(s string) []byte {
	data, err := base64.RawURLEncoding.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return data
}

func Base64UrlSafeEncode(src []byte) string {
	data := base64.RawURLEncoding.EncodeToString(src)
	return data
}

func Base64Encode(s string) []byte {
	data, err := base64.StdEncoding.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return data
}

func Base64Decode(src []byte) string {
	data := base64.RawURLEncoding.EncodeToString(src)
	return data
}
```

## 编码后出现换行符
刚开始出现这个问题，也是比较懵逼，查询一些资料，得出的初步结论是：根据RPC822规定，base64 encode每编码76个字符，需要加上一个换行符，这些回答者都是java的开发者，说明可能部分base64编码的java库还按照这个标准执行。

ruby和golang并没有执行RPC822标准，ruby中每编码60个字符就要加上一个换行符，而golang的标准库干脆就不加换行符。

那服务间调用岂不是会出错？其实不会，这些标准库都会对换行符进行预处理，这个结论基本符合我们现在服务间数据交互的情况。
