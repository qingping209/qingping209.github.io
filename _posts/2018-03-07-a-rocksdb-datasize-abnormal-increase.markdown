---
layout: post
title:  "rocksdb数据异常增长"
date:   2018-03-07 10:38
categories: rocksdb
tags: 坑
---

现网有个应用侧的实例，在写入量并不是很大的情况下，以每天100G的速度不断增大，很是诡异。 于是分析了下该实例的数据访问模式如下:

0. 有一个hash结构， 该hash有2200个key， 并且每个key对应的value比较大， 200K左右
1. 应用层轮流对该hash的2200个key进行更新， 周而复始； 在一天的时间内， 该hash的每个key都要被更新很多次
2. 由于每次更新一个key都有一条binlog产生，且所有的binlog的序号是单调递增的; 在rocksdb中， binlog的key的范围是不断扩大的

改进了下sst_dump_tool.cc， 使用ldb dump_live_file分析该实例的rocksdb数据，发现如下现象：

0. 该实例的rocksdb一共有25926个sst文件
1. 完全由写binlog数据的key/value组成的sst文件，命名为binlog_put_sst, 有23163个
2. 完全由删除binlog数据的key/value组成的sst文件, 命名为binlog_del_sst, 有1334个
3. 其中binlog_del_sst的分布为: 1280个在level4, 54个在level5, level4层的1280个文件的大小最小为4.5K, 最大为479K，level4中比479K大的文件有1289个， level5上的54个文件，最大为37K, 最小为1.1K， level5中大于37K的文件有12860个;
4. 其中binlog_put_sst的分布为: 52个在level4, 12812个在level5, 10263个在level6

跟进一下rocksdb中跟合并相关的代码，一个sst文件被选中参与一次合并的条件:

0. 对每层,  计算一个分数：当前文件大小的和/当层能容量的总数据量大小，对这些分数排序，取分数最大的层
1. 选定需要合并的层以后， 从该层中选定最大的文件， 作为合并的文件之一(还会做其他的计算把当前层的其他文件和下一层的一些文件也包含进来)

根据这个合并条件，即使应用侧对binlog都执行了删除操作，位于level6的binlog_put_sst文件被释放的机会也很渺小； level5中的binlog_put_sst文件被释放的机会稍微大一些，但还是小。 因此导致binlog被应用侧删除， 空间确无法被真正释放。
