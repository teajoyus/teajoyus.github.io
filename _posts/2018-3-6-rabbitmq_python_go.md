---
layout: post
title: RabbitMQ消息队列跨语言demo（go和python）
date: 2018-03-06
tags: 消息队列 go  
---

 #前言
本文代码主要参考 RabbitMQ官方demo
[RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html%20%20RabbitMQ%20%20Tutorials)

其中发送消息使用 python,接收消息使用go

基本条件：
| 	类型|	名称 | 内容	|
|:---:    |:----:|:--- : |
|Operate System|Ubuntu |     16.04|
|Message Queue|RabbitMQ|   3.6.15|
|MQ Host| http address|192.168.105.132|
|MQ User| admin| 123456|

##发送消息
使用 pyhon  RabbitMQ库为pika
首先使用 python pip工具安装pika

 > pip intsall pika

 显示已经安装软件
 > pip list

完整源码sendmq.py
```
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='hello')
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

##接收消息
笔者go运行环境在window 系统中
使用go   RabbitMQku 为 Go amqp client
运行go安装
> go get github.com/streadway/amqp

完整源码 receive.go
```
package main

import (
	"log"

	"github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}

func main() {
	conn, err := amqp.Dial("amqp://admin:123456@192.168.105.132:5672/")
	failOnError(err, "Failed to connect to RabbitMQ")
	defer conn.Close()

	ch, err := conn.Channel()
	failOnError(err, "Failed to open a channel")
	defer ch.Close()

	q, err := ch.QueueDeclare(
		"hello", // name
		false,   // durable
		false,   // delete when unused
		false,   // exclusive
		false,   // no-wait
		nil,     // arguments
	)
	failOnError(err, "Failed to declare a queue")

	msgs, err := ch.Consume(
		q.Name, // queue
		"",     // consumer
		true,   // auto-ack
		false,  // exclusive
		false,  // no-local
		false,  // no-wait
		nil,    // args
	)
	failOnError(err, "Failed to register a consumer")

	forever := make(chan bool)

	go func() {
		for d := range msgs {
			log.Printf("Received a message: %s", d.Body)
		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever
}
```
测试结果 截图
![send mq](/images/posts/rabbitmq_python_go/receivemq.png)
![receive mq](/images/posts/rabbitmq_python_go/sendmq.png)

Go语言主要参考代码
[rabbitmq tutorial-one-go ](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)



![微信公众号](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/88408382.jpg)