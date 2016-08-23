+++
date = "2016-08-11T17:44:27+08:00"
draft = true
title = "nginx模块开发那些事"

+++

[参考链接](http://yanyiwu.com/work/2014/09/21/nginx-module-development-stuff.html)

* nginx为每个连接分配内存池
	* r->pool是r的连接池，所有内存操作都在这里完成；请求结束后，该pool将会释放，因此没有内存泄露问题

* nginx的异步通信
	* 回调函数是nginx实现异步操作的方式
	* 以ngx读取POST请求的request body为例，每次epoll监听到socket有数据进来的时候，就非阻塞的调用recv接收数据并累计，直到数据大于等于Header头部中的"Content-Length"(HTTP请求的Header部分此时已经被处理)，然后调用模块的回调函数对所有POST数据进行处理。

* nginx的sendfile、TCP_NODELAY和TCP_NOPUSH
	* 互联网早期，由于网络链路质量不好，在发送数据时，如果只是很小的数据包，TCP/IP协议栈将会等待200ms，收集更多数据包后一次发出，可以提高吞吐量，这个算法成为Nagle算法；随着通信技术进步，Nagle算法逐渐不合实际，在nginx中甚至需要TCP_NODELAY来禁用socket的Nagle算法，要求有数据包后立刻发出
	* TCP_NOPUSH显然是与TCP_NODELAY相冲突的，那为什么它存在呢？
	* 对于sendfile on这一配置而言，由于系统调用sendfile有着“零拷贝”的优势，在内核中从in_fd复制到out_fd，不足之处是in_fd只能是文件fd，因此sendfile只能用于发送文件到网络IO；当TCP_NOPUSH、TCP_NODELAY和sendfile配合起来，就很有意思了：在调用sendfile之前，将out_fd这一socket设置为TCP_NOPUSH的，sendfile将文件数据写到socket缓冲区，在sendfile完成后，去掉out_fd的TCP_NOPUSH选项，将文件数据一次发出，可以提高性能。

* nginx的sendfile，还有读取用户请求的post数据等，灵活的操作内存缓冲区，代码质量很高，值得品味


[参考链接](https://t37.net/nginx-optimization-understanding-sendfile-tcp_nodelay-and-tcp_nopush.html)

* sendfile, tcp_nodelay和tcp_nopush是怎样对nginx产生影响的呢？
	* 
