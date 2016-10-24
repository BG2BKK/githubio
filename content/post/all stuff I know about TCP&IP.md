+++
date = "2016-05-30T13:58:17+08:00"
draft = true
title = "effective tips about tcp ip on Linux"

+++

* [Linux内核参数注释](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
	* [TCP参数注释及调优](https://tonydeng.github.io/2015/05/25/linux-tcpip-tuning/)
	* man tcp

* [内核为tcp维护的四个定时器](http://network.51cto.com/art/201412/459352.htm)
	* 重传定时器
		* 用于重传没有收到确认的报文；在发送报文时启动重传定时器，如果在定时器时间内收到确认，则撤销定时器；若在收到确认前已经超时，则重传报文并复位定时器。
	* 坚持定时器
		* 当接收方的接收通告窗口是0时，会向发送方发送零窗口报文段；稍后接收方窗口不为0时，会向发送方发送窗口更新报文（非0报文段）；如果发送方没有收到这个报文，则一直会认为接收方还是窗口为0所以继续等待；而接收方此时并不知道这个报文丢失，所以处于等待发送状态；通信双方陷入了死锁状态
		* 因此TCP为每个连接设置坚持定时器，当发送方收到零长窗口的包时，会启动该定时器；当该定时器到期，发送方会发送探测报文段（只有一个字节，有序号，但是该序号永远不需要确认，也会被忽略掉），探测报文提醒接收端，问问是不是还是零长窗口
		* 坚持计时器的初始时长是重传时间；第一次探测报文发出，如果没有收到接收端的窗口更新响应，则在坚持计时器超时后，复位该计时器，计时时间加倍，然后发送新的探测报文，重复这个过程；直到定时器时长超过60s为止，开始每60s重复一次。
	* 保活定时器
		* 保活定时器是TCP的keepalive设置；如果客户端故障，链路上没有数据，而服务端不能一直等待，所以需要保活定时器。服务端每次收到对端报文，都会重置保活定时器，默认是7200s，之后再每隔75s重复发送探测报文，连续10次没有回应的话就放弃这个连接。
	* 2MSL定时器
		* 第一种情况：被动关闭方发出FIN包并将状态从CLOSE_WAIT转换到LAST_ACK后，主动关闭方从FIN_WAIT2转到TIME_WAIT状态，并回复ACK；这个ACK到达接收方最大需要一个MSL时间，接收方收到ACK后，可以进入CLOSED状态；而如果接收方没有收到ACK，则会认为发送方可能没有收到刚才发的FIN包，所以会从新发送FIN包，这个FIN包到达主动关闭方需要一个MSL时间；因此，如果主动关闭方能等待2个MSL时间，可以在应付被动关闭方再次发送过来的FIN包并回复ACK，保证二者都能进入CLOSED状态
		* 第二种情况：由于等待了2个MSL时间，之前双方发送的报文都会消失不见，避免下一个采用同样地址和端口的新连接遇到这些不速之客；否则如果等待时间短，新连接可以很快使用这个端口和地址，遇到之前的FIN包什么的，造成误解。

* [tcp的超时重连机制]()
	* tcp_syn_retries

> Number of times initial SYNs for an active TCP connection attempt will be retransmitted. Should not be higher than 127. Default value is 6, which corresponds to 63seconds till the last retransmission with the current initial RTO of 1second. With this the final timeout for an active TCP connection attempt will happen after 127seconds.

对于一个活跃的TCP连接来说，SYN报文的重传次数。这个次数应该小于127，默认是6次，初始RTO是1s，这意味着在63s后最后一次重传，所以在第127s的时候这个tcp连接会超时o

我们将会在1s、2s、4s、8s、16s、32s、64s进行重传

* [tcp的报文超时重传机制](rto and retransmission)
	* google 'tcp 超时处理'
	* [computing tcp's retransimission timer](https://tools.ietf.org/html/rfc6298)
	* 内核有两个重要的超时重传选项
		 
```bash
tcp_retries1 - INTEGER
	This value influences the time, after which TCP decides, that
	something is wrong due to unacknowledged RTO retransmissions,
	and reports this suspicion to the network layer.
	See tcp_retries2 for more details.

	RFC 1122 recommends at least 3 retransmissions, which is the
	default.

tcp_retries2 - INTEGER
	This value influences the timeout of an alive TCP connection,
	when RTO retransmissions remain unacknowledged.
	Given a value of N, a hypothetical TCP connection following
	exponential backoff with an initial RTO of TCP_RTO_MIN would
	retransmit N times before killing the connection at the (N+1)th RTO.

	The default value of 15 yields a hypothetical timeout of 924.6
	seconds and is a lower bound for the effective timeout.
	TCP will effectively time out at the first RTO which exceeds the
	hypothetical timeout.

	RFC 1122 recommends at least 100 seconds for the timeout,
	which corresponds to a value of at least 8.

```

* tcp_retries1
	* 在发生超时，tcp没有收到报文ack的情况下，会进行重传；重传次数取决于该选项，默认是3次；初次重传时间间隔是0.2s，每次重传的时间间隔加倍，0.2、0.4、0.8、1.6、3.2s
	* 超过这个时间后，则认为发生了超时，向网络层报告这个怀疑，底层的IP和ARP开始接管，去寻找所要连接的对端机器
* tcp_retries2
	* 当RTO超时重传，仍然没有被回复确认包时。给定一个值N，一个假想的TCP连接会随着初始RTO时间进行指数增长补偿，重传N次，在第N+1次时连接被干掉。
	* 默认值是15次，即924.6s，这是一个有效超时的下限；TCP将在超出这个时间后的第一个RTO中超时；
	* RFC1122建议至少这个值是100s，就是说重传8次
	* 关于RTO的计算，参考RFC和以下链接：
		* [RTO和RTO在linux上的实现](http://www.orczhou.com/index.php/2011/10/tcpip-protocol-start-rto/)
		* [RTO的计算方法一](http://www.pagefault.info/?p=430)
		* [RTO的计算方法二](http://perthcharles.github.io/2015/09/06/wiki-rtt-estimator/)
		* [RTO的修改方法](http://weibo.com/p/1001603821691477346388)
			* rfc1323
			* [格式较好的地址](https://blog.gesha.net/archives/568/)
	* [聊一聊重传次数](http://perthcharles.github.io/2015/09/07/wiki-tcp-retries/)
		* 值得一看，目前我对下文还有疑问，并且认为超时时间只和retries2有关。
	* [min_rto设置可以通过ip route设置](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Release_Notes/U7/ppc/ar01s04.html)

```bash
   如果RTT较大，比如RTO初始值计算得到的是1000ms
   那么根本不需要重传15次，重传总间隔就会超过924600ms。
   比如我测试的一个RTT=400ms的情况，当tcp_retries2=10时，仅重传了3次就放弃了TCP流
```
		* RTO_MIN是动态的一个值
	<!-- * 超时后，对端发送RST包，终止连接；connect返回结果是Connection time out 110 -->

	* [导致TCP重传的情况](http://www.voidcn.com/blog/gogokongyin/article/p-5803363.html)
		* 报文中途被丢弃，或者ttl到期
		* 报文的ACK在中途丢失
		* 接收端异常，并不发送ACK给发送端
	* 判断一个报文是重传报文
		* 序列号突然下降。由于是累计确认的，某个数据包丢失的话，后面的数据包都不会被确认接收；所以序列号下降很有可能是被重传的报文
		* 如果某一时刻出现两个序列号、长度等相同的包，则其中一个包肯定是重传的

* [TCP连接占用内存大小](http://blog.csdn.net/russell_tao/article/details/18711023)
	* tcp_rmem min mid max 单个TCP连接的读内存占用，系统限制；单位是byte，大小是几KB到十几MB
	* tcp_wmem min mid max 单个TCP连接的写内存占用
	* tcp_mem min mid max 系统中tcp整体内存使用，单位是4k或者8k大小的页

SO_SNDBUF和SO_RCVBUF可以基于setsockopt来限制单个tcp的读写缓存，但是会受限于上述的限制；高于或者低于限制时，会被替换

读写缓存用于缓存对端的TCP报文，读缓存用于缓存两种报文：一种是无序的落在接收滑动窗口中的TCP报文，等待有序报文；另一种是有序报文，但是没有来得及被应用程序读取的TCP报文；写缓存也一样

读写缓存不是固定的，是在使用中根据使用情况分配的；如果TCP连接非常清闲，读写缓存的大小会降到0；而如果TCP连接非常繁忙，以读缓存为例，如果缓存报文超过缓存大小，则新来报文将被丢弃；持续一段时间后，将向对端发送接收窗口为0的报文

* [TCP长连接](http://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)
	* [SO_KEEPALIVE套接口选项](http://blog.csdn.net/apn172/article/details/8030230)
		* SO_KEEPALIVE保持连接检测对方主机是否崩溃，如果2小时内该socket在任一方向都没有数据交换，TCP自动发送给对方一个存活探测分节（keepalive probe）
		* 存活探测分节是一个对方必须响应的TCP分节，返回有三种情况：
			* 对方接收一切正常：以期望的ACK响应
			* 对方已崩溃并已重新启动：以RST响应，socket错误为ECONNRESET，socket被关闭
			* 对方无任何响应：开始根据tcp_keepalive_time设置的重传次数开始重传，相隔75s一次，默认9次,11分钟多之后放弃。socket错误是ETIMEOUT，socket被关闭。
		* 因此如果我们使用系统的保活探测机制，可能在2小时之后才能感知到tcp连接不存在。（方法是设置SO_KEEPALIVE为1）
		* 所以可以通过setsockopt来设置这个保活机制，TCP_KEEPIDLE、TCP_KEEPINTVL和TCP_KEEPCNT，可以让时间小一点。
		* 默认设置

		net.ipv4.tcp_keepalive_intvl = 75

		net.ipv4.tcp_keepalive_probes = 9

		net.ipv4.tcp_keepalive_time = 7200

* [传输层协议 TCP/UDP](http://xuelinf.github.io/2016/03/09/TCP-IP-%E5%AE%8C%E5%85%A8%E8%A7%A3%E8%AF%BB1/)


* TCP连接的保活和异常检测
	* [TCP通讯时对侧主机崩溃或者网络异常的退出](http://blog.csdn.net/apn172/article/details/8034236)

如果链路空闲，此时如果是系统的保活机制，空闲7200s后开始发送探测报文，每隔75s发送一次，共持续9次；那么我们可能需要在2小时之后才能发现这个tcp连接断开，这是不可想象的

此时tcp的其他机制可以起作用：[tcp超时重传机制](rto and retransmission)。tcp_retries2参数和TCP_RTO_MIN共同确定的超时时间，如果tcp_retries2是15的话，这个时间是924.6s；如果设置为8，则是100s左右；设置为5的话，46.5s左右就能检测出超时了。
	因此在本地网络正常，对侧网络断开或者对侧主机崩溃时，发出探测包，此时本地socket正常，所以写操作没有问题；如果该报文在tcp_retries2设置的超时时间后没有回应，则判断超时，可以关闭socket了。

	* [TCP保活报文](http://www.netis.com/flows/2012/11/01/tcpkeepalive/)
		* 保活探测报文是将最近TCP报文的序号减1，并设置1个byte，payload为00的数据报文
		* 保活回复报文将是对其的回应

* [深入理解socket网络异常](http://blog.csdn.net/wy5761/article/details/17232495)
	* 客户端连接了服务端未开放未监听的端口
		* 这种情况下服务端会对收到的SYN回应一个RST（RFC793）；客户端收到RST后终止连接，进入CLOSED状态；connect返回ECONNREFUSED 111错误，错误信息：connect refused
	* 客户端与服务端之间网络不通
		* connect返回主机不可达
			* 如果给出的是一个不可访问的地址，比如不存在的本地网络地址，或者DNS解析失败时，会返回这个错误。在Linux上，错误码是EHOSTUNREACH 113，错误信息：No route to host
		* connect返回连接超时
			* 连接超时，客户端的SYN包消失，或者没有收到ACK，就会超时重传SYN；重传间隔时间是1s、2s、4s、8s、16s、32s，默认为6次，第6次SYN会经过64s后超时；也就是说127s后，connect返回ETIMEDOUT
	
	* 通信过程中会遇到哪些问题呢：
		* 通信双方之间网络断开，双方也不互相发送数据
			* 双方都对网络断开无感知，在没有SO_KEEPALIVE的情况下，双方都会保持ESTABLISHED
		* 网络断开，一方给另一方发送数据
			* 首先接收方是对网络断开无感知的；理论上讲发送方在重传一定次数后，报超时错误；而实际上，可能发送方会显示发送成功，但接收方一定没有收到数据，原因可能是：
				* 写在本机的TCP写缓存中；要么缓存写满后不能再往缓存中写数据，向客户端报错，当然这种情况比较少见；更多情况是写缓存中有一定数据后，启动发送，多次重传后报超时错误
				* 缓存在本机网络的某个NAT的缓存中，由NAT回复了ACK包
			* 因此这里需要说的是，并不是TCP显示发送成功，对方就一定能收到；这个时候只是内核或者本机所处网络的NAT收到了应用程序的数据
			* TCP可以保证的是可靠、有序的传输，这个意思是TCP保证收到有序数据包时可以保证可靠传输，而不是发送成功数据包时可靠传输
		* 网络断开，一方等待另一方发送数据
			* 等待的接收方是不能感知到网络已断开的，如果采用阻塞型的read系统调用等，会一直阻塞下去；当有SO_KEEPALIVE设置时，可以依赖系统的keepalive机制；应用程序可以设置超时时间来确保read不要被永久阻塞

		* TCP协议栈在各平台的实现不尽相同，在Linux下，Ctrl+C结束程序，协议栈会发送close，而Windows会发送RST段；如果没有调用close而结束，Linux会发送FIN包，而Win会发送RST包

		* crash的一端发送FIN包，相当于调用close，
			* 未崩溃的一方继续 接收 数据
				* Linux上，当对端crash后，相当于调用close，会发送FIN包；那么在这个tcp连接上读取数据的接收方，read系统调用会立刻返回失败
			* 未崩溃的一方继续 接收 数据
				* 对方crash的时刻，tcp连接不会立刻感知；第一次write该tcp连接，可以写成功；而当这次数据包发送到对端时，由于应用程序crash，不论有没有重启，对端都会发送RST包；再次调用write会返回-1，错误码是EPIPE 32，错误信息是Broken Pipe；应用程序会收到SIGPIPE信号，崩溃，没有coredump文件产生（可以通过忽略SIGPIPE避免这个问题）。
		* crash的一端发送RST包
			* 未崩溃的一方继续 接收 数据
				* 调用recv会返回-1，错误码ECONNRESET，错误信息 Connection reset by peer
			* 未崩溃的一方继续 发送 数据
				* 调用send会返回-1，错误码ECONNRESET，错误信息 Connection reset by peer
		* crash的一端没有发送RST或者FIN包
			* 调用recv的话会一直阻塞
			* 调用send的话会尝试重传直到超时
		* 可见对端崩溃时，本端如何感知，取决于对端是否发送RST包

	* [RFC 793](https://tools.ietf.org/html/rfc793)

	* [RST包的产生](http://blog.csdn.net/wy5761/article/details/17244061)
		* [向不存在的端口发起连接，对方回复RST](http://yaowhat.com/2014/07/28/tcp-rst.html)
		* 一些延迟的包，比如旧的SYN包的副本，导致连接无法正常建立
		* 程序异常终止，有的系统会发送RST
		* 设置了SO_LINGER选项支持异常关闭，并调用close，会发送RST包
		* 已经关闭的端口收到了数据，反馈回一个RST包
			* 这就是对端crash后，第一次本端向crash方发送数据是可以写成功的，但是当数据到达crash方后，crash已经是关闭了的，所以会回复RST，这个时候tcp就会出错，Broken Pipe
	* [RST包的处理方式](http://blog.csdn.net/wy5761/article/details/17244061)
		* Linux-2.6.18源码：

```cpp
/* When we get a reset we do this. */
static void tcp_reset(struct sock *sk)
{
	/* We want the right error as BSD sees it (and indeed as we do). */
	switch (sk->sk_state) {
		case TCP_SYN_SENT:
			sk->sk_err = ECONNREFUSED;
			break;
		case TCP_CLOSE_WAIT:
			sk->sk_err = EPIPE;
			break;
		case TCP_CLOSE:
			return;
		default:
			sk->sk_err = ECONNRESET;
	}

	if (!sock_flag(sk, SOCK_DEAD))
		sk->sk_error_report(sk);

	tcp_done(sk);
}
```

	* 当本端这个tcp连接处于TCP_SYN_SENT状态，也就是说本地主动发起连接，缺不幸收到了RST包，这个时候会向应用程序报告ECONNREFUSED连接被拒绝错误
	* 当本地这个tcp处于TCP_CLOSE_WAIT状态，就是说本地从ESTA状态收到对方的FIN包后转为CLOSE_WAIT状态时，对端已经关闭了；如果这个时候本地仍然继续向对端发送数据，对端会返回一个RST包，这就导致程序进入这个case分支，因此向上层应用程序报EPIPE错误
	* 默认清空下错误码都是ECONNRESET，connection reset by peer了。


* TCP连接中的一些常见的“常数”
	* mss 1460
		* 以太网的MTU是1500bytes，而IP头是20bytes，TCP头部是20bytes，所以很多时候mss是1460bytes，这个比较常见
	* win 5792
		* 在tcp初始化后，接收窗口大小会被设置成4个mss大小，在linux3.0之后是10个mss大小；
		* 以4个mss大小为例，由于有的时候tcp的头部中option字段可能会带有时间戳，头部额外占用12字节，所以初始窗口大小是 4 * (1460 - 12) = 5792大小
	
* broken pipe相关的问题
	* 


* [linux内核工程导论：TCP](http://www.lai18.com/content/1409216.html)

* [TCP的write/read系统调用和send/recv的区别](http://blog.csdn.net/petershina/article/details/7946615)
