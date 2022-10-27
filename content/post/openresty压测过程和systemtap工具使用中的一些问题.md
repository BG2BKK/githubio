+++
date = '2016-02-22T11:15:36+08:00'
draft = false
title = 'openresty压测过程和systemtap工具使用中的一些问题'

+++

一、tegine编译+高版本的httpluamodule
-----------------------------------------------

* tengine-2.1.0的ngx_lua模块随着tengine软件发布，和以往版本的tengine不一样。
* tengine自带的ngx_lua模块版本太老，与openresty相比要差几个版本，导致openresty里的一些好的软件工具不可用。
    * 编译tengine+lua时需要手动指定ngx_lua模块和LuaJIT2.1
        * 新版本ngx_lua能弥补openrestysystemtap在probe某些函数时的错误
            * 关键脚本ngx-lua-exec-time.sxx中第58行@pfunc(ngx_http_lua_free_fake_request)函数找不到
        * 使用LuaJIT2.1能解决lua执行时的stap问题。

```bash
#关键脚本stapxx/samples/ngx-lua-exec-time.sxx中第58行@pfunc(ngx_http_lua_free_fake_request)函数找不到的问题：
2> sudo ./samples/ngx-lua-exec-time.sxx -x 11070
semantic error: while resolving probe point: identifier 'process' at <input>:58:7
        source:       process("/usr/local/nginx/sbin/nginx").function("ngx_http_lua_free_fake_request")
                      ^

semantic error: no match (similar functions: ngx_http_lua_get_request, ngx_http_create_request, ngx_http_free_request, ngx_http_lua_post_subrequest, ngx_http_scgi_create_request)
Pass 2: analysis failed.  [man error::pass2]
```

```bash
#tegine编译选项
CFLAGS="-g -O2" ./configure  --add-module=/path/ngx_openresty-1.7.10.1/bundle/ngx_lua-0.9.15/ --with-luajit-lib=/usr/local/lib/ --with-luajit-inc=/usr/local/include/luajit-2.1/ --with-ld-opt=-Wl,-rpath,/usr/local/lib

make -j16

#新版本ngx_lua在openresty的bundle的ngx_lua-0.9.15/里，也可以从https://github.com/openresty/lua-nginx-module获得

#LuaJIT2.1的源代码也在bundle里，也可以从https://github.com/openresty/luajit2获得。

LuaJIT2.1的编译选项是：
make CCDEBUG=-g -B -j8
make -j16
```

二、axel下载工具和debuginfo.centos.org
-----------------------------------------------

* 我们在安装kernel的debuginfo包的时候，由于只有debuginfo.centos.org上面有rpm包，所以yum源设置为它，但是在国内访问实在是慢，因此推荐使用axel或者mwget下载。mwget顾名思义是多线程的wget工具，下载rpm包非常快，值得推荐。
* 当我们为了系统中的systemtap安装时，需要安装kernel-debuginfo，这时需要注意严格按照自己的kernel版本来，centos系的需要注意是否2.6.32-431.11.2.el6.toa.2.x86_64后面有toa之类的

```bash
#debian系的可以使用dpkg -l linux-image*命令，查看具体内核版本；

huang@ThinkPad-X220:~$ dpkg -l linux-image*
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                   Version          Architecture     Description
+++-======================-================-================-==================================================
un  linux-image            <none>           <none>           (no description available)
un  linux-image-3.0        <none>           <none>           (no description available)
ii  linux-image-3.13.0-24- 3.13.0-24.47     amd64            Linux kernel image for version 3.13.0 on 64 bit x8
ii  linux-image-3.13.0-24- 3.13.0-24.47     amd64            Linux kernel debug image for version 3.13.0 on 64 
ii  linux-image-extra-3.13 3.13.0-24.46     amd64            Linux kernel extra modules for version 3.13.0 on 6
ii  linux-image-generic    3.13.0.24.28     amd64            Generic Linux kernel image

#特别要注意的是3.13.0-24.46、 47、 48，小版本很重要的。
```

三、centos6.4的gcc版本过低导致的问题
-----------------------------------------------
* 在使用stapxx的工具追踪nginx运行情况时，发现有如下情况，比如：

```bash
2> sudo ./samples/ngx-rewrite-latency-distr.sxx -x 11070
semantic error: not accessible at this address [man error::dwarf] (0x44a53b, dieoffset: 0x13a891): identifier '$r' at <input>:67:9
        source:     r = $r
                        ^

Pass 2: analysis failed.  [man error::pass2]

# 这个r是没有问题的，在nginx-systemtap-toolkit/ngx-active-reqs中有类似定义：
my $c = '@cast(c, "ngx_connection_t")';
my $r = '@cast(r, "ngx_http_request_t")';
my $u = '@cast(u, "ngx_http_upstream_t")';
my $p = '@cast(p, "ngx_event_pipe_t")';
```

* 产生原因
    * tengine的DWARF信息不完整
        * 低版本的gcc在O2时优化掉很多东西，而高版本gcc智能的多

* 解决方法有两种
    * 一种方法是将gcc的编译优化选项降低为O0，以前是O2，则这个ngx_http_request_t的符号，即nginx的DWARF信息可以保留下来。
    * 另一种方法是升级gcc，考虑到在线上服务器升级gcc不太好，这里介绍一种[暂时升级gcc的办法](http://ask.xmodulo.com/upgrade-gcc-centos.html)

```bash
$ sudo wget http://people.centos.org/tru/devtools-1.1/devtools-1.1.repo -P /etc/yum.repos.d
$ sudo sh -c 'echo "enabled=1" >> /etc/yum.repos.d/devtools-1.1.repo'
$ sudo yum install devtoolset-1.1
$ scl enable devtoolset-1.1 bash
$ gcc --version
# 通过devtoolset工具可以暂时提高gcc版本，而不更改之前服务器的配置，这个很有效果，高版本的gcc会智能保留symbol。
```

四、apache的压测工具ab升级高版本
-----------------------------------------------

在压测过程中，我们想微观的通过tcpdump抓包分析通信过程。

* 发现ab工具在发起keepalive请求时，在完成-n的请求数后额外的发出一个请求，随后又发出F包关闭连接，导致通信过程多出一次。
* 已有的httpd 2.3自带的ab工具有这个bug，升级httpd2.4后这个问题修复。

五、tcpdump抓取fin包
-----------------------------------------------

```bash
# tcp包里有个flags字段表示包的类型，tcpdump可以根据该字段抓取相应类型的包：
# tcp[13] 就是 TCP flags (URG,ACK,PSH,RST,SYN,FIN)
# Unskilled 32
# Attackers 16
# Pester     8
# Real       4
# Security   2
# Folks      1

#抓取fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 1 != 0 ' -s0  -w fin.cap -vvv
#抓取syn+fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 3 != 0 ' -s0  -w syn_fin.cap -vvv
#抓取rst包：
tcpdump -ni any port 9001 and 'tcp[13] & 4 != 0 ' -s0  -w rst.cap -vvv
```
[参考链接](http://babyhe.blog.51cto.com/1104064/1395489)

六、redis内部监测工具 redis-faina
-------------------------------------------

[redis-faina](https://github.com/facebookarchive/redis-faina)是facebook出品的，用于监控redis内部情况统计的一个工具。使用方法是：


```bash
redis-cli  -p 6379 MONITOR | head -n 100000 | ./redis-faina/redis-faina.py
```

七、压测过程中发现lua获取用户特征时，如果用户特征不存在，比如headers中的UID参数，事实上它是处于ngx.req.get_headers()函数返回值(table类型)中，如果lua提取用户特征时，找不到UID，则会扩大范围去上一级里找，此时性能会大大下降
