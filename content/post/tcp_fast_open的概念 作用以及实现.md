+++
date = '2016-05-09T15:53:34+08:00'
draft = false
title = 'tcp_fast_open的概念 作用以及实现'

+++

引言
------------------

> 三次握手的过程中，当用户首次访问server时，发送syn包，server根据用户IP生成cookie，并与syn+ack一同发回client；client再次访问server时，在syn包携带TCP cookie；如果server校验合法，则在用户回复ack前就可以直接发送数据；否则按照正常三次握手进行。

> TFO提高性能的关键是省去了热请求的三次握手，这在充斥着小对象的移动应用场景中能够极大提升性能。

Google研究发现TCP 二次握手是页面延迟时间的重要部分，所以提出TFO

TFO的fast open标志体现在TCP报文的头部的[OPTION字段](http://www.iana.org/assignments/tcp-parameters/tcp-parameters.xhtml)

TCP Fast Open的标准文档是[rfc7413](http://tools.ietf.org/html/rfc7413)

TFO与2.6.34内核合并到主线，[lwn通告地址](https://lwn.net/Articles/508865/)

TFO的使用目前还是有些复杂的，从[linux的network文档来看](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)：

TFO的配置说明：

```bash
tcp_fastopen - INTEGER
	Enable TCP Fast Open feature (draft-ietf-tcpm-fastopen) to send data
	in the opening SYN packet. To use this feature, the client application
	must use sendmsg() or sendto() with MSG_FASTOPEN flag rather than
	connect() to perform a TCP handshake automatically.

	The values (bitmap) are
	1: Enables sending data in the opening SYN on the client w/ MSG_FASTOPEN.
	2: Enables TCP Fast Open on the server side, i.e., allowing data in
	   a SYN packet to be accepted and passed to the application before
	   3-way hand shake finishes.
	4: Send data in the opening SYN regardless of cookie availability and
	   without a cookie option.
	0x100: Accept SYN data w/o validating the cookie.
	0x200: Accept data-in-SYN w/o any cookie option present.
	0x400/0x800: Enable Fast Open on all listeners regardless of the
	   TCP_FASTOPEN socket option. The two different flags designate two
	   different ways of setting max_qlen without the TCP_FASTOPEN socket
	   option.

	Default: 1

	Note that the client & server side Fast Open flags (1 and 2
	respectively) must be also enabled before the rest of flags can take
	effect.

	See include/net/tcp.h and the code for more details.
```

为了启用 tcp fast open功能

	- client需要使用sendmsg或者sento系统调用，加上MSG_FASTOPEN flag，来连接server端，代替connect系统调用。
	- 对server端不做要求。

linux系统（高版本内核）默认tcp_fastopen为1：

```bash
$ sysctl -a | grep fastopen

net.ipv4.tcp_fastopen = 1
```

测试代码： server.c

```cpp
// reference: http://blog.csdn.net/hanhuili/article/details/8540227

#include <string.h>         
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
int main(){
    int portno = 6666;
    socklen_t clilen;
    char buffer[256];
    struct sockaddr_in serv_addr, cli_addr;
    int cfd;
    int sfd = socket(AF_INET, SOCK_STREAM, 0);   // Create socket
    
    bzero((char *) &serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = INADDR_ANY;
    serv_addr.sin_port = htons(portno);
    bind(sfd, &serv_addr,sizeof(serv_addr));      // Bind to well known address
    
    int qlen = 5;                 // Value to be chosen by application
    int err = setsockopt(sfd, IPPROTO_TCP/*SOL_TCP*/, 23/*TCP_FASTOPEN*/, &qlen, sizeof(qlen));
    
    listen(sfd,1);                // Mark socket to receive connections

	while(1){
		cfd = accept(sfd, NULL, 0);   // Accept connection on new socket

		while(1){
			int len = read(cfd,buffer,256);
			if(len)
				printf("tcp fast open: %s\n",buffer);
			else
				break;
			// read and write data on connected socket cfd
		}

		memset(buffer, 0, 256);
		close(cfd);
	}

}

```

测试代码：client.c

```cpp

#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h> 
#include <errno.h>
int main(){
    struct sockaddr_in serv_addr;
    struct hostent *server;

    char *data = "Hello, tcp fast open";
    int data_len = strlen(data);    
    
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    server = gethostbyname("localhost");
    
    bzero((char *) &serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    bcopy((char *)server->h_addr, 
         (char *)&serv_addr.sin_addr.s_addr,
         server->h_length);
    serv_addr.sin_port = htons(6666);

// /usr/src/linux-headers-4.4.0-22/include/linux/socket.h:#define MSG_FASTOPEN	0x20000000	/* Send data in TCP SYN */

//  int len = sendto(sfd, data, data_len, 0x20000000/*MSG_FASTOPEN*/, 

    int len = sendto(sfd, data, data_len, MSG_FASTOPEN/*MSG_FASTOPEN*/, 
                (struct sockaddr *) &serv_addr, sizeof(serv_addr));
	if(errno != 0){
		printf("error: %s\n", strerror(errno));
	}
    close(sfd);
}

```

通信过程：tcpdump

```bash

$ sudo tcpdump -i any port 6666 -X

# 第一次 ./client.o

00:29:34.820187 IP localhost.51388 > localhost.6666: Flags [S], seq 755101042, win 43690, options [mss 65495,sackOK,TS val 17053 ecr 0,nop,wscale 7,unknown-34,nop,nop], length 0
	0x0000:  4500 0040 afef 4000 4006 8cc6 7f00 0001  E..@..@.@.......
	0x0010:  7f00 0001 c8bc 1a0a 2d01 ed72 0000 0000  ........-..r....
	0x0020:  b002 aaaa fe34 0000 0204 ffd7 0402 080a  .....4..........
	0x0030:  0000 429d 0000 0000 0103 0307 2202 0101  ..B........."...
00:29:34.820284 IP localhost.6666 > localhost.51388: Flags [S.], seq 3725111481, ack 755101043, win 43690, options [mss 65495,sackOK,TS val 17053 ecr 17053,nop,wscale 7], length 0
	0x0000:  4500 003c 0000 4000 4006 3cba 7f00 0001  E..<..@.@.<.....
	0x0010:  7f00 0001 1a0a c8bc de08 b0b9 2d01 ed73  ............-..s
	0x0020:  a012 aaaa fe30 0000 0204 ffd7 0402 080a  .....0..........
	0x0030:  0000 429d 0000 429d 0103 0307            ..B...B.....
00:29:34.820372 IP localhost.51388 > localhost.6666: Flags [P.], seq 1:21, ack 1, win 342, options [nop,nop,TS val 17053 ecr 17053], length 20
	0x0000:  4500 0048 aff0 4000 4006 8cbd 7f00 0001  E..H..@.@.......
	0x0010:  7f00 0001 c8bc 1a0a 2d01 ed73 de08 b0ba  ........-..s....
	0x0020:  8018 0156 fe3c 0000 0101 080a 0000 429d  ...V.<........B.
	0x0030:  0000 429d 4865 6c6c 6f2c 2074 6370 2066  ..B.Hello,.tcp.f
	0x0040:  6173 7420 6f70 656e                      ast.open
00:29:34.820433 IP localhost.6666 > localhost.51388: Flags [.], ack 21, win 342, options [nop,nop,TS val 17053 ecr 17053], length 0
	0x0000:  4500 0034 f227 4000 4006 4a9a 7f00 0001  E..4.'@.@.J.....
	0x0010:  7f00 0001 1a0a c8bc de08 b0ba 2d01 ed87  ............-...
	0x0020:  8010 0156 fe28 0000 0101 080a 0000 429d  ...V.(........B.
	0x0030:  0000 429d                                ..B.
00:29:34.859246 IP localhost.6666 > localhost.51388: Flags [.], ack 22, win 342, options [nop,nop,TS val 17063 ecr 17053], length 0
	0x0000:  4500 0034 f228 4000 4006 4a99 7f00 0001  E..4.(@.@.J.....
	0x0010:  7f00 0001 1a0a c8bc de08 b0ba 2d01 ed88  ............-...
	0x0020:  8010 0156 fe28 0000 0101 080a 0000 42a7  ...V.(........B.
	0x0030:  0000 429d                                ..B.

# 第二次 ./client.o

00:29:39.271936 IP localhost.51398 > localhost.6666: Flags [S], seq 2362540136, win 43690, options [mss 65495,sackOK,TS val 18166 ecr 0,nop,wscale 7,exp-tfo cookiereq], length 0
	0x0000:  4500 0040 c69e 4000 4006 7617 7f00 0001  E..@..@.@.v.....
	0x0010:  7f00 0001 c8c6 1a0a 8cd1 8068 0000 0000  ...........h....
	0x0020:  b002 aaaa fe34 0000 0204 ffd7 0402 080a  .....4..........
	0x0030:  0000 46f6 0000 0000 0103 0307 fe04 f989  ..F.............
00:29:39.271986 IP localhost.6666 > localhost.51398: Flags [S.], seq 3703577773, ack 2362540137, win 43690, options [mss 65495,sackOK,TS val 18166 ecr 18166,nop,wscale 7], length 0
	0x0000:  4500 003c 0000 4000 4006 3cba 7f00 0001  E..<..@.@.<.....
	0x0010:  7f00 0001 1a0a c8c6 dcc0 1cad 8cd1 8069  ...............i
	0x0020:  a012 aaaa fe30 0000 0204 ffd7 0402 080a  .....0..........
	0x0030:  0000 46f6 0000 46f6 0103 0307            ..F...F.....
00:29:39.272038 IP localhost.51398 > localhost.6666: Flags [P.], seq 1:21, ack 1, win 342, options [nop,nop,TS val 18166 ecr 18166], length 20
	0x0000:  4500 0048 c69f 4000 4006 760e 7f00 0001  E..H..@.@.v.....
	0x0010:  7f00 0001 c8c6 1a0a 8cd1 8069 dcc0 1cae  ...........i....
	0x0020:  8018 0156 fe3c 0000 0101 080a 0000 46f6  ...V.<........F.
	0x0030:  0000 46f6 4865 6c6c 6f2c 2074 6370 2066  ..F.Hello,.tcp.f
	0x0040:  6173 7420 6f70 656e                      ast.open
00:29:39.272072 IP localhost.6666 > localhost.51398: Flags [.], ack 21, win 342, options [nop,nop,TS val 18166 ecr 18166], length 0
	0x0000:  4500 0034 5a58 4000 4006 e269 7f00 0001  E..4ZX@.@..i....
	0x0010:  7f00 0001 1a0a c8c6 dcc0 1cae 8cd1 807d  ...............}
	0x0020:  8010 0156 fe28 0000 0101 080a 0000 46f6  ...V.(........F.
	0x0030:  0000 46f6                                ..F.
00:29:39.311280 IP localhost.6666 > localhost.51398: Flags [.], ack 22, win 342, options [nop,nop,TS val 18176 ecr 18166], length 0
	0x0000:  4500 0034 5a59 4000 4006 e268 7f00 0001  E..4ZY@.@..h....
	0x0010:  7f00 0001 1a0a c8c6 dcc0 1cae 8cd1 807e  ...............~
	0x0020:  8010 0156 fe28 0000 0101 080a 0000 4700  ...V.(........G.
	0x0030:  0000 46f6                                ..F.

```

奇怪的是，在代码中启用tcp_fastopen的结果和不启用，并没有区别。那这是什么原因呢？

通过搜索，发现在介绍[tcp fast open优化shadowsocks](https://github.com/shadowsocks/shadowsocks/wiki/TCP-Fast-Open)时，设置net.ipv4.tcp_fastopen为3，虽然奇怪，但是可以试试：

```bash
$ sysctl -a | grep fastopen

net.ipv4.tcp_fastopen = 3
```

```bash

# 第一次，server返回cookie unknown-34 0x38af51c10bf41ca4

00:36:36.667932 IP localhost.52220 > localhost.6666: Flags [S], seq 3662514892, win 43690, options [mss 65495,sackOK,TS val 122515 ecr 0,nop,wscale 7,unknown-34,nop,nop], length 0
	0x0000:  4500 0040 545f 4000 4006 e856 7f00 0001  E..@T_@.@..V....
	0x0010:  7f00 0001 cbfc 1a0a da4d 8acc 0000 0000  .........M......
	0x0020:  b002 aaaa fe34 0000 0204 ffd7 0402 080a  .....4..........
	0x0030:  0001 de93 0000 0000 0103 0307 2202 0101  ............"...
00:36:36.667990 IP localhost.6666 > localhost.52220: Flags [S.], seq 3186866007, ack 3662514893, win 43690, options [mss 65495,sackOK,TS val 122515 ecr 122515,nop,wscale 7,unknown-34 0x38af51c10bf41ca4,nop,nop], length 0
	0x0000:  4500 0048 0000 4000 4006 3cae 7f00 0001  E..H..@.@.<.....
	0x0010:  7f00 0001 1a0a cbfc bdf3 b757 da4d 8acd  ...........W.M..
	0x0020:  d012 aaaa fe3c 0000 0204 ffd7 0402 080a  .....<..........
	0x0030:  0001 de93 0001 de93 0103 0307 220a 38af  ............".8.
	0x0040:  51c1 0bf4 1ca4 0101                      Q.......
00:36:36.668050 IP localhost.52220 > localhost.6666: Flags [P.], seq 1:21, ack 1, win 342, options [nop,nop,TS val 122515 ecr 122515], length 20
	0x0000:  4500 0048 5460 4000 4006 e84d 7f00 0001  E..HT`@.@..M....
	0x0010:  7f00 0001 cbfc 1a0a da4d 8acd bdf3 b758  .........M.....X
	0x0020:  8018 0156 fe3c 0000 0101 080a 0001 de93  ...V.<..........
	0x0030:  0001 de93 4865 6c6c 6f2c 2074 6370 2066  ....Hello,.tcp.f
	0x0040:  6173 7420 6f70 656e                      ast.open
00:36:36.668109 IP localhost.6666 > localhost.52220: Flags [.], ack 21, win 342, options [nop,nop,TS val 122515 ecr 122515], length 0
	0x0000:  4500 0034 69cb 4000 4006 d2f6 7f00 0001  E..4i.@.@.......
	0x0010:  7f00 0001 1a0a cbfc bdf3 b758 da4d 8ae1  ...........X.M..
	0x0020:  8010 0156 fe28 0000 0101 080a 0001 de93  ...V.(..........
	0x0030:  0001 de93                                ....
00:36:36.707264 IP localhost.6666 > localhost.52220: Flags [.], ack 22, win 342, options [nop,nop,TS val 122525 ecr 122515], length 0
	0x0000:  4500 0034 69cc 4000 4006 d2f5 7f00 0001  E..4i.@.@.......
	0x0010:  7f00 0001 1a0a cbfc bdf3 b758 da4d 8ae2  ...........X.M..
	0x0020:  8010 0156 fe28 0000 0101 080a 0001 de9d  ...V.(..........
	0x0030:  0001 de93                                ....


# 第二次，client发送请求时，将cookie写在syn包中，同时带上发送的数据；server端校验后(kernel和tcp/ip协议栈做校验)后返回成功，如此在3次握手中节省了一次rtt时间

00:36:38.744954 IP localhost.52226 > localhost.6666: Flags [S], seq 1820632025:1820632045, win 43690, options [mss 65495,sackOK,TS val 123034 ecr 0,nop,wscale 7,unknown-34 0x38af51c10bf41ca4,nop,nop], length 20
	0x0000:  4500 005c 4343 4000 4006 f956 7f00 0001  E..\CC@.@..V....
	0x0010:  7f00 0001 cc02 1a0a 6c84 a3d9 0000 0000  ........l.......
	0x0020:  d002 aaaa fe50 0000 0204 ffd7 0402 080a  .....P..........
	0x0030:  0001 e09a 0000 0000 0103 0307 220a 38af  ............".8.
	0x0040:  51c1 0bf4 1ca4 0101 4865 6c6c 6f2c 2074  Q.......Hello,.t
	0x0050:  6370 2066 6173 7420 6f70 656e            cp.fast.open
00:36:38.745022 IP localhost.6666 > localhost.52226: Flags [S.], seq 3848342665, ack 1820632046, win 43690, options [mss 65495,sackOK,TS val 123034 ecr 123034,nop,wscale 7], length 0
	0x0000:  4500 003c 0000 4000 4006 3cba 7f00 0001  E..<..@.@.<.....
	0x0010:  7f00 0001 1a0a cc02 e561 0c89 6c84 a3ee  .........a..l...
	0x0020:  a012 aaaa fe30 0000 0204 ffd7 0402 080a  .....0..........
	0x0030:  0001 e09a 0001 e09a 0103 0307            ............
00:36:38.745072 IP localhost.52226 > localhost.6666: Flags [.], ack 1, win 342, options [nop,nop,TS val 123034 ecr 123034], length 0
	0x0000:  4500 0034 4344 4000 4006 f97d 7f00 0001  E..4CD@.@..}....
	0x0010:  7f00 0001 cc02 1a0a 6c84 a3ee e561 0c8a  ........l....a..
	0x0020:  8010 0156 fe28 0000 0101 080a 0001 e09a  ...V.(..........
	0x0030:  0001 e09a                                ....
00:36:38.745127 IP localhost.52226 > localhost.6666: Flags [F.], seq 1, ack 1, win 342, options [nop,nop,TS val 123034 ecr 123034], length 0
	0x0000:  4500 0034 4345 4000 4006 f97c 7f00 0001  E..4CE@.@..|....
	0x0010:  7f00 0001 cc02 1a0a 6c84 a3ee e561 0c8a  ........l....a..
	0x0020:  8011 0156 fe28 0000 0101 080a 0001 e09a  ...V.(..........
	0x0030:  0001 e09a                                ....
00:36:38.747232 IP localhost.6666 > localhost.52226: Flags [.], ack 2, win 342, options [nop,nop,TS val 123035 ecr 123034], length 0
	0x0000:  4500 0034 ec10 4000 4006 50b1 7f00 0001  E..4..@.@.P.....
	0x0010:  7f00 0001 1a0a cc02 e561 0c8a 6c84 a3ef  .........a..l...
	0x0020:  8010 0156 fe28 0000 0101 080a 0001 e09b  ...V.(..........
	0x0030:  0001 e09a                                ....

```

* 上述通信过程中
	* 第一次，server返回cookie unknown-34 0x38af51c10bf41ca4
	* 第二次，client发送请求时，将cookie写在syn包中，同时带上发送的数据；server端校验后(kernel和tcp/ip协议栈做校验)后返回成功，如此在3次握手中节省了一次rtt时间
* 也就是说，在net.ipv4.tcp_fastopen设置为3时，tcp fastopen特性使能

关于如何使能TFO，在前文中的TFO的配置说明中，我们可以看到，

```bash
	The values (bitmap) are
	1: Enables sending data in the opening SYN on the client w/ MSG_FASTOPEN.
		使能client端的TFO特性
	2: Enables TCP Fast Open on the server side, i.e., allowing data in
	   a SYN packet to be accepted and passed to the application before
	   3-way hand shake finishes.
		使能server端的TFO特性
	4: Send data in the opening SYN regardless of cookie availability and
	   without a cookie option.

```

并且这个标志是位操作，如果我在本机做实验，将本机作为sever端和client端的话，需要两个位都使能，所以应该将该值设置为3.

同时我们可以看到，tcp fast open是非常向后兼容的，升级成本不高，需要高于3.7+版本内核，但总体来说值得采用。

nginx 1.5.18（2013年）开始支持tcp fast open

TODO LIST
--------------

* TFO在移动端场景中的性能体现：android+nginx
* tcp fast open 在内核中的实现


参考链接
---------------

* [TFO---google tcp fast open protocol](http://blog.sina.com.cn/s/blog_583f42f101011veh.html)
* [wikipedia](https://en.wikipedia.org/wiki/TCP_Fast_Open)
* [TFO简介](http://www.pagefault.info/?p=282)
* [tfo的golang实现(github)](https://github.com/bradleyfalzon/tcp-fast-open)
* [上一行项目的作者bradley falzon](https://bradleyf.id.au/nix/shaving-your-rtt-wth-tfo/)
* [google关于tfo的论文](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/zh-CN/us/pubs/archive/37517.pdf)

