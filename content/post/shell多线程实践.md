+++
date = "2016-02-19T10:54:28+08:00"
draft = true
title = "linux shell控制并发进程数实践"

+++

shell脚本多线程应用
====================

* 同事将新上线APP的一部分log交给我，让我统计下这些log中供出现了哪些deviceid
    * 采用awk就可以实现这部分匹配和统计功能，还是比较简单的
    * 挑战在于，这批log文件非常多非常大，单进程工作处理起来非常的慢。因此我想到了多进程方式。
    * 以往需要使用shell来实现多进程时，采用以下模板

```bash
for seq; do
    {
        task
    }&
done
```
    * 当任务较为简单，并发数不多时，这招很管用。然而现在log文件有好几K个，grep处理文件非常耗CPU，上述模板将会按文件数启动进程，系统的CPU Load一跃而起，泪目。
    * 此时的情况是，解决问题的思路和方向没有错，方式上还需要改进。关键在于：控制并发任务，合理使用CPU。
    * 如何在shell中控制并发进程数呢，我找到这样一个帖子
    * http://blog.sciencenet.cn/blog-548663-750136.html

* shell脚本中控制并发任务数的大体方式是：
    * 初始化token池，形成一定token空间，又能在为空时阻塞想拿token的进程
        * 生成一个数组，执行任务前先从数组中获得一个元素，能够获得就继续执行，否则阻塞。数组大小最好为CPU核心数。任务执行完成后将元素放回，以供别的进程使用。
        * 生成一个阻塞访问的管道pipe，先向管道中写入若干行，任务执行前从管道中获取token，任务结束后放回。
* 最终脚本如下所示
    * 我在理解这个脚本的时候感到吃力，比如exec、read等既熟悉又陌生的指令，毕竟没写过shell脚本
    * 后来我发现，man bash和man sh是第一手消息资料
        * if -p $directory中，-p是什么意思，在man bash的CONDITIONAL EXPRESSIONS中
        * read -u999中，-u又是什么意思，在man bash的 SHELL BUILTIN COMMANDS的read指令中
        * exec 999<>$Pfifo中，man bash的REDIRECTION小节中

```bash
#!/bin/sh

awk=/usr/bin/awk
uniq=/usr/bin/uniq

Nproc=24

#$$是进程pid
Pfifo="/tmp/$$.fifo"
mkfifo $Pfifo

#以999为文件描述符打开管道,<>表示可读可写
exec 999<>$Pfifo
rm -f $Pfifo

#向管道中写入Nproc行,作为令牌
for((i=1; i<=$Nproc; i++)); do
    echo
done >&999

echo '' > out
echo '' > ooo
filenames=`ls *.log`
for filename in $filenames; do
#从管道中取出1行作为token，如果管道为空，read将会阻塞
#man bash可以知道-u是从fd中读取一行
    read -u999

    {
    #所要执行的任务
        `$awk -F',' '/did/ {for(i=1;i<=NF;i++) if($i ~ /did/) print $i i}' $filename | $awk -F':' '{print $2}' | $awk -F'"' '{print $2}'  | $uniq | $awk '{count[$1]++}END{for(name in count)print name >> "out"}'` && {
            echo "$filename done"
        } || {
            echo "$filename error"
        }
        sleep 1
    #归还token
        echo >&999
    }&

done

#等待所有子进程结束
wait 

#关闭管道
exec 999>&-

echo `$awk '{count[$0]++}END{for(name in count)print name}' out > ooo; awk 'END{print NR}' ooo`
```
* 单进程方式处理这些log需要3个小时，而控制并发进程数的话只需要10分钟不到。
    * 可见大部分计算资源都浪费在CPU切换上了。


参考链接
====================

linux shell 和 lsof 等工具使用的一些tips
---------------------------------
[linux shell数据输入输出的重定向分析](http://www.cnblogs.com/chengmo/archive/2010/10/20/1855805.html)      
[lsof 一切皆文件](http://linuxtools-rst.readthedocs.org/zh_CN/latest/tool/lsof.html)

文件描述符与进程间通信
-------------------------------
[IO重定向和文件描述符](http://adelphos.blog.51cto.com/2363901/1598570)        
[文件描述符与进程间通信的关联](http://www.cnblogs.com/GODYCA/archive/2013/01/05/2845618.html)

linux shell cocurrency并发控制
-----------------------------
[Bash脚本实现批量作业并行化](http://blog.sciencenet.cn/blog-548663-750136.html)
[A script for running processes in parallel in Bash](https://pebblesinthesand.wordpress.com/2008/05/22/a-srcipt-for-running-processes-in-parallel-in-bash/)

awk&sed
-----------------
[sed&awk入门 一](http://kodango.com/sed-and-awk-notes-part-1)       
[sed&awk入门 二](http://kodango.com/sed-and-awk-notes-part-2)        
[awk用法汇总](http://www.cnblogs.com/dong008259/archive/2011/12/06/2277287.html)


