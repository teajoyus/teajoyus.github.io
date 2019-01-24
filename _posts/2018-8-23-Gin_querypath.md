---

layout: post
title: Gin框架入门03--处理简单参数和表格
date: 2018-08-23
tags: http,http.request
---





```
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	fmt.Printf("launch")
	r := gin.Default()

	r.GET("/user/:name/*action", GetParamPath)

	r.GET("/welcome",GetQueryParmeters)

	r.POST("/form_post",Formpost)

	r.Run(":9000")
}

func GetParamPath(c *gin.Context)  {
	name := c.Param("name")
	action := c.Param("action")
	message := name + " is " + action
	c.String(http.StatusOK, message)

}

func GetQueryParmeters(c *gin.Context)  {
	firstname := c.DefaultQuery("firstname", "Guest")
	lastname := c.Query("lastname") // shortcut for c.Request.URL.Query().Get("lastname")
	c.String(http.StatusOK, "Hello %s %s", firstname, lastname)

}

func Formpost(c *gin.Context)  {
	message := c.PostForm("message")
	nick := c.DefaultPostForm("nick", "anonymous")

	c.JSON(200, gin.H{
		"status":  "posted",
		"message": message,
		"nick":    nick,
	})

}

```



## 示例一：带简单参数请求

http://localhost:9000/user/mike/drink

![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-25/64997062.jpg)

## 示例二：对请求参数设置默认值

http://localhost:9000/welcome?firstname=jack&lastname=lee
![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-25/73457672.jpg)

#### 请求参数为firstname，lastname

http://localhost:9000/welcome?lastname=lee

对于缺失参数firstname情况的处理![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-25/2179572.jpg)



## 示例三：处理简单表格传输

http://localhost:9000/form_post
![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-25/65480832.jpg)



备注：简单参数意味着只有几个参数，简单表格意味着是key-value一对一

![](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/88408382.jpg)



