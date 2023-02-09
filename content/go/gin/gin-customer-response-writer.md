---
title: "Gin 自定义response Writer"
tags: 
- go
- gin
- responseWriter
weight: 100
description: "Gin 自定义response Writer"
---

## 1.问题
开发日志中间件的时候，如何打印返回数据，并且不影响上下文继续执行呢？
    
## 2.log middleware
```go
func LOGS() gin.HandlerFunc {
	return func(ctx *gin.Context) {
            // 自定义ResponseWriter,将ctx中Writer进行包装一下
            crw := utils.NewCustomerResponseWriter(ctx)
            ctx.Writer = crw

            // 请求参数
            buffer, _ := io.ReadAll(ctx.Request.Body)
            ctx.Request.Body.Close()
            ctx.Request.Body = io.NopCloser(bytes.NewBuffer(buffer))
            req := struct {
                Method string `json:"method"`
                Url    string `json:"url"`
                Body   string `json:"body"`
            }{
                Method: ctx.Request.Method,
                Url:    ctx.Request.URL.Path,
                Body:   string(buffer),
            }
            logs, _ := json.Marshal(req)
            logger.Info(string(logs))

            ctx.Next()
            // 响应参数
            logger.Info(crw.Body.String())
	}
}

```

## 3. Customer Response Writer
```go
package utils

import (
	"bytes"

	"github.com/gin-gonic/gin"
)

type CustomerResponseWriter struct {
	Body *bytes.Buffer
	gin.ResponseWriter
}

func (c CustomerResponseWriter) Write(b []byte) (int, error) {
	c.Body.Write(b)
	return c.ResponseWriter.Write(b)
}

func (c CustomerResponseWriter) WriteString(s string) (int, error) {
	c.Body.WriteString(s)
	return c.ResponseWriter.WriteString(s)
}

func NewCustomerResponseWriter(c *gin.Context) *CustomerResponseWriter {
	return &CustomerResponseWriter{
		Body:           bytes.NewBufferString(""),
		ResponseWriter: c.Writer,
	}
}
```