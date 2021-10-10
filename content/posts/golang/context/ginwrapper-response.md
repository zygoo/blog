---
title: "ginWrapper v1.0: 统一响应格式"
date: 2021-10-06T10:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---

## 前言
* 经历过前（web, 客户端，嵌入式，etc）后端联调的同学都知道，指定规范的交互协议是非常重要的。如果没有一定的规范，协议的格式必然是五花八门的，前端的小伙伴需要适配各种消息结构，带来巨大的工作量
* 在后端开发的方法论中，少写重复的代码一直是我们追求的目标。该模块的初衷也是帮我们快速构造符合协议的数据结构，减少代码量 
* 在http请求中，最常用的格式为json。所以在的设计中，只考虑了json。后期有需要，会考虑更多的Content-Type
* 总结一下，我们目标是：1. 统一前后端数据交互格式 2. 抽象出公共代码，快速构造响应数据

## 响应结构体

```json
{
  "Result": {
    "Code": 0,
    "Message":"success"
  },
  "Data": null
}
```
* code中传递的是业务码，0为成功，1为失败，还可以声明其他业务码，有兴趣的同学可以参照代码
* message为业务响应信息
* data用于传递业务数据
* 前端的同学可以按照这个数据结构完成数据的解析

## request.go
```go
type GinContext struct {
	*gin.Context
}

func NewGinContext(c *gin.Context) *GinContext {
	return &GinContext{
		Context: c,
	}
}

type GinContextFunc func(ctx *GinContext)

func WrapGinContext(handler GinContextFunc) gin.HandlerFunc {
	return func(ctx *gin.Context) {
		handler(NewGinContext(ctx))
	}
}
```
## response.go
```go
type ResponseInfo struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

type Response struct {
	Result ResponseInfo `json:"result"`
	Data   interface{}  `json:"data"`
}

const (
	CodeSuccess = 0
	CodeFailure = 1

	CodeBindError        = 10
	CodeRecordNotFound   = 11
	CodePermissionDenied = 12
)

var ResponseMap = map[int]ResponseInfo{
	CodeSuccess:   {http.StatusOK, http.StatusText(http.StatusOK)},
	CodeFailure:   {http.StatusInternalServerError, http.StatusText(http.StatusInternalServerError)},
	CodeBindError: {http.StatusInternalServerError, "params bind error"},
}

// SendResponse code为可变参数，code[0]为business code, code[1]为http code
func (c *GinContext) SendResponse(data interface{}, codes ...int) {
	httpCode := http.StatusOK
	response, exist := ResponseMap[codes[0]]
	if !exist {
		response = ResponseMap[CodeFailure]
	}

	result := ResponseInfo{codes[0], response.Message}
	if len(codes) > 1 {
		httpCode = codes[1]
	} else if result.Code <= 20 {
		httpCode = result.Code
	}
	c.JSON(httpCode, Response{result, data})
}

func (c *GinContext) OK(data interface{}) {
	c.SendResponse(data, CodeSuccess)
}

func (c *GinContext) BindFailure() {
	c.SendResponse(nil, CodeBindError)
}

func (c *GinContext) Failure(err error) {
	c.JSON(http.StatusInternalServerError, Response{
		Result: ResponseInfo{
			Code:    CodeFailure,
			Message: err.Error(),
		},
		Data: nil,
	})
}
```

## 举个栗子
```go
package main

import (
	"ginWrapper"
	"github.com/gin-gonic/gin"
	"github.com/sirupsen/logrus"
)

func main() {
	r := gin.Default()
	r.Use(ginWrapper.APILogger(ginWrapper.GinLog{})) // 在注册路由之前引入中间件
	r.POST("/health", ginWrapper.WrapGinContext(healthHandler))
}

func healthHandler(ctx *GinContext) {
	ctx.OK(nil)
}
```
有需要完整代码的同学可以参考[github](https://github.com/zygoo/ginWrapper/blob/master/request.go)