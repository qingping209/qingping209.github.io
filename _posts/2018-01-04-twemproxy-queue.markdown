---
layout: post
title:  "twemproxy队列机制"
date:   2018-01-04 22:21:10
categories: redis
tags: 原理
---

twemproxy作为redis的代理程序，为客户端提供通过M个twemproxy访问N个后端redis节点(M << N)的能力。 

twemproxy为每个后端redis节点建立不超过server_connections个连接server_conn;   对于每个server_conn， twemproxy维护一个队列，记录从这个server_conn上面发到后端redis的请求。  对于每个客户端连接client_conn， twemproxy也维护一个队列，记录从这个客户端发到twemproxy的请求。  例如client_conn A向twemproxy发送了3个请求R3, R2, R1,  那么twemproxy在client_conn  A的队列CQ中记录[R1, R2, R3].  twemproxy发现， 其中R1需要被转发到redis节点N1,  R2和R3需要被转发到redis节点N2；那么R1通过server_conn1转发到N1, R2和R3通过server_conn2转发到N2;   server_conn1的发送队列SQ1中记录[R1], server_conn2的发送队列SQ2中记录[R2, R3]。 我们知道，后端redis是单线程模型，一次只能处理一个请求，那么从server_conn上面先发到后端redis的请求， server_conn一定先收到对应的回包。 所以，当twemproxy从server_conn1上面收到一个来自N1的一个回包RSP时，它会从SQ1队列的头部把R1 pop出来， 把RSP保存到R1的buff中， 并且找到R1所属的客户端连接client_conn A， 检查其请求队列CQ的头部请求，也就是最先发送到twemproxy的请求R3，发现后端还没有返回R3的回包， 此时就不把RSP返回给client_conn A对应的远端了； twemproxy又从server_conn2收到RSP'时， 将R3从SQ2的头部pop出来， 将RSP'保存到R3的buf中， 并且找到R3所属的客户端连接client_conn A， 发现其请求队列的头部请求R3，已经收到回包了， 就把RSP'返回给client_conn A对应的远端， 将R3从 CQ pop出去； 然后继续检查CQ的头部请求， 发现是R2，还没收到回包，放着不管先。 此时， SQ1为空， SQ2队列中剩下[R2], CQ队列中为[R1 R2], 其中R1已经收到回包， R2没有收到回包。  最后twemproxy 从server_conn2上又收到一个RSP'',  将SQ2的头部请求R2 pop出来， 把RSP''保存到R2的buff中， 找到R2所属的客户端连接client_conn A，发现CQ的头部请求是R2， 已经收到了回包了， 就把RSP''发送给client_conn A对应的远端， 将R2从CQ中pop出去, CQ中剩下[R1]； 继续检查CQ的头部请求，发现是R1, 已经收到了回包了，将R1的回包RSP发送到client_conn A对应的远端， 将R1从CQ中pop出去， CQ为空，所有所有请求处理完毕。

从以上流程我们可以看到， 通过为客户端连接维护请求队列，无论这些请求被后端节点处理的先后顺序如何， twemproxy确保给客户端回包的顺序一定与该客户端发送的请求的顺序一致。 另外，twemproxy利用redis的单线程特性，确保了当前从server connection上面收到的回包，一定属于位于server connection的请求队列头部的请求。 

根据这个队列机制，我们可以得出结论：twemproxy支持异步访问， 支持pipeline
