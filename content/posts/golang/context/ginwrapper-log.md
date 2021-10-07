---
title: "ginWrapper v1.0: 日志中间件"
date: 2021-10-05T10:18:06+08:00
draft: false
toc: true
images:
tags:
- golang
---
## 前言
日志中间件是一个特别重要的模块，特别是在http的请求中，记录一次完整的request数据可以帮助我们快速定位问题。
在以前的一些工作经历中，接触过两个log中间件，感觉它们存在一些缺陷：
1. 请求日志和响应日志分开打印，不利于数据的聚合，影响问题分析的速度
2. 将响应信息塞在gin的context上下文中，对代码有侵入性的改造

在使用gin的过程中，希望总结出一个较优的实践，所以写下了这篇文章。

## GinLog结构体定义
```go
type GinLog struct {
     FilterParams []string   // 需要过滤的参数，在日志中不做展示
     EncryptParam []string   // 参数加密展示
     SkipPaths    []string   // 跳过路径
}
```
首先定义了一个GinLog的结构体，里面了包含一些数据处理，过滤规则。在1.0版本中，暂时没有用到，后面可以补充。

## 重写接口方法
```go
func (w bodyLogWriter) Write(b []byte) (int, error) {
	w.body.Write(b)
	return w.ResponseWriter.Write(b)
}

func (w bodyLogWriter) WriteString(s string) (int, error) {
    w.body.WriteString(s)
    return w.ResponseWriter.WriteString(s)
}
```

## 拼装参数信息
```go
func getRequestInfo(c *gin.Context) gin.H {
	params := map[string]interface{}{}
	if c.Request.Method != "GET" {
		switch c.ContentType() {
		case "application/json":
			data, err := ioutil.ReadAll(c.Request.Body)
			if err != nil {
				defer c.Request.Body.Close()
			}
			c.Request.Body = ioutil.NopCloser(bytes.NewReader(data))
			m := map[string]interface{}{}
			err = json.Unmarshal(data, &m)
			if err != nil {
				params["raw"] = string(data)
			} else {
				for k, v := range m {
					params[k] = fmt.Sprint(v)
				}
			}
		case "application/x-www-form-urlencoded":
			_ = c.Request.ParseForm()
			if c.Request.Form != nil {
				for k, v := range c.Request.Form {
					params[k] = v
				}
			}
		}
	}
	return params
}
```
* 不同Content-Type, 获取参数信息的方式是不一样的，需要区别对待
* 针对GET方法，目前采用的方式是打印raw query, 没有必须再单独获取参数信息
## 打印日志
```go
func APILogger(ginLog GinLog) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		bodyLogWriter := &bodyLogWriter{
			body:           bytes.NewBufferString(""),
			ResponseWriter: c.Writer,
		}
		c.Writer = bodyLogWriter
		c.Next()
		end := time.Now()
		logger.WithFields(logger.Fields{
			"p":              bodyLogWriter.body.String(),
			"q":              getRequestInfo,
			"client_ip":      c.ClientIP(),
			"status":         c.Writer.Status(),
			"method":         c.Request.Method,
			"path":           c.FullPath(),
			"raw_query":      c.Request.URL.Query(),
			"user_agent":     c.Request.UserAgent(),
			"end":            end.Format("2006/01/02-15:04:05"),
			"duration":       end.Sub(start).Seconds(),
			"content_type":   c.ContentType(),
			"content_length": c.Request.ContentLength,
			"x-forward-for":  c.GetHeader("X-Forward-For"),
		}).Info("APILogger")
	}
}
```
需要使用完整代码的同学，请查看[github](https://github.com/zygoo/ginWrapper/blob/master/log.go)。

## 举个栗子

```go
package main

import (
	"ginWrapper"
	"github.com/gin-gonic/gin"
	"github.com/sirupsen/logrus"
	"os"
)

func main() {
	logrus.SetFormatter(&logrus.JSONFormatter{})
	logrus.SetOutput(os.Stdout)
	r := gin.Default()
	r.Use(ginWrapper.APILogger(ginWrapper.GinLog{})) // 在注册路由之前引入中间件
}
```

## 参考
* [gin-日志使用](https://www.kancloud.cn/lhj0702/sockstack_gin/1805368)
* [go-gin-api 路由中间件 - 日志记录](https://cloud.tencent.com/developer/article/1498038)
* [Gin中间件之日志模块](https://www.cnblogs.com/lxmhhy/p/13518211.html)


