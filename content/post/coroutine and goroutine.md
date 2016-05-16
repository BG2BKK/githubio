+++
date = "2016-03-13T17:59:29+08:00"
draft = false
title = "coroutine and goroutine"

+++


* 协程是什么？ what is coroutine ?
	* subroutine
	* [Coroutines in C](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)
	* [协程](http://coolshell.cn/articles/10975.html)
* ucontext
	* [ucontext版本的实现](http://courses.cs.vt.edu/~cs3214/spring2016/examples/threads/coroutines.c)
		* man getcontext and setcontext
		* man makecontext and swapcontext

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

	* [ucontext的使用](man makecontext)
	* [云风的实现](https://github.com/cloudwu/coroutine) 

* setjump & longjmp
	* [setjump实现](https://github.com/nasrallahmounir/context-switch-setjmp-longjm)
	* [setjump实现](https://github.com/mrquincle/event-abbey)

* 其他一些应用
	* [coroutine-libevent](https://github.com/colaghost/coroutine_event)
		* 在libevent里通过协程实现同步
* lua的协程

* golang的协程(待续)


* lua不支持那种真正的多线程（共享同一地址空间的抢占式线程），原因是
    - ANSI C没有原生的多线程，所以lua不能直接调用实现
    - 最重要的原因是，我们不认为多线程在lua中是个好主意

* 多线程是提供给底层编程的。多线程的同步机制，比如信号量和监控都是在操作系统上下文实现的，而非应用程序。调试多线程比较麻烦。而且，由于程序临界区的同步和竞争，多线程也会引起性能下降。
* 多线程引起的问题，主要是抢占式线程和共享内存导致的，lua解决这两个问题的方法是：lua coroutine是协作式的，非抢占式的，所以能避免线程切换导致的问题；lua coroutine之间不共享内存。

* 我见过的最lua的[lua代码和博客](https://techsingular.org/2012/12/22/programming-in-lua%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%8D-nil-%E5%92%8C-list/)

* lua的[coroutine and stack](https://techsingular.org/2013/05/09/programming-in-lua%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%8D-coroutine-lua-stack/)

* lua的c runtime stack和lua runtime stack是[什么样子的](https://techsingular.org/2013/07/14/programming-in-lua%EF%BC%88%E5%85%AD%EF%BC%89%EF%BC%8Dcontinuation/)


* [consumer-producer](https://www.lua.org/pil/9.2.html)

