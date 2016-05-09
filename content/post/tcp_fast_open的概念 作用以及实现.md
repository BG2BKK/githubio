+++
date = "2016-05-09T15:53:34+08:00"
draft = false
title = "tcp_fast_open的概念 作用以及实现"

+++

引言
------------------

> 三次握手的过程中，当用户首次访问server时，发送syn包，server根据用户IP生成cookie，并与syn+ack一同发回client；client再次访问server时，在syn包携带TCP cookie；如果server校验合法，则在用户回复ack前就可以直接发送数据；否则按照正常三次握手进行。

> TFO提高性能的关键是省去了热请求的三次握手，这在充斥着小对象的移动应用场景中能够极大提升性能。

Google研究发现TCP 二次握手是页面延迟时间的重要部分，所以提出TFO

TFO的fast open标志体现在TCP报文的头部的[OPTION字段](http://www.iana.org/assignments/tcp-parameters/tcp-parameters.xhtml)

TCP Fast Open的标准文档是[rfc7413](http://tools.ietf.org/html/rfc7413)

TFO与2.6.34内核合并到主线，[lwn通告地址](https://lwn.net/Articles/508865/)



参考链接
---------------

* [TFO---google tcp fast open protocol](http://blog.sina.com.cn/s/blog_583f42f101011veh.html)
* [wikipedia](https://en.wikipedia.org/wiki/TCP_Fast_Open)
* [TFO简介](http://www.pagefault.info/?p=282)
* [tfo的golang实现(github)](https://github.com/bradleyfalzon/tcp-fast-open)
* [上一行项目的作者bradley falzon](https://bradleyf.id.au/nix/shaving-your-rtt-wth-tfo/)
* [google关于tfo的论文](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/zh-CN/us/pubs/archive/37517.pdf)
