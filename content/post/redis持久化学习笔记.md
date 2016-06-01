+++
date = "2016-02-26T14:56:41+08:00"
draft = false
title = "redis持久化学习笔记"

+++

* 虽然网络上关于redis持久化的相关内容数不胜数，但是一来作为我的学习笔记，好记性不如烂笔头；二来除去官方redis之外，很多有意义的修改或补充都非常值得讨论，所以我想做一下记录。

* 持久化用于重启后的数据恢复，而持久化的引入导致了redis可能产生的性能抖动

* redis持久化的[ 两种方法 ](http://www.cnblogs.com/zhoujinyi/archive/2013/05/26/3098508.html)
    * RDB方式
        * RDB方式是redis的默认持久化方式
        * 快照RDB持久化过程:
            * redis调用fork，产生子进程
            * 父进程继续接受用户请求；子进程负责将内存内容写入临时文件，写入完成后rename为dump文件，实现替换
            * 在子进程写内存内容期间，父进程如果要修改内存数据，os将会通过写时复制为父进程创建副本，所以此时子进程写入的仍然是fork时刻的整个数据库内容
        * 不足之处在于:
            * 如果redis出现问题崩溃了，此时的rdb文件可能不是最新的数据，从上次RDB文件生成到redis崩溃这段时间的数据全部丢掉。
            * 产生快照时，redis最多将占用2倍于现有数据规模的内存，因此当内存占用过多时，RDB方式可能导致系统负载过高，甚至假死。（有个说法是，当redis的内存占用超过物理内存的3/5时，进行RDB主从复制就比较危险了）

        * 主从复制过程
            * 第一次同步
                * slave向master发送sync同步请求，master先dump出rdb文件，并将其全量传输给slave；master将产生rdb文件之后这段时间内的修改命令缓存起来，并发送给slave。首次同步完成。
            * 第二次及以后的同步实现方式：
                * master将变量的快照（有修改的变量）直接实时发送给slave。
                * 如果发生断开重连，则重复第一步第二步
            * reids-2.8版本之后，重连后进行第一步时，不用全量更新了。

    * AOF方式
        * AOF方式持久化过程
            * Append Only File
            * Redis将每次收到的命令都追加到文件中，类似于mysql的binlog；当redis重启时重新执行文件中的所有命令来重建数据
            * 如果将所有命令不加甄别的都写入文件中，持久化文件会越来越大，比如INCR test命令执行100次，效果与SET test 100一样。此时需要进行rewrite，合并命令。
            * Redis提供了bgrewriteaof命令，执行过程与产生RDB文件的机制类似，fork出的子进程将内存中的数据以命令的方式重写持久化文件。本质上讲，该命令是将数据库中所有数据内容以命令的方式重写进新的AOF文件

        * AOF方式之我的想法
            * AOF方式是redis在收到命名后将命令写入文件内，如果redis发生故障，重启时直接读取AOF文件重新执行命令即可恢复，可以克服RDB方式的缺点
            * bgrewriteaof指令是对AOF方式的一次优化，执行bgrewriteaof命令时是根据此时数据库内容来写入AOF文件，并替换旧的AOF文件。这个过程与RDB快照产生方式一样

    * RDB方式和AOF方式的对比
        * RDB方式恢复起来快，而AOF方式需要一条条命令执行
        * RDB文件不需要经过编码，是数据库内容的直接克隆，所以文件比较小；而AOF文件内是一条条命令，需要依次执行
        * RDB文件可能会丢失部分数据，而AOF则专门解决这个问题

    * 选择哪种方式
        * 官方推荐：
            * 如果想要很高的数据保障，则同时使用两种方式
            * 如果可以接受数据丢失，则仅使用RDB方式
        * 通常的设计思路是
            * 利用replication机制弥补持久化在性能和设计上的不足
            * master上不做RDB和AOF，保证读写性能
            * slave同时开启两种方式，保证数据安全性


* Redis数据恢复过程
    * AOF优先级高于RDB方式，如果同时配置了AOF和RDB，AOF生效

```cpp
    void loadDataFromDisk(void) {
        long long start = ustime();
        if (server.aof_state == REDIS_AOF_ON) {
            if (loadAppendOnlyFile(server.aof_filename) == REDIS_OK)
                redisLog(REDIS_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
        } else {
            if (rdbLoad(server.rdb_filename) == REDIS_OK) {
                redisLog(REDIS_NOTICE,"DB loaded from disk: %.3f seconds",
                    (float)(ustime()-start)/1000000);
            } else if (errno != ENOENT) {
                redisLog(REDIS_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
                exit(1);
            }
        }
    }
```


* [持久化和RDB文件格式](http://blog.nosqlfan.com/html/4039.html)
