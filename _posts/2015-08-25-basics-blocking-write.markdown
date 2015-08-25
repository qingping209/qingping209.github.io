---
layout: post
title: "一个非阻塞写的坑"
date: 2015-08-25 22:23
categories: networking 
---
先看一份代码:
 
    // 通过iter遍历数据库，逐条将数据发送到远程服务器上
    Iterator *iter = beginIterator();
    while (isValid(iter)) {
        size_t keylen_, vallen_;
        char *key_ = (char *)iterKey(iter, &keylen_);
        char *val_ = (char *)iterValue(iter, &vallen_);
            
        // 从key_和val_建立一个数据缓冲区buf, 大小为buflen
        size_t n = write(fd, buf, buflen);
        
        iterNext(iter);
    }

说下背景: 服务器和客户端是使用行文本协议交互通信，上面的代码在调用write之前，已经从key\_, val\_构建了一个符合协议规范的文本行；远端服务器收到数据后，会逐行解析。 这两个逻辑没有问题。 但是，在远端服务器上偶尔能观察到，行断掉的现象。

什么情况? 不检查系统调用的返回函数不是个好习惯，于是在write下面加上：

    if (n < buflen) {
        fprintf(stdout, "%s\n", strerror(errno);
    }

首先第一反应是可能写缓冲区不够了，数据没写完。 改了再运行一次，输出“resource temporarily unavailable”，确实是的。没写完的话，正常情况下应该是要阻塞的，难道没有阻塞么？ 看看fd的属性, 
    
    int flags = fcntl(fd, F_GETFL);
    fprintf(stdout, "%s", flags & O_NONBLOCK);

没错，fd被设置为非阻塞状态。改为阻塞的：

    int flags = fcntl(fd, F_GETFL);
    flags &= ~O_NONBLOCK;
    fcntl(fd, F_SETFL, flags);

上述现象消失。
-- eof --
    





