---
layout: post
title:  "redis 事务实现"
date:   2015-08-12 12:21:10
categories: redis
---

看下redis的文档，对于事务的介绍如下：

> It is possible to group commands together so that they are executed as a single transaction. 
> 
> All the commands in a transaction are serialized and executed sequentially. 
> 
> Either all of the commands or none are processed, so a Redis transaction is also atomic. 

通过MULTI, EXEC, DISCARD和WATCH可以在redis中使用事务。MULTI开启一个事务，EXEC执行一个事务，DISCARD中断一个事务，WATCH为事务执行设置一个条件。具体每个指令是干什么的就不展开，可以参考 [Transactions](http://redis.io/topics/transactions)。

说到事务，必须扯到ACID, 即原子性，一致性，隔离性，持久性。由于redis单线程的架构，不存在对数据的并发操作，因此天然就具有一致性，隔离性。 至于原子性， 就是让一组命令要么全部执行要么都不执行; 而持久性就是事务完成以后，数据更新不会丢失。 我们主要探究这原子性和持久性如何实现。

#### 实现 ####
*** 相关数据结构 ***

事务的实现位于src/multi.c中。 与multi和exec相关的数据结构有：

```
/* Client MULTI/EXEC state */
typedef struct multiCmd {
   robj **argv;
   int argc;
   struct redisCommand *cmd;
} multiCmd;

typedef struct multiState {
    multiCmd *commands;     /* Array of MULTI commands */
    int count;              /* Total number of MULTI commands */
    int minreplicas;        /* MINREPLICAS for synchronous replication */
    time_t minreplicas_timeout; /* MINREPLICAS timeout as unixtime. */
} multiState;
```

multiState记录着一个事务所需要执行的所有命令。 

与watch相关的数据结构有：

redisClient->watched_keys /* Keys WATCHED for MULTI/EXEC CAS */
redisDb->watched_keys /* WATCHED keys for MULTI/EXEC CAS */ 
redisClient->watched_keys是一个list, list的元素类型为:

typedef struct watchedKey {
    robj *key;
redisDb *db;
} watchedKey;

redisClient->watched_keys记录着所关注的每个key和key所属的redisDb。

redisDb->watched_keys是一个dict, dict的键为redis key，值为一个list, list中元素为redisClient。redisDb->watched_keys记录着一个redisDb中被关注着的redis key和关注一个redis key的所有redisClient。


*** 实现流程 ***

流程挺简单，客户端发起multi命令开始一个事务，redis将对应的redisClient标记为REDIS_MULTI，并将multi之后的命令保存在multiState中，直至收到discard或者exec命令。discard命令会中断事务，将redisClient的multiState内容清空，清除REDIS_MULTI, REDIS_DIRTY_CAS以及REDIS_DIRTY_EXEC这三种状态；exec命令则挨个顺序执行所有保存在multiState中的命令，注意exec在执行命令之前会检查redisClient的状态是否被设置为REDIS_DIRTY_CAS或者REDIS_DIRTY_EXEC，如果是则中断事务，清除所有状态。

上面提到REDIS_DIRTY_CAS状态和REDIS_DIRTY_EXEC状态。 什么时候redisClient会被置为REDIS_DIRTY_CAS状态呢？在redisClient发起一个事务前，如果该redisClient使用watch命令关注过redis key被修改了，可能是自己修改也可能是别的redisClient修改，那么该redisClient会被设置为REDIS_DIRTY_CAS状态。 具体而言，watch命令将一个redis key和发起该watch命令的redisClient记录到redisDb->watched_keys中，将redis key和该key所属的redisDb记录到redisClient->watched_keys中。执行任意具有写语义的命令时，signalModifiedKey会被调用，signalModifiedKey继而调用touchWatchedKey，touchWatchedKey会从redisDb->watched_keys中将关注某个redis key的所有redisClient找出来，设置他们的状态为REDIS_DIRTY_CAS。 redisClient->watched_keys是个辅助数据结构，用于记录一个redisClient是否关注过某个redis key。那为什么不用dict而用list, dict的效率不是更高么？如果使用dict，那就得确定键和值。由于不同redisDb中可以有相同的redis key，那无论键是什么，值都会是个list。如果键是redis key，则值是redisDb的list, 如果使用到的redisDb不多，不失为一种方式？ 如果键是redisDb，则值是redis key的list。 我个人觉得，如果在redis中使用大量的watch， 或者使用大量的redisDb，都不是明智的做法，因此在通常情况下使用dict和list的性能就差不多了，由于dict的值是个list，那么直接使用list作为redisClient->watched_keys就更加间接方便。 
