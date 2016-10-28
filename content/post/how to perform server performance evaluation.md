+++
date = "2016-10-23T16:43:44+08:00"
draft = false
title = "怎样尽可能全面的评估一台服务器的性能"

+++

给你一台服务器，怎样能够全面评估它的性能？需要测试哪些指标？请写出每个指标的具体测试原理和测试代码。假设这台服务器完全处于线下。

这个问题看似平常，但是细细审题的话，发现还是不一样的。我们过多关注线上机器的性能，但是如果单独拿出来一台服务器，它的性能怎样呢？这个问题网上还真没有答案，只能根据自己的理解查资料来解决。

评估和压榨一台服务器性能的话，找到评估的指标，然后进行压测加观察的方式，得到性能参数。

1. 观察哪些指标，意义何在？
2. 如何观察这些指标，测试原理，观测工具/统计代码
3. 如何进行压测，实现测试场景，压测工具/压测代码

评估服务器性能，需要：

1. 主要观察CPU、内存、磁盘和网络IO这四个指标

2. 首先看硬件配置，CPU核数/主频/超线程，内存带宽/大小/访问速度，磁盘类型/转速，网卡千兆/万兆/多队列等
	* 知晓极限值，针对硬件选取场景。

3. 然后知道kernel关于这些设备的可tuning的配置选项，以及kernel自身配置，如vm策略、进程调度设置等
	* 默认服务器设备的驱动都是厂商调优好的，或者采用kernel自带驱动

4. 不同指标的通用观测方法
	* 首选成熟工具：top、sar、vmstat、mpstat、netstat、iptraf等
	* 从 ***/proc*** 定制
	* 动态追踪，perf、systemtap等，在精细观察时使用

5. 根据软硬件变量的结合，搭配设计纯理论场景并实现，采用对应方法检测
	* 先把linux server主流常用的配置配好，采用简单方法做性能测试，确定各个指标的性能，作为参照基准
	* 然后设计场景，体现要突出的指标，确定极限值，与参照基准做对比
		* kernel对相关资源进行优化
		* 选择压测工具，必要时写代码实现
	* 如果能够引入内核的一些新技术，就某个指标进行优化，可以进行后续测试。

### CPU计算能力

* 硬件
	* 主频/核数/超线程：/proc/cpuinfo
	* 各级缓存大小

* 测试场景
	* 评估计算能力
		* 方式一：选择高CPU型应用压测，采用专业benchmark软件观察;[计算圆周率](http://wushank.blog.51cto.com/3489095/1585927)
		* 方式二：采用开源代码，或者手写计算型压测代码，通过systemtap等工具统计时长
			* [参考sysbench](https://github.com/akopytov/sysbench)

	* 评估各级CPU高速缓存L1/L2/L3失效对性能的影响
		* 场景：
			* 通过代码创造cache miss情况
		* 方法：
			* 采用systemtap等相关工具统计时长
			* 获取CPU各级缓存速度，需要多少cycle读取
				* 查文档或者其他方式，如[lmbench](http://www.bitmover.com/lmbench/)

	* 评估进程调度和切换能力
		* 场景：并发大量CPU繁忙任务
		* 方法：
			* 查看CPU负载
			* 统计context switch/ interrupt stats，通过sar等工具
			* 进程切换平均时间统计，在不同负载下：[谁在做进程调度&&lmbench](http://blog.yufeng.info/archives/753)
			* 进程调度能力
				* load average，通过统计工具
				* 动态跟踪方法动态确定处于调度队列中的任务规模
				* taskset设置多cpu亲和性运行

* TODO
	* 评估CPU对软硬中断的响应处理能力
		* 一段时间内的中断分布情况，CPU响应时间，需要动态追踪
			* 多长时间响应中断
			* 多长时间处理完中断
			* 如何分配，如何均衡处理

### 内存资源使用
* 硬件
	* 总线宽度/内存读写速度/内存颗粒主频
		* [dmidecode获取主板信息](http://blog.opskumu.com/dmidecode.html)
		* [内存带宽测试](http://www.latelee.org/using-gnu-linux/linux-memory-bandwidth-test-note.html)

* 内存资源使用情况

	* 评估系统的内存分配能力

		* 系统自身需要的内存
			* page entry
			* slab/vma小内存块
			* 值得具体了解
			* 通过内核代码对各种分配函数进行统计，分配大小、位置和目的

		* 最多能分配多少内存给某个资源
			* 分配socket memory
				* 据说系统在已有大规模socket的情况下，对其分配内存的策略是惰性的，待查
			* 分配page cache/page buffer
				* [cache的使用，cache的重复使用](http://liwei.life/2016/01/22/cgroup_memory/)
				* swap内存

		* 内存换页率和脏页情况
			* /proc/meminfo等
			* swap in和swap out
			* [sar -B 1中的主缺页中断和次缺页中断](http://chuansong.me/n/285622451424)

		* [采用huge page的性能影响](https://www.ibm.com/developerworks/cn/linux/l-cn-hugetlb/)

### 磁盘读写能力
* 硬件
	* 磁盘类型/转速
	* 查看kernel能够tuning的选项
		* 磁盘块IO大小
		* [调度策略](http://scoke.blog.51cto.com/769125/490546)
	
* 评估磁盘IO性能
	
	* 评估不同读写文件方式的性能
		* 直接读写文件性能
		* 经过文件系统读写速度

	* 评估读小文件时的性能
		* 读大文件速度
			* 较大文件的读取效率，考验IO能力
		* 读小文件速度
			* 大量小文件
	
	* 评估文件缓存对磁盘性能的影响
		* 首次读
			* 可测试文件从磁盘读到kernel内存直到用户进程这一过程
				* 微观角度，systemtap等脚本动态追踪
		* 非首次读
			* 测试文件在缓存中后的读效率
				* 文件系统缓存命中率

	* 评估不同调度策略对磁盘性能的影响
		* 场景：
			* 不同策略适应不同场景
		* 方法：
			* 写随机文件内容，记录参数
			* bio调度队列的大小
			* 输入输出能力
				* iostat
				* iotop

### 网卡吞吐能力

* 硬件
	* 千兆/万兆
	* 多队列
	* 网卡缓冲区大小
* 查看kernel能够tuning的选项
	* [ethtool、网卡硬件信息和优化技术](http://www.blogjava.net/yongboy/archive/2015/01/30/422592.html)
	* 网卡驱动的缓冲队列

* 评估linux server的网络性能

	* 评估网络子系统的吞吐量
		* 场景：
			* 测试网卡吞吐能力
		* 方法：
			* 少量TCP连接，发起大规模数据传输
			* 网络压测工具netperf/iperf，或者写代码调整
			* 测试过程中调整sysctl配置、网卡驱动、网卡硬件配置
			* 检测工具：tcpdump/sar等工具
			* 统计socket缓冲大小
			* 在局域网内两台设备间执行

	* 评估网络子系统的响应能力
		* 场景：
			* 在网络子系统繁忙时，对外服务的响应能力
		* 方法:
			* 采用iperf等工具发起大量tcp连接，发出巨量小包
			* 检测工具，tcpdump/wireshark统计服务质量
			* systemtap分析软中断响应时间
			* 统计TCP状态分布情况
			* 统计内存分配情况

### 内核本身

* 管道
* IPC性能
* unix domain socket性能

### 参考链接

* [context switch definition](http://www.linfo.org/context_switch.html)
* [查询本机CPU相关信息](http://smilejay.com/2011/03/linux_cpu_core_thread/)
* [海量小文件](http://blog.csdn.net/liuaigui/article/details/9981135)
* [单机负载评估](http://www.jianshu.com/p/db8e8a2884ef)、[性能分析](http://www.jianshu.com/p/fd6e35f529c1)
* [性能评估](http://blog.csdn.net/hguisu/article/details/39373311)
* [Linux性能优化--CPU](http://kumu-linux.github.io/blog/2014/04/21/performance-cpu/)
	* 参考该post前我也想到了CPU压测的几个关注点：待调度执行任务数、CPU负载和进程切换
	* 该post有很大启发
* [SAR的一些使用](http://www.thegeekstuff.com/2011/03/sar-examples/?utm_source=feedburner)
* [Linux磁盘调度策略](http://blog.itpub.net/27425054/viewspace-768224/)
* [其实没想到卖VPS的写性能测试挺专业](http://www.vpsee.com/2009/11/linux-system-performance-monitoring-cpu/)
* [coolshell的性能调优攻略](http://coolshell.cn/articles/7490.html)
* [linux下网络环境性能测试](http://www.samirchen.com/linux-network-performance-test/)



bw_mem
----------------


* init_loop

* get_enough
	* init_timing
		* compute_enough()
			* 计算可以得到计算机准确执行时间的时间段，在该时间段内计算机的执行时间能够稳定
		* t_overload()
			* 用循环计算 gettimeofday() 的开销
		* l_overload()
			* 计算仅用于循环的时间开销
		* init_timing()在一次benchmakr中只执行一次，除非第二次运行程序

	* compute_enough()
		* 预估5000us、10000us、50000us和100000us，比如在5000us内计算机用时能够稳定，那么认为运行想运行的任务，5000us足够使计算机测量稳定下来，那么就返回5000us；如果四个都不行，则返回SHORT，也就是1000000us，一个足够大的数据。理论上讲其实计算机执行一个非常快或者用时普通的任务，在循环足够次数，运行足够时间，平均下来都能使单次时间稳定，但是如果能找到合适的时长，benchmark过程可以更快，且执行时间短意味着受到外界干扰小。
		* compute_enough() 调用 test_time() 依次评估每个预估值
		* test_time(enough)
			* find_N(enough)，得到执行任务A，执行时长为enough us，需要跑多少次，得到N。
				* find_N 首先执行 N = 10000, time_N(N)，看看执行10000次A的运行时间t，如果运行时间t与enough误差在2%以下，那么就返回N；如果运行时间t过小，低于1000us，说明执行N次还不够，需要加码，扩大10倍，N = N * 10；如果t不低于1000us，说明执行N次还可以，不过要按照运行时间t和enough的比值，再结合运行N次需要t us，折算出运行enough时间大概需要  N = enough * N / t;
				* 拿着新的N再次find_N(N)，直到执行时间与enough相差不到2%时，可以认为，计算机执行N次任务A，需要用时enough us。
				* 这个逼近过程循环10次，返回结果；如果不能收敛在2%里，则认为enough时长还不足以让CPU运行稳定，需要加长。
					* time_N(N)通过执行执行duration(N)，执行Nx10次 long **p; p = (long **) *p;这样一个赋值运算，汇编是[ move (%eax), %eax ]，从内存中读值，写入寄存器；读p地址的值只是读L1 cache中，所以在CPU内运行，不会有外界干扰。在不同平台不同性能的机器上，大家都执行这个duration(N)，通过执行N次达到时间u，作为cpu的基准。这样还是蛮科学的
					* 执行TRIES次 time_N(N)，取最小值返回
					* duration(N)就是任务A。
			* test_time(enough)中，拿到find_N(enough)所需要的执行次数N后，再次计算执行N次用时，time_N(N)，以这次结果为baseline；然后将N扩大到1.015倍、1.020倍和1.035倍，计算执行所需要时间time_N(N * factor)，得到结果与baseline * factor比较，误差在0.25%内的话，说明执行enough时间，需要N次，这个经过更新的N存储在全局变量中(save_n())，也就是存储在全局变量iterations里。
			* 如果误差仍然较大，那么我们选用一个较大的时间，比如SHORT=1000000；此时的N并无意义。

	* t_overload()
		* 如果在之前的compute_enough()没有计算得到enough时间，返回的是SHORT，或者即使得到了，但是大于50000，执行时间足够长，那么我们认为gettimeofday的时间开销可以忽略不计，记为0。
		* 如果得到一个时间enough小雨50000us，那么认为gettimeofday的开销有必要列入，需要在循环中确定它的调用所需时长
	* l_overload()
		* 在需要循环很多次的benchmark中，loop的开销很有必要列入统计中，再bench总时长中排除loop的影响
		* loop的计算比较巧妙，假设loop开销是overhead，在同样的loop下，loop循环体执行一个任务和两个任务，分别进行bench，循环n1次和n2次，得到执行时长为u1、u2
			* u1 = n1 * ( overhead + work)
			* u2 = n2 * ( overhead + 2 * work)
			* overhead = 2*u1/n1 - u2/n2
			* overhead即为单次loop的开销，一般很小，但是累计起来比较客观。
* benchmp框架
	* fork出child进程，作为bench执行体；parent作为控制体；
	* parent和child通过pipe互相通信，child通知parent准备好，parent通知child开始，child执行完毕后通知parent完成；parent取完执行结果，通知child结束。
	* 应用程序中child首先执行init程序，将准备工作做好；比如如果是bw_mem中的cp benchmark，需要首先分配好src内存块，然后分配dst内存块，准备好后即返回，init结束；
	* init结束后，benchmp框架进行下一步，while循环中执行benchmark进入warmup状态
	* child执行benchmark是以状态机方式实现的
		* 首先是warmup状态，在该状态中childparent发送ready信号，阻塞等待parent的start信号。child收到start后，状态设置为timing_interval，取iterations为1，执行benchmark程序；执行完后重入while循环；
		* 由于已经执行以此iterations为1的benchmark，重入循环时满足state->need_warmup == 0，所以开始统计上一次benchmark开始到现在的用时，并刨去t_overload和l_overload；进入状态机的timing_interval状态，该状态评估本次benchmark的用时是否符合期望，如果符合，将本次bench结果记录，并设置状态为cooldown；如果不符合期望，分情况，如果用时result小于150，说明单次benchmark用时太短，需要多次迭代得到总结果，因此将iterations扩大8倍去计算；如果用时大于150，说明单次benchmark执行时间是够的，但是需要微调迭代次数，按照 enough * 1.1 / (iterations / result)；所以说timing_interval状态是用来在bench过程中微调迭代次数以使执行时间符合预期的过程。
		* 在cooldown状态，child阻塞等待取结果信号，并将结果写回response管道中；执行清理函数后，得到parent发出的exit信号，结束自己。
	* 最终结果存储在全局变量iterations和stop_tv中，其实是通过get_time()和get_n()，以及set_time()/save_n()或者set_results()中，并在用户程序的结束时进行计算。

	* 如果进行多任务
		* 1
		* 2
		* 3
