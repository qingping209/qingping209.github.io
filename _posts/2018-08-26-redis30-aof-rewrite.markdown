---
layout: post
title: "redis3.0以后的aof rewrite优化"
date: 2018-08-26
categories: redis
tags: 坑
---

### AOF Rewrite优化点

AOF rewrite的时候，需要把从fork那一刻开始的增量修改命令保存起来，待新的aof文件生成以后，把这些增量追加到新的aof。 redis采用双buff,  aof\_buf和aof\_rewrite\_buf\_blocks， 其中aof\_buf里面的数据正常刷到当前的aof文件中，而aof\_rewrite\_buf\_blocks里保存的增量修改在所有内存数据rewrite完成以后，被追加到新的aof文件。这样做的好处是，即使child进程rewrite失败了，parent进程还是保有一个可用的aof文件，并且数据是完整。

相比对redis 2.8.17, redis 3.0以后的版本增加了一个pipe,  通过这个pipe，parent进程把rewrite过程中累计的增量修改，也就是aof\_rewrite\_buf\_blocks中的数据发给child进程，child进程收到这些数据以后保存到 aof\_child\_diff中，child进程在rewrite结束的时候，把aof\_child\_diff中的数据追加到新的aof。 parent进程等待child进程结束以后， 需要把对新的aof文件做rename，在rename之前，parent进程需要把aof\_rewrite\_buf\_blocks中的残留数据(从子进程不再通过pipe读取aof\_rewrite\_buf\_blocks开始，到parent调用doneHandler期间， 新产生的增量修改)追加到新的aof中。 这样一来，parent进程中需要追加到新的aof的数据就会大大减少，parent进程阻塞的几率就降低了很多。在此过程中，内存使用量并没增加，因为parent把aof\_rewrite\_buf\_blocks发给child以后，就释放已发送给child的数据所占用的内存。

### 测试一把

```
redis2.8.17 vs redis4.0.11

启动命令：  ./src/redis-server  --appendonly yes
AOF policy:  everysec
压测命令： ./src/redis-benchmark -r 1000000000 -n 20000000 lpush mylist__rand_int__ rand_int__

测试结果：

redis2.8.17
...
[24596] 17 Aug 23:37:30.933 * Parent diff
successfully flushed to the rewritten AOF (6616355 bytes)
...
[24596] 17 Aug 23:37:51.738 * Parent diff
successfully flushed to the rewritten AOF (10946132 bytes)
...
[24596] 17 Aug 23:38:33.967 * Parent diff
successfully flushed to the rewritten AOF (38479070 bytes)
...
[24596] 17 Aug 23:40:07.592 * Parent diff
successfully flushed to the rewritten AOF (98919020 bytes)
...
81309.07 requests per second


redis 4.0.11:

...
23349:M 17 Aug 23:32:45.306 * Residual
parent diff successfully flushed to the rewritten AOF (0.45 MB)
...
23349:M 17 Aug 23:33:06.939 * Residual
parent diff successfully flushed to the rewritten AOF (0.52 MB)
...
23349:M 17 Aug 23:33:52.114 * Residual
parent diff successfully flushed to the rewritten AOF (1.48 MB)
...
23349:M 17 Aug 23:35:46.264 * Residual
parent diff successfully flushed to the rewritten AOF (1.83 MB)
...

89217.60 requests per second
```

不难发现， redis4.0.11的主线程在aof rewrite完成以后， 所需要的flush的数据量是远小于redis2.8.17的。 由于redis的单线程架构，主线程中的慢操作越少，那么服务的抖动就越小，服务就越稳定。
