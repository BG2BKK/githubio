+++
date = '2016-03-14T16:36:14+08:00'
draft = false
title = 'openresty学习过程中的一些tips'

+++

openresty中如何写redis或者mysql的wraper
--------------------------------------

* [参考资料](http://zivn.me/2013/08/05/Something-about-Openresty/)

ngx_lua获取post字段参数
-------------------------------------

在用户请求为POST方式时，如果想获取post中的各参数字段，比如post数据为 "uid=100&ip=10.13.112.53"，此时想获取该字段的话，可以调用[ngx.req.get_post_args](https://github.com/openresty/lua-nginx-module#ngxreqget_post_args)函数。按惯例返回table类型，post数据的各字段为table的key

```lua
function get_uid()
	local args = ngx.req.get_post_args()
	local uid = args['uid']
	return uid
end
```

关于HTTP的POST提交数据的方式，网上有很多讨论

* [四种常见的POST提交数据方式](https://imququ.com/post/four-ways-to-post-data-in-http.html)
	* application/x-www-form-urlencoded
	* multipart/form-data
	* application/json
	* text/xml
* HTTP [header头的一些字段](http://jaseywang.me/2012/03/03/http-headers-%E9%83%A8%E5%88%86%E5%AD%97%E6%AE%B5%E7%AC%94%E8%AE%B0/)

ngx_lua同样提供了读写HTTP请求中[uri参数](https://github.com/openresty/lua-nginx-module#ngxreqget_uri_args)，读写HTTP请求中的[HEADER头部](https://github.com/openresty/lua-nginx-module#ngxreqget_uri_args)，这些在ngx_lua开发中为我们提供了丰富的工具，非常好的功能。

最后回到主题，当我读出uid字段后，有时候会发现报错"requesty body in temp file not supported"，原因在于nginx会将用户请求的body字段缓存起来，如果超出缓存大小，则将用户body数据写到文件中；而ngx.req.get_post_args()是不支持从文件中读取数据的。因此解决办法是：适当加大 [nginx 的 client_body_buffer_size 配置](http://wiki.nginx.org/HttpCoreModule#client_body_buffer_size), 当 client_body_buffer_size 配置为和 client_max_body_size 一样大时，nginx就不会把请求体缓冲到文件系统了（但也要仔细内存占用）。

对于client_max_body_size来说，

```bash
Syntax:		client_max_body_size size;
Default:	client_max_body_size 1m;
Context:	http, server, location

Sets the maximum allowed size of the client request body, specified in the “Content-Length” request header field. If the size in a request exceeds the configured value, the 413 (Request Entity Too Large) error is returned to the client. Please be aware that browsers cannot correctly display this error. Setting size to 0 disables checking of client request body size.
```
设置允许的client body最大值，对http server来说是种保护。

对于client_body_buffer_size来说，

```bash
Syntax:		client_body_buffer_size size;
Default:	client_body_buffer_size 8k|16k;
Context:	http, server, location

Sets buffer size for reading client request body. In case the request body is larger than the buffer, the whole body or only its part is written to a temporary file. By default, buffer size is equal to two memory pages. This is 8K on x86, other 32-bit platforms, and x86-64. It is usually 16K on other 64-bit platforms.
```
如果client body size大于默认值，则nginx将会把body缓存在文件中。将client body buffer size设置为和client_max_body_size一样大，nginx将不会把它写进文件中。

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

