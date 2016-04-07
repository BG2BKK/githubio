+++
date = "2016-03-14T16:36:14+08:00"
draft = true
title = "openresty学习过程中的一些tips"

+++

ngx_lua中判断table为空
-------------------------------------
* lua的table中，有两类kv，一类是以数字为index，比如{'abc', 'efg'}中，{k=1, v=abc}, {k=2, v=efg}，另一类以自己kv存储，比如{['abc'] = 'efg'}，k为abc的元素，v为efg
* #table中，#标识符是返回以数字为index的key，从1开始算，连续的key的数量

```lua
local t = {}
t[1] = 'a'
t[2] = 'b'
t[20] = 'c'

print(#t)
2
```

* 因此#号不能获得table的真实大小，也不能用于判断table是否为空
* table.maxn(tab)，maxn返回table中以数字为key的元素中，数字最大的那个

```lua
local t = {}
t[1] = 'a'
t[2] = 'b'
t[20] = 'c'

print(table.maxn(t))
20
```

* next就是pairs遍历table时用来取下一个内容的函数，因此next(tab)可以用来判断table是否为空，如果next(tab)返回为nil的话，说明第一个元素不存在，所以该table为空

* [参考链接](https://moonbingbing.gitbooks.io/openresty-best-practices/content/lua/not_nill.html)

ngx_lua中的参数获得
-------------------------------------

ngx_lua中获得req参数有如下几个方法：

* ngx.var.arg_city 获取city参数
    * host:port/uri?city=abc    ***ngx.var.arg_city = abc***
    * host:port/uri?city=       ***ngx.var.arg_city = ''***, and its length is 0
    * host:port/uri?city        ***ngx.var.arg_city = nil***，cause it doesn't exist yet

* ngx.req.get_headers()['city']，获取http请求头中的city参数
    * host:port/uri -H 'city:abc'   ***结果为abc***
    * host:port/uri -H 'city:'      ***结果为nil***


- [thread](https://groups.google.com/forum/#!topic/openresty/fQvG_TvDAvU)

    - 虽然init_worker_by_lua阶段不能使用cosocket，不过可以先通过一个timer（定时时间为0让其立即调用）来发出对外的socket io操作，以实现一些初始化的目的。
    - openresty的两个缓存中，ngx shared dict是跨worker共享的，是一个单纯的kv缓存；预计接下来会有patch能够支持lpush等redis操作；lua-resty-lrucache是每个worker的Lua VM空间内缓存，不能跨worker共享，优点是可以存储所有lua对象，比如table，而不需要序列化和反序列化

- worker 启动时，upstream 是空的，即 _M.data={}，所以这个时候是不能提供服务的。所以每次 reload config 都会导致一段时间内服务不可访问。
    - 在 init_worker_by_lua 执行 cosocket 相关的 API 是不允许的（后期可能会添加支持），但可以调用标准 SOCKET 完成初始化加载，例如借助 luasocket 完成数据源获取并初始化 _M 。

- 不清楚 init_worker_by_lua 里是否可以进行文件操作？
    - 是可以的，这个确定。

- 在ngx lua性能分析方面，agentzh提出一系列的工具，主要是nginx-systemtap-toolkit和stapxx两个工程。我们使用的脚本，[说明文档](https://groups.google.com/forum/#!topic/openresty/bOwgPymXQzg)

```bash
引用：
	我们有一整套的基于 systemtap 的工具链可以用于在线或者离线的性能分析。 
	你的 nginx 进程的 CPU 使用率如果很高的话，可以使用 C 级别的 on-CPU 时间火焰图工具对你最忙的 nginx worker 进程进行采样： 
	    https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt 
	如果你的 nginx 进程的 CPU 很低，但请求延时很高，则有两种可能： 
	1. 你的 nginx 阻塞在了某些阻塞的 IO 操作（比如文件 IO）或者系统的同步锁上，此时你可以使用 C 级别的 off-CPU 
	时间火焰图工具对某个典型的 nginx worker 进程进行采样： 
	   https://github.com/agentzh/nginx-systemtap-toolkit#sample-bt-off-cpu 
	如果你发现 Lua 代码占用了大部分的 CPU 时间，则可以进一步使用 ngx-lua-exec-time 工具加以确认： 
	    https://github.com/agentzh/stapxx#ngx-lua-exec-time 
	进一步地，你可以使用 Lua 代码级别的 on-CPU 火焰图工具在 Lua 层面上分析 CPU 时间的分布。如果你使用的是 LuaJIT 
	2.0.x，则可以使用下面这个工具进行采样： 
	    https://github.com/agentzh/nginx-systemtap-toolkit#ngx-sample-lua-bt 
	如果你使用的是 LuaJIT 2.1，则可以使用 lj-lua-stacks 工具进行采样： 
	    https://github.com/agentzh/stapxx#lj-lua-stacks 
	2. 你的 nginx 通过 ngx_lua 的 cosocket 或者 ngx_proxy 这样的 upstream 
	模块和上游服务进行通信时，上游服务的延时过大。此时你可以分别使用 
	ngx-lua-tcp-recv-time、ngx-lua-udp-recv-time 以及 ngx-single-req-latency 
	工具进行分析： 
	    https://github.com/agentzh/stapxx#ngx-lua-tcp-recv-time 
	    https://github.com/agentzh/stapxx#ngx-lua-udp-recv-time 
	    https://github.com/agentzh/stapxx#ngx-single-req-latency  



    - 我们主要使用四个工具来生成火焰图以分析性能，sample-bt、sample-bt-off-cpu、ngx-sample-lua-bt 和 lj-lua-stacks。
    - 实际使用中命令：
```

```bash
1、sudo ./sample-bt -p 8736 -t 20 -u -a '-DMAXSKIPPED=10000' > a.bt
2、sudo ./sample-bt-off-cpu -p 8736 -t 20 -u > b.bt
3、sudo ./ngx-sample-lua-bt --luajit20 -p 44252 -t 20 > c.bt
4、sudo ./samples/lj-lua-stacks.sxx --skip-badvars -x 44250 -I tapset/ > d.bt
```

    - 通过a.bt生成火焰图

```bash
	../FlameGraph/stackcollapse-stap.pl a.bt | ../FlameGraph/flamegraph.pl > a.svg
```

    - 通过脚本ngx-lua-conn-pools来追踪ngx lua connection pool的工作情况。		

```bash
    sudo ./ngx-lua-conn-pools  --luajit20 -p 28261
```

