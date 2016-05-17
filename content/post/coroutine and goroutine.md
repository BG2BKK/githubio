+++
date = "2016-03-13T17:59:29+08:00"
draft = false
title = "coroutine and goroutine"

+++


#### 协程是什么？ what is coroutine ?

* [协程的概念](https://en.wikipedia.org/wiki/Coroutine)

> Coroutines are computer program components that generalize subroutines for nonpreemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations. Coroutines are well-suited for implementing more familiar program components such as cooperative tasks, exceptions, event loop, iterators, infinite lists and pipes.

> According to Donald Knuth, the term coroutine was coined by Melvin Conway in 1958, after he applied it to construction of an assembly program.[1] The first published explanation of the coroutine appeared later, in 1963.[2]

协程是为实现非抢占式多任务而提出的计算机子程序，通过提供多个程序入口使得程序可以在特定地址挂起和恢复执行。协程天生的支持实现常见程序组件，比如协作式任务、异常、时间循环、迭代器、无边界列表和管道等。

根据祖师爷高纳德节说，协程的概念由Melvin Conway在1958年提出，随后他将协程应用在编写汇编程序上。协程的第一个公开发表的解释出现在1963年。

可见协程的概念比多线程还早，而且按照Knuth的说法，[”子例程是协程的特例“，一次子例程调用就是一次子函数调用](http://coolshell.cn/articles/10975.html)，协程是类函数一样的组件，我们可以在单线程中创建N多个协程，只要内存够用。

	* 协程与子例程的[区别](https://en.wikipedia.org/wiki/Coroutine)
		* 子例程只有一个调用入口起点，子例程退出后，执行结束；子例程只返回一次，在两次调用之间不保存状态；
		* 协程有多个入口，调用起始点、或者从上一次返回点接着执行；从协程自己的角度来看，他放弃执行时不是退出，而是去调用另一个协程，或者说将CPU主动让给另一个协程；协程保存状态;
			* 计算机科学中，[yield](https://en.wikipedia.org/wiki/Yield_(multithreading))用于使处理器放弃当前运行的线程thread，并将它放入运行队列的末尾
			* 协程coroutine中的yield需要显式调用
		* 每个子例程可以转换为一个不带yield的协程
	
协程有关的四个概念：[coroutine](https://en.wikipedia.org/wiki/Coroutine)、[yield](https://en.wikipedia.org/wiki/Yield_(multithreading)、[Continuation](https://en.wikipedia.org/wiki/Continuation)、[cooperative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking)

* [Coroutines in C](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)
	* [给你一个直观的认识](http://www.hawkwithwind.net/blog/2011/02/18/%E5%8D%8F%E7%A8%8B%E7%9A%84c%E5%AE%9E%E7%8E%B0/)

```cpp
#include <stdio.h>
#include <stdlib.h>
int function(void) {
	static int i, state = 0;
	switch (state) {
		case 0: goto LABEL0;
		case 1: goto LABEL1;
	}
LABEL0: /* start of function */
	for (i = 0; i < 10; i++) {
		printf("\t\t\ti = %d\n", i);
		state = 1; /* so we will come back to LABEL1 */
		return i;
LABEL1:
		; /* resume control straight after the return */
	}
}

int main(){
	int j = 0;
	for ( j = 0; j < 21; j++){
		printf("j = %d, func = %d\n", j, function());
	}
}

```

执行结果

```bash
			i = 0
j = 0, func = 0
			i = 1
j = 1, func = 1
			i = 2
j = 2, func = 2
			i = 3
j = 3, func = 3
			i = 4
j = 4, func = 4
			i = 5
j = 5, func = 5
			i = 6
j = 6, func = 6
			i = 7
j = 7, func = 7
			i = 8
j = 8, func = 8
			i = 9
j = 9, func = 9
j = 10, func = 10
j = 11, func = 11
j = 12, func = 12
j = 13, func = 13
j = 14, func = 14
j = 15, func = 15
j = 16, func = 16
j = 17, func = 17
j = 18, func = 18
j = 19, func = 19
j = 20, func = 20
```

* 执行结果分析
	* 变量i 和 state 都是 static类型的，是文件全局作用域的
	* 首次执行function函数时，state = 0，goto 到 LABEL0，然后进行正常循环和返回
	* 当function调用次数超过十次后，每次进入function函数内部时，由switch分发到LABEL1；执行完循环体后，对i增1，然后进行判断是否 i < 10，发现不满足，退出程序
	* 可能我们会比较纠结function程序在 i > 10后不再执行return i语句，为什么还会返回自增后的结果呢？
		* 说实话我也并不能很好解释。我通过查看汇编代码，发现仍然会执行将i值转到ecx寄存器，我们的function返回值就从ecx拿到的。

* 仍然有两种更优化的写法: [左耳朵耗子的例子](http://coolshell.cn/articles/10975.html)和[上例](http://www.hawkwithwind.net/blog/2011/02/18/%E5%8D%8F%E7%A8%8B%E7%9A%84c%E5%AE%9E%E7%8E%B0/)的来源都是天才程序员  [imon Tatham](http://www.chiark.greenend.org.uk/~sgtatham/)对[协程做的尝试](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)

* [协程实现的基础](http://www.colaghost.net/os/unix_linux/341)

* 推荐的一些协程库
	* [protothread](http://dunkels.com/adam/pt/)
	* 其实c里面最好的协程库，就是嵌入LuaVM啦。

setjump & longjmp
--------------------------

##### man longjmp

```bash
NAME
       longjmp, siglongjmp - nonlocal jump to a saved stack context

SYNOPSIS
       #include <setjmp.h>

       void longjmp(jmp_buf env, int val);
       void siglongjmp(sigjmp_buf env, int val);

DESCRIPTION
       longjmp()  and  setjmp(3) are useful for dealing with errors and interrupts encountered in a low-level subroutine of a program.  longjmp() restores the environment saved by the last call of setjmp(3)
       with the corresponding env argument.  After longjmp() is completed, program execution continues as if the corresponding call of setjmp(3) had just returned the value val.  longjmp() cannot cause 0 to
       be returned.  If longjmp() is invoked with a second argument of 0, 1 will be returned instead.

	   longjmp和setjmp在处理程序调用子例程过程中遇到错误或中断时非常有用。longjmp将恢复最近一次调用setjmp时通过env参数保存的上下文环境。longjmp完成后，原来调用setjmp的地方将会返回，并且setjmp的返回值是longjmp的val参数。longjmp不会返回0。如果longjmp调用时第二个参数是0，那么它将会返回1.

       siglongjmp()  is  similar  to longjmp() except for the type of its env argument.  If, and only if, the sigsetjmp(3) call that set this env used a nonzero savesigs flag, siglongjmp() also restores the
       signal mask that was saved by sigsetjmp(3).

RETURN VALUE
       These functions never return.

```

##### man setjmp

```bash
NAME
       setjmp, sigsetjmp - save stack context for nonlocal goto

SYNOPSIS
       #include <setjmp.h>

       int setjmp(jmp_buf env);
       int sigsetjmp(sigjmp_buf env, int savesigs);
	 
DESCRIPTION
       setjmp()  and  longjmp(3)  are  useful for dealing with errors and interrupts encountered in a low-level subroutine of a program.  setjmp() saves the stack context/environment in env for later use by
       longjmp(3).  The stack context will be invalidated if the function which called setjmp() returns.

       sigsetjmp() is similar to setjmp().  If, and only if, savesigs is nonzero, the process's current signal mask is saved in env and will be restored if a siglongjmp(3) is later performed with this env.

RETURN VALUE
       setjmp() and sigsetjmp() return 0 if returning directly, and nonzero when returning from longjmp(3) or siglongjmp(3) using the saved context.

```

##### 示例代码

[代码地址](http://blog.linux.org.tw/~jserv/archives/001848.html) 博客评论区

```cpp
#include <stdio.h>                                                                                  
#include <unistd.h>
#include <setjmp.h>

jmp_buf jmpbuf_th0;
jmp_buf jmpbuf_th1;

static int cnt1 = 0;
static int cnt0 = 0;

static void thread_0()
{
	printf("%s \n\n", __FUNCTION__);
	sleep(1);
	longjmp(jmpbuf_th0, cnt0++);
}

static void thread_1()
{
	printf("%s \n\n", __FUNCTION__);
	sleep(1);
	longjmp(jmpbuf_th1, cnt1++);
}


int main()
{
	int rc0, rc1 = 0;

entry_thread_0:
	rc0 = setjmp(jmpbuf_th0);
	printf("rc0 = %d\n", rc0);
	if (rc0 != 0)
		thread_1();
entry_thread_1:
	rc1 = setjmp(jmpbuf_th1);
	printf("rc1 = %d\n", rc1);
	thread_0();

	return 0;
}
```

执行结果

```bash
rc0 = 0
rc1 = 0
thread_0 

rc0 = 1
thread_1 

rc1 = 1
thread_0 

rc0 = 1
thread_1 

rc1 = 1
thread_0 

rc0 = 2
thread_1 

rc1 = 2
thread_0 

rc0 = 3
thread_1 

rc1 = 3
thread_0 

```

* 执行过程分析
	* setjmp和longjmp都是基于程序空间中额外的jmpbuf
	* setjmp将当前环境存储在jmpbuf中，longjmp到同一个jmpbuf时，setjmp将会再次返回，返回值是longjmp的第二个参数val
	* 如果setjmp不是因为longjmp返回的，返回值为0
	* 不知道为什么执行结果中，自增的cnt0和cnt1在值为1的时候停顿了一次？


* [setjump & longjmp 进阶实现](https://github.com/nasrallahmounir/context-switch-setjmp-longjm)
* [setjump & longjmp 进阶实现](https://github.com/mrquincle/event-abbey)

ucontext
-------------------------

##### man getcontext

```bash
In  a  System  V-like  environment,  one has the two types mcontext_t and ucontext_t defined in <ucontext.h> and the four functions getcontext(), setcontext(), makecontext(3), and swapcontext(3) that
allow user-level context switching between multiple threads of control within a process.

The mcontext_t type is machine-dependent and opaque.  The ucontext_t type is a structure that has at least the following fields:

    typedef struct ucontext {
        struct ucontext *uc_link;
        sigset_t         uc_sigmask;
        stack_t          uc_stack;
        mcontext_t       uc_mcontext;
        ...
    } ucontext_t;

with sigset_t and stack_t defined in <signal.h>.  Here uc_link points to the context that will be resumed when the current context terminates (in case the current context was created  using  makecon‐
text(3)),  uc_sigmask is the set of signals blocked in this context (see sigprocmask(2)), uc_stack is the stack used by this context (see sigaltstack(2)), and uc_mcontext is the machine-specific rep‐
resentation of the saved context, that includes the calling thread's machine registers.

The function getcontext() initializes the structure pointed at by ucp to the currently active context.

The function setcontext() restores the user context pointed at by ucp.  A successful call does not return.  The context should have been obtained by a call  of  getcontext(),  or  makecontext(3),  or
passed as third argument to a signal handler.

If the context was obtained by a call of getcontext(), program execution continues as if this call just returned.

If the context was obtained by a call of makecontext(3), program execution continues by a call to the function func specified as the second argument of that call to makecontext(3).  When the function
func returns, we continue with the uc_link member of the structure ucp specified as the first argument of that call to makecontext(3).  When this member is NULL, the thread exits.

If the context was obtained by a call to a signal handler, then old standard text says that "program execution continues with the program instruction following the instruction interrupted by the sig‐
nal".  However, this sentence was removed in SUSv2, and the present verdict is "the result is unspecified".
```

##### man makecontext

```bash
In  a  System  V-like  environment,  one has the type ucontext_t defined in <ucontext.h> and the four functions getcontext(3), setcontext(3), makecontext() and swapcontext() that allow user-level context switching between multiple threads of control within a
process.

在类System V环境下，结构体ucontext_t和四个函数 getcontext、setcontext、makecontext和swapcontext提供了在一个进程内通过用户级上下文切换的方式实现多线程的方式

For the type and the first two functions, see getcontext(3).

The makecontext() function modifies the context pointed to by ucp (which was obtained from a call to getcontext(3)).  Before invoking makecontext(), the caller must allocate a new stack for this context and assign its address to ucp->uc_stack, and  define  a
successor context and assign its address to ucp->uc_link.

makecontext函数将修改ucp指向的 ucontext_t（必须是从getcontext获得的对象），在调用makecontext前，调用者必须为改上下文分配一个新的栈，并将ucp->uc_stack指向栈的地址，同时将ucp->uc_link指向下一个context

When  this  context  is  later activated (using setcontext(3) or swapcontext()) the function func is called, and passed the series of integer (int) arguments that follow argc; the caller must specify the number of these arguments in argc.  When this function
returns, the successor context is activated.  If the successor context pointer is NULL, the thread exits.

当该context稍后被激活(通过setcontext或者swapcontext)，makecontext函数参数中的func将被调用，同时向func传递argc个参数。当func返回时，下一个context将被激活调用。如果下一个context是null的话，该线程结束。

The swapcontext() function saves the current context in the structure pointed to by oucp, and then activates the context pointed to by ucp.

swapcontext函数将当前context保存在oucp结构体，同时激活ucp指向的context。
```

##### man page 中的例子

```bash
#include <ucontext.h>
#include <stdio.h>
#include <stdlib.h>

static ucontext_t uctx_main, uctx_func1, uctx_func2;

#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)

static void
func1(void)
{
    printf("func1: started\n");
    printf("func1: swapcontext(&uctx_func1, &uctx_func2)\n");
    if (swapcontext(&uctx_func1, &uctx_func2) == -1)
        handle_error("swapcontext");
    printf("func1: returning\n");
}

static void
func2(void)
{
    printf("func2: started\n");
    printf("func2: swapcontext(&uctx_func2, &uctx_func1)\n");
    if (swapcontext(&uctx_func2, &uctx_func1) == -1)
        handle_error("swapcontext");
    printf("func2: returning\n");
}

int
main(int argc, char *argv[])
{
    char func1_stack[16384];
    char func2_stack[16384];

    if (getcontext(&uctx_func1) == -1)
        handle_error("getcontext");
    uctx_func1.uc_stack.ss_sp = func1_stack;
    uctx_func1.uc_stack.ss_size = sizeof(func1_stack);
    uctx_func1.uc_link = &uctx_main;
    makecontext(&uctx_func1, func1, 0);

    if (getcontext(&uctx_func2) == -1)
        handle_error("getcontext");
    uctx_func2.uc_stack.ss_sp = func2_stack;
    uctx_func2.uc_stack.ss_size = sizeof(func2_stack);
    /* Successor context is f1(), unless argc > 1 */
    uctx_func2.uc_link = (argc > 1) ? NULL : &uctx_func1;
    makecontext(&uctx_func2, func2, 0);

    printf("main: swapcontext(&uctx_main, &uctx_func2)\n");
    if (swapcontext(&uctx_main, &uctx_func2) == -1)
        handle_error("swapcontext");

    printf("main: exiting\n");
    exit(EXIT_SUCCESS);
}


// http://stackoverflow.com/questions/20778735/is-the-type-stack-t-no-longer-defined-on-linux

```

执行结果：

```bash
$ gcc context.c -o context.o

$ ./context.o 
main: swapcontext(&uctx_main, &uctx_func2)
func2: started
func2: swapcontext(&uctx_func2, &uctx_func1)
func1: started
func1: swapcontext(&uctx_func1, &uctx_func2)
func2: returning
func1: returning
main: exiting
```

通过manpage中提供的ucontext_t源码，结合man page中的解释，我们对此做一分析。

* 执行过程详解
	* getcontext(&uctx_func2) 用当前上下文初始化ucontext_t结构体对象：uctx_func2
	* 在调用makecontext前，调用者为该上下文分配一个新的栈，并将成员uc_stack指向栈的地址，同时将uc_link指向下一个context。此时argc为0，所以uctx_func2的uc_link指向uctx_func1，也就是说当uctx_func2执行完后，会自动激活uctx_func1执行
	* makecontext(&uctx_func2, func2, 0) 函数设置当uctx_func2激活时，调用func2函数，参数为0个；当func2返回时，下一个context将被激活，在这里是uctx_func1
	* swapcontext(&uctx_main, &uctx_func2) 函数将进程当前执行的上下文保存在uctx_main中，并激活uctx_func2；uctx_func2开始执行，首先被执行的是func2函数，
		* func2: started
		* func2: swapcontext(&uctx_func2, &uctx_func1)
	* 随后func2函数调用 swapcontext(&uctx_func2, &uctx_func1)，将当前上下文环境保存在uctx_func2中，激活uctx_func1；uctx_func1开始执行，首先执行的是func1，打印出：
		* func1: started
		* func1: swapcontext(&uctx_func1, &uctx_func2)
	* 随后func1函数调用swapcontext(&uctx_func1, &uctx_func2)，将当前上下文环境保存在uctx_func1中，并激活uctx_func2；上一次uctx_func2保存的上下文环境将被回复，然后进程执行返回到上次被swap_context的点，打印出:
		* func2: returning
	* func2函数返回后，uctx_func2也就返回了，下一个context将被激活，也就是uctx_func1；打印出：
		* func1: returning
	* uctx_func1也执行结束，而它的下一个context是uctx_main，在main函数中最开始调用swap_context(&uctx_main, &uctx_func2)时，当时程序执行的上下文的地址就是这句，那么在main中继续执行，打印出：
		* main: exiting

```bash
$ ./context.o x
main: swapcontext(&uctx_main, &uctx_func2)
func2: started
func2: swapcontext(&uctx_func2, &uctx_func1)
func1: started
func1: swapcontext(&uctx_func1, &uctx_func2)
func2: returning
```

* 执行过程详解
	* 当argc个数不为0时，uctx_func2的uc_link为NULL，所以如果uctx_func2结束生命周期，那么整个进程将会退出
	* 在第一次swap_context(&uctx_main, &uctx_func2)时开始调用func2，func2中打印出：
		* func2: started
		* func2: swapcontext(&uctx_func2, &uctx_func1)
	* 随后swap执行uctx_func1，打印：
		* func1: started
		* func1: swapcontext(&uctx_func1, &uctx_func2)
	* 随后swap返回uctx_func2，打印：
		* func2: returning
	* 而func2返回时，由于uctx_func2->uc_link为NULL，所以整个进程退出，并不会返回到之前保存过的uctx_main执行上下文中。

通过以上分析，您有没有对协程有一个直观的认识呢？本质上，协程调度只是将当前执行的上下文保存起来；调度协程的时候就是将两个执行上下文context切换；指定context的下一个context，在本context执行结束后自动激活下一个context，实现协作；本context执行过程中，通过swap_context主动让出CPU，而不是被抢占放弃CPU。

* 可以参考下[进一步的实现](http://courses.cs.vt.edu/~cs3214/spring2016/examples/threads/coroutines.c)，加深下印象：

```cpp
#include <stdio.h>
#include <stdbool.h>
#include <ucontext.h>

static char stack[2][65536];            // a stack for each coroutine
static ucontext_t coroutine_state[2];   // container to remember context

// switch current coroutine (0 -> 1 -> 0 -> 1 ...)
static inline void
yield_to_next(void) 
{
    static int current = 0;

    int prev = current;
    int next = 1 - current;

    current = next;
    swapcontext(&coroutine_state[prev], &coroutine_state[next]);
}

static void
coroutine(int coroutine_number)
{
    int i;
    for (i = 0; i < 5; i++) {
        printf("Coroutine %d counts i=%d (&i=%p)\n", coroutine_number, i, &i);
        yield_to_next();
    }
}

int
main()
{
    ucontext_t return_to_main;

    // set up
    int i;
    for (i = 0; i < 2; i++) {
        // initialize ucontext_t
        getcontext(&coroutine_state[i]);
        // set up per-context stack
        coroutine_state[i].uc_stack.ss_sp = stack[i];
        coroutine_state[i].uc_stack.ss_size = sizeof(stack[i]);
        // when done, resume 'return_to_main' context
        coroutine_state[i].uc_link = &return_to_main;
        // let context[i] perform a call to coroutine(i) when swapped to
        makecontext(&coroutine_state[i], (void (*)(void))coroutine, 1, i);
    }

    printf("Starting coroutines...\n");
    swapcontext(&return_to_main, &coroutine_state[0]);
    printf("Done.\n");
    return 0;
}
```

* [云风的实现](https://github.com/cloudwu/coroutine) 

* 其他一些应用
	* [coroutine-libevent](https://github.com/colaghost/coroutine_event)
		* 在libevent里通过协程实现同步

Lua的协程
-----------------

* lua不支持那种真正的多线程（共享同一地址空间的抢占式线程），原因是
    - ANSI C没有原生的多线程，所以lua不能直接调用实现
    - 最重要的原因是，我们不认为多线程在lua中是个好主意

* 多线程是提供给底层编程的。多线程的同步机制，比如信号量和监控都是在操作系统上下文实现的，而非应用程序。调试多线程比较麻烦。而且，由于程序临界区的同步和竞争，多线程也会引起性能下降。
* 多线程引起的问题，主要是抢占式线程和共享内存导致的，lua解决这两个问题的方法是：lua coroutine是协作式的，非抢占式的，所以能避免线程切换导致的问题；lua coroutine之间不共享内存。

* 我见过的最lua的[lua代码和博客](https://techsingular.org/2012/12/22/programming-in-lua%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%8D-nil-%E5%92%8C-list/)

* lua的[coroutine and stack](https://techsingular.org/2013/05/09/programming-in-lua%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%8D-coroutine-lua-stack/)

* lua的c runtime stack和lua runtime stack是[什么样子的](https://techsingular.org/2013/07/14/programming-in-lua%EF%BC%88%E5%85%AD%EF%BC%89%EF%BC%8Dcontinuation/)


* [consumer-producer](https://www.lua.org/pil/9.2.html)

golang的协程(待续)
-----------------------

stm32/ contiki/ coroutine
----------------------------

[stm32上的协程实现](http://wiki.csie.ncku.edu.tw/embedded/Lab2)
	* http://blog.linux.org.tw/~jserv/archives/001848.html


