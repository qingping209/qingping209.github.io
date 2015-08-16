---
layout: post
title:  "redis list blocking操作实现"
date:   2015-08-12 12:21:10
categories: redis
---

###相关数据结构###

Redis的list支持blocking操作，即如果目标list没有元素，客户端操作会阻塞，直到其他客户端向list中增加元素。
这个功能很有用，看一下是怎么实现的。

实现list的blocking操作，需要四个数据结构:

- redisDb.blocking\_keys
- redisClient.bpop
- redisServer.ready\_keys
- redisDb.ready\_keys

***redisDb.blocking\_keys***类型是dict，键是redis的key, 值是一个装着redisClient的list，也就是说redisDb.blocking\_keys记录了在某个redisDb中，有哪些client在等待哪些key上的数据。

***redisClient.bpop***的类型是blockingState:

    typedef struct blockingState {
        dict *keys; 
        time_t timeout;
        robj *target; 
    } blockingState;

其中keys记录着redisClient在等待哪些key上的数据，timeout为等待时间。target只有在做blpoprpush使用，指定lpop得到的值应该被rpush到哪。

***redisServer.ready\_keys***是一个list，记录了全局中有哪些list上有数据了，可以响应被阻塞在这些list上的redisClient。redisServer.ready_keys中元素的结构为:

    typedef struct readyList {
        redisDb *db;
        robj *key;
    } readyList;

记录了一个list的键和其所属redisDb。

***redisDb.ready\_keys***是一个dict, 键是redis key，值为NULL。redisDb.ready_keys是一个辅助数据结构，用于加速判断一个list是否已经在redisServer.ready_keys中。

----------

###实现流程###

1)  阻塞

当某客户端redisClient  A对某list发起blpop, brpop或者brpoplpush操作时，如果被pop的list中没有数据，则该list的key被加入到redisClient的blockingState的keys中， 把该redisClient加入到key所属redisDb的blocking\_keys中； 在blockingState中更新redisClient将被阻塞的时长，将redisClient的状态改为REDIS_BLOCKED。 这样，redis就记住了redisClient阻塞在key上，key阻塞了redisClient。

2) 唤醒

什么时候“唤醒”被阻塞的redisClient呢？ 有两种方式:

- clientsCronHandleTimeout作为clientsCron的一部分每秒执行一次，检查是否有处于REDIS\_BLOCKED状态的redisClient已经过了等待时间，如果有，则调用unblockClientWaitingData清除阻塞状态：将该redisClient的blockingState结构清空，将该redisClient从其当前操作的redisDb的blocking_keys的删除。

- redis每执行完一条命令以后会处理被阻塞的redisClient：
       
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();

    具体做法是：遍历server.ready\_keys，每个元素是一个readyList结构，包含一个key和一个redisDb， key上面存储一个list。 从redisDb的blocking_keys获取阻塞在key上面的redisClients，将存储在key上的list中的数据逐一pop出来，返回给各个redisClient，每个获取到数据的redisClient的阻塞状态都通过unblockClientWaitingData被清除。

3) 唤醒条件

唤醒客户端条件之一是server.ready\_list不为空。如果一个list阻塞过客户端，那么在往这个list上push数据时，调用signalListAsReady，将list的key和所在redisDb加入server.ready\_list。在push完成以后，就可以调用handleClientsBlockedOnLists()了。注意，signalListAsReady只在dbAdd中被调用:

    void dbAdd(redisDb *db, robj *key, robj *val) {
        sds copy = sdsdup(key->ptr);
        int retval = dictAdd(db->dict, copy, val);

        redisAssertWithInfo(NULL,key,retval == REDIS_OK);
        if (val->type == REDIS_LIST) signalListAsReady(db, key);
    }

由于空的list会被redis从redisDb中删除，那么push数据到一个不存在于redisDb中的list，则需要新建一个list，且调用dbAdd将其加入到redisDb中。 也就是说，将一个list加入到redisDb中，就意味着该list由空变为不为空了。

最后， 从上面的描述中可以看出来，blpop, brpop, blpoprpush是不能跨redisDb操作的。

-- eof --

