---
layout: post
title:  "twemproxy支持任意multibulk的回包"
date:   2018-10-02 15:00:10
categories: twemproxy 
tags:源码
---

我们知道twemproxy回后端回包的解析支持不够全面的, 很大原因是当初在开发的时候，redis命令返回的结果是足够简单的，只有简单的字符串，数字，错误，以及对hscan/scan/zscan的支持(简单的两元素multibulk: 一个integer + 一个simlebulk)。后来，感觉twemproxy的维护不是那么活跃了，以至于后来redis增加了一些新的命令, twemproxy是支持不了的。

但是，twemproxy依然是有它独特的价值的。个人看来，redis集群的去中心化方案(redis cluster)和使用代理的中心的方案(twemproxy)， 各自有各自的有点，也各自有各自的毛病此处就不一一展开。让twemproxy保持对redis新命令的支持，也是有持续的价值的。

https://github.com/twitter/twemproxy/pull/565， 我最近提的这个pr, 让twemproxy可以支持任意合法格式的redis回包，以后对于任意redis新增命令的支持，只需要在nc\_message.h新增相应的msg\_req类型，在proto/nc\_redis.c里面新增该命令解析的支持，理论上就可以工作了。顺便增加了对geo命令的支持。 同时，在编写测试用例的时候，给redis-py提了个pr: https://github.com/andymccurdy/redis-py/pull/1035, 修了一个无法处理geodis返回$-1的bug。

多说一句， twemproxy确实不怎么活跃了，我提了PR#565以后，一个韩国同行，给我回复了，提示我PR#565跟PR#395重复了，这个确实蛮尴尬的。不过我后来我抱着好奇的心态， 把PR#395弄下来测试了一把， 发现我构造的有几个特殊的case是跑不过的，这些case我作为附件上传到了PR#565了。

估计这个PR在可以想象的将来不会被merge的。有兴趣或者有问题的朋友可以一起在github上面交流。。。

