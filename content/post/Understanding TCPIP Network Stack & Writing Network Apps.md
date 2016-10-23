+++
date = "2016-10-20T22:21:55+08:00"	
draft = false
title = "Understanding TCPIP Network Stack & Writing Network Apps"

+++

理解TCP/IP协议栈 实现网络应用
===================================

遇到好文章我就想给翻译下来，觉得写的很好，现在cubrid的一篇TCP/IP相关的文章[详细介绍了TCP协议栈，以及收发数据包的流程，非常有启发意义](http://www.cubrid.org/blog/dev-platform/understanding-tcp-ip-network-stack/)，所以我就想翻译一下，做个记录。

我们不能想象没有TCP/IP，互联网服务将会是何种情况。所有我们开发和使用的Internet服务都基于一个坚实的基础：TCP/IP。理解数据如何在网络中传输可以帮助你通过优化和调试的方式来提升程序性能，引入新的技术。

本文将通过在linux操作系统和硬件层面的数据流和控制流来描述网络技术栈的整体操作流程。

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
	* 发送方都想尽可能的发送数据给接收方，但是发送方也得能够有能力接收，因此接收方要发送自己能够接收的最大数据量给发送方知道，最终发送方发出长度数据量由接收方的***接收窗口***决定。
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

让我们大致看看user area，首先应用程序准备好数据，然后调用***write()***系统调用发送数据。假设所用的socket(图中write调用的参数fd)合法，那么当发起系统调用后，发送流程切换到kernel area。

POSIX系列操作系统例如Linux和Unix通过一个file descriptor，即文件描述符fd向应用程序暴露所用的socket。在POSIX系系统中，socket也是一种文件，应用程序使用的fd在进程中有其对应的file structure，与socket对应，图1中的文件曾进行简单的检查，然后通过调用socket的相关函数。--

内核的socket有两个buffer：

1. 一个是send socket buffer，发送缓冲区，用于发送
2. 一个是receive socket buffer，接收缓冲区，用于接收

当***write***系统调用被调用时，待发送数据从用户空间复制到内核内存中，然后添加进发送缓冲区的末尾。这样就可以按顺序发出数据。图一中的'Sockets'那层对应的右边灰色的小格子指向socket buffer中的数据。

然后，TCP被调用了。

每个tcp类型的socket都有一个***TCP Control Block(TCB)***tcp控制块的数据结构，包括了用于处理一个TCP连接所需要的元素，比如connection state连接状态(LISTEN, ESTABLISHED, TIME_WAIT等)、receive window接收窗口，congestion window拥塞窗口、sequence number包序号和resending timer重传定时器等。 
如果当前TCP状态允许数据传输，会新建一个新的TCP segment(packet，报文)；否则系统调用结束并返回错误码。

下图是一个TCP报文，包括两个TCP片段：TCP header和Payload，如图2所示

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/tcp_frame_structure.png" width="70%" height="70%"><p>Figure 2: TCP Frame Structure .</p></div>

payload部分是待发送的数据，从系统的未确认(unACK) socket发送缓冲区中获得，payload的最大长度由对方接收窗口大小、拥塞窗口大小和maximum segment size（MSS，最大报文长度）共同决定。

-- 计算packet的checksum，目前由NIC用硬件实现

然后TCP报文进入下一层IP层处理，IP层添加IP头部，并进行IP路由选择。IP路由选择是一个选择下一跳的过程。当IP层计算并添加IP头部校验checksum后，将数据包发送到下一层Ethernet层，即数据链路层。Ethernet层采用ARP协议搜索查询下一跳IP的MAC地址，然后向报文添加Ethernet头部。添加完Ethernet头部后，host部分的报文就处理完毕了。

在IP路由选择执行完毕后，根据选择结果选择哪个NIC作为传输接口，然后调用NIC驱动发送数据。

此时，如果一个抓包软件比如tcpdump或者wireshark正在运行，kernel将报文从内核态复制一份到这些软件内存中。同样的，如果是抓接收到的包，也同样是从NIC驱动这里抓取的。

NIC驱动程序通过约定的通信协议向NIC请求发送packet。

NIC收到发送网络包请求后，将报文复制到自己的内存中然后发送到网络。发送前，还要修改一些标志，包括packet的CRC校验码，IFG（Inter-Frame Gap）--（ At this time, by complying with the Ethernet standard, it adds the IFG (Inter-Frame Gap), preamble, and CRC to the packet. The IFG and preamble are used to distinguish the start of the packet (as a networking term, framing), and the CRC is used to protect the data (the same purpose as TCP and IP checksum). Packet transmission is started based on the physical speed of the Ethernet and the condition of Ethernet flow control. It is like getting the floor and speaking in a conference room.）（待翻译）


当NIC发送一个数据报文，NIC向CPU发出中断，每个中断有其自己的中断号，操作系统根据中断号调用对应的驱动程序处理中断。NIC驱动启动时注册中断回调函数，OS调用中断服务程序，然后中断服务程序返回已发送的数据包给OS。

至此我们讨论了应用程序数据发送的流程，贯穿kernel和NIC设备。而且，即使没有应用程序的写请求，kernel可以调用TCP层直接发送数据包。例如，当收到一个ACK后并且得知对端接收窗口扩大，kernel将自动的把仍在发送缓存中的数据打包，直接发出。

数据接收
------------------------

现在我们看看数据的接收流程，当数据包到来的时候，网络栈是如何处理的，如图3所示。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/operation_process_by_each_layer_of_tcp_ip_for_data_received.png" width="70%" height="70%"><p>Figure 3: Operation Process by Each Layer of TCP/IP Network Stack for Handling Data Received.</p></div>

首先，NIC将数据包写入自身内存，检查该包是否CRC合法，然后将该包发送给host的内存，host的内存是NIC驱动事先向kernel申请的内存，用于接收数据包，当host分配成功，通过NIC驱动告诉NIC这块内存的地址和大小。如果NIC driver没有实现分配好内存，NIC收到数据包后会直接丢弃。

当NIC将数据包写入到host的内存缓冲区后，NIC向host 操作系统发出中断信号。

-- 然后，NIC驱动检查它是否可以处理这个新包，至此都是NIC和NIC驱动之间的 （删除）

当驱动需要将数据包发送到上一层时，这个数据包必须被包装成OS可以理解的包格式。比如，linux上的sk_buff，BSD系列内核的mbuf结构，或者MS系统的NET_BUFFER_LIST结构。NIC驱动将封装后的数据包转给上层处理。

Ethernet层检查数据包是否合法，然后根据数据包头部的ethertype值选择上层网络协议。IPV4类型的值为0x0800。本层的工作就是去掉数据包的Ethernet头部，传送给上层IP层。

IP层同样首先检查数据包合法性，采用检查IP头部的checksum字段的方式。在逻辑上决定是否进行IP路由选择，本机操作系统处理这个包，还是转发给另一个系统。如果本机处理数据包，那么IP层将根据IP头部的协议proto值选择上层传输层协议，比如TCP协议的proto值是6.本层的工作就是移除IP头部，发送给上层TCP层。

同样的，TCP层检查数据包的checksum是否正确。之前说过，TCP的checksum也是由NIC计算得到的。（难道这些检查都是在NIC层面上做，这一步只是逻辑上放在这里的？）

然后开始搜索这个数据包对应的TCP Control Block，采用IP:PORT四元组作为标志查找。找到TCP控制块后，根据包协议处理数据包。如果是收到新数据，那么将其加入socket接收缓冲区中。根据TCP状态，协议栈发送TCP回复包（比如ACK包）。现在TCP/IP的接收数据流程完成了。

socket接收缓冲区的大小是TCP接收窗口大小。数据接收时，TCP接受窗口扩大时TCP的吞吐能力增大；在此之前，socket的缓冲区大小由应用程序或者操作系统配置来调整，而现在新的网络栈具有自动调整接受缓冲区大小的功能。

当应用程序调用read系统调用时，从user area切换到kernel area，数据从socket的缓冲区复制到user area，然后从socket缓冲区中去除，然后TCP层被调用。当socket缓冲区仍有空间时，TCP增大接受窗口。And it sends a packet according to the protocol status. If no packet is transferred, the system call is terminated.（待翻译）

网络栈开发方向
-------------------------

以上描述的网络栈各层的功能都是一些基本的功能。90年代早期的网络栈功能比以上描述的还少。不过，目前最新的网络栈的功能更加丰富，复杂度更高，这些新功能根据用途有如下分类：

* Packet Processing Procedure Manipulation, 控制修改包处理流程

类似于Netfilter(firewall, NAT)和流量控制。通过在数据包基本处理流程中插入用户代码可以实现不同功能。

* Protocol Performance, 协议性能提升

目的是在同样的网络质量情况下，提升吞吐量、降低时延，提高稳定性。多种拥塞控制算法和附加TCP功能比如SACK（选择确认）就是这类功能。通过协议提升性能在本文中不作重点讨论。

* Packet Processing Efficiency, 数据包处理效率

包处理效率相关的功能旨在提升每秒能够处理最大量的数据包，通过降低用于处理数据包的CPU使用率、内存占用和内存访问次数。目前有多种降低系统时延的尝试，包括并行处理、头部预测、零拷贝、单一副本、免校验、TSO、LRO和RSS等。

网络栈的控制流
-------------------------

现在我们可以从更细节的角度观察linux网络栈的内部流程。就像其他非网络栈的子系统，linux的网络站以事件驱动的方式，当网络事件发生时进行相应处理，也就是说网络栈内只有一个进程或者控制流处理运行（可修饰）。上文的图1和图3简单的表示了控制流的数据包，图4将显示更多细节。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/control_flow_in_stack.png" width="70%" height="70%"><p>Figure 4: Control Flow in the Stack.</p></div>

图4的控制流(1)中，应用程序通过系统调用比如read()和write()使用TCP，此处没有数据包的传输。控制流(2)与控制流(1)的不同之处在于，它用于调用TCP层后发送数据包，它创造一个数据包然后将该包发送到NIC驱动前的一个队列中，然后队列的实现方式决定何时将该包发送给NIC驱动。这是linux中的队列方式，linux的流量控制功能就是操作这个队列实现的，默认的操作方式是FIFO，先进先出。通过使用另一种队列控制方式，linux可以实现多种效果，比如人工控制丢包、包延迟和流量限制等等功能。在控制流(1)和(2)中，应用程序的进程也将调用NIC驱动。

控制流(3)表示TCP用到的一些定时器，比如当TIME_WAIT定时器超时后，TCP协议栈将响应并删除超时的连接。

与控制流(3)类似，(4)表示超时后TCP将处理一系列待处理的数据包。比如，当重传定时器超时后，未得ACK确认的包将被重传。

控制流(3)和(4)显示定时器软中断的处理流程。

当NIC驱动收到NIC中断，它将释放已传输的数据包。大部分情况下，NIC驱动的处理流程在这里就终止了（莫非是发送数据完成后收到NIC的通知？）。控制流(5)表示数据包在传输队列中累积，NIC驱动请求软中断，然后软中断处理函数从发送队列中将累积的数据包发送给NIC驱动（请结合(5)左边的黑线）。

当NIC驱动收到中断并且收到一个新的数据包，它将请求软中断。处理接收数据包的软中断调用NIC驱动并将收到的数据包传给上层处理。在LInux中，如上描述的处理接受数据包的处理方式称为New API(NAPI)。NAPI与轮询类似，因为NIC驱动并不直接向上层发送数据，而是上层从NIC驱动中拿数据包，这段代码称为NAPI poll(NAPI轮询)。

控制流(6)显示TCP协议栈接受数据包的完整处理流程，控制流(7)表示请求更多数据包进行发送到过程。控制流(5)、(6)、(7)都是由NIC发起中断，软中断处理NIC中断实现的。

怎样处理中断然后接受数据包
---------------------------------

中断处理是复杂的，毕竟你需要理解与接受数据包有关的各个环节。图5显示了中断处理流程图。


<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/processing_interrupt_softirq_and_received_packet.png" width="70%" height="70%"><p>Figure 5: Processing Interrupt, softirq, and Received Packet.</p></div>

想象下CPU 0正在执行应用程序，这个时候NIC收到一个数据包，向CPU 0产生一个中断。然后CPU执行内核中断处理程序。内核通过中断号调用中断处理程序，调用相应驱动的中断处理程序。NIC驱动释放已发送完成的数据包，然后调用napi_schedule()函数去接受数据包，该函数请求软中断（参考图4中(6)右边的黑线）。NIC驱动的中断处理程序结束，返回，控制权交回内核中断处理程序，内核中断处理程序执行刚才NIC驱动调用的napi_schedule()产生的软中断。硬中断上下文执行完成后，软中断开始执行（这里是内核的tasklet或者work_queue了吧），软硬中断上下文都是由同一个进程执行的（linux kernle吧）。不过，软硬中断的执行栈不一样，硬中断将会屏蔽硬件中断，软中断执行期间是不屏蔽的（老生常谈）。

软中断处理程序调用net_rx_action()处理收到的数据包，这个函数调用驱动的poll()方法。poll()方法调用netif_receive_skb()方法收取数据包，然后讲其逐层向上层传送。处理完软中断后，应用程序从断点返回，此时可以开始调用系统调用比如read读取数据。

这就是CPU从收到硬中断到完成接收数据包的完整过程，Linux、BSD和MS Windows系统都是大同小异的。

当你查看服务器CPU使用率时，有事你可以看到只有一个CPU执行软中断，这个现象我们上文描述的可以解释，只有CPU 0在响应网卡中断，使用多队列网卡、RSS和RPS（在软件层面模拟实现硬件的多队列网卡功能）可以解决这个问题，将软中断绑定到多个CPU上。

相关数据结构
------------------

下文列出一些关键性的数据结构

## sk_buff structure

首先，sk_buff结构体或者skb结构体表示一个数据包，图6表示sk_buff结构体的主要部分，足以说明sk_buff相关的通用方法。

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/packet_structure_sk_buff.png" width="70%" height="70%"><p>Figure 6: Packet Structure sk_buff.</p></div>

### sk_buff包括数据包的Data部分和元数据部分

sk_buff直接包括包数据部分，或者用指针指向它。在图6中，一些
