+++
date = "2016-05-09T16:55:42+08:00"
draft = false
title = "SO_REUSEADDR和SO_REUSEPORT"

+++

结论
------------------
    
    1. SO_REUSEPORT用于多个socket监听同一个TCP链接
    2. SO_REUSEADDR可用于多个进程bind同一端口，但需要TCP连接的四元组不一样。
	3. SO_REUSEPORT比SO_REUSEADDR更加扩展，但是也带来了隐患，需要额外注意

引言
------------------------

nginx 1.9.1引入了 SO_REUSEPORT选项，在高版本（linux kernel 3.9以上）系统上可用。该选项允许多个socket监听同一个IP:PORT组合，

* SO_REUSEPORT可以[简化服务器编程](http://freeprogrammersblog.vhex.net/post/linux-39-introdued-new-way-of-writing-socket-servers/2)
* prefork模式：master预先分配进程池，每一个client连接用一个进程处理
    * 省资源，不用每次都fork，然后再回收
    * 可控制，预先分配的进程池大小是固定的
* SO_REUSEPORT使得多进程时不用使master再做管理工作，比如管理子进程，设置信号等等，设置不需要一个master进程，只需要子进程监听同一个端口就行。操作系统做了大部分工作。
    * 这里还有个好处是，C写的server模块，python写的server模块，它们可以共存监听同一个端口，灵活性更好


听听linux kernle[维护者怎么说](https://lwn.net/Articles/542629/)
--------------------------------------------------------------------

* 允许多个进程绑定host上的同一端口
* 只需要第一个绑定端口的进程指定SO_REUSEPORT选项，后继者都可以绑定该端口，所以需要担心的是端口劫持，不希望恶意程序能accept该端口的连接。
* 方法是后继者要与第一次绑定端口的进程的USER ID一样，比如用root和普通用户启动程序绑定同一个端口，会报address already in use
* SO_REUSEPORT的负载均衡性能更好
<!--		* 这里的负载均衡可能指的不是主动分配的，而是当多个线程监听同一端口时，如果某个线程在忙，那么新来的请求自然会被load较低的线程处理，间接的达到均衡效果 -->
* TCP和UDP都可以用
	* UDP场景中，在DNS server的应用比较有意义，可以负载均衡的处理dns请求
	* 作者指出，SO_REUSEADDR虽然也能让UDP连接绑定同一端口，但是SO_REUSEPORT可以防止劫持，并能将请求均衡的分配给监听的线程

传统多线程的工作模式的缺点
----------------------------

* 1. 传统的多线程server都是有一个listener线程绑定端口并接受所有的请求，然后传递给其他线程，而这个listener往往会成为瓶颈
* 2. master绑定端口，每个slave轮流accept从该端口获取连接（nginx）
    * 缺点是有可能导致每个slave不能平均的处理连接，unblanced；有的slave处理的过多，有的slave处理的过少，导致cpu资源不能充分利用
    * SO_REUSEPORT的实现可以使请求平均的分配给堵塞在accept上的各个进程

SO_REUSEPORT的应用举例
---------------------------

* server.py

```python
import socket
import os

SO_REUSEPORT = 15

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, SO_REUSEPORT, 1)
s.bind(('', 10000))
s.listen(1)
while True:
    conn, addr = s.accept()
    print('Connected to {}'.format(os.getpid()))
    data = conn.recv(1024)
    conn.send(data)
    conn.close()
```

启动两个进程，都绑定10000端口；使用nc作为client

```bash
$ python server.py&
[1] 12649
$ python server.py&
[2] 12650
$ echo data | nc localhost 10000
Connected to 12649
data
$ echo data | nc localhost 10000
Connected to 12650
data
$ echo data | nc localhost 10000
Connected to 12649
data
$ echo data | nc localhost 10000
Connected to 12650
data
```

再启动一个新的进程显然也是可以的

```bash
$ python server.py&
[3] 14021
$ echo data | nc localhost 10000
Connected to 12650
data
$ echo data | nc localhost 10000
Connected to 14021
data
```

SO_REUSEPORT 和 SO_REUSEADDR 对比（待续）
-------------------------------------

* 前者可以防止端口被恶意进程劫持
* 前者可以使请求平均分配给各个进程

参考链接
----------------------

* [lwn: the SO_REUSEPORT socket option](https://lwn.net/Articles/542629/)
* [topic on so_reuseaddr and so_reuseport on stackoverflow](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t)

