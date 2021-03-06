---
layout: post
title: "redis 3.0 集群 之 高可用"
date: 2015-12-06
categories: redis
tags: 集群
---

### 3. 高可用 ###
redis集群的高可用是通过自动故障切换来实现。具体而言，就是redis集群的每个master节点都有至少一个slave节点作副本,  在master故障的时候， 集群会自动从该master的选一个slave， 替换故障的master节点，以恢复服务。 
首先明确一下，我们的故障场景：

- redis master本身不可达， 主机挂掉，进程挂掉，或者网络不可达
- redis集群发生分裂，所有节点被分为两部分（我们只讨论这一类最简单场景），一部分有占多数的master节点，我们称之为多数派子集群，另一部分中有占少数的master节点，我们称之为少数派子集群

下面分析一下，这两种故障情况对集群访问的可用性有何影响，集群如何做故障恢复。

#### 3.1 异步复制和写安全 ####

Redis节点之间使用异步复制，即master不等待slave确认收到数据，即给客户端返回写成功。异步复制会导致两种丢失数据的情况：

- master收到了一个写请求，master对客户端返回OK，但由于slave和master之间使用异步复制，该写请求可能在master回复client的时候，还没有同步到slave。如果在写请求同步到slave之前，master挂了，且在它的一个slave被提升为新的master之前一直不可达(master一段时间, TIMEOUTs, 不可达，某个slave会成为新的master)，那么这个写操作就永远的丢失了。这种master节点失效导致丢失数据的情况比较少见，这需要master的失效恰好发生在回复客户端写成功后，写请求复制到slave前，时间窗口比较小。 实际运营中，这取决于master和slave的数据同步延迟。
- 另外一种理论上可能导致写操作丢失的集群失效模式如下：

    *  一个master由于集群分裂而不可达
    * 系统自动故障切换，不可达的master的某一个slave成为新的master，接管了服务
    *  一段时间后，老的master可达了
    * 一个客户端如果使用的是旧的路由表，那么之前挂掉又恢复的master在正式成为slave之前，会错误的接受到写请求


第二种失效模式不太可能发生，因为这要求一个master在与集群失去联系达TIMEOUT s后，在被故障替换成为slave之前，由客户端由于没有更新路由表，而将写请求继续发往该节点，这种可能性不是很大。 
在集群分裂时，发往少数派集群中master节点的写请求丢失的机率大一些，因为集群分裂时间够长的话，它们会被其位于多数派子集群的slave节点故障替换，而这些slave节点在集群分裂期间无法同步master上的新数据。

值得指出的是，一个master节点只有在超过NODE_TIMEOUT 秒无法与被集群通信时，才能被故障替换。如果集群分裂在NODE_TIMEOUT秒内被修复，就不会有任何写请求丢失。当集群分裂持续超过NODE_TIMEOUT秒，那么在集群分裂的头NODE_TIMEOUT秒内，所有发给少数派子集群的写请求会丢失。但是，超过NODE_TIMEOUT无法与多数派通信的话，少数派子集群会拒绝接受写操作，所以数据丢失的最大时间窗口是NODE_TIMEOUT。

#### 3.2 集群分裂丢失数据的概率 ####

在一个有N个master节点的Redis集群中，每个master节点都只有一个slave。当只有一个节点， 无论slave还是master, 被割裂出去，集群完全可用(如果某个slave与集群失去联系，无影响； 如果某个master与集群失去联系，写请求会失败，但不会丢失数据）；当有两个节点被割裂出去，集群可用的概率是1-(1/(N*2-1))(当第一个节点失效后，集群中还剩下N\*2-1个节点，那个没有slave的master节点失效的概率是1/(N\*2-1)， 一旦一个master和其slave同时和集群失去联系，则某些数据就无法访问，新数据也无法写入，导致集群不可用)。例如，一个有5个master节点的集群，每个节点配置一个slave节点，那么两个节点从集群割裂出去后，整个集群不可用的概率是 1/(5\*2-1) = 11.11% 。

#### 3.3 副本迁移 #####

##### ___3.3.1 副本迁移背景和原理___ #####
副本迁移是指，集群将有多个slave的master的某个slave节点分配给集群中没有slave节点的master当slave节点，副本迁移特性在很多场合下进一步增强了集群的可用性。通常在在一个集群中，如果每个master节点都有一个slave节点， 只要任意一个master和它的slave不同时出故障，这个集群就可以正常服务。 但是，由于经年累月积累的软硬件问题，造成某一类故障频发（比如，采购了一批服务器，随着使用时间变长，硬盘故障率越来越高），形成如下场景：

- master节点A由一个slave A1
- master A失效了，A1成为新的master
- 几个小时后， A1也失效了（A1失效和A失效没有因果关系），这个时候，没有其它的节点可以成为新的master, 因为A仍处于宕机状态。此时，整个集群无法对外服务。

如果master和slave之间的关系时固定的，那么解决这个问题的唯一方案就是，为每个master节点挂更多的slave。这个方案不经济，需要运行更多的redis实例，使用更多的内存等等。 然而，如果允许不同的master节点有不同数量的slave，且集群可以在运行过程中调整每个master的slave的数量，就可以通过副本方案解决这个问题。 比如， 集群中由三个master节点A, B, C，A和B分别只有一个slave, A1和B1, 而C节点不同，它有两个slave节点， C1和C2，就可以应用副本迁移：

- masterj节点A失效了， A1成为新的master, 取代了A
- C2被迁移为A1的副本
- 几个小时后A1失效
- C2成为新的master, 取代了A1
- 集群可以继续对外服务

##### ___3.3.2 副本迁移算法___ #####
副本迁移算法保证集群中每个master节点最终都至少有一个slave节点。副本迁移算法分为以下几步：

- 每个slave节点会检测集群中是否存在无good slave的master节点（good slave是指状态不为FAIL的slave节点）
- 当有slave节点探测到满足条件的master节点时，从这些slave节点中选取一个子集（通常只有一个节点）作为副本迁移的候选slave节点， 选取规则如下：

   * 候选slave节点的master节点在集群中拥有最多的good slave节点
   * 候选slave节点有最小的node ID

例如，集群中10个master节点有1个slave， 2个master节点有5个slave， 那么被迁移的slave节点将是2 * 5个slave节点中node ID最小的那个。由于候选节点的确定不涉及选举和协商，只使用节点本地存储的集群拓扑信息，所以当集群尚不稳定时， 有可能不止一个slave节点会认为自己是有最小node ID（实际中不可能发生）的good slave。当这种情况发生时，这些slave都会参与副本迁移，脱离原来master，成为新master的slave，这个会导致它们原来的master失去所有的slave； 当集群稳定时，迁移算法会将一个slave迁移给它原来的master， 最终，所有的master都至少会有一个slave。这种情况对集群影响不大，只是会带来不必要的迁移。 正常情况下，只会把一个slave从它的master(当然该master有多个slave节点）迁移给一个没有slave节点的master。 
这个算法的行为由一个参数cluster-migration-barrier控制， 该参数规定了一个master节点最少得有几个slave节点，才能将其中的一个slave节点迁走。
