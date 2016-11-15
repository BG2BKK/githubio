+++
date = "2016-10-20T22:21:55+08:00"	
draft = false
title = "Understanding TCPIP Network Stack & Writing Network Apps"

+++

理解TCP/IP协议栈 实现网络应用
===================================

> 遇到好文章我就想给翻译下来，觉得写的很好，现在cubrid的一篇TCP/IP相关的文章[详细介绍了TCP协议栈，以及收发数据包的流程，非常有启发意义](http://www.cubrid.org/blog/dev-platform/understanding-tcp-ip-network-stack/)，所以我就想翻译一下，做个记录。将TCP/IP协议栈在一篇文章内讲明白是不可能的，所以本文能够做到的是讲清楚TCP/IP协议栈收发数据包的流程，我们要做的是首先了解大致流程，然后尝试根据TCP/IP协议和拿着代码去理解。由于我本人能力十分有限，译文都是我个人理解，所以会有大量错误，希望您能帮我纠正。如果您想转载，我必须提醒一句我这个译文是自己学习之用，并未取得版权方同意，因此首先做免责声明。


我们不能想象没有TCP/IP，互联网服务将会是何种情况。所有我们开发和使用的Internet服务都基于一个坚实的基础：TCP/IP。理解数据如何在网络中传输可以帮助你通过优化和调试的方式来提升程序性能，引入和使用新的技术。

本文将通过在linux操作系统和硬件层面的数据流和控制流来描述网络技术栈的整体执行流程。

TCP/IP的关键字
-----------------------

***我如何设计网络协议，能够在保持数据不丢失不乱序的情况下，快速传输数据？***

TCP/IP协议为这些考虑而设计，如下是理解TCP/IP协议栈需要了解的关键字

	确切的说，TCP和IP是两个不同的层，理应分开描述；不过惯例上一直将他俩合成一个概念来讲

1. CONNECTIONI-ORIENTED，面向连接的
	* 首先通信双方需要建立一条连接，一条TCP连接的标识符是***local IP address, local port number***和***remote IP address, remote port number***组成的四元组
2. BIDIRECTIONAL BYTE STREAM, 双向数据流传输
	* 使用字节流实现双向传输
3. IN-ORDER DELIVERY，顺序发送
	* 接收方按照数据从发送方发送的顺序接收；采用32bit整型作为数据包的序号，以实现顺序传输
4. RELIABILITY THROUGH ACK，通过ACK实现可靠性
	* 当发送方发送数据后，没有收到接收方传来的该包的ACK，发送方将重新发送该数据。因此，发送方的TCP需要将未被ACK的数据缓存起来。
5. FLOW CONTROL，流控
	* 发送方都想尽可能的发送数据给接收方，但是对端也得能够有能力接收，因此接收方要发送自己能够接收的最大数据量给发送方知道，最终发送方发出数据量由接收方的***接收窗口***决定。
6. CONGESTION CONTROL，拥塞控制
	* 拥塞窗口是除接收窗口之外的另一个通过限制在途数据流大小以防止网络拥塞的方法。发送方尽可能多的发出拥塞窗口允许的数据量，该窗口大小有诸多方法可以实现，Vegas、Westwood、BIC或者CUBIC。不同于流控中的接收窗口，拥塞窗口是由发送方单独确定的。


数据发送
------------------------

如下图所示，一个网络栈有很多层，图中包含各层类型。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/operation_process_by_each_layer_of_tcp_ip.png" width="70%" height="70%"><p>Figure 1: Operation Process by Each Layer of TCP/IP Network Stack for Data Transmission.</p></div>

图中虽然有多层，但可以简要分为3类：

1. User area 用户区
2. Kernel area 内核区
3. Device area 设备区

在user area和kerne area处理的任务都是由CPU完成的，所以user area和kernel area统称为***host***来与device area加以区分。在这里的device是***Network Interface Card(NIC)***，也就是网卡，用于收发数据，NIC是一个比我们常用的"局域网网卡"更准确的术语。

让我们大致看看user area，首先应用程序准备好数据(右上角的user data灰色框)，然后调用***write()***系统调用发送数据。假设所用的socket(图中write调用的参数fd)合法，那么当发起系统调用后，发送流程切换到kernel area。

POSIX系列操作系统例如Linux和Unix在应用程序通过一个file descriptor，即文件描述符fd来表示所用的socket。在POSIX系系统中，socket也是一种文件，应用程序使用的fd在进程中有其对应的file structure，与socket对应（file->private_data指向对应的struct socket，此处不影响理解），图1中的文件层进行简单的检查(VFS对write()的权限检查)，然后通过调用socket的相关函数最终实现write()。

内核中每个socket有两个buffer：

1. 一个是send socket buffer，发送缓冲区，用于发送
2. 一个是receive socket buffer，接收缓冲区，用于接收

当***write***系统调用被调用时，待发送数据从用户空间复制到内核内存中，然后添加进发送缓冲区的链表末尾。这样就可以按顺序发出数据。图一中的'Sockets'那层对应的右边灰色的小格子指向socket send buffer中的数据。然后，调用TCP/IP协议栈。

每个tcp类型的socket都有一个***TCP Control Block(TCB)***tcp控制块的数据结构，TCB包括了一个TCP连接所需要的成员，比如connection state连接状态(LISTEN, ESTABLISHED, TIME_WAIT等)、receive window接收窗口，congestion window拥塞窗口、sequence number包序号和resending timer重传定时器等。可以认为一个TCB 代表一条TCP连接。

如果当前TCP状态允许数据传输，会新建一个新的TCP segment(packet，报文)；否则系统调用结束并返回错误码。

下图是一个TCP报文，包括两个TCP片段：TCP header和Payload，如图2所示

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/tcp_frame_structure.png" width="70%" height="70%"><p>Figure 2: TCP Frame Structure .</p></div>

payload部分是待发送的数据，处于socket的未确认(unACK)发送缓冲区，每个包的payload的最大长度由对方接收窗口大小、拥塞窗口大小和maximum segment size（MSS，最大报文长度）共同决定。

然后计算packet的checksum校验码，实际上，checksum计算目前由NIC用硬件实现，放在这里只是为了逻辑通顺。

然后TCP报文进入下一层IP层处理，IP层添加IP头部和checksum，并进行IP路由选择。IP路由选择是选择下一跳的过程。当IP层计算并添加IP头部校验checksum后，将数据包发送到下一层Ethernet层，即数据链路层。Ethernet层采用ARP协议搜索查询下一跳IP的MAC地址，然后向报文添加Ethernet头部。添加完Ethernet头部后，host部分的报文就处理完毕了。

在IP路由选择执行完毕后，根据结果选择哪个NIC作为传输接口；在host处理完报文后，调用NIC驱动发送数据。（一定要注意，NIC和NIC驱动不是一体的，前者是NIC网卡硬件，后者是运行在host和内核的驱动程序，硬件是CPU）

此时，如果一个抓包软件比如tcpdump或者wireshark正在运行，kernel将报文从内核态复制一份到这些软件内存中。同样的，如果是抓接收到的包，也同样是从NIC驱动这里抓取的。一般来说，流量整形工具也是在这一层实现的。

NIC驱动程序通过厂商制定的网卡与主机的通信协议向NIC请求发送packet。

NIC收到发送网络包请求后，将报文复制到自己的内存中然后发送到网络。发送前，为遵守以太网标准，还要修改一些标志，包括packet的CRC校验码，IFG（Inter-Frame Gap）包内间隔和报文头等标志；CRC校验码用于数据保真，其他二者用于区分其实包还是中间包（需要翻译调整）。数据包传输速度根据网络物理速度和以太网流控制条件来调整，一般取低值，并留有一定余量。

当NIC发出一个数据包，NIC向CPU发出中断；每个中断有其自己的中断号，操作系统根据中断号调用对应的驱动程序处理中断，驱动的中断处理函数是NIC驱动在OS启动时注册中断回调函数；当中断发生时，OS调用中断服务程序，然后中断服务程序向OS返回发送完成的数据包（编号）。

至此我们讨论了应用程序数据发送的流程，贯穿kernel和NIC设备。而且，即使没有应用程序的写请求，kernel可以调用TCP/IP协议栈直接发送数据包。例如，当收到一个ACK后并且得知对端接收窗口扩大，kernel将自动的把仍在发送缓存中的数据打包，直接发出。

数据接收
------------------------

现在我们看看数据的接收流程，当数据包到来的时候，网络栈是如何处理的，如图3所示。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/operation_process_by_each_layer_of_tcp_ip_for_data_received.png" width="70%" height="70%"><p>Figure 3: Operation Process by Each Layer of TCP/IP Network Stack for Handling Data Received.</p></div>

首先，NIC将数据包写入自身内存，检查该包是否CRC合法，然后将该包发送给host的内存，host的内存是NIC驱动事先向kernel申请的内存，用于接收数据包，当host分配成功，通过NIC驱动告诉NIC这块内存的地址和大小。如果NIC driver没有实现分配好内存，NIC收到数据包后会直接丢弃。

当NIC将数据包写入到host的内存缓冲区后，NIC向host 操作系统发出中断信号。

然后，NIC驱动来确认它是否可以处理这个新包，这个过程使用的是NIC和NIC驱动之间的通信协议。

当驱动需要将数据包发送到上一层时，这个数据包必须被包装成OS可以理解的包格式。比如，linux上的sk_buff，BSD系列内核的mbuf结构，或者MS系统的NET_BUFFER_LIST结构。NIC驱动将封装后的数据包转给上层处理。

链路层Ethernet层检查数据包是否合法，然后根据数据包头部的ethertype值选择上层网络协议。IPV4类型的值为0x0800。本层的工作就是去掉数据包的Ethernet头部，传送给上层IP层。

IP层同样首先检查数据包合法性，采用检查IP头部的checksum字段的方式。在逻辑上进行IP路由选择，决定是否本机操作系统处理这个包，还是转发给另一个系统。如果本机处理数据包，那么IP层将根据IP头部的协议proto值选择上层传输层协议，比如TCP协议的proto值是6.本层的工作就是移除IP头部，发送给上层TCP层。

同样的，TCP层检查数据包的checksum是否正确。之前说过，TCP的checksum也是由NIC计算得到的。（可以理解这些CRC校验的工作都是由NIC硬件实现的，如果硬件层没有校验通过，可以直接在网卡丢弃）

然后开始采用IP:PORT四元组作为标志搜索这个数据包对应的TCP Control Block。找到TCP控制块后就找到了TCP连接，根据包协议处理数据包。如果是收到新数据，那么将其加入socket接收缓冲区中。根据TCP状态，协议栈发送TCP回复包（比如ACK包）。现在TCP/IP的接收数据流程完成了。

socket接收缓冲区的大小是TCP接收窗口大小。数据接收时，TCP接受窗口扩大时TCP的吞吐能力增大；在此之前，socket的缓冲区大小由应用程序或者操作系统配置来调整，而现在新的网络栈具有自动调整接受缓冲区大小的功能。

当应用程序调用read系统调用时，从user area切换到kernel area，数据从socket的缓冲区复制到user area，然后从socket缓冲区中释放。然后调用TCP协议栈；因为socket缓冲区有了新的空间，所以TCP增大接受窗口；然后根据该连接的状态发送ACK包或者其他包比如RST。如果进行read系统调用时没有新数据包，那么read()就终止返回。

网络栈进化方向
-------------------------

以上描述的网络栈各层的功能都是一些基本的功能。90年代早期的网络栈功能比以上描述的还少。不过，目前最新的网络栈的功能更加丰富，复杂度更高，这些新功能根据用途有如下分类：

* Packet Processing Procedure Manipulation, 控制修改包处理流程

类似于Netfilter(firewall, NAT)和流量控制。通过在数据包基本处理流程中插入用户代码可以实现不同功能。

* Protocol Performance, 协议性能提升

目的是在同样的网络质量情况下，提升吞吐量、降低时延，提高稳定性。多种拥塞控制算法和附加TCP功能比如SACK（选择确认）就是这类功能。通过协议提升性能在本文中不作重点讨论。

* Packet Processing Efficiency, 数据包处理效率

包处理效率相关的功能旨在提升每秒能够处理最大量的数据包，通过降低单机处理数据包的CPU时间、内存占用和内存访问次数。目前有多种降低系统时延的尝试，包括并行处理、头部预测、零拷贝、单一副本、免校验、TSO、LRO和RSS等。

网络栈的控制流
-------------------------

现在我们可以从更细节的角度观察linux网络栈的内部流程。就像其他非网络栈的子系统，linux的网络站以事件驱动的方式，当网络事件发生时进行相应处理，也就是说网络栈内只有一个进程或者控制流处理运行（其实就是kernel）。上文的图1和图3表示数据包的简化版控制流，图4将显示更多细节。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/control_flow_in_stack.png" width="70%" height="70%"><p>Figure 4: Control Flow in the Stack.</p></div>

图4的控制流(1)中，应用程序通过系统调用比如read()和write()调用TCP/IP协议栈，在这里没有数据包的发送，需要经过协议栈传输。控制流(2)与控制流(1)的不同之处在于，它要求调用TCP后直接发送数据包，参考[raw socket](http://blog.chinaunix.net/uid-23069658-id-3280895.html)的[用法](http://abcdxyzk.github.io/blog/2015/04/14/kernel-net-sock-raw/)。它创造一个数据包然后将该包发送到NIC驱动前的一个队列中，然后队列的实现方式决定何时将该包发送给NIC驱动。这其实就是linux中的队列方式(queue discipline, qdisc)，linux的流量控制功能就是操作这个队列实现的，默认的操作方式是FIFO，先进先出。通过使用其他队列控制方式，linux可以实现多种效果，比如人工控制丢包、包延迟和流量限制等等功能~~待查：同一个队列的不同控制方式discipline，还是多个队列~~。在控制流(1)和(2)中，应用程序的处理流程最终将调用NIC驱动。

控制流(3)表示TCP用到的一些定时器，比如当TIME_WAIT定时器超时后，TCP协议栈将响应并删除超时的连接。

与控制流(3)类似，(4)表示超时后TCP将处理一系列待处理的数据包。比如，当重传定时器超时后，未得ACK确认的包将被重传。

控制流(3)和(4)显示定时器软中断的处理流程。

当NIC驱动收到NIC中断，它将释放已传输完成的数据包。大部分情况下，NIC驱动的处理流程在这里就终止了。控制流(5)表示数据包在传输队列中累积，NIC驱动请求软中断，然后软中断处理函数从发送队列中将累积的数据包发送给NIC驱动（请结合(5)左边的黑线）。

当NIC驱动收到中断并且发现一个新的数据包到来，它将请求软中断。处理接收数据包的软中断调用NIC驱动接收并将收到的数据包传给上层处理。在LInux中，如上描述的处理接受数据包的处理方式称为New API(NAPI)。NAPI与轮询类似，因为NIC驱动并不直接向上层发送数据，而是上层从NIC驱动中主动拿数据包，这段代码称为NAPI poll(NAPI轮询)。~~这里的实现方式有很多，比如NIC收到大量包时，就不会采用中断方式接收，而是改为轮询方式，总之实现比较精妙，值得看看~~

控制流(6)显示TCP协议栈接受数据包的完整处理流程，控制流(7)表示请求额外数据包发送的过程~~比如ACK包？~~。控制流(5)、(6)、(7)都是由NIC发起中断，软中断服务程序处理NIC中断实现的。

怎样处理中断然后接收数据包
---------------------------------

中断处理是复杂的，毕竟你需要理解与接受数据包有关的各个环节。图5显示了中断处理流程图。


<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/processing_interrupt_softirq_and_received_packet.png" width="70%" height="70%"><p>Figure 5: Processing Interrupt, softirq, and Received Packet.</p></div>

想象下CPU 0正在执行应用程序，这个时候NIC收到一个数据包，向CPU 0产生一个中断。然后CPU执行内核中断处理程序。内核通过中断号调用中断处理程序，调用相应驱动的中断处理程序。NIC驱动释放已发送完成的数据包，然后调用napi_schedule()函数去接受数据包，该函数请求软中断（参考图4中(6)右边的黑线）。NIC驱动的中断处理程序结束，返回，控制权交回内核中断处理程序，内核中断处理程序执行刚才NIC驱动调用的napi_schedule()产生的软中断。硬中断上下文执行完成后，软中断开始执行（这里是内核的tasklet或者work_queue了吧，处于中断上下文的话，只能是tasklet，涉及到阻塞方法，比如与NIC设备的通信，就需要wait_queue了），软硬中断上下文都是由同一个进程执行的（linux kernle）。不过，软硬中断的执行栈不一样，硬中断将会屏蔽硬件中断，软中断执行期间是不屏蔽的（老生常谈）。

软中断处理程序调用net_rx_action()处理收到的数据包，这个函数调用驱动的poll()方法。poll()方法调用netif_receive_skb()方法收取数据包，然后讲其逐层向上层传送。处理完以上软中断后，应用程序从断点继续执行，此时可以开始调用系统调用比如read读取数据。

这就是CPU从收到硬中断到完成接收数据包的完整过程，Linux、BSD和MS Windows系统都是大同小异的。

当你查看服务器CPU使用率时，有时你看到只有一个CPU在辛苦的执行软中断，这个现象我们上文描述的可以解释，只有CPU 0在响应网卡中断，使用多队列网卡、RSS和RPS（在软件层面模拟实现硬件的多队列网卡功能）可以解决这个问题，将软中断绑定到多个CPU上。

相关数据结构
------------------

下文列出一些关键性的数据结构

## sk_buff structure

首先，sk_buff结构体或者skb结构体表示一个数据包，图6表示sk_buff结构体的主要部分。虽然sk_buff的功能越来越丰富，也越来越复杂，但图6足以说明sk_buff相关的通用方法。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/packet_structure_sk_buff.png" width="70%" height="70%"><p>Figure 6: Packet Structure sk_buff.</p></div>

### sk_buff包括数据包的Data部分和元数据部分

sk_buff直接包括数据包的数据部分，或者用指针指向它。在图6中，sk_buff结构体中的data成员指向一个skb_shared_info结构体的Ethernet到buffer成员之间的内存，而额外的数据由skb_shared_info的frags成员指向具体的内存页。

sk_buff的一些基本信息，比如头部信息和包数据长度存在元数据区域(元数据待理解，初步认定是skb_shared_info)。如图6所示，链路层头部mac_header、网络层头部network_header和传输控制层头部transport_header都有对应的指针依次指向从元数据开始的地方。这种方式使得TCP协议处理更容易一些。

### 如何添加或删除头部

当在网络协议栈各层处理时，数据包的头部将会被添加或者删除，此时采用指针来移动到不同层的header位置最为方便高效，比如如果要删除Ethernet头部，只需要将头部指针，即sk_buff的head成员指向上一层IP头部位置即可。

### 怎样合并或者分解数据包

在向socket缓冲区高效添加或者删除数据包的数据量时，采用链表的方式最为方便。sk_buffer的next和prev指针成员的目的就是在此。

### 快速分配或者释放内存

当创建一个packet时，一个sk_buff结构体要被分配，这里需要快速分配器。比如，如果数据在万兆网卡传输，那么每秒最多将超过百万包被分配和释放。

## TCP Control Block

第二，需要有一个结构体来表示一条TCP连接。此前，它被笼统的称为TCP控制块。Linux使用tcp_sock结构体表示TCP Control Block，如图7所示，你可以看到socket、tcp_socket和struct file之间的关系。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/tcp_connection_structure.png" width="70%" height="70%"><p>Figure 7: TCP Connection Structure.</p></div>

当一个系统调用执行时，首先搜索进程的fd对应的struct file结构，对于类Uinx操作系统来说，一个socket、一个文件或者一个设备对于普通文件系统来说都抽象成struct file结构。因此，文件系统包括了基本信息，对于一个socket结构体来说，struct socket包含了与socket相关的信息，以及一个file指针，该socket结构体[同样有一个struct sock类型的指针sk成员，struct sock可强制类型转换到struct tcp_sock（参考tcp_sk函数）](http://lxr.free-electrons.com/source/include/linux/tcp.h)。

```cpp
// socket 结构体, include/linux/net.h

/**
 *  struct socket - general BSD socket
 *  @state: socket state (%SS_CONNECTED, etc)
 *  @type: socket type (%SOCK_STREAM, etc)
 *  @flags: socket flags (%SOCK_NOSPACE, etc)
 *  @ops: protocol specific socket operations
 *  @file: File back pointer for gc
 *  @sk: internal networking protocol agnostic socket representation
 *  @wq: wait queue for several uses
 */
struct socket {
	socket_state		state;

	kmemcheck_bitfield_begin(type);
	short			type;
	kmemcheck_bitfield_end(type);

	unsigned long		flags;

	struct socket_wq __rcu	*wq;

	struct file		*file;
	struct sock		*sk;
	const struct proto_ops	*ops;
};

```

socket结构体指向的tcp_sock结构体除了支持TCP协议类型外还有别的比如sock，inet_sock等类型，这点可以视为某种意义上的多态。

所有TCP协议的状态信息保存在tcp_sock结构体中，比如TCP的序号sequence number、接受窗口receive window、拥塞窗口和重传定时器等。

socket发送缓冲区和socket接收缓冲区都是tcp_sock的sk_buff链表；tcp_sock的dst_entry成员，存储IP路由选择结果，避免再次进行路由选择，dst_entry可以进行ARP结果的快速检索，比如对端MAC地址。dst_entry是路由表的一部分，由于路由表的复杂结构，所以不在本文讨论，总之记住dst_entry可用来选择传输本数据包的网络设备NIC，NIC就是dst_entry指向的net_device成员。

因此，通过struct file我们可以很容易的找到与TCP连接的所有信息，只占用少量内存，几KB而已（有很多功能添加进来，所以这部分内存从过去到现在不断增长）。

最后，我们看下TCP连接的查找表，这是一个用于查找所收到数据包对应的TCP连接的哈希表，索引通过数据包的IP:PORT四元组进行Jenkins哈希算法计算得到。选择这个算法的原因是考虑到防范对此哈希表的攻击（待查）。

追踪代码：如何发送数据
---------------------------

我们通过追踪阅读Linux kernel源码来学习TCP/IP协议栈如何执行，通过常用的读数据和写数据来观察。

首先，应用程序调用write()来实现数据发送

```cpp
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, ...)
{
	struct file *file;
	[...]
	file = fget_light(fd, &fput_needed);
	[...] ===>
	ret = filp->f_op->aio_write(&kiocb, &iov, 1, kiocb.ki_pos);
 
struct file_operations {
	[...]
	ssize_t (*aio_read) (struct kiocb *, const struct iovec *, ...)
	ssize_t (*aio_write) (struct kiocb *, const struct iovec *, ...)
	[...]
};
 
static const struct file_operations socket_file_ops = {
	[...]
	.aio_read = sock_aio_read,
	.aio_write = sock_aio_write,
	[...]
};
```

当应用程序调用write()系统调用，内核在文件层执行write()，首先找到fd对应的struct file，然后调用file_operations中的aio_write()，它是一个函数指针，最终调用的是socket_file_ops的sock_aio_write()方法。kenerl中通过函数表的方式实现接口，这点十分常见，但是对于TCP的具体指向方式，可以以后详查(TODO LIST)。

接下来是sock_aio_write()的具体调用过程。

```cpp
static ssize_t sock_aio_write(struct kiocb *iocb, const struct iovec *iov, ..)
{
	[...]
	struct socket *sock = file->private_data;
	[...] ===>
	return sock->ops->sendmsg(iocb, sock, msg, size);
 
struct socket {
	[...]
	struct file *file;
	struct sock *sk;
	const struct proto_ops *ops;
};
 
const struct proto_ops inet_stream_ops = {
	.family = PF_INET,
	[...]
	.connect = inet_stream_connect,
	.accept = inet_accept,
	.listen = inet_listen,
	.sendmsg = tcp_sendmsg,
	.recvmsg = inet_recvmsg,
	[...]
};
 
struct proto_ops {
	[...]
	int (*connect) (struct socket *sock, ...)
	int (*accept) (struct socket *sock, ...)
	int (*listen) (struct socket *sock, int len);
	int (*sendmsg) (struct kiocb *iocb, struct socket *sock, ...)
	int (*recvmsg) (struct kiocb *iocb, struct socket *sock, ...)
	[...]
};
```
sock_aio_write()函数从struct file中获取struct socket，然后调用socket的sendmsg方法，sendmsg依然是个函数指针，指向的是struct socket中的proto_ops函数表的sendmsg，IPv4协议族的TCP类型的proto_ops操作表是inet_stream_ops，将sendmsg实现为tcp_sendmsg。

```cpp
int tcp_sendmsg(struct kiocb *iocb, struct socket *sock, struct msghdr *msg, size_t size)
{
	struct sock *sk = sock->sk;
	struct iovec *iov;
	struct tcp_sock *tp = tcp_sk(sk);
	struct sk_buff *skb;

	[...]

	mss_now = tcp_send_mss(sk, &size_goal, flags);

	/* Ok commence sending. */
	iovlen = msg->msg_iovlen;
	iov = msg->msg_iov;
	copied = 0;

	[...]

	while (--iovlen >= 0) {
		int seglen = iov->iov_len;
		unsigned char __user *from = iov->iov_base;
		iov++;
		while (seglen > 0) {
			int copy = 0;
			int max = size_goal;
	
			[...]
		
			skb = sk_stream_alloc_skb(sk, select_size(sk, sg), sk->sk_allocation);
			if (!skb)
				goto wait_for_memory;
			/*
			* Check whether we can use HW checksum.
			*/
			if (sk->sk_route_caps & NETIF_F_ALL_CSUM)
				skb->ip_summed = CHECKSUM_PARTIAL;
		
			[...]
			skb_entail(sk, skb);
		
			[...]
			/* Where to copy to? */
			if (skb_tailroom(skb) > 0) {
		
				/* We have some space in skb head. Superb! */
				if (copy > skb_tailroom(skb))
					copy = skb_tailroom(skb);
				if ((err = skb_add_data(skb, from, copy)) != 0)
					goto do_fault;
				[...]
			
				if (copied)
					tcp_push(sk, flags, mss_now, tp->nonagle);
			
				[...]
}
```

tcp_sendmsg()首先从参数struct socket \*sock获取tcp_sock，即TCP Control Blcok，然后将应用程序请求发送的数据复制到socket的发送缓冲区中。当复制数据到sk_buff中前，首先获取socket的Maximum Segment Size(MSS，最大消息长度)，MSS代表一个TCP包可携带的最大数据量（当然如果支持TSO或者GSO的话可以大于MSS），然后创建数据包，即sk_stream_alloc_skb()函数创建一个新的sk_buff，返回skb，skb_entail()函数将新建的skb添加到socket的发送缓冲区中（前文提到该缓冲区是一个链表）。skb_add_data函数将应用程序的数据复制到skb的buffer中。所有的数据将在重复调用这一过程中复制完成。此时，socket的发送缓冲区将以链表形式组织起MSS大小的若干个sk_buff。最后，调用tcp_push()函数将可发送的数据以数据包的形式发送出去，实现write()掉用的完整流程。

```cpp
static inline void tcp_push(struct sock *sk, int flags, int mss_now, ...)
	[...] ===>
	static int tcp_write_xmit(struct sock *sk, unsigned int mss_now, ...)
	int nonagle,
	{
		struct tcp_sock *tp = tcp_sk(sk);
		struct sk_buff *skb;
		[...]
		while ((skb = tcp_send_head(sk))) {
			[...]
			cwnd_quota = tcp_cwnd_test(tp, skb);
			if (!cwnd_quota)
			break;
			 
			if (unlikely(!tcp_snd_wnd_test(tp, skb, mss_now)))
			break;
			[...]
			if (unlikely(tcp_transmit_skb(sk, skb, 1, gfp)))
			break;
			 
			/* Advance the send_head. This one is sent out.
			* This call will increment packets_out.
			*/
			tcp_event_new_data_sent(sk, skb);
			[...]

```

tcp_push()函数尽可能的将TCP允许发送的sk_buff按序号发送出去。首先调用tcp_send_head()函数获取发送缓冲区队列头的sk_buff，然后tcp_cwnd_test()和tcp_snd_wnd_test()函数用来检查拥塞窗口和接收窗口是否允许新的数据包发送，如果可以，调用tcp_transmit_skb()函数新建网络数据包，用于发送。

```cpp

static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,int clone_it, gfp_t gfp_mask)
{
	const struct inet_connection_sock *icsk = inet_csk(sk);
	struct inet_sock *inet;
	struct tcp_sock *tp;

	[...]

	if (likely(clone_it)) {
		if (unlikely(skb_cloned(skb)))
		skb = pskb_copy(skb, gfp_mask);
		else
		skb = skb_clone(skb, gfp_mask);
		if (unlikely(!skb))
		return -ENOBUFS;
	}

	[...]

	skb_push(skb, tcp_header_size);
	skb_reset_transport_header(skb);
	skb_set_owner_w(skb, sk);

	/* Build TCP header and checksum it. */
	th = tcp_hdr(skb);
	th->source = inet->inet_sport;
	th->dest = inet->inet_dport;
	th->seq = htonl(tcb->seq);
	th->ack_seq = htonl(tp->rcv_nxt);

	[...]

	icsk->icsk_af_ops->send_check(sk, skb);

	[...]

	err = icsk->icsk_af_ops->queue_xmit(skb);
	if (likely(err <= 0))
		return err;
	tcp_enter_cwr(sk, 1);
	return net_xmit_eval(err);
}

```

tcp_transmit_skb()首先调用pskb_copy()创建待发送sk_buff的副本，仅复制sk_buff的元数据；然后调用skb_push()锁定tcp头部区域，然后填充头部字段，send_check()计算TCP头部的checksum。最后，queue_xmit()将数据包skb转移到下一层IP层，IPv4的queue_xmit指针指向函数ip_queue_xmit()。

```cpp

int ip_queue_xmit(struct sk_buff *skb)
{
	[...]
	rt = (struct rtable *)__sk_dst_check(sk, 0);
	[...]

	/* OK, we know where to send it, allocate and build IP header. */
	skb_push(skb, sizeof(struct iphdr) + (opt ? opt->optlen : 0));
	skb_reset_network_header(skb);
	iph = ip_hdr(skb);
	*((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (inet->tos & 0xff));
	if (ip_dont_fragment(sk, &rt->dst) && !skb->local_df)
		iph->frag_off = htons(IP_DF);
	else
		iph->frag_off = 0;
	iph->ttl = ip_select_ttl(inet, &rt->dst);
	iph->protocol = sk->sk_protocol;
	iph->saddr = rt->rt_src;
	iph->daddr = rt->rt_dst;
	[...]
	res = ip_local_out(skb);
	[...] ===>
	int __ip_local_out(struct sk_buff *skb)
	[...]
	ip_send_check(iph);
	return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,	skb_dst(skb)->dev, dst_output);

[...] ===>
int ip_output(struct sk_buff *skb)
{
		struct net_device *dev = skb_dst(skb)->dev;
		[...]
		skb->dev = dev;
		skb->protocol = htons(ETH_P_IP);
		return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
		ip_finish_output,

[...] ===>
static int ip_finish_output(struct sk_buff *skb)
{
	[...]
	if (skb->len > ip_skb_dst_mtu(skb) && !skb_is_gso(skb))
		return ip_fragment(skb, ip_finish_output2);
	else
		return ip_finish_output2(skb);
```
ip_queue_xmit()方法负责执行IP层的任务，__sk_dst_check()检查缓存的路由结果是否合法。如果此时没有缓存的路由，或者缓存路由结果过期，就会进行IP路由查找。然后调用skb_push()锁定IP包头，填充IP包头部字段。接着调用ip_send_check()计算IP头的checksum，然后使用nf_hook调用netfilter模块，nf_hook()方法设置回调函数为dst_output，该函数被调用时，作为函数指针指向的是ip_output()函数。在ip_output()函数中，设置ip_finish_output()为回调函数，当发送数据包需要被分片发送时，进行分片，否则调用ip_finish_output2()，添加Ethernet Header，进入链路层Ethernet层。这样，一个数据包最终生成。

```cpp
int dev_queue_xmit(struct sk_buff *skb)
[...] ===>

static inline int __dev_xmit_skb(struct sk_buff *skb, struct Qdisc *q, ...)
[...]
	if (...) {
		....
	} 
	else if ((q->flags & TCQ_F_CAN_BYPASS) && !qdisc_qlen(q) && qdisc_run_begin(q)) 
	{
		[...]
		if (sch_direct_xmit(skb, q, dev, txq, root_lock)) {

[...] ===>
int sch_direct_xmit(struct sk_buff *skb, struct Qdisc *q, ...)
	[...]
	HARD_TX_LOCK(dev, txq, smp_processor_id());
	if (!netif_tx_queue_frozen_or_stopped(txq))
		ret = dev_hard_start_xmit(skb, dev, txq);
	HARD_TX_UNLOCK(dev, txq);
	[...]
}

int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev, ...)
	[...]
	if (!list_empty(&ptype_all))
		dev_queue_xmit_nit(skb, dev);
	[...]
	rc = ops->ndo_start_xmit(skb, dev);
	[...]
}
```

上层最终生成数据包后，函数dev_queue_xmit()将数据包发送出去。首先，数据包以qdisc方式传递过去；如果采用默认数据包入队规则（FIFO）并且队列为空，sch_direct_xmit()函数将直接把数据包发送给网卡驱动，越过缓冲队列；该函数调用dev_hard_start_xmit()函数选择对应的驱动并发送。在调用网卡驱动前，设备的TX队列将被加锁，防止在多任务同时访问网络设备。由于kernel已经向设备的TX队列加锁，所以设备驱动的发送代码不需要额外加锁。这和接下来要讨论的并行处理紧密相关。

ndo_start_xmit()函数负责调用NIC驱动代码。在调用之前你可以看到ptype_all和dev_queue_xmit_nit()语句，ptype_all是一个包括处理模块的链表，比如抓包模块，如果一个抓包程序在运行中，这个数据包将被ptype_all复制给这个程序。所以，类似于tcpdump这类软件显示的数据包，其实是发送给网卡驱动的；响应的tcpdump显示收到的数据包也是从这层拿到的。此时数据包未携带checksum，或者如果此时TSO使能的话，NIC将会操作编辑这个包。所以tcpdump抓到的包和实际发到网络的包还是有一定区别的。当完成发送数据包后，网卡驱动的终端处理程序返回发送的sk_buff。

追踪代码：如何接收数据
---------------------------

姑且认为接收代码的流程与发送代码区别不大，所以我们先进行下一部分。

NIC和NIC驱动是怎样通信的
-----------------------------

NIC和NIC驱动之间的通信属于网络栈的最底层，往往被人忽视。然而NIC目前在网络性能方面承担着越来越多的任务，了解基本的操作模式将帮你学习到更多。

驱动和NIC之间是异步通信的。首先，NIC驱动请求发送一个数据包后，CPU转向执行其他任务，并不阻塞等待响应；然后NIC发送数据包，通知CPU；最后NIC驱动将发送完成的数据包返回给上层。与发送一样，数据接收也一样是异步的，首先，NIC驱动请求接收一个包，然后CPU转而执行其他任务，然后NIC收到包后，通知CPU，NIC驱动处理收到的包处理并返回（由之前图4所示，NIC驱动注册时会请求kernel提前分配好缓存收到数据包的内存，NIC接受指令将数据包写入这部分内存）。

所以，有一个空间，用于存放请求和响应是很必要的。大部分NIC使用环状结构(ring structure)，环状结构与普通队列结构类似，有固定的容量，一个单位存储一个请求或者响应。使用时也是按序处理，区别在于到达队列末尾后重头开始，形成一个环。

如图8所示的包发送流程图，我们可以看到ring是怎样工作的。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/driver_nic_communication_how_to_transmit_packet.png" width="70%" height="70%"><p>Figure 8: Driver-NIC Communication: How to Transmit Packet.</p></div>

NIC驱动收到上层发来的数据包后，创建NIC可以理解的发送描述符(send descriptor)，发送描述符包括包大小、物理地址等信息；NIC要求通过物理地址访问NIC驱动的内存，所以NIC驱动需要将包的虚拟地址转换为物理地址。然后NIC驱动将send descriptor加入到发送环状缓冲(Send TX Ring Buffer)，如图8的流程(1)所示。

然后，NIC驱动通知NIC有新的发送请求，见流程(2)，NIC驱动直接将这个请求写入到NIC的内存地址中，在这里，CPU采用Programmed I/O(PIO)的方法，直接将数据写到设备（其实这里如果开发过Linux 设备驱动，比如以前开发过的
PCIE驱动就知道，将PCIE设备的配置寄存器映射到Host的内存空间中，kernel可以像访问自身内存一样读写这些地址，进而将控制指令写入设备中）。被通知的NIC从host的内存（发送环状内存缓冲区）以DMA方式获取发送描述符，见流程(3)。拿到发送描述符后，获得数据包在host内存的的物理地址和大小，然后将数据包以DMA方式读出。

NIC取得发送数据包后，计算包的checksum然后加到数据包里，然后发送，见流程(5)。发送完成后，NIC将发送的数据包数量写回host内存(流程6)；然后向CPU发起中断(流程7)。NIC驱动独处发送了哪些数据包后，将数据包返回。(The NIC sends packets (5) and then writes the number of packets that are sent to the host memory (6). Then, it sends an interrupt (7). The driver reads the number of packets that are sent and then returns the packets that have been sent so far.)

如图9所示，我们可以看到读取数据包流程图.

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/driver_nic_cmomunication_how_to_receive_packets.png" width="70%" height="70%"><p>Figure 9: Driver-NIC Communication: How to Receive Packets.</p></div>

首先，NIC驱动在host分配用于存储接收数据包和接收描述符的内存。接收描述符包括缓冲区大小和物理地址，与发送描述符一样都是物理地址，用于DMA传输。然后，NIC驱动将接收描述符添加到RX ring中(流程1)。通过PIO，NIC驱动将新的接收描述符地址写入NIC中(流程2)，NIC从Rx ring中以DMA方式获取接收描述符，获得用于接收数据包的缓冲区的大小和物理地址并存储(流程3)。

NIC收到数据包后(流程4)，NIC将数据包写入实现分配好的host内存中(流程5),如果网卡有计算数据包checksum的功能，那么NIC此时计算数据包的checksum。接收数据包的大小、checksum和其他信息存储在另一个环状buffer(the receive return ring，接收返回环 )中(流程6)。接收返回环也存储NIC处理接收到数据包的结果，比如返回包。然后NIC发出中断(流程7)，NIC驱动从接收返回包中获取包的信息，然后处理数据包。如果必要的话，NIC驱动还会继续分配内存并重复流程(1)和(2).

在调优网络栈的时候，大家都认为环状缓存大小和中断设置要互相匹配。当发送环状缓存Tx ring比较大时，可以一次发出较多请求；当Rx ring比较大时，可以一次收到较多数据包。大Ring buffer可以并发大量发送操作，提高工作能力；实际实现中，NIC使用一个定时器定期收集处理中断，减少CPU中断的次数，以免CPU为处理中断而分心。

缓存和流控制
--------------------

流控制是网络栈各层通力合作实现的。图10显示发送数据时网络栈的各级缓存。首先，应用程序创建数据，添加到socket发送缓存中，如果缓存没有内存可用，则send/write系统调用返回失败或者堵塞。因此，应用程序流向kernel的数据流速由socket缓冲区大小来限制。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/buffers_related_to_packet_transmission.png" width="70%" height="70%"><p>Figure 10: Buffers Related to Packet Transmission.</p></div>

TCP协议栈创建和发送数据包，通过发送队列transmit queue(qdisc)向NIC驱动发送。这是个典型的FIFO队列，队列长度可以由ifconfig工具配置，执行ifconfig工具结果中的txqueuelen的值，一般为1000，意味着缓存1000个数据包。

环状发送队列(TX ring)处于NIC驱动和NIC之间，正如上一章提到的，Tx ring可被认为是发送请求队列。如果Tx ring满，此时NIC驱动不能发出发送请求，那么待发送的数据包将会累积在TCP/IP协议栈和NIC驱动之间的qdisc中，如果累积数据包超过qdisc大小，那么再想发送新包，会被直接丢弃。

NIC将待发送数据包存储在自身缓存中，包速率主要由NIC的物理速度决定。而且由于链路层Ethernet layer的流控制，如果NIC的接收缓冲区没有空间，那么发送数据包也将停止（可以猜测原因是自身停止发送后，对端将不会再发送数据包过来，有助于NIC和NIC驱动将拥塞在接受缓冲区的数据包处理完）。

当发送自kernel的数据包速度大于发送自NIC的数据包速度时，包将拥堵在NIC的缓存中。如果NIC自身缓存没有多余空间，NIC将不会从Tx ring中去取发送请求request；这样的话，越来越多的发送请求累积在Tx ring中，最终Tx ring也堵满；此时NIC驱动再也不能发起新的发送请求，并且要发送的新包将堵塞在qdisc中；就这样，性能衰退从底向上传递。（感觉这里我们可以通过检测各级buffer的堵塞情况，判断程序堵塞在哪一步）

图11显示接受数据包的传递过程。首先，收到的数据包将缓存在NIC自身缓存中。从流控制的角度来看，NIC和NIC驱动之间的Rx ring队列作为缓存，NIC驱动从Rx ring中将已接受数据包的请求取出，发给上层，在这里NIC驱动和协议栈之前没有缓冲区，因为这里是通过kernel调用NAPI去poll已收到的数据包的。(需要想想怎么翻译).这里可以认为上层直接从Rx ring中获取数据包。网络包的数据部分将上传缓存在socket的接收缓冲区中，应用程序随后从socket的接受缓冲区中读取数据。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/buffers_related_to_packaet_receiving.png" width="70%" height="70%"><p>Figure 11: Buffers Related to Packet Receiving.</p></div>

To Be Continued
------------------------

[ldd](https://static.lwn.net/images/pdf/LDD3/ch17.pdf)

