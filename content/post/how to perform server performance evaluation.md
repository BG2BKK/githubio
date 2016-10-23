+++
date = "2016-10-23T16:43:44+08:00"
draft = true
title = "怎样尽可能全面的评估一台服务器的性能"

+++

* cpu使用率
	* 进程调度
	* softirq
	* CPU load balance
	* context switch/ interrupt stats
		* 频繁切换很要命

* 内存使用情况
	* 内存使用情况分布
		* page entry/sk_buff
	* 内存换页率
	* 脏页情况
		* /proc/meminfo等
	* 小内存块分布情况
	* 虚拟内存使用情况
		* vmstat
	* 通过内核代码对各种分配函数进行统计，分配大小、位置和目的

* 进程调度
	* 进程数

* 网络带宽
	* 带宽瓶颈
	* 小包率
	* 网卡中断率
	* 重传率
	* 输入输出
	* DNS时间
	* TCP情况统计
	* socket内存使用率
	* socket缓冲队列
	* 从sysctl配置开始，然后统计

* 磁盘
	* 输入输出
		* iostat
	* 磁盘换页
	* 读盘写盘
	* 读文件
	* 数据库
	* io调度队列的大小

* 文件
	* 打开文件数

* 设备包括网卡、磁盘

分析某个进程或者任务的热点

USE

resource
usage
saturation
error
