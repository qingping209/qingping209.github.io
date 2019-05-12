---
layout: post
title: "redis混合存储"
date: 2019-05-09
categories: redis
tags: 源码
typora-root-url: ../images
---

最近redis混合存储很火，不断有云厂商或者互联网公司推出redis混合存储的方案。 

1. 阿里巴巴的[混合redis存储](https://mp.weixin.qq.com/s/z8NyiNo1m5voXEuvF5kg-Q)，已经发了[好几篇软文](https://yq.aliyun.com/articles/629173?spm=5176.10695662.1996646101.searchclickresult.6509ed80sL11DT)了，目前还没看到[可售卖版本](https://help.aliyun.com/document_detail/68930.html?spm=a2c4g.11186623.6.541.6ZsGYg)
2. 新浪微博的redrocks方案， 已经在十几个业务小范围应用
3. RedisLabs的[Redis Enterprise](https://docs.redislabs.com/latest/rs/concepts/memory-architecture/redis-flash/)方案

几家的思路都非常类似，即把访问频率高的key留在RAM中，把访问频率低的key放到次一级的存储介质中，一般是FLASH，如果应用程序中的数据有明显的冷热分明的模式，这种可以极大的节省成本， 代价就是偶尔的缓存不命中带来的性能抖动。

redis混合存储并不是新的东西，其最早的起源是redis 2.2中的vm模块，在2.2.15中已经决定放弃这个特性，让redis server专注于做一个内存存储，这部分特性成为了[Redis Enterprise的一部分](https://redis.io/topics/faq)。 另外， redis 2.2中vm的实现是基于文件系统的，而现在市面上的实现方案存储都是 rocksdb了。

实现冷热分离架构，无外乎要解决几个问题，什么时候swap out/in,   如何swap out/in？ swap out的触发可以根据内存使用比来决策，例如只允许X%的key存放在内存中， 或者只允许Y个key存放在内存中，类似这种策略； swap in就很直接了，如果访问到一个key，其不在内存，而在flash上，就加载进来。swap out/in的时候，自然不能阻塞工作线程，因此开启I/O线程来干这活。 在几个地方干这活：

- server cron中，主动把内存中超额的key交换到vm去
- 访问到一个不在内存中key的时候， blockClientOnSwappedKeys会把客户端挂起到， 被挂起的客户端不会再接受新的数据
- I/O线程把key加载到内存时，会唤醒已经挂起的客户端, 把这些客户端加入到io_ready_clients,  event_loop会在beforeSleep(在每次epoll_wait之前执行)中执行被唤醒的客户端的当前命令
- swap in完成以后，通过pipe通知主线程把相关客户端唤醒， 等下一次调用epoll_wait之前调用beforeSleep

Reference

1. <http://oldblog.antirez.com/post/redis-virtual-memory-story.html>
2. <https://www.redhat.com/magazine/001nov04/features/vm/>
3. <https://redis.io/topics/internals-vm>