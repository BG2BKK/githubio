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

bench也分macro bench和micro bench

对于macro bench，很多时候我们得到的是一个整体而粗略的结果，通过top我们可以看到系统负载，这些对于我们定位线上问题，分析应用程序的性能热点很有帮助，然而这并不能精确衡量一台Linux Server的性能；应用程序多种多样，线上系统目的各不相同，所以macro bench一般用于case by case的性能分析，[用于解决应用的性能瓶颈]()；而micro bench可以定量的分析一台机器+操作系统的性能，采用相同的测试基准，通过无干扰的大量重复基本操作，比如从L1 Cache读取单个字节，耗时一个CPU时钟，在大量重复中得到较为宏观的结果，再排除loop损耗、取运行时间的损耗，可以用于基本衡量一台server的运行效率。

./bin/x86_64-linux-gnu/enough结果为n=322229, u=5172，执行322229次 TEN( p = *p )，TEN(T)表示循环展开执行10次任务T，可使loop开销对单次执行结果的影响降低1/10；总共用时5172000ns，单次平均0.623ns；而CPU主频1999.830MHz，折合一个cycle 0.500ns，数据基本可信。

目的是把CPU喂饱，所有的性能都是从CPU的角度来衡量；内存读写快慢，单纯比较数据从内存的一个位置移动到另一个位置，这是设备厂商用来做广告用的，不是计算机系统来评估性能的，不把数据从内存读到CPU，衡量这段时间的性能；把数据从CPU读一遍，然后写到另一个地址，数据流经过CPU后算是经过了计算机系统，测量这段时间才是有意义的。基于此，其他的bench，比如pipe的性能，需要排除数据流动的时间，排除其余的时间，才是单纯pipe的性能，；如context switch，需要bench时用pipe来驱动，这里需要排除掉数据流动的时间，pipe通信的时间，其他时间，才是context switch的时间

性能评估主要对CPU、memory、disk IO和network IO四个指标，从带宽和时延两个角度评估。

所谓带宽，不仅仅是硬件上的读写速度，而是数据从源头到达CPU，然后CPU将其送往目的地的速度，考验的是传输能力；所谓时延，更多的评估传输的效率，读取一定量的数据，数据可以在多长时间内从内存读取到CPU。


带宽性能测试
----------------

> 带宽包括了数据从内存到CPU的带宽；从磁盘文件读取数据的带宽；从系统提供的IPC机制，比如pipe的带宽；从网络读取数据的带宽；

* mem 
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
			 

* pipe
	* 测量方式：父子进程阻塞读写pipe
	* tips
		* 用bw_pipe，默认是64kb的块，发现采用32kb更快一些，16kb和32kb没区别；
		* 通过sar -w 1发现，64kb的时候的csw是32kb时的10倍以上
		* 最后发现，块大小是32kb - 1 和32kb +1是分水岭，
		* 正在查原因

* mmap方式读取文件
	* mmap方式读取文件，首先要打开文件，然后通过mmap将fd映射到匿名内存页，mmap的内存页在读取时才会真正分配
	* 测量方式：
		* 一、多次循环中，open、mmap，然后读取内容，最后close
		* 二、在测量前open文件，并进行mmap；在多次循环中，每次读取目标大小的文件数据；
		* 前者可以得到通过读取文件数据时，mmap的纯开销；后者更贴近实际情况
	* 测量结果
		* 读取1024m数据；测试采用-C标志，复制文件后再进行，可以以冷数据的方式避开文件缓存
		* 结果一：3223.58 MB/s
		* 结果二：7948.87 MB/s

* read方式读取文件
	* 测量方式
		* 一、以及包含open、read和close的带宽
		* 二、测试单纯读文件(read)的带宽，
	* 测量结果
		* 读取1024m数据；测试采用-C标志，复制文件后再进行，可以以冷数据的方式避开文件缓存
		* 一、5526.29 MB/s
		* 二、5442.29 MB/s

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
