---

layout: post
title: Gin框架入门02--HTTP常用请求方法示例
date: 2018-08-22
tags: http,http.request
---

## HTTP

HTTP协议最常用的方法是Get方法和Post方法，任何一个了解过Web开发的程序员，都耳熟能详，那么还有哪些方法是HTTP协议支持的呢？

- GET     请求获取Request-URI所标识的资源 

- POST    在Request-URI所标识的资源后附加新的数据 

- HEAD    请求获取由Request-URI所标识的资源的响应消息报头 

- PUT     请求服务器存储一个资源，并用Request-URI作为其标识 

- DELETE  请求服务器删除Request-URI所标识的资源 

- OPTIONS 请求查询服务器的性能，或者查询与资源相关的选项和需求

## RESTful

架构中使用了四种HTTP方法作为对资源(URI)的操作

| 方法名称 |             含义              |
| :------: | :---------------------------: |
|   GET    |           获取资源            |
|   POST   | 新建资源（也可以用于更新资源) |
|   PUT    |           更新资源            |
|  DELETE  |         用来删除资源          |


使用Gin框架实现响应常见HTTP方法的示例
```
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
	"io/ioutil"
	"fmt"
)

func main() {

	r := gin.Default()
	r.GET("/get",HandleGet)
	r.POST("/post",HandlePost)
	r.PUT("/put",HandlePut)
	r.DELETE("/delete",HandleDelete)
	r.PATCH("/patch",HandlePatch)
	r.HEAD("/head",HandleHead)
	r.OPTIONS("/options",HandleOptions)
	r.Run(":9000")
	
}

func HandleGet(c *gin.Context)  {
	c.JSON(200,gin.H{
		"receive":"65536",
	})

}

func HandlePut( C *gin.Context)  {
	body,_ := ioutil.ReadAll(C.Request.Body)
	C.String(http.StatusOK,"Put,%s",body)

}

func HandleDelete( C *gin.Context)  {
	body,_ := ioutil.ReadAll(C.Request.Body)
	C.String(http.StatusOK,"Delete,%s",body)

}

func HandleOptions( C *gin.Context)  {
	body,_ := ioutil.ReadAll(C.Request.Body)
	C.String(http.StatusOK,"Options,%s",body)

}

func HandlePatch( C *gin.Context)  {
	body,_ := ioutil.ReadAll(C.Request.Body)
	C.String(http.StatusOK,"patch,%s",body)

}

func HandlePost( C *gin.Context)  {
	body,_ := ioutil.ReadAll(C.Request.Body)
	C.String(http.StatusOK,"post,%s",body)

}

func HandleHead( C *gin.Context)  {
    // http head only response  header
	fmt.Printf("head http \r\n")

}
```
其中handlefunc函数分别用来响应Get/Post/Head/Put/Delete/Options方法

如下对post函数进行的测试

![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/74809242.jpg)


![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/88408382.jpg)



