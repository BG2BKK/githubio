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
