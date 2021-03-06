+++
date = "2016-10-23T16:43:44+08:00"
draft = false
title = "怎样尽可能全面的评估一台服务器的性能"

+++

给你一台服务器，怎样能够全面评估它的性能？需要测试哪些指标？请写出每个指标的具体测试原理和测试代码。假设这台服务器完全处于线下。

这个问题看似平常，但是细细审题的话，发现还是不一样的。我们过多关注线上机器的性能，但是如果单独拿出来一台服务器，它的性能怎样呢？

评估和压榨一台服务器性能的话，找到评估的指标，然后进行压测加观察的方式，得到性能参数。

1. 观察哪些指标，意义何在？
2. 如何观察这些指标，测试原理，观测工具/统计代码
3. 如何进行压测，实现测试场景，压测工具/压测代码

bench可以分类为macro bench和micro bench;对于macro bench，很多时候我们得到的是一个整体而粗略的结果，通过top我们可以看到系统负载，这些对于我们定位线上问题，分析应用程序的性能热点很有帮助，然而这并不能精确衡量一台Linux Server的性能；应用程序多种多样，线上系统目的各不相同，所以macro bench一般用于case by case的性能分析，[用于解决应用的性能瓶颈](https://github.com/BG2BKK/githubio/blob/master/content/post/macrobench.md)；而micro bench可以定量的分析一台机器+操作系统的性能，采用相同的测试基准，通过无干扰的大量重复基本操作，比如从L1 Cache读取单个字节，耗时一个CPU时钟，在大量重复中得到较为宏观的结果，再排除loop损耗、取运行时间的损耗，可以用于基本衡量一台server的运行效率。

```bash
./bin/x86_64-linux-gnu/enough

结果为n=322229, u=5172，执行322229次 TEN( p = *p )，

TEN(T)表示循环展开执行10次任务T，可使loop开销对单次执行结果的影响降低1/10；

总共用时5172000ns，单次平均0.623ns；CPU主频1999.830MHz，折合一个cycle 0.500ns，数据基本可信。
```

提升性能主要是把CPU喂饱，所有的性能都是从CPU的角度来衡量；内存读写快慢，单纯比较数据从内存的一个位置移动到另一个位置，这是设备厂商用来做广告用的，不是计算机系统来评估性能的；把数据从内存读到CPU，然后写到另一个地址，数据流经过CPU即是经过了计算机系统，测量这段时间才是有意义的。以数据流动为基础，其他的bench，比如pipe的性能，需要排除数据流动的时间，排除loop等时间，才是单纯pipe的带宽性能；比如context switch速度，在做bench时，需要用pipe来驱动切换进程，这里需要排除掉数据流动的时间，pipe通信的时间，其余开销时间，才是context switch的时间。

性能评估主要对CPU、memory、disk IO和network IO四个指标，从带宽和时延两个角度评估。

所谓带宽，不仅仅是硬件上的读写速度，而是数据从源头到达CPU，然后CPU将其送往目的地的速度，考验的是传输能力；所谓时延，更多的评估传输的效率，读取一定量的数据，数据可以在多长时间内从内存读取到CPU。

所谓时延，其实也是另一种意义的速度，比如context switch，并没有吞吐量这个概念，但是通过将多个进程切换N次，得到总体时间，平均后可以获得单次切换用时，用以评估context switch的latency。

测试前的准备
-------------------

很多很重要的选项，比如设计测试场景，从逻辑上说通一个测试中都包含哪些时间开销，如何测量

* 测量数据块大小
	* 测量从内存经过CPU拷贝到另一块内存时，如果数据量过小，比如32KB，可能这个数据只在最次L2 Cache中流动，那么测量结果将会比真实数据大；如果数据量过大，又有可能被从内存换到磁盘上
	* 为此的应对方法是，在循环中逐渐增大一倍数据量，列出不同数据量大小的测试数据，我们其实可以分辨出哪些是L1 Cache，哪些是Cache已经失效，因为他们之间的速度差异是巨大的；另外，当我们每次跨过一页访问该页内存，如果访问时间需要好几个us，那么说明这个内存页不在内存中

* 测量时间
	* 不论用多么精确的时钟，由于benchmark时单次任务执行时间都非常短，因此用多次loop中求整体运行时间，取平均后能够得到误差较小的结果。
	* lmbench的时间机制写的很精妙，比如针对不同的任务，在每次bench的时候，都会预估执行当前任务执行比如500000us，需要执行多少次，这就是需要loop的次数；比较精妙，能够照顾到即使是相同计算机，在负载不一样的时候，对不同任务有一定的适应能力，在运行足够时间后使得系统表现稳定，得到稳定的结果；使用lmbench对相同任务做bench的时候，每次执行结果的误差都不大，低于2%，这个结果我想还是很稳定的。

* 考虑多进程，以及编译器可能导致的问题
	* 测试时运行的benchmark也都比较小，其实可以视为和单处理器没有区别；或者我们可以将进程绑定到某个CPU上。
	* 用gcc编译benchmark，优化级别为 -O，可以避免优化过度；注意一些load指令，如果load结果没有被用到的话，可能会被优化掉


带宽性能测试
----------------

* Memory: 数据从内存到CPU的带宽
	* rd
		* 单次读取512Byte数据，即128个int整数，并做相加操作以防止编译器优化；循环展开，而非在for中挨个相加
		* use_int(sum)等指令也是防止编译器优化的
		* 每次读取512Byte，直到读完，算是一次读取完成
		* 如果rd的数据块太小，比如32MB，很快被读完，这时需要调整连续循环读取iterations次，然后求平均；iterations的取值取决于根据系统负载情况，实时计算需要执行的次数。
		* 我的CPU是[至强E5-2620](http://www.cpu-world.com/CPUs/Xeon/Intel-Xeon%20E5-2620.html)，包含6*32KB的8路组相连数据L1缓存，6*256KB的8路组相连L2,15MB的20路共享L3cache
		* 根据读取的数据块大小，我们可以逻辑上推断该数据处于哪级缓存；本机的L1/L2/L3分别为192kb/1536kb/15360kb，
			* 数据块16KB，带宽34999.46MB/s
			* 数据块32KB，带宽19191.28MB/s
			* 数据块320KB，带宽10669.75MB/s
			* 数据块8MB，带宽6394.37MB/s
			* 数据块16MB，带宽4993.96MB/s
			* 数据块1024MB，带宽4979.76MB/s
				* 纯内存带宽
	* wr
		* 向内存块每一个4字节写入1
	* rdwr
		* wr与rd的结合，性能略差
			 

* pipe：系统提供的IPC机制的带宽
	* 测量方式：父子进程阻塞读写pipe

* mmap方式读取文件：从磁盘文件读取数据的带宽
	* mmap方式读取文件，首先要打开文件，然后通过mmap将fd映射到匿名内存页，mmap的内存页在读取时才会真正分配
	* 测量方式：
		* 一、多次循环中，open、mmap，然后读取内容，最后close
		* 二、在测量前open文件，并进行mmap；在多次循环中，每次读取目标大小的文件数据；
		* 前者可以得到通过读取文件数据时，mmap的纯开销；后者更贴近实际情况
	* 测量结果
		* 读取1024m数据；测试采用-C标志，复制文件后再进行，可以以冷数据的方式避开文件缓存
		* 结果一：3223.58 MB/s
		* 结果二：7948.87 MB/s

* read方式读取文件：从磁盘文件读取数据的带宽
	* 测量方式
		* 一、以及包含open、read和close的带宽
		* 二、测试单纯读文件(read)的带宽，
	* 测量结果
		* 读取1024m数据；测试采用-C标志，复制文件后再进行，可以以冷数据的方式避开文件缓存
		* 一、5526.29 MB/s
		* 二、5442.29 MB/s
	
* 注：带宽测试中使用的计算机为 至强E5-2620；之后机器收回，我采用我的一台闲置笔记本Thinkpad X220进行时延性能测试，CPU为 i5-2520M

时延性能测试
--------------------

计算机所有的时延几乎都跟memory时延有关，做context switch时首先要store当前进程状态，然后load下一个进程。可以说准确测量计算机内存时延是评估其他时延的前提，虽然内存时延的准确测量不太容易。

计算机系统中存在的时延主要有内存访问时延、调用操作系统组件如读写文件和系统调用等、进程创建的时延，以及进程切换导致的上下文切换时延。

### 内存延时

* 定义:
	* 刨去硬件相关的内存芯片和系统总线时延外，从总线和内存空闲时读数据，和连续读数据这两种场景的差异值得探讨。
	* 总线空闲时读取内存数据
		* 处理器等待从内存中取数据的时间
			* 这个时间通常是一种标称值，有些处理器取数据时并不等待和停顿，因此测量延时可能显得小于标称值
			* 而压测时，由于突发读取导致cache miss，导致实际延时又比这个值大
			* 因此采用这个值也不是那么合理
	* 连续繁忙读内存数据
		* 连续读内存时，每次load都会跟着一个load。
		* 连续读可能会导致时延高于空闲读，有些系统会有"关键字优先"的机制，读取某个字时不等待整个cache line填充就把该line中的要读取数据喂给CPU，然而此时cache依然处于busy状态；如果此时再有第二次load，会因为当前cache busy而停顿等待。在UltraSPARC中空闲读和连续读的差异可达35%.
	* 所以lmbench采用的是测试连续读时的内存时延，一来连续读的测量比空闲读容易些，二来连续读更贴近实际情况。由于处理器速度很快，即使发生cache miss，引起的load latency也和连续读更接近。

* 测量原理
	* 采用不同内存大小、不同读取跨度stride来评估、探测系统的各级内存：L1、L2、L3以及主存的时延
	* 读取内存，每跨读一步，如果跨度很小，比如64B/128B小于Cache line，那么时延会很小，如果跨度很大，比如1M，可能就需要从主存中将该地址内容读出，时延会相应增大
	* ~~每次测量前先通过分配一块内存将cache内容全部替换掉~~
	* 与bw_mem不同的是，并不读取数据块，而是通过地址链表跨stride去访问内存，以此测试内存时延，可见目的和测试bandwidth不一样
		* 地址链表的实现方式是，比如512KB的内存块buf，编址从0开始，以512 Byte为一跳
		* buf = 0x77889900, buf + 512 = 0x77889b00
		* *buf = 0x77889b00，即用内存块存放下一跳地址，这个时候 buf = 0x77889900, *(long *)buf = 0x77889b00, buf[0] = 0x00, buf[1] = 0x9b等等
		* 这个地址链表非常高效，因为我们不关心buf存放内容，所以就利用buf的空间来存储下一跳地址；类似于如下代码
	* 对一块内存进行跨行访问，依次采用不同大小内存和不同跨度，比如size=32KB，stride=512B，根据访问次数64次，和用时time，可以得出该次访问用时为time/64.
	* 对于L1 Cache 32KB，L2 Cache 256KB和L3 Cache 3072KB来说，通过内存大小可以限定访问主要集中在哪级Cache，控制stride降低各级Cache miss，可以推断出访问时间。

```cpp

	char *buf = (char *)malloc(sizeof(char ) * 1024);
	memset(buf, 0, 1024);
	*(char **)&(buf[0]) = (char *)&(buf[512]);
	printf("%p\t%p\n", buf, buf + 512);
	char **p = (char *)&buf[0];
	printf("%p\t%p\n", p, *p);

```


* 测量结果
	* 测量结果如下图所示，[原图见](https://raw.githubusercontent.com/BG2BKK/githubio/master/static/lat_mem_latency.png)，原始数据[见文件](https://github.com/BG2BKK/githubio/blob/master/static/data.set)

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/lat_mem_latency.png" ><p>多级内存的读写速度，从L1、L2、L3到主存</p></div>

	图例：
		横轴读取的内存块大小，从4KB到8MB，横轴以0.5MB为单位
		纵轴是load平均延迟，单位为ns，从1ns到50ns
		不同颜色的线表示不同的Stride，即每次读内存时跨越的数据长度
		系统L1 Cache 32KB、L2 Cache 256KB、L3 Cache 3072KB
		getconf命令可以获取系统的必要信息，包括各级Cache

* intel CPU的各级缓存latency的[官方数据](https://software.intel.com/sites/products/collateral/hpc/vtune/performance_analysis_guide.pdf)和[其他解释](https://software.intel.com/en-us/forums/intel-manycore-testing-lab/topic/287236)

```bash
Core i7 Xeon 5500 Series

Data Source Latency (approximate)

L1 CACHE hit, ~4 cycles

L2 CACHE hit, ~10 cycles

L3 CACHE hit, line unshared ~40 cycles

L3 CACHE hit, shared line in another core ~65 cycles

L3 CACHE hit, modified in another core ~75 cycles

remote L3 CACHE ~100-300 cycles

Local Dram ~60 ns

Remote Dram ~100 ns

```
	
#### 对于测量各级Cache的latency来说，需要对每级进行特定分析。

##### L1 Cache Latency

对于intel i5-2520M来说，L1 Cache的32KB容量，cache line长64B，8路组相连，每路4KB大小，有64组cache line供选择；在不考虑其他因素的情况下，每32KB连续数据中一定会产生L1 miss，每4KB连续数据一定会有一次组内选择哪路cache line存储数据，可能产生cache miss；一旦产生cache miss，会进行L2乃至下一级的读取，造成时延加大，影响L1 cache的latency测量。因此为了避免cache miss带来的影响，在测量L1时尽量采用小步长，小内存块进行逼近，得到尽可能精确的L1 Cache的latency。

##### L2 Cache Latency

对于L2 Cache来说，所用内存块和步长应该加大，尽量使L1 Cache失效，访问L2；也应该注意步长不能过大，造成L2 失效；所采用的内存块大小倒不是关键，因为即使再大的内存块也需要按照步长读取

##### L3 Cache Latency

L3 Cache的latency反而难以测量，原因一是L3可能是多核共享的，容易受干扰，二是相比L1/L2和DRAM的性能差异，L3与DRAM的访问差异显得不那么大。此外，TLB的因素也应该考虑在内。

拜读完[CacheMemory.pdf](yuhaozhu.com/CacheMemory.pdf)的第一章后，发现自己考虑的太少了，所以以上的测量方法其实只能说明一个大概，并不能极具说服力的得出时延数据就是L1 Latency。

#### TODO
---------------
* measure cache line
* measure tlb
    * [tlb和cache区别](http://blog.chinaunix.net/uid-26009500-id-3089718.html)
* [lwn](https://lwn.net/Articles/252125/)
* [IBM关于lmbench对mem latency的深度benchmark](https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/W51a7ffcf4dfd_4b40_9d82_446ebc23c550/page/Untangling%20memory%20access%20measurements%20-%20memory%20latency)
* [stackoverflow_1](http://stackoverflow.com/questions/4087280/approximate-cost-to-access-various-caches-and-main-memory) [stackoverflow_2](http://stackoverflow.com/questions/10274355/cycles-cost-for-l1-cache-hit-vs-register-on-x86)
* 读内存过程
	* [CPU读数据的一系列过程](http://yuhaozhu.com/CacheMemory.pdf)
	* [What Your Computer Dos While You Wait](http://duartes.org/gustavo/blog/post/what-your-computer-does-while-you-wait/)
	* [译文](http://www.cnblogs.com/xkfz007/archive/2012/10/08/2715163.html)
	* [intel: Cache相关的问题](https://software.intel.com/sites/default/files/m/1/1/7/0/a/12645-6.7__Cache_e7_9b_b8_e5_85_b3_e9_97_ae_e9_a2_98.pdf)


### 调用系统组件（系统调用） 

调用操作系统的入口，一般指系统调用，比如读写设备的read/write，比如getpid()或getimeofday()。

对于前者的bench，选择操作/dev/null设备，每次写一个字，一般为4字节，经历了用户进程发起系统调用、转入内核态、查询文件描述符、VFS层等一整个过程，测量这个过程的时间；选择该设备的原因是所有操作系统都没有对该设备进行优化

而后者，系统调用比如getpid、gettimeofday，各平台不同优化，甚至是实现在用户层，不过在Linux上仍然是实现在内核态的。所以可以通过getpid()了解基本开销，通过写/dev/null设备了解整体开销。

* 原理：
	* 在循环中调用getpid()和write(fd, *buf, 1)
* 结果：
	* getpid	
		* Simple syscall: 0.0540 microseconds
	* write to /dev/null
		* Simple write: 0.0897 microseconds

### 信号处理耗时

* 建立信号sigaction耗时
	* 测量原理：
		* 在本进程内部，调用sigaction建立信号
		* time = sig_installation
	* 测量结果：
		* Signal handler installation: 0.1492 microseconds

* 发送信号耗时 kill(pid, sig)
	* 测量原理：
		* 设置不捕获信号，进程内向自己发送信号，kill(pid_self, SIGUSR1)
		* time = sig_send 
	* 测量结果：
		* 0.1240 microseconds

* 捕获并处理信号耗时		
	* 测量原理：
		* 设置捕获信号，向自己发送信号，kill(pid_self, SIGUSR1)
		* sig_handle = total_time - sig_installation - sig_send
	* 测量结果：
		* Signal handler overhead: 0.9763 microseconds

* 捕获信号耗时：
	* 测量原理：
		* 在bench进程中以只读方式mmap一段内存，如果试图写这块内存，则会一直触发SIGBUS和SIGSEGV信号
		* 在触发信号前设置SIGBUS和SIGSEGV的处理函数，在处理函数中不执行任务，只动态调整捕获次数；捕获次数达到一定数量时，评估单次处理用时
		* 由于重复捕获信号，并且信号处理函数里基本没有任务，可以认为这段时间是捕获信号耗时，或者说是信号传递耗时
	* 测量结果:
		* Protection fault: 0.4928 microseconds

* 补充：
	* 由于第三项与第四项采用不相同的测试场景测量，前三项是一个场景，进程设置信号捕获，并向自身发送信号，然后捕获处理，第四项是产生page fault后重复触发信号，bench程序统计捕获次数和耗时计算得来。

### 创建进程开销

进程相关的benchmark主要是衡量几个进程原语：创建新进程、执行新的程序以及上下文切换。

* 创建进程的开销：fork process
	* 测量原理：
		* 通过fork创建子进程，子进程直接执行exit退出；所以最后测量的时间是：fork() + exit()
			* 其实这里的开销还包括了父进程调用wait系统调用和父子进程上下文切换的开销，前者在本机50ns、后者在本机1.93us，相对进程创建来说非常小
	* 测量结果：
		* Process fork+exit: 107.1001 microseconds
* 创建进程并执行新程序的开销：fork + execlp
	* 测量原理：
		* 不光测量fork的时间开销，还要通过execlp系统调用加载新程序/tmp/hello，打印一条hello world信息
	* 测量结果：
		* Process fork+execve: 372.2349 microseconds

### 上下文切换的开销

* 上下文切换会受到多个因素影响：切换进程数量、每个进程自身的内存大小，以及Cache在其中的影响

* 测量结果：



intel的超线程技术
--------------------

[CPU的超线程机制](https://www.ibm.com/developerworks/cn/linux/l-htl/)通过复制、分区和共享 Intel NetBurst 微结构管道中的资源，使得一个物理处理器能包含两个逻辑处理器。逻辑处理器有自己的处理器状态、指令指针、重命名逻辑以及一些较小的资源，共享的资源有乱序执行引擎和高速缓存。超线程机制HT利用各资源的速度差异，在时间上并行模拟出两份计算资源，理论上讲提供多一倍的计算能力，而代价是，既然涉及到共享，那么在一些操作中会因为共享和竞争而有性能损耗。下表是超线程对Linux API的影响，采用lmbench测试结果，这份bench报表在运行有linux-2.4.19内核的Intel Xeon处理器上，主频1.60GHz。

<h5 id="N100C0"> 超线程对 Linux API 的影响</h5>
<table border="1" cellpadding="5" cellspacing="1" class="ibm-data-table" summary=""><thead xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"><tr><th><strong>内核函数</strong></th><th><strong>2419s-noht</strong></th><th><strong>2419s-ht</strong></th><th><strong>加速</strong></th></tr></thead><tbody xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"><tr><td>简单的 syscall</td><td>1.10</td><td>1.10</td><td>0%</td></tr><tr><td>简单的 read</td><td>1.49</td><td>1.49</td><td>0%</td></tr><tr><td>简单的 write</td><td>1.40</td><td>1.40</td><td>0%</td></tr><tr><td>简单的 stat</td><td>5.12</td><td>5.14</td><td>0%</td></tr><tr><td>简单的 fstat</td><td>1.50</td><td>1.50</td><td>0%</td></tr><tr><td>简单的 open/close</td><td>7.38</td><td>7.38</td><td>0%</td></tr><tr><td>对 10 个 fd 的选择</td><td>5.41</td><td>5.41</td><td>0%</td></tr><tr><td>对 10 个 tcp fd 的选择</td><td>5.69</td><td>5.70</td><td>0%</td></tr><tr><td>信号处理程序安装</td><td>1.56</td><td>1.55</td><td>0%</td></tr><tr><td>信号处理程序开销</td><td>4.29</td><td>4.27</td><td>0%</td></tr><tr><td>管道延迟</td><td>11.16</td><td>11.31</td><td>-1%</td></tr><tr><td>进程 fork+exit</td><td>190.75</td><td>198.84</td><td>-4%</td></tr><tr><td>进程 fork+execve</td><td>581.55</td><td>617.11</td><td>-6%</td></tr><tr><td>进程 fork+/bin/sh -c</td><td>3051.28</td><td>3118.08</td><td>-2%</td></tr><tr><td colspan="4">注：数据用微秒表示：越小越好。</td></tr></tbody></table>

我认为上表中，ht没有影响的API是一些lmbench单进程可以测试的项，比如read、write等，而ht有影响的选项，比如管道延迟、进程fork+exit等，是lmbench需要发起多个进程进行bench，这些进程在逻辑处理器间有竞争，导致性能有些下降。从报表中可以看到，开启ht的bench结果在有些项目中耗时加长，但是比较小，但是整个系统获得了一倍的计算资源。

intel的turbo技术
---------------------

[linux的睿频工具](https://magiclen.org/linux-intel-cpu/)可以实时看到CPU的运行频率

```bash
sudo i7z
```

以笔记本CPU i5-2520M 为例，在CPU空闲时主频低至
 
```bash
        Core [core-id]  :Actual Freq (Mult.)      C0%   Halt(C1)%  C3 %   C6 %  Temp      VCore
        Core 1 [0]:       1366.96 (13.72x)      24.9    81.6    4.69       0    62      1.1008
        Core 2 [2]:       1381.01 (13.86x)      23.3    79.2    7.89       0    62      1.1008
```

甚至更低

运行lmbench时，CPU满载，主频可达

```bash
        Core [core-id]  :Actual Freq (Mult.)      C0%   Halt(C1)%  C3 %   C6 %  Temp      VCore
        Core 1 [0]:       2989.17 (30.00x)      16.9    73.4    6.37       0    69      1.1409
        Core 2 [2]:       3050.32 (30.61x)      99.1       0       0       0    73      1.1409
```

可见睿频的主要作用是动态调整CPU主频适应处理任务，当我们运行lmbench时，理所应当的CPU将会满载运行，所以计算CPU主频时完全不用担心睿频导致的频率变化，总是接近最高值的。

lmbench测试框架
----------------------

* fork出child进程，作为bench执行体；parent作为控制体；
* parent和child通过pipe互相通信，child通知parent准备好，parent通知child开始，child执行完毕后通知parent完成；parent取完执行结果，通知child结束。
* 应用程序中child首先执行init程序，将准备工作做好；比如如果是bw_mem中的cp benchmark，需要首先分配好src内存块，然后分配dst内存块，准备好后即返回，init结束；
* init结束后，benchmp框架进行下一步，while循环中执行benchmark进入warmup状态
* child执行benchmark是以状态机方式实现的
	* 首先是warmup状态，在该状态中childparent发送ready信号，阻塞等待parent的start信号。child收到start后，状态设置为timing_interval，取iterations为1，执行benchmark程序；执行完后重入while循环；
	* 然后切换到timing_interval状态。由于之前在warmup状态执行以iterations为1的benchmark，重入循环时满足state->need_warmup == 0，所以开始统计上一次benchmark开始到现在的用时，并刨去t_overload和l_overload；进入状态机的timing_interval状态，该状态评估本次benchmark的用时是否符合期望，如果符合，将本次bench结果记录，并设置状态为cooldown；如果不符合期望，分情况，如果用时result小于150，说明单次benchmark用时太短，需要多次迭代得到总结果，因此将iterations扩大8倍去计算；如果用时大于150，说明单次benchmark执行时间是够的，但是需要微调迭代次数，按照 enough * 1.1 / (iterations / result)；所以说timing_interval状态是用来在bench过程中微调迭代次数以使执行时间符合预期的过程。
	* 在cooldown状态，child阻塞等待取结果信号，并将结果写回response管道中；执行清理函数后，得到parent发出的exit信号，结束自己。
* 最终结果存储在全局变量iterations和stop_tv中，其实是通过get_time()和get_n()，以及set_time()/save_n()或者set_results()中，并在用户程序的结束时进行计算。


