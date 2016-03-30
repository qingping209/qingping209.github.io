---
layout: post
title: "redis的output-buffer-limit配置"
date: 2015-11-02
categories: redis
tags: 源码
---

### **配置介绍** ###

redis的client output buffer limit可以用来强制断开那些无法从redis服务器端读取数据
足够快的客户端。一个常见的案例就是一个Pub/Sub中，消费者的速度无法和消息生产者的
速度相匹配。client output buffer limit可以应用在三类的客户端中：

- 普通client，包括monitor
- slave用来同步master数据的client
- Pub/Sub模式中订阅了至少一个channel或者模式的client

该配置的语法是:

    client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>

连接到redis server的客户端，如果它不能快速消费服务端返回给他的数据，其在server
端占用的内存(output buffer size)会越来越多，一旦大小超过hard limit, 或者超过
soft limit一段时间，server端会立即断开该客户端，并且释放该客户端所占用的资源。
例如，若hard limit配置为32M， soft limit配置为10M/10s，一旦output buffer使用
的内存到达30M，或者连续超过10M的时间维持10s，那么客户端的连接会被立刻断开。

默认情况下，对普通客户端的output buffer size不设置限制，因为server只有在其请求
数据的时候才会发送；只有异步客户端请求数据比接收快的时候，才会产生out buffer 
size膨胀的问题。对于用作pubsub和slave的客户端，由于server会主动把数据推送给
它们，需要设置output buffer size的限制。

可以通过将hard limit/soft limit都设置为0的方式禁用该配置项，如：

    client-output-buffer-limit normal 0 0 0 
    client-output-buffer-limit slave 256mb 64mb 60 
    client-output-buffer-limit pubsub 32mb 8mb 60 

### **应用场景** ###

主从同步是我在生长环境中遇到的跟该配置关系比较密切的场景。我们知道redis做主从同步时，
首先master将全量数据保存为rdb文件传输给slave，在master保存rdb，传输给slave，slave加载
rdb完成之前，master收到的所有的写命令，都要保存在slave的client中。如果保存，传输，加载
持续的时间太长，且此期间写的QPS很高，则有可能使slave的client使用的内存超过限制，造成
master断开slave，同步失败。

### **实现** ###

redis server通过addReply把数据发送给客户端:

    void addReply(redisClient *c, robj *obj) {
        if (prepareClientToWrite(c) != REDIS_OK) return;
    
        if (obj->encoding == REDIS_ENCODING_RAW) {
            if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != REDIS_OK)
                _addReplyObjectToList(c,obj);
        } else if (obj->encoding == REDIS_ENCODING_INT) {
            /* Optimization: if there is room in the static buffer for 32 bytes
             * (more than the max chars a 64 bit integer can take as string) we
             * avoid decoding the object and go for the lower level approach. */
            if (listLength(c->reply) == 0 && (sizeof(c->buf) - c->bufpos) >= 32) {
                char buf[32];
                int len;
    
                len = ll2string(buf,sizeof(buf),(long)obj->ptr);
                if (_addReplyToBuffer(c,buf,len) == REDIS_OK)
                    return;
                /* else... continue with the normal code path, but should never
                 * happen actually since we verified there is room. */
            }
            obj = getDecodedObject(obj);
            if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != REDIS_OK)
                _addReplyObjectToList(c,obj);
            decrRefCount(obj);
        } else {
            redisPanic("Wrong obj->encoding in addReply()");
        }
    }

我们看到，prepareClientToWrite(redisClient *c)判断是否要把数据写入客户端的output buffer。我们
看下要满足什么条件数据才会被写入客户端缓存中:

    /* If the client should receive new data (normal clients will) the function
     * returns REDIS_OK, and make sure to install the write handler in our event
     * loop so that when the socket is writable new data gets written.
     *
     * If the client should not receive new data, because it is a fake client,
     * a master, a slave not yet online, or because the setup of the write handler
     * failed, the function returns REDIS_ERR. */
    int prepareClientToWrite(redisClient *c) {
        if (c->flags & REDIS_LUA_CLIENT) return REDIS_OK;
        if ((c->flags & REDIS_MASTER) &&
            !(c->flags & REDIS_MASTER_FORCE_REPLY)) return REDIS_ERR;
        if (c->fd <= 0) return REDIS_ERR; /* Fake client */
        if (c->bufpos == 0 && listLength(c->reply) == 0 &&
            (c->replstate == REDIS_REPL_NONE ||
             c->replstate == REDIS_REPL_ONLINE) && !c->repl_put_online_on_ack &&
            aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
            sendReplyToClient, c) == AE_ERR) return REDIS_ERR;
        return REDIS_OK;
    }
    
该段代码的注释说得很清楚， 如果客户端是个fakeClient（用于加载aof文件），或者是个master,
或者是个处于REDIS\_REPL\_ONLINE状态的slave但为其且安装写回掉函数失败时，则不应该将数据
写入客户端output buffer。 我们知道，在master保存和发送rdb文件期间，slave的状态是:

    #define REDIS_REPL_WAIT_BGSAVE_START 6    
    #define REDIS_REPL_WAIT_BGSAVE_END 7      
    #define REDIS_REPL_SEND_BULK 8            

中的一种，所以这期间的写请求都会保存到slave的output buffer；但由于并没有安装写回调函数，
数据并不会发到slave；直到master把rdb文件完全发送给slave，master通过sendBulkToSlave中的
一段代码将slave的状态改为REDIS\_REPL\_ONLINE:

     // rdb完全发送给slave了
     if (slave->repldboff == slave->repldbsize) {
          close(slave->repldbfd);
          slave->repldbfd = -1;
          aeDeleteFileEvent(server.el,slave->fd,AE_WRITABLE);
          putSlaveOnline(slave);
      }

    // 将slave的状态改为REDIS_REPL_ONLINE, repl_put_online_on_ack置为0，
    // 并且安装写回调函数
    void putSlaveOnline(redisClient *slave) {
        slave->replstate = REDIS_REPL_ONLINE;
        slave->repl_put_online_on_ack = 0;
        slave->repl_ack_time = server.unixtime;
        if (aeCreateFileEvent(server.el, slave->fd, AE_WRITABLE,
                    sendReplyToClient, slave) == AE_ERR) {
            freeClient(slave);
            return;
        }
        // ......
    }

putSlaveOnline中安装的写回掉函数会将此前积累在slave的output buffer中的写操作全部发送到
slave上。而且slave的状态使得后续master的写操作可以正常地push到slave上。

--- eof ---
