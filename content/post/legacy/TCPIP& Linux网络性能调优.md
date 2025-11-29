+++
date = '2016-05-06T16:03:46+08:00'
draft = true
title = 'TCPIP& Linux网络性能调优'

+++

time_wait相关
-----------------------

* tcp_tw_reuse 
	* 允许重用

* tcp_tw_recycle
	* 允许快速回收

* net.ipv4.tcp_max_tw_buckets
	* 系统可以保持timewait状态socket连接的最大数量。桶需要小一点，强迫系统回收端口

```bash
	Maximal number of timewait sockets held by system simultaneously.
	If this number is exceeded time-wait socket is immediately destroyed
	and warning is printed. This limit exists only to prevent
	simple DoS attacks, you _must_ not lower the limit artificially,
	but rather increase it (probably, after increasing installed memory),
	if network conditions require more than default value.
```

* net.ipv4.tcp_timestamps 和 net.ipv4.tcp_tw_recycle
	* 对于客户端处于NAT环境下时，多个客户端通过一个出口发出请求，对服务端而言这些请求的来源IP是一样的；如果设置timestamps，TCP缓存每个主机（IP）最近的时间戳，如果后续请求时间戳小雨缓存的时间戳的话，将视为无效，该请求的数据包会被丢弃；多个客户端发出的请求数据包的到达是无序的，所以有可能导致请求被丢弃，得不到服务端的响应。所以这个时候我们需要关闭timestamp机制，而如果tcp_tw_recycle开启的话，这种机制将被激活，唯一的解决办法就是关闭tcp_tw_recycle。
	* tcp_tw_recycle的作用是快速回收端口，目的是让主机有端口可用；我们通过设置tcp_max_tw_buckets，将其限制到小与65536的值，比如50000，那么系统在分配超过50000端口后将尝试回收端口，效果和tcp_tw_recycle是一样的，所以设置tcp_max_tw_buckets也能实现目的。但是该值不能设置太小，因为一旦超过这个值，timewait状态的socket将被销毁，只输出警告。

* [linux doc](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
* reference
	* [huoding](http://huoding.com/2012/01/19/142)
	* [linux内核调优参数对比和解释](http://nosmoking.blog.51cto.com/3263888/1684114)
	* [sysconf和limits.conf原理](http://wsgzao.github.io/post/sysctl/)


tcp fast open 相关
------------------------

so_reuse_addr 和 so_reuse_port相关
----------------------------------

