---
layout: post
title: "tcp-delay-accept"
date: 2016-07-12
categories: tcp/ip
tags: 坑 
---

google上面搜索TCP_DEFER_ACCEPT会出现这几篇文章：

 - http://blog.csdn.net/fullsail/article/details/4429102
 - http://www.pagefault.info/?p=346
 - http://blog.csdn.net/zhangskd/article/details/42614793

如果你的客户端向服务器发起连接， 出现如下现象：

 - client向svr发起连接，成功发送SYNC包
 - svr不断向client重传SYNC+ACK
 - netstat在client上可以看到连接状态为established，而在svr端看不到连接
 - 在client连接返回时立马给svr发送数据成功， 但是过一定时长，再进行第一次数据发送， 则收到svr返回的RST包

那么svr端设置了TCP_DEFER_ACCEPT是八九不离十了， 详细分析，见http://www.pagefault.info/?p=346 和 http://blog.csdn.net/zhangskd/article/details/42614793。 解决方法很简单，在需要发数据的时候，再建立连接。 ：）
