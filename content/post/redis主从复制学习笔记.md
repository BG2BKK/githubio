+++
date = "2016-02-26T16:15:31+08:00"
draft = false
title = "redis主从复制学习笔记"

+++

* redis[主从复制](http://qifuguang.me/2015/10/18/Redis%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6/)
    * 在介绍REDIS的RDB持久化方式时，我们提到了主从复制的实现过程
        * 第一次同步，slave发送sync命令开始同步，master生成快照全量发送给slave，快照生成之后的变更命令缓存起来，也一块发送给slave
        * 第二次及以后的同步，master收到命令后修改数据，并将数据修改后的结果同步给slave；如果此时发生断开重连情况，则重新进行第一步操作；
        * redis-2.8版本后，如果发生断开重连，则进行增量传输，而不是全量传输
    * 这里有两个值得介绍的地方
        * 不论是否设置RDB持久化，主从复制都会产生快照
        * redis.conf中不设置save 900 1等配置时，只是不自动产生快照，如果执行save，还是会产生快照的
        * 主从复制时，master执行完命令后会立刻将结果返回client，而不是等待同步给slave后再返回给client。这里可能会有一个不一致窗口，如果主从在master执行完指令和同步给client之间断开，这里会发生不一致现象，需要注意
    * 主从复制的常见设计思路
        * 用于保证数据持久化
            * master正常读写，不设置RDB或者AOF的持久化
            * slave设置RDB和AOF方式持久化，保证数据安全
        * 基于实用目的的主从复制
            * master设置为只写模式，将结果同步给slave
            * slave设置为只读模式，作为系统缓存


一般而言，了解以上知识，只能算是对redis的主从同步机制有了整体和粗糙的认识，而主从复制过程中发生了什么，如果能充分挖掘，也能感受到作者当时的苦心孤诣，学习到新的知识。

好吧，装逼的话说完了，我想说，面试58的分布式存储工程师的时候，我被问到了这些问题，这些问题问的非常好，我需要补补课！

replication of redis
-----------------------------

[replication](http://redis.io/topics/replication)

抓包发现，首次同步时，master将rdb文件整体发给slave，之后master有变动的话，master将会同步命令给salve，比如直接同步set key val命令。


v2.8之后的redis增量复制
-------------------------

* redis slave有6个状态:
	* REDIS_REPL_NONE
		* 收到 slaveof host port，或者初始化进行主从同步时，进入下一状态
	* REDIS_REPL_CONNECT
		* redis会周期的执行replicationCron函数，而在该状态下，replicationCron函数将创建socket连接master，进入下一状态
	* REDIS_REPL_CONNECTING
		* 同步socket将发送PING消息给master，随后关闭该socket的写事件，进入下一状态
	* REDIS_REPL_RECEIVE_PONG
		* socket收到PONG后，进行redis认证，认证成功后发送SYNC命令请求同步；新建临时文件接收rdb文件；进入下一状态
	* REDIS_REPL_TRANSFER
		* socket读取rdb数据，写到rdb文件
	* REDIS_REPL_CONNECTED
		* 完全获取rdb文件后，进入CONNECTED状态；把master当做一个特殊的client，接收写命令

* redis master 有4个状态
	* REDIS_REPL_WAIT_BGSAVE_START
	* REDIS_REPL_WAIT_BGSAVE_END_
	* REDIS_REPL_WAIT_BGSAVE_BULK
	* REDIS_REPL_WAIT_BGSAVE_ONLINE

* 增量复制
	* 增量复制的区别，同步时，客户端先做增量复制，不然时间太久也只能全量复制了。
	* [rdb文件格式](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-RDB-Dump-File-Format)
