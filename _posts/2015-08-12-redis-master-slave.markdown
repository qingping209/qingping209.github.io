---
layout: post
title:  "redis 主从同步过程"
date:   2015-08-12 12:21:10
categories: redis, replication
---
我们来看看几个问题，探索下为何做这些调整以后，同步就成功了。


***Q1：如何在两个Redis实例之间建立主从关系?***

A1： 向Redis实例A发送命令slaveof master\_host master\_port，master\_host和master\_port分别是Redis实例B的ip和端口。这样，实例A和实例B就建立了主从关系，其中实例A为slave，实例B为master。但值得注意的是，slaveof命令并不直接为slave建立到master的物理连接，相反，它主要做的事情是状态清理。slaveof命令实际做的事情如下：

-   接收slaveof命令的Redis实例保存slaveof命令提供的master的ip和端口
-   如果该实例之前和其它Redis实例建立了主从关系，断开到已有mater的连接  
-   如果该实例本身是master，断开和它的slaves之间的连接
-   清除cached master，避免该实例向新的master发送psync命令
-   清除backlog，防止该实例的slave发psync命令
-   如果该实例已经在和其他Redis实例建立连接或者传输数据，通通停止，将复制的初始状态置为REDIS\_REPL\_CONNECT, 将master\_repl\_offset置为0。

***Q2: slave到master的物理连接由谁来建立？ 如何建立？***

***A2***：Redis有一个复制相关的定时任务replicationCron，每秒钟执行一次(执行周期不可配置，在代码中写死了）。replicationCron会根据当前实例的身份（slave或者master），以及当前实例所处的状态（slave和master在复制的过程中分别维护了几个状态）来决定干什么事情。 前面一个QA中提到，Redis实例在接收并处理了slaveof命令后，处于REDIS\_REPL\_CONNECT状态。replicationCron再次执行时，发现实例处于REDIS\_REPL\_CONNECT状态，就通过connectWithMaster以非阻塞的方式建立到master的物理连接，得到一个文件描述符fd。然后将fd加入到事件循环中，监听该fd的读写事件，且设置回调函数为syncWithMaster。最后，该Redis实例的状态更新为REDIS\_REPL\_CONNECTING，记录repl\_transfer\_lastio为当前时间。

***Q3：slave处于REDIS\_REPL\_CONNECTING状态了， 和master的连接建立好了么？ 什么时候才能建立好？***

A3：slave处于REDIS\_REPL\_CONNECTING，表明slave已经向master发起了连接，但连接并没有建立好。此时，有几种可能性: 

- 连接超时。replicationCron执行时会比较repl\_transfer\_lastio和当前时间来检查连接(connect调用)是否超时，若超时，就通过undoConnectWithMaster取消连接。下次replicationCron执行时再次尝试连接。
- 连接成功。回调函数syncWithMaster被执行，确认fd上没出错且Redis实例状态为REDIS\_REPL\_CONNECTING时，将Redis实例状态设置为REDIS\_REPL\_RECEIVE\_PONG， 且向master以同步的形式发送一个PING命令。
- 连接失败。检查到fd上出错了，取消连接。

***Q4：为什么slave在建立好到master的连接后要向master发送一个PING命令并进入REDIS\_REPL\_RECEIVE\_PONG状态？***

A4：slave成功建立到master的连接并不是能够进行数据同步的充分条件。因为，进入数据传输阶段，超时时间比较长，所以在进入下一阶段之前，slave需要确认master能够响应和处理命令。slave在发送收到master回复的PONG后，可能需要向master发送auth命令，进行身份认证，这一步是否要进行取决于master是否有masterauth这个选项。然后就可以进入synchronization了。
