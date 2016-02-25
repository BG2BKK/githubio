+++
date = "2016-02-19T10:54:28+08:00"
draft = true
title = "shell多线程实践"

+++

shell脚本多线程应用
====================


shell&cocurrency
------------------

http://blog.sciencenet.cn/blog-548663-750136.html
http://blog.chinaunix.net/uid-8478094-id-3995108.html

awk&sed
-----------------

http://kodango.com/sed-and-awk-notes-part-1
http://kodango.com/sed-and-awk-notes-part-2
http://www.cnblogs.com/dong008259/archive/2011/12/06/2277287.html

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
#从管道中取出1行
    read -u999

    {
        `$awk -F',' '/did/ {for(i=1;i<=NF;i++) if($i ~ /did/) print $i i}' $filename | $awk -F':' '{print $2}' | $awk -F'"' '{print $2}'  | $uniq | $awk '{count[$1]++}END{for(name in count)print name >> "out"}'` && {
            echo "$filename done"
        } || {
            echo "$filename error"
        }
        sleep 1

        echo >&999
    }&

done

wait 
exec 999>&-

echo `$awk '{count[$0]++}END{for(name in count)print name}' out > ooo; awk 'END{print NR}' ooo`

```

linux shell数据输入输出的重定向分析
--------------------------------------
http://www.cnblogs.com/chengmo/archive/2010/10/20/1855805.html
http://www.cnblogs.com/GODYCA/archive/2013/01/05/2845618.html

lsof使用的一些tips
---------------------------------
http://linuxtools-rst.readthedocs.org/zh_CN/latest/tool/lsof.html
http://linuxtools-rst.readthedocs.org/zh_CN/latest/tool/pstack.html

