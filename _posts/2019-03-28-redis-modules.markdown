---
layout: post
title: "redis modules"
date: 2019-03-28
categories: redis
tags: redis
---



### Redis Modules简介

redis module是redis 4.0推出的一项重大功能， redis 模块的出现，使得redis的生态进一步的繁荣了。 redis module的核心理念是，以模块的方式，为redis server开发新的功能，新功能的代码和redis server核心功能的代码完全隔离，互不影响， 换句话说，**redis server的代码量并不会随着功能的增多而膨胀, 核心模块的开发和质量可控，同时又能满足社区对功能的多样化的需求**。如何做到这点？ redis server通过暴露一系列的核心API， 为模块开发者提供redis server的核心基础能力, 这些基础能力包括，获取key, 写入key, 复制，持久化，集群，lua, pubsub, blocking等等。开发者通过核心API，实现特定功能，通过加载模块的方式， 赋予redis server新的能力，满足自身的业务需求。

理解一个模块如何正常工作，我们需要回答如下几个问题：

- 如何加载模块到redis-server中，只有加载了，模块才能工作？
- 如何接受请求， 如何把命令映射到模块中的函数，调用模块中的逻辑?
- 如何把数据放到redis-server中，如果从redis-server中获取key?
- 如何把请求封装好，返回给客户端?
- 如何做持久化，如何做主从同步，如何过期，如何选择db?
- 如何使用lua script, 如何实现blocking, 如果做pubsub等?
- 为什么要单独封装一套API?

简单来说，就是在redis-server中能做的事情， 在module中也能做， 不能少， 这样module才能作为一等公民发挥出redis的全部能力。 那么，如何做？ 很简单， 把redis的核心基础能力以API的方式开放出来，给module调用。

### 理解Redis Module的几个问题

#### Q1: 如何加载模块到redis-server中，只有加载了，模块才能工作？
A1:  使用module load命令从so中加载模块。module load会去dlsym这个so, 找到so中的RedisModule_OnLoad函数， 然后调用这个onload，把模块注册到redis-server中。所以每个redis模块必须实现一个RedisModule_OnLoad的函数。在RedisModule_OnLoad里面，做两件事情，初始化模块和注册新模块中的命令。初始化模块通过调用RedisModule_Init，注册新命令通过RedisModule_CreateCommand。 RedisModule_Init通过调用RM_GetApi， 把所有的核心API注册到server.moduleapi中，如果没有注册的话； 通过RM_SetModuleAttribs，为新模块分配一个RedisModule对象，做好初始化。 例如设置模块名， 设置版本号， API版本等等。 初始化完成以后，就注册模块中的命令， 通过调用RM_CreateCommand,  注册新模块中的命令命令和注册原生命令本质上是一样的，dictAdd(server.commands, cmdname, redisCommand). 但是注册模块的命令时需要保存模块相关的信息， 使用RedisModuleCommandProxy来记录这些信息

`struct RedisModuleCommandProxy {
    struct RedisModule *module;               // 所属模块
    RedisModuleCmdFunc func;                  // 真正实现命令逻辑的函数
    struct redisCommand *rediscmd;            // 实际被注册到server.commands中的对象
};
cp = zmalloc(sizeof(*cp));
cp->module = ctx->module;
cp->func = cmdfunc;
cp->rediscmd = zmalloc(sizeof(*rediscmd));
cp->rediscmd->name = cmdname;
cp->rediscmd->proc = RedisModuleCommandDispatcher;
cp->rediscmd->arity = -1;
cp->rediscmd->flags = flags | CMD_MODULE;
cp->rediscmd->getkeys_proc = (redisGetKeysProc*)(unsigned long)cp;
cp->rediscmd->firstkey = firstkey;
cp->rediscmd->lastkey = lastkey;
cp->rediscmd->keystep = keystep;
cp->rediscmd->microseconds = 0;
cp->rediscmd->calls = 0;
dictAdd(server.commands,sdsdup(cmdname),cp->rediscmd);
dictAdd(server.orig_commands,sdsdup(cmdname),cp->rediscmd);`

#### Q2: 如何接受请求， 如何把命令映射到模块中的函数，调用模块中的逻辑?

A2: 我们知道， redis的主题工作流程很简单， 从客户端的tcp socket中读取数据， 根据resp解析出命令，根据命令名到server.commands中找到对应的redisCommand对象，调用其proc函数(proc是个函数指针，其指为实际实现具体命令的函数)。module中的命令得redisCommand对象的proc函数并不是指向实际实现该命令逻辑的函数，而是指向RedisModuleCommandDispatcher这个函数:

`void RedisModuleCommandDispatcher(client *c) {
    RedisModuleCommandProxy *cp = (void*)(unsigned long)c->cmd->getkeys_proc;
    RedisModuleCtx ctx = REDISMODULE_CTX_INIT;
    ctx.module = cp->module;
    ctx.client = c;
    cp->func(&ctx,(void**)c->argv,c->argc);
    moduleHandlePropagationAfterCommandCallback(&ctx);
    moduleFreeContext(&ctx);`

getkeys_proc在redis-server中是用来取key的位置的，在module的命令中，不需要这个，用它来存放RedisModuleCommandProxy对象;  通过命令名获取到command以后，从command中获取RedisModuleCommandProxy,  从RedisModuleCommandProxy获取实际实现命令得函数，以及模块的环境，这些执行命令所必须的参数，调用即可。

#### Q3: 如何把数据放到redis-server中，如果从redis-server中获取key?

A3:  数据放到redis-server中通过dbAdd; 从redis-server中获取key通过RM_OpenKey, RM_OpenKey中通过lookupRead或者lookupWrite来获取key, 并且把获取到的key放到RedisModuleKey的环境中:

`struct RedisModuleKey {
    RedisModuleCtx *ctx;
    redisDb *db;
    robj *key;      /* Key name object. */
    robj *value;    /* Value object, or NULL if the key was not found. */
    void *iter;     /* Iterator. */
    int mode;       /* Opening mode. */
    uint32_t ztype;         /* REDISMODULE_ZSET_RANGE_* */
    zrangespec zrs;         /* Score range. */
    zlexrangespec zlrs;     /* Lex range. */
    uint32_t zstart;        /* Start pos for positional ranges. */
    uint32_t zend;          /* End pos for positional ranges. */
    void *zcurrent;         /* Zset iterator current node. */
    int zer;                /* Zset iterator end reached flag
                               (true if end was reached). */
};`

#### Q4 - Q6: 如何在模块中回包，选择DB, 设置过期，执行脚本，持久化， 做blocking操作等等？

A4-A6: 都有相应的核心API提供响应的能力

#### Q7: 为什么要单独封装一套API?

A7:  在执行module中的命令时，需要保存module的context

