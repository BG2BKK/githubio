+++
date = '2016-08-18T11:51:16+08:00'
draft = true
title = 'redis client buffer'

+++


* [redis客户端的两个buffer](https://zhuoroger.github.io/2016/07/30/redis-client-two-buffers/)
	* query buffer
		* 相当于redis为客户提供的输入buffer，不论用户执行get，还是set，都会将命令和参数写到该buffer
	* output buffer
		* 相当于redis为客户提供的输出buffer，所以对客户的输出，不论大还是小，都会写到该buffer然后输出

* 查询redis client属性
	* redis命令：client list

```bash
10.13.112.54:6379> client list
id=222464 addr=10.13.112.54:46957 fd=8 name= age=46008 idle=1 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=expire
id=222543 addr=10.13.112.54:34158 fd=7 name= age=41737 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=select
id=223496 addr=10.13.112.54:36934 fd=10 name= age=593 idle=2 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=expire
id=223514 addr=10.75.12.239:60522 fd=26 name= age=32 idle=32 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=hincrby
id=223505 addr=10.13.112.54:43949 fd=16 name= age=64 idle=64 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=hincrby
id=223506 addr=10.13.112.54:44754 fd=17 name= age=61 idle=61 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=select
id=223519 addr=172.16.193.186:32256 fd=31 name= age=22 idle=22 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=hincrby
id=223504 addr=172.16.193.186:40687 fd=15 name= age=68 idle=33 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=hincrby
```

* qbuf: query buffer 大小
* qbuf-free: query free buffer 大小

* obl: 定长 output buffer的使用字节数
* oll: 可变 output buffer的对象个数
* omem: 可变 output buffer的使用字节数

query buffer
---------------------------

redis动态调整每个客户端的query buffer大小，范围0~1GB之间，当某个客户端query buffer使用超过1GB，redis会立刻关闭它，以防OOM

* query buffer的大小限制是硬编码的：

```cpp
server.h#163
/* Protocol and I/O related defines */
#define PROTO_MAX_QUERYBUF_LEN  (1024*1024*1024) /* 1GB max query buffer. */
```

* query buffer大小不受maxmemory限制
	* 文章中作者模拟100个客户端，连续写入500MB的Key，此时内存占用43GB，而maxmemory限制为4GB，因此这相当于是一个bug。
	* 虽然maxmemory不能限制该buffer，但是该buffer大小却会计入maxmemory，此时会触发redis的LRU淘汰机制，或者无法写入


output buffer
--------------------

客户端output buffer有两种：静态大小buffer和动态buffer

* 静态大小buffer
	* 定长16KB，用于存储返回小结果
* 动态大小buffer
	* 存储大的结果
	* redis提供配置动态output buffer的指令

```
client-output-buffer-limit normal 10mb 5mb 60	配置普通客户端的output buffer大小，硬限制是10mb，软限制为5mb/持续60s
client-output-buffer-limit slave 256mb 64mb 60	配置redis从机的output buffer大小，硬限制是256mb，软限制为64mb/持续60s
client-output-buffer-limit pubsub 32mb 8mb 60	配置pub/sub客户端的output buffer大小，硬限制是32mb，软限制为8mb/持续60s
```

client-output-buffer-limit的解读是，如果buffer大小超过硬限制，redis立刻断开客户端；如果buffer大小超过软限制，并且持续时长超过60s，则redis断开客户端连接

如果硬限制和软限制设置为0，则redis将不会主动断开，显然这是不安全的

redis主动断开的设置，在设置timeout时，如果设置为0，也不会主动断开客户，而是等待客户主动断开

由于每个客户端都会有大小不等的output buffer，所以当客户端总量很大时，占用内存也是非常客观的；因此开发时客户端应该节省使用redis，尽量使用小一点的key；限制slave的buffer大小，避免返回被杀；还有就是，如果监控redis发现 used memory 抖动严重，那么极有可能是client的大量来袭或者请求访问的动态变化导致。
