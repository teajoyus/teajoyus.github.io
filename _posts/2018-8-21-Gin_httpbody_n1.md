---
layout: post
title: Gin框架入门01--Http请求Body和Header的获取
date: 2018-08-21
tags: http,http.request
---

## 前言

Gin是一个轻量级的Web开发框架，与重量级代表Beego的区别在于，Gin专注于Web 中Http协议处理，数据、表格解析，路由与中间件等，而Beego相对大而全，完整MVC模式，不仅包含了Web协议处理的内容，也包含了数据库的CURD(Beego光数据库的驱动都有三种 mysql/Sqlite/Postgres)

首先对于Gin框架的安装 简要

> ```
> go get -u github.com/gin-gonic/gin
> ```

## Gin官方Demo

Gin教程的官方入门程序 example.go

```
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```

运行 example.go，可以通过浏览器来访问   http://localhost:8080/ping

如果一切运行正常，那么可以看到浏览器中有Json返回 



## Gin获取Http请求头Header和Body

　　一个HTTP报文由3部分组成，分别是:

　　(1)、起始行(start line)

　　(2)、首部(header)

　　(3)、主体(body)

本次主要关注的是发起请求的报文，使用Postman做为测试工具，发起Http请求

```
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"io/ioutil"
)

func main() {

	fmt.Printf("launch Gin")

	r := gin.Default()
	r.GET("/get",HandleGet)
	r.POST("/getall",HandleGetAllData)
	
	//如果使用浏览器调试，那么响应Get方法
	//r.GET("/getall",HandleGetAllData)
	r.Run(":9000")
	
}

func HandleGet(c *gin.Context)  {
	c.JSON(200,gin.H{
		"receive":"65536",
	})

}

func HandleGetAllData(c *gin.Context)  {
   //log.Print("handle log")
	body,_ := ioutil.ReadAll(c.Request.Body)
	fmt.Println("---body/--- \r\n "+string(body))

	fmt.Println("---header/--- \r\n")
	for k,v :=range c.Request.Header {
		fmt.Println(k,v)
	}
	//fmt.Println("header \r\n",c.Request.Header)

	c.JSON(200,gin.H{
		"receive":"1024",
	})

}

```

运行示例：

postman请求内容

> Method：post

> Body type: raw

> Body: username=123

![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-21/86828159.jpg)


其中红色框内为HTTP  Request Body

橙色框内为 HTTP  Request Header



![微信公众号](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/88408382.jpg)