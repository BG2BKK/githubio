+++
date = '2016-10-13T13:57:31+08:00'
draft = false
title = 'futex和linux的线程同步机制'

+++


* [futex初体验](http://blog.csdn.net/nellson/article/details/5400360#)
* [阿里基础架构事业群的博客关于futex的文章](http://blog.sina.com.cn/s/blog_e59371cc0102v29b.html)
* [linux线程同步机制](http://www.dbafree.net/?p=1128)
	* Linux中的线程同步机制(二)–In Glibc
		* 大部分的glibc的同步方式，mutex或者semaphore，大多基于futex的方式，首先进行用户态检查，未果的话进行futex系统调用。这是我疑惑为什么futex这么常用却在代码层面上看不到它，原因是我们使用的都是基于futex的机制
	* Linux中的线程同步机制(三)–Practice
		* pthread库中的pthread_join也是基于futex的哦，当父进程执行pthread_join它的某一个子线程时，如果子线程已经执行完毕，则父进程不会调用futex系统调用，如果子线程仍然执行中，那么父进程调用futex系统调用进行FUTEX_WAIT休眠，等待子线程的唤醒
	* 好文章，值得深挖和思考
* man page
	* 先通过__sync_bool_compare_and_swap等原子操作比对futex的值是否有变化，如果没有，说明没有进程竞争。这里都是用户态执行的
	* 如果有变化，说明有进程竞争了，所以这时系统调用futex进行FUTEX_WAIT，使得本进程休眠或休眠一段时间，直到有别的进程FUTEX_WAKE它
	* 说白了，futex针对有些同步场景中，尽管没有竞争发生，但是还要陷入内核态去获得锁或者标志位然后同步的情况，futex可以仅通过原子性的内核态即可实现线程安全
	* 提供futex_demo.c
		* 如果将futex1和futex2互换下，画面太美不敢看，CPU暴涨100%

```bash
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
futex(0x7f99372d3004, FUTEX_WAIT, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
```


