+++
date = "2016-03-13T17:59:29+08:00"
draft = true
title = "coroutine and goroutine"

+++


* lua不支持那种真正的多线程（共享同一地址空间的抢占式线程），原因是
    - ANSI C没有原生的多线程，所以lua不能直接调用实现
    - 最重要的原因是，我们不认为多线程在lua中是个好主意

* 多线程是提供给底层编程的。多线程的同步机制，比如信号量和监控都是在操作系统上下文实现的，而非应用程序。调试多线程比较麻烦。而且，由于程序临界区的同步和竞争，多线程也会引起性能下降。
* 多线程引起的问题，主要是抢占式线程和共享内存导致的，lua解决这两个问题的方法是：lua coroutine是协作式的，非抢占式的，所以能避免线程切换导致的问题；lua coroutine之间不共享内存。

* 我见过的最lua的[lua代码和博客](https://techsingular.org/2012/12/22/programming-in-lua%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%8D-nil-%E5%92%8C-list/)

* lua的[coroutine and stack](https://techsingular.org/2013/05/09/programming-in-lua%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%8D-coroutine-lua-stack/)

* lua的c runtime stack和lua runtime stack是[什么样子的](https://techsingular.org/2013/07/14/programming-in-lua%EF%BC%88%E5%85%AD%EF%BC%89%EF%BC%8Dcontinuation/)

* lua的[gc](https://techsingular.org/2013/10/27/lua-%E7%9A%84%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/)

* man getcontext

* man swapcontext

* man setjmp

* man longjmp

* https://github.com/cloudwu/coroutine

* http://www.colaghost.net/os/unix_linux/321
