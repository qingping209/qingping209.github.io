---
layout: post
title: "redis pipeline"
date: 2017-02-19
categories: redis
tags: 原理
---

redis有种客户端交互模式，叫做pipeline模式， 可以大幅度提高系统的吞吐量， 官方文档在这里,  https://redis.io/topics/pipelining，

### pipeline如何工作 ###
在普通模式下， 客户端不需要发一个请求给redis-server，收到回复，再发送下一个命令。而在pipeline模式下， 客户端可以一口气将***你想发几条发几条的命令***一口气扔给redis-server，redis-server收到以后， 一股脑的把所有的回复的返回给客户端。 这个模式的好处是， 可以增加提高吞吐量。 简单来说，IPV4中， 一个tcp包可以带536字节的数据， 在普通模式下，请求的tcp包和回复的tcp的都是小包，处理同样的请求，  需要的包量比较多； 而在pipeline模式下， 请求的tcp包和回复的tcp都是比较大的包，处理同样的请求，需要的包量比较少。 因此pipeline模式可充分利用带宽，提高系统的吞吐。

### pipeline需要redis-server支持么 ###
pipeline模式，是一种client/server的交互模式， 要实现pieline模式， 需要server端有如下功能:
- server端按照fcfs的模式处理数据， 也就是，必须把先来的请求的回复，先发给客户端
- server需要能够维护一个请求队列

redis的单线程模型，天然就满足这两个条件。  对于客户端， 只需要把N个请求写到一个buffer里，然后把这个buffer里面所有的数据发到server， 然后接受N个请求即可。

### twemproxy集群模式下的pipeline ###
twemproxy不能完整支持pipeline。 由于twemproxy到一个后端redis-server可能会有多个连接， 考虑如下场景, 在pipeline中先后有两个命令set foo bar 和 get foo; 由于这两个命令可以通过两个不同的连接发送到同一个后端redis-server,  那么这两个命令在redis-server上的执行顺序是没法得到保证的。 因此， twemproxy集群环境下，要让pipeline模式正确工作， 需要:  ***把到后端redis-server的连接设置为1***，或者: ***pipeline里面的命令都访问不同的key***。 对于批量数据导入，如果没有重复的key，默认部署是可行的，对于其他情况， 得注意这个坑。

小结一下:  ***twemproxy可以保证客户端收到回复的顺序和其发请求的顺序是一致的，但是在twemproxy到后端redis-server配置多个连接时，发到同一个redis-server的请求的执行顺序是不确定的。***

### Ref ###
- https://github.com/twitter/twemproxy/issues/193
- https://github.com/twitter/twemproxy/blob/master/notes/recommendation.md#server_connections--1

