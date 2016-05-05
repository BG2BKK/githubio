+++
date = "2016-03-25T20:16:51+08:00"
draft = false
title = "nginx gdb utils的编译 安装和使用"

+++


* 1、想用[nginx-gdb-utils](https://github.com/openresty/nginx-gdb-utils)来监控ngx_lua的内存使用情况
* 2、在CentOS 6.5上，gdb为7.2，python为2.6，没有一个符合的，想强上，没上了，只能在7上搞
* 3、CentSO 7的gdb是7.6，版本也很老，对于nginx-gdb-utils来说。python倒是2.7，可以搞
* 4、[gdb](http://www.linuxfromscratch.org/blfs/view/svn/general/gdb.html)需要编译安装，首先下载gdb-7.11

```bash
    cd gdb-7.11
    ./configure --with-python=python2.7
    报错，报python2.7找不到，换成

    ./configure --with-python=/usr/bin/python2.7
    依然报错
    很奇怪，难道不是要python2.7吗，怎么报找不到。
    后来才知道，需要python，要的不是python2.7的可执行文件，而是python的库文件等待

    sudo yum install python2.7-devel

    make -j24 --with-python
    注意我这里不用写--with-python=blahblah了
    终于不报错了

    make -C gdb install 
    报错，报没有makeinfo的错误，经查，makeinfo是texinfo的一部分，用来生成说明文档的，因为它而不能安装，蛋疼

    sudo yum install texinfo

    make -C gdb install
    安装在/usr/local/bin/gdb

```

* 5、将nginx-gdb-utils写入gdb初始化文件中，这样以后就不用每次加载py文件了

```bash
vim ~/.gdbinit

directory /path/to/nginx-gdb-utils

py import sys
py sys.path.append("/path/to/nginx-gdb-utils")

source luajit20.gdb
source ngx-lua.gdb
source luajit21.py
source ngx-raw-req.py
set python print-stack full
```

但其实一般而言我们都是用root用户的，所以在sudo或者直接是root用户下时，需要重新写~/.gdbinit，这时应该是在/root/.gdbinit了

* 6、/usr/local/bin/gdb -p 12345

* 7、lgcstat

发现报一些函数或者变量找不到，比如Lgref找不到，这个原因是相关软件没有把用 -g 选项把符号编译进去

* 8、对于LuaJit而言，make CCDEBUG=-g -B -j8
* 9、对于lua-cjson而言，make CCDEBUG=-g -B -j8
* 10、对于tengine、nginx或者openresty而言，CFLAGS="-g -O2" ./configure 
* 
* 11、自此就可以愉快的玩耍了。这些工具还是很有意思的。
