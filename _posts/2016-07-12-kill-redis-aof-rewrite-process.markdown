---
layout: post
title: "杀死redis后台io进程"
date: 2016-07-12
categories: redis
tags: 坑 
---

Redis的aof rewrite机制在如下两种情况下会启用：
1. 用户给Redis发BGREWRITEAOF命令
2. aof文件的大小增长超过一定比例，且aof文件实际大小超过一定值

条件一对应触发aof rewrite代码：

<pre>
 /* Start a scheduled AOF rewrite if this was requested by the user while
  * a BGSAVE was in progress. */
 if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
     server.aof_rewrite_scheduled)
 {
     rewriteAppendOnlyFileBackground();
 }
</pre>

条件二对应触发aof rewrite代码：

<pre>
 /* Trigger an AOF rewrite if needed */
 if (server.rdb_child_pid == -1 &&
     server.aof_child_pid == -1 &&
     server.aof_rewrite_perc &&
     server.aof_current_size > server.aof_rewrite_min_size)
 {
    long long base = server.aof_rewrite_base_size ?
                    server.aof_rewrite_base_size : 1;
    long long growth = (server.aof_current_size*100/base) - 100;
    if (growth >= server.aof_rewrite_perc) {
        redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
        rewriteAppendOnlyFileBackground();
    }
 }
</pre>

实际运营的过程中， 通过条件二自动触发aof rewrite，从而达到数据持久化的目的。 但是，也会有极端情况发生： 被触发的aof rewrite最终失败了， 由于条件二一直满足， aof rewrite会不断发生， 不断失败， 生成大量的aof 临时文件，可能把磁盘塞满， 导致服务故障。 （aof rewrite成功后， aof\_rewrite\_base\_size会被重置， 使growth小于server.aof\_rewrite\_perc, 就没有这个问题）。 

所以， 一旦发现线上aof rewrite 的进程影响了线上服务， 且kill不掉的时候，可以先通过config set调整

```
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

这两个配置，将auto-aof-rewrite-percentage调大，或者auto-aof-rewrite-min-size调大， 然后kill掉bg-aof-rewrite进程，必要的时候，可以通过config set appendonly no停掉aof rewrite。
