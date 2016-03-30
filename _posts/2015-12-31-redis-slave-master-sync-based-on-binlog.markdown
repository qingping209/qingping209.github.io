---
layout: post
title: "redis基于binlog的主从同步"
date: 2015-12-31 16：20
categories: redis 
tags: 主从同步
---

### 1. 背景 ###

上半年我们使用RocksDB做存储引擎，实施了Redis数据实时落地的项目，实现了在兼容redis协议的前提下，管理超出内存大小的数据集。

在主从同步方面，我们沿用了Redis原有的方案，该方案在实际运营中应对不良的网络状况显得很无力，因此我们在数据落地的基础上，为Redis开发了一套新的主从同步机制。

### 2. Redis原生同步方式 ###

主从数据同步分一般两步走：同步已有的全量数据，和同步增量数据。同步的全量数据必须是master数据集的point-in-time的快照；增量数据则从快照完成的瞬间算起。如图(一），

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/1.png)
 ***图 （一）*** Redis原生主从同步机制
 
slave给master发送sync命令请求同步数据，master收到sync命令后，将内存数据镜像(point-in-time)保存为dump.rdb文件，同时将新进来的写请求保存到client-buffer中；master首先将生成的内存镜像文件dump.rdb同步给slave（全量数据同步），完成后，将client-buffer中累积的写请求同步slave（增量数据同步）。这个机制足够简单，但是在实际运营中会遇到一些问题：

- slave一旦同master断开，重连后如果不能做psync（实际运营中psync基本没有成功过），就要同步master的全量数据
- 实际运营中一般会做多实例部署，一台Z3服务器部署20 - 40 不等的实例数，一旦网络闪断，导致几十个master实例同时fork做bgsave，场面感人； 通常结果就是内存不够，做bgsave的进程被kill掉
- 如果同步全量数据时，master实例上写请求量大，使client-buffer占用内存超限，master会主动断开和slave的连接，这样之前的全量数据就白做了

这里的根本矛盾是Redis本身定位和我们的要求有偏差。Redis所有操作都是基于内存， 数据存在内存，用于断开重连的backlog保存在内存，主从同步时的增量数据保存在内存，可每台服务器内存就那么多，能存储的数据量是有限的；而我们在更多场景下，要求Redis在提供足够性能的前提下，存储尽量多的数据。我们需要在性能和数据量，以及可运营性之间做折中。

### 3. 基于binlog的数据主从同步 ###

前面已经提到，我们已经实现Redis数据实时落地到RocksDB中。 这里进一步，我们将数据的修改记录以一定的格式(binlog）落入RocksDB中，并实现基于binlog的主从数据同步。

#### 3.1 落地binlog ####

我们对Redis每种具有写语义的命令定义一种binlog，注意是一种，而不是一条。 总共定义了16中binlog， 如图(二）：
![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/2.png)
 ***图 （二）*** 十六种binlog类型

所有类型binlog都需要满足两个条件， 原因后面展开：

* 每条binlog需要记录它所修改的数据存储于RocksDB中的key
* 一条binlog修改的数据不能对应RocksDB中多对key/value


#### 3.2 基于binlog的主从同步 ####

前面提到过，主从数据同步分两步，全量数据同步和增量数据同步。master收到一个新的slave的同步请求时，master给RocksDB做个快照，快照中数据分两部分，Redis的数据集和写数据时生成的binlog。master把快照中数据集发给slave即实现全量数据同步；master记录快照中最大的binlog序列号MaxSeq, 把序列号从MaxSeq+1开始的binlog逐条发给slave即可实现增量数据同步。流程很简单，如下图：

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/3.png)
 ***图 （三）***  基于binlog的主从数据同步

#### 3.3 主从连接断开处理 ####

我们已经知道Redis主从同步机制要求slave断开重连master后，同步(psync绝大多数情况下不管用）master的全量数据。这个机制的问题上面已经提过。 如何解决呢？ 很直观，如果slave重连后，能够从断开的位置开始，继续同步master数据，这是最好的。 为了实现这一点，我们以master的视角将slave的同步过程划分为四个状态，图(四）描绘了四个状态之间的转换图：

- REPL\_PASTE，master和slave做全量同步
- REPL\_RESUME，slave全量同步时与master断开后重连
-  REPL\_FOLLOW，slave和master做增量同步
- REPL\_OUTSYNC，master无法找到slave所需要的binlog

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/4.png)
***图（四）***  slave同步状态转换图

具体看下每个状态下slave和master断开后重连，如何做续传的：

___case 1: 正在做全量同步的slave断开与master连接，重连后的续传___

一个空slave实例连上master开始做全量数据同步， slave处于REPL_PASTE状态。 master创建snapshot 1，如下图，

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/5.png)
***图（五）*** 空的slave实例连上master同步全量数据

此时，master将snapshot 1的数据集同步（逐个key/value对发送)给slave，并且把snapshot 1最大的binlog序列号MaxSeq发给slave。slave记录（持久化到RocksDB中）自己的同步状态为REPL_PASTE，和收到最大的binlog序列号为MaxSeq，并实时更新最后接收的key。假设slave处理完LastKey（此时master的数据集尚未完全同步给salve）即与master断开，稍后重新连上，进入REPL_RESUME状态。此时，master创建snapshot 2，如下图：

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/6.png)
***图（六）*** 做全量同步的slave断开重连master后做“续传” 
__（注意：snapshot 1和snapshot 2是RocksDB不同时刻的两个状态，不可能同时存在，这里这样画图只为了方便比较）__

断开时slave同步到snapshot 1的LastKey处，收到的最大binlog序列号为MaxSeq。 重连后，我们要在不清空slave数据集和不全量同步snapshot 2的前提下，把slave的数据更新为snapshot 2，达到“续传”的效果。 先来个拍脑袋的做法：在snapshot 2中找到LastKey， 从紧跟在它后面的LastKey’开始的key/value对起继续同步。 这样做不对，有两个原因：
1）  LastKey可能不存在，它可能在slave断开期间被删除
2）  在LastKey之前，可能有key/value在slave断开期间被修改，也有可能新的key/value被写入

因此这种做法会导致丢失一部分写操作。为了解锁正确的姿势， 我们先找到snapshot 1和snapshot 2之间的差异：

	snapshot 2 = snapshot 1 + binlog[ MaxSeq + 1 ... MaxSeq’]

也就是， 把序列号从MaxSeq+1开始到MaxSeq’结束的binlog应用到snapshot 1上，即可得到snapshot 2。进一步，由于RocksDB中key/value按照key的字典序有序存储，以LastKey为界，这些binlog一定是一部分应用到位于LastKey之前(含LastKey)的key/value对，另一部分应用到位于LastKey之后的key/value对。所以，可以把snapshot 2的数据集分成两部分处理，一部分位于LastKey之前(含LastKey)，一部分位于LastKey之后。由于slave在断开前已经同步了snapshot 1位于LastKey之前(含LastKey)的数据，我们只需要把这部分数据更新到与snapshot 2一致即可。做法：把序列号位于[MaSeq+1, MaxSeq’]范围内，且只应用到LastKey之前(含LastKey)的key/value对的binlog同步到slave。此时slave进入REPL\_PASTE状态。由于snapshot 2中位于lastKey之后（lastKey’紧随lastKey之后）的数据对于slave而言，是全新的，将这部分数据同步给slave后，snapshot 2的数据集就完全同步到slave了。__此时slave进入REPL_FOLLOW状态__。

3.1中提到，一条binlog不能修改多于一对底层key/value，这里解释下原因。在前面的处理中，要求把做snapshot 1后， 做snapshot 2之前master上生成的binlog应用到slave已同步的数据集（位于snapshot 1的LastKey之前）。 假设其中一条binlog既修改了LastKey之前的k1/v1，也修改LastKey之后的k2/v2，比如rpoplpush对应的binlog。我们知道k1/v1是snapshot 1中的数据，对其应用binlog可以将其更新为snapshot 2的数据；而k2/v2本身就是snapshot 2的数据，对其应用binlog会破坏snapshot 2的point-in-time特性。因此，rpoplpush操作需要落两条binlog，RPOP和LPUSH。

___case 2：处于REPL_RESUME状态的slave和master断开后重连___

重连时，slave将其断开前所处的同步状态，最后同步的key和最后同步的binlog序号发给master, slave进入REPL_RESUME状态，和case 1
的处理方式一致。

___case 3: 处于REPL_FOLLOW状态的slave和master断开后重连___

slave重连将后进入REPL_FOLLOW状态。 处于这个状态的slave在做增量同步，master不需要为其生成快照。 由于slave重连时把最后同步的binlog序号MaxSeq发给了master，master只需要找到序号从MaxSeq+1开始的binlog，将它们同步给slave即可，如图：

![](https://raw.githubusercontent.com/paralleld/paralleld.github.io/master/images/binlog/7.png)
***图 （七）*** 做增量同步的slave断开重连master后做“续传”

### 4. 小结 ###

我们为Redis增加了一种新的基于binlog的主从数据同步方式，实现了主从断开后slave和master间数据“续传”，避免了每次slave都要全量同步master数据，给master带来过大压力。 另外，由于有了binlog，我们可以做历史数据恢复，这里就不具体展开了。
