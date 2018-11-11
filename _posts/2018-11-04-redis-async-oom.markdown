---
layout: post
title: "一个异步访问redis的内存问题"
date: 2018-11-04
categories: redis
---

遇到一个redis实例突然内存飙高的案例， 具体症状如下：
- 客户端使用异步访问模式
- 单个请求的回包很大，hgetall一个8M的key

由于访问量比较大，已经登录不上redis了， 看不到具体在做什么做操， 因此使用perf来看下调用栈， 此处且按下不表。

为何内存会飙高呢，我们线下重现一下：
```python
import redis
import time

r=redis.Redis("127.0.0.1", 9988)
pipe = r.pipeline()
key="8Mhashkey"
for i in range(100):
    pipe.hgetall(key)
rsp=pipe.execute()
time.sleep(1)

```
执行这个脚本若干次， 我们可以发现9988的内存瞬间升高了， 而9988本身只有一个key, 8Mhashkey。

执行一下client list， 得到：
![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/oom.png)

可以发现，是client 的output buffer占用的大量的内存。  那为什么会出现这个现象呢？  根本原因是： redis并不是使用ping/pong的模式来处理请求。对于来自同一个连接上面的请求，并不是走如下模式：

```css
1. 接受客户端请求
2. 处理请求，生成回包
3. 给客户端发送回包，跳转1
```

而是这种模式:

```css
// 处理请求
1. 接受客户端请求
2. 处理请求生成回包， 跳转1

// 处理回复
while 有回包
    把回包发给客户端
```

也就是说， 也就是说对于同一个连接上的请求，不需要等上一个请求的回包都被发送到客户端了，才去处理下一个请求； 即使上一个请求的回包没有被发送到客户端， redis也可以去接受并处理下一个请求。 redis把这些请求的回包都保存在内存里面了，内存大小通过如下参数配置：

```css
client-output-buffer-limit normal 0 0 0
```

epoll\_wait在返回时， 如果客户端连接可写，则向这些客户端上面上送回包， 已经发送给客户端的回包所对应的内存会被释放。

到此， 原因就明显了。 异步方式访问redis时，如果生成回包的速度大于客户端读取回包的速度，redis的内存就会上涨。

突然想起来上次在SACC上面有人问我异步访问redis有什么问题，当时由于时间关系没有解释很充分。 现在看来，redis的这种网络模型，收包和回包互不阻塞的方式，在多链接的情况下，不同链接上的请求上的延时是能保证的。 唯一的问题，可能就是这个内存问题。
