---
layout: post
title: linode 服务器延迟测试powershell脚本
date: 2018-03-01
tags: 工具    
---

有个需求需要测试一下linode不同节点的服务器延迟大小，如果每次手工输命令测试，那效率太低。因此做一个简单的脚本
来测试不同节点的网络延迟

[linode speed test](https://www.linode.com/speedtest)

![Speed test](/images/posts/powershell/linode.png)

完整代码如下

	ping  speedtest.newark.linode.com	-n 10 |Out-File "C:\linode.txt"
	
	ping speedtest.atlanta.linode.com	-n 10 |Out-File "C:\linode.txt" 	
	
	ping speedtest.dallas.linode.com	-n 10 |Out-File "C:\linode.txt" 	
	
	ping speedtest.fremont.linode.com	-n 10 |Out-File "C:\linode.txt"  	
	
	ping speedtest.frankfurt.linode.com	-n 10 |Out-File "C:\linode.txt"  
	
	ping speedtest.london.linode.com	-n 10 |Out-File "C:\linode.txt"  	
	
	ping speedtest.singapore.linode.com	-n 10 |Out-File "C:\linode.txt"

主要功能是 ping 一个主机看 返回延时数据，同时把数据保存到txt文本

某次测试结果如下：

	正在 Ping speedtest.frankfurt.linode.com [139.162.130.8] 具有 32 字节的数据:
	来自 139.162.130.8 的回复: 字节=32 时间=300ms TTL=52
	来自 139.162.130.8 的回复: 字节=32 时间=301ms TTL=52
	来自 139.162.130.8 的回复: 字节=32 时间=301ms TTL=52
	来自 139.162.130.8 的回复: 字节=32 时间=301ms TTL=52
	请求超时。
	来自 139.162.130.8 的回复: 字节=32 时间=300ms TTL=52
	请求超时。
	来自 139.162.130.8 的回复: 字节=32 时间=299ms TTL=52
	来自 139.162.130.8 的回复: 字节=32 时间=304ms TTL=52
	来自 139.162.130.8 的回复: 字节=32 时间=300ms TTL=52
	139.162.130.8 的 Ping 统计信息:
		数据包: 已发送 = 10，已接收 = 8，丢失 = 2 (20% 丢失)，
	往返行程的估计时间(以毫秒为单位):
		最短 = 299ms，最长 = 304ms，平均 = 300ms


单次测试数据结果：

|区域      | 主机名   |  平均延时值  |
| -------- | :-----:  | :----:  |
| US East|newark |123ms|
|US South|atlanta|320ms|
|US Central|dallas|179ms|
|US West|fremont|248ms|
|Frankfurt|frankfurt|300ms|
|London|london|262ms|
|Singapore|singapore|300ms|
|Tokyo 2, Japan|shg1|107ms|

