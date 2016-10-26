+++
date = "2016-10-26T10:34:31+08:00"
draft = true
title = "lmbench usenix"

+++

* 数据在CPU、内存、网络、文件系统和磁盘之间的传输
	* 带宽
	* 时延

lmbench与其他benchmark工具不一样的地方
------------------------------------

* IO(disk)
	* IOstone
		* 侧重于压测内存子系统的速度
	* IObench
		* 侧重于测量子系统：文件系统和磁盘的性能，这样会比较复杂且笨重
	* 另一篇论文
		* 看到很多IO性能测试历程，有不足：运行时间长、对于简单问题提出的解决方案太复杂
	* lmbench
		* lmdd
			* 测试顺序IO和随机IO，运行速度快
			* Chen和Patterson通过不同大小的数据量测试系统IO性能，而我们更偏向于在单个请求中的CPU消耗，较少关注测试中系统整体性能
* Berkeley Software Distribution's miscrobench suite
	* BSD出品一个可扩展的benchmark，用于BSD系统的回归测试，包括质量和性能。
	* 我们没有用它作为出发点(借鉴了观点)，原因是
		* 缺乏某些测试，比如内存时延
		* 有些测试太多，可能会使测试结果淹没在大量数据中
		* 采用了BSD license

* Ousterhout's Operating System benchmark
	* 提出一些测试用例
		* 测试系统调用延迟
		* 上下文切换时间
		* 文件系统性能等
	* 我们借鉴其idea作为工作基础，并在此之上进行扩展
	* 干的漂亮，待翻译。

* 网络测试
	* netperf测试网络带宽和时延，lmbench涵盖一个更小，复杂度较低的benchmark，打到同样效果
	* ttcp是应用广泛的benchmark，我们实现的版本在相同测试用例下，带宽差距小于2%，因此我们的测试用例是可信的

* McCalpin's Stream benchmark

* 总之
	* 我们搞了一套自己的，因为我们想要一个简单、可移植的benchmakr，想准确全面的测量我们认为对今天计算机系统性能的关键点。
	* 也借鉴了其他benchmark的idea。

benchmark注意事项
------------------------------

* 测试用例的大小问题
	* 比如做内存复制测试内存速度，如果太小，会被缓存，这样侧出来的数据将比数据在内存中快10倍多；然而如果过大的话，数据可能会被换页到磁盘，这样又会造成速度过慢，从而让benchmark过程显得漫长无结果。
	* lmbench采用如下两个方式解决：
		* 所有benchmakr使用循环的话都会被cache大小影响，所以循环同时增大数据量(2的倍数)直到到达最大值。这个结果可以打印出来，当benchmark不再符合cache长度的时候
		* benchmark确定系统内存足够。用一个小测试程序分配尽可能多的内存，然后清空这些内存，然后每次越过一页内存，计算每次结果。如果每次应用都需要ms级别时间，说明该页已经不在内存中了。然后测试程序从小的开始，然后干活直到有充足内存或者到达内存限制。
	
* 编译器的时间问题
	* 所有benchmark工具都是-O级别编译优化的；除了计算时钟速度和上下文切换时间时不能用优化，以产生正确结果。
	* 没有其他的优化选项，我们想从应用程序作者的角度观察性能问题。

* 多处理器相关
	* 多处理器系统在做benchmark时和单处理器一样，没有区别。有些系统允许用户将程序绑定到某个CPU上，为了更好的重用cache。
	* 我们benchmark时不会将程序绑定，因为这种方式与多处理器调度机制相悖。某些情况下，这个决定会导致一些有意思的结果。


* 计时相关
	* 时钟分辨率
		* benchmark软件计算用时是通过调用gettimeofday()获取系统时钟的，在一些系统中这个接口的分辨率是10ms，而大部分测试用例耗时10ms到千ms，所以这个接口太耗时了。为补偿这个定时器，benchmark软件一般通过多次运行来评估，一般来说就是在循环里执行，如果循环体非常快的话就用循环展开的方式多执行几遍，最后求平均。
	* 缓存
		* 如果benchmark软件期待数据在暂存缓存中，那么benchmark就多运行几次，只有最后一次记录结果就行。
		* 如果benchmakr不想测试cache性能，那就将数据大小设置的比cache大。比如，做bcopy测试时，采用复制8MB数据，这个值比当前计算机的二级缓存还大。
	* 结果可变性
		* 很多benchmark结果，比如context switch的性能测试，有一个趋势是，结果会有一定的跳动，可达30%之多。我们怀疑这是因为系统没有使用同一片物理内存，所以遇到了缓存冲突，换页等因素。所以我们多运行几次，选取最小的结果。
		* 用户做benchmark时，应该成为该系统的唯一用户。

* 使用lmbench数据库

* Bandwidth
	* Memory bandwidth
	* IPC bandwidth
	* Cached I/O bandwidth

* Latency measurements
	* Memory read latency background
	* Memory read latency
	* Operating system entry
	* Signal handling cost
	* Process creation costs
	* Context switching
	* Interprocess communication latencies
	* File system Latency
	* Disk latency


* Memory bandwidth
	* 以前评测性能时首先看MFLOPS，但是现在CPU的浮点单元并不是瓶颈，反而将数据从内存中读出来计算是瓶颈
	* 所以现在主要看的是内存的复制、读、写，通过不同大小的数据关心大规模内存传输
