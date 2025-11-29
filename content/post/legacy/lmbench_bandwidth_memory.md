+++
date = '2016-10-31T11:07:28+08:00'
draft = true
title = 'lmbench_bandwidth_memory'

+++

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
