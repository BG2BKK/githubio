+++
date = '2016-08-11T18:07:19+08:00'
draft = true
title = 'nginx接收用户请求的过程'

+++

* [nginx从启动worker到处理一个用户请求](http://www.voidcn.com/blog/fengmo_q/article/p-2425250.html)

* 1、nginx启动时，在init_process阶段首先注册事件和处理方法。首先为每个listen fd分配一个ngx_connection_t，并为它设置读时间处理函数，ngx_event_accept
	* 如果nginx没有开启accept_mutex，则直接将ngx_event_accept挂载nginx的事件处理模型epoll上
	* 否则等到init_process阶段结束，在worker的事件处理循环中竞争到锁之后才挂载用于接收新请求的读事件

* 2、当worker监听到读事件，nginx就可以接收客户端的请求；用户向nginx发起请求后，nginx的事件处理模型收到读事件，然后调用ngx_event_accept处理
	* ngx_event_accept中，nginx调用accept，从TCP协议栈的已连接队列中取到一个连接和对应的socket，接着分配一个ngx_connection_t，将其与该socket对应
	* 接着初始化该连接：
		* src/event/ngx_event_accept.c
		* 为该连接分配一个256B大小的内存池
		* 初始化读写事件对应的处理和回调函数handler: c->rev = ngx_recv等
		* 分配log结构，以便后续log系统使用
		* 分配一个套接口地址sockaddr，将对端tcp地址保存在其中
		* 将本地套接口地址保存在local_sockaddr中；因为有时候从监听结构ngx_listenging_t中获得的监听地址可能是通配符*******，而本地套接地址是真实地址
		* 设置ready为1，即设置该连接的写事件就绪
		* 如果socket设置了TCP_DEFER_ACCEPT属性，则表示该连接上已经有数据包了，于是设置事件为读就绪
		* 将sockaddr保存的对端地址格式化为可读字符串
		* 最后调用ngx_http_init_connection初始化该连接的其他部分

```bash
       TCP_DEFER_ACCEPT (since Linux 2.4)
              Allow a listener to be awakened only when data arrives on the
              socket.  Takes an integer value (seconds), this can bound the
              maximum number of attempts TCP will make to complete the
              connection.  This option should not be used in code intended
              to be portable.

		允许只有当监听事件者只有当有数据时才被唤醒；参数是整数，可以限制一个TCP为了处理完成一个连接所做的最多尝试次数
```

		* ngx_http_init_connection
			* 初始化读写事件处理函数
				* 写事件：ngx_http_empty_handler
				* 读事件：ngx_http_init_request
					* 如果连接上有数据过来，则调用该函数处理数据
					* 否则设置定时器，等待数据到来或者超时，再在该函数中处理
	* 初始化用户请求：ngx_http_init_request
		* 它是一个事件处理函数，唯一的参数是ngx_event_t *；
		* 进入该事件处理函数后，判断是否超时，如果超时，则直接关闭连接并返回；否则继续
		* 在连接的内存池中分配一个ngx_http_request_t结构，用于保存请求的所有信息，并设置为该连接的request字段
		* nginx根据该请求的端口和地址，找到一个默认vhost(第一个定义的server)，或者根据域名来区分监听相同端口和地址的vhost，临时使用，最终找到真正的目标vhost；
		* 进入vhost后，将连接的读事件处理函数ngx_http_process_request_line函数，解析请求行；为这个请求分配缓冲区，保存header头在request->header_in，默认1024B，在该请求所在连接的内存池中分配
		* 将请求的引用计数count字段设置为1；将当前时间设置为start_sec和start_mset；nginx以接收到客户端的第一个数据包的时间作为起始，apache则以接收到整个request line开始
		* 初始化请求的其他字段，比如将uri_changes设置为11，表示最多可以将该请求的uri改写10次，subrequests被设置为201，表示一个请求最多可以发起200个子请求； 
	
