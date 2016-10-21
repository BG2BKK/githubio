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


数据传输
------------------------





