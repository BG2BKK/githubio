+++
date = "2016-08-24T00:01:05+08:00"
draft = true
title = "检测TCP连接是否断开"

+++

1. [setsockopt方式设置keepalive，可以设置空闲多长时间后开始发送探测报文；](http://blog.sina.com.cn/s/blog_49aeff0b01016nob.html)
[原理是](http://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)，一定时间空闲后，发送keepalive probe报文，如果对端回复ACK，那么可以认为该连接仍然有效；如果对端不回复，则认为该连接已经失效；这时对该连接进行读写操作，返回码-1，错误描述是timedout

那么现在的问题是，keepalive probe包是否也和其他普通tcp包一样，重传收不到ack的话，会根据tcp_retries1和tcp_retries2重传，这样的话，时间会很长吧

上面那句的担忧是没有必要的，因为keepalive probe是一个必须被回应的报文，如果没有回复，那就报timedout错误

[rfc](https://tools.ietf.org/html/rfc1122#page-101)

2. 加入heartbeat机制，定期发送心跳包，从应用层保证联通

[参考链接](http://www.cnblogs.com/youxin/p/4056041.html)


[close后，两端的行为](http://blog.csdn.net/jnu_simba/article/details/9068059)

[读一个已经是RST状态的tcp，返回值-1，错误描述connection reset by peer](http://cs.baylor.edu/~donahoo/practical/CSockets/TCPRST.pdf)

[关于如何处理RST包](https://www.snellman.net/blog/archive/2016-02-01-tcp-rst/)
