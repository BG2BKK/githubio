+++
date = "2016-10-13T13:57:31+08:00"
draft = true
title = "futex和linux的线程同步机制"

+++


* [futex初体验](http://blog.csdn.net/nellson/article/details/5400360#)
* [阿里云](http://blog.sina.com.cn/s/blog_e59371cc0102v29b.html)
* [linux线程同步机制](http://www.dbafree.net/?p=1128)
* [man page]()
	* 先通过__sync_bool_compare_and_swap等原子操作比对futex的值是否有变化，如果没有，说明没有进程竞争。这里都是用户态执行的
	* 如果有变化，说明有进程竞争了，所以这时系统调用futex进行FUTEX_WAIT，使得本进程休眠或休眠一段时间，直到有别的进程FUTEX_WAKE它
	* 说白了，futex针对有些同步场景中，尽管没有竞争发生，但是还要陷入内核态去获得锁或者标志位然后同步的情况，futex可以仅通过原子性的内核态即可实现线程安全
