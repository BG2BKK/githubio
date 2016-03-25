+++
date = "2016-03-22T01:48:41+08:00"
draft = false
title = "Epoll在Linux Kernel中的新发展"

+++

EPOLL在linux 内核中的新发展
==============================

epoll是linux专有的系统调用，用于快速高效监听大量文件描述符。这个API从kernel-2.5版本合并进主线后一直使用至今，即使如此，和其他的接口一样，epoll仍然有提升空间。现在有两个patch为此epoll系列系统调用添加了新的功能。

epoll概述
------------

epoll的功能与select或者poll类似，但是epoll在应对处理大规模文件描述符时拥有更灵活的选项和更高的性能。每次调用select和poll，都会重新生成新的文件描述符集合，所以内核需要检查每一个描述符是否合法，是否IO就绪，然后将相关进程添加到相应的等待队列。但是实际上一般情况下，在两次select或者poll调用之间，有事件产生的fd并不多，所以对每个fd都进行前述流程实际上有很多不必要的重复性操作。因此，Epoll将设置监听fd和监听fd是否就绪这两个任务分开，从而解决这一问题。

使用epoll的话，必须首先新建epoll fd用于轮询，新建epfd通过如下接口：

```cpp
    #include <sys/poll.h>

    int epoll_create(int size);
    int epoll_create1(int flags);
```
两者都返回epoll fd，而epoll_create的size参数已经不再有意义，epoll_create1的flag参数设置CLOSE_ON_EXEC标志。

第二步是添加所有被监听的fd，通过函数：

```cpp
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
参数op是EPOLL_CTL_ADD时，fd将被添加进epfd的监听集合中，event参数用于指定哪个类型的事件被轮询，读事件、写事件或者其他事件，详情参考man page。

最后，等待集合中某个fd是否就绪的工作由以下函数实现：

```cpp
    int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
    int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, 
                                                int timeout, const sigset_t *sigmask);
```
有事件发生时，epoll_wait将返回，产生的时间存在参数events中，最多maxevents个事件。如果timeout时间内没有事件发生，epoll_wait也将返回，timeout的单位是ms。epoll_pwait可以在监听时使用信号集sigmask来屏蔽特定信号。

patch 1：epoll_ctl_batch() 和 epoll_pwait1()
---------------------------------------------

Fam Zheng为epoll引入了两个新的系统调用。第一个函数用来解决一个性能问题：每次调用epoll_ctl，都只能添加、修改和删除一个fd，如果有大量fd需要修改，那么需要调用相应次数的epoll_ctl来实现，这会导致大量系统调用发生，而这个场景却是经常发生的。本次引入的epoll_ctl_batch()通过在一个系统调用中添加多个fd来解决这个问题：

```cpp
    int epoll_ctl_batch(int epfd, int flags, int ncmds, struct epoll_ctl_cmd *cmds);
```
结构体epoll_ctl_cmd用于描述一个待添加的事件，可以看作是epoll_ctl参数的一次打包：

```cpp
truct epoll_ctl_cmd {
      /* Reserved flags for future extension, must be 0. */
      int flags;
      /* The same as epoll_ctl() op parameter. */
      int op;
      /* The same as epoll_ctl() fd parameter. */
      int fd;
      /* The same as the "events" field in struct epoll_event. */
      uint32_t events;
      /* The same as the "data" field in struct epoll_event. */
      uint64_t data;
      /* Output field, will be set to the return code after this
       * command is executed by kernel */
      int result;
};
```
将一个epoll_ctl_cmd数组cmds传入，则epoll_ctl_batch可以在一次系统调用中添加多个fd。

Fam的第二个系统调用是

```cpp
    struct epoll_wait_params{
        int clockid;
        struct timespec timeout;
        sigset_t *sigmask;
        size_t sigsetsize;
    }

    int epoll_pwait(int epfd, int flags, struct epoll_event *events, int maxevents, 
                                                struct epoll_wait_params *params);
```
本版本的epoll_pwait1()添加了一个flags参数，但是并未定义任何flag值，所以flags置为0即可。所哟参数都写在params参数中，目的是为应用程序提供更精细的时间控制。对于epoll_wait()来说，毫秒级的时钟分辨率已经被证明在一些场景中过于粗糙，新的系统供调用提供了纳秒级别的精度，解决了这个问题。

多线程环境下更好的性能
-------------------------

Jason Baron主要解决一个相对来说不那么常见的场景下，epoll现有的一个问题。通常情况下，只有一个进程轮询一个给定的fd，但是在Jason的场景中，会有多个进程轮询一个fd集合。在这个场景设定下，一个fd有事件产生时将会唤醒所有进程，即使最后只有一个进程能够得到处理该事件的机会，这就是所谓的“惊群”问题。

Jason的解决方案是通过epoll_ctl向被轮询的fd添加一对新的flags，第一个是EPOLLEXCLUSIVE，保证只有一个进程能被唤醒然后处理事件，这个flag使得在epoll_ctl添加轮询fd时，简单的用add_wait_queue_exclusive()代替add_wait_queue()。很明显，所有轮询同一个fd的进程都要通过互斥模式被唤醒。

不过，这个变化不能完全解决问题，因为这会导致当有事件发生时，唤醒的都是同一个进程。而Epoll存在的一个原因是在epoll_wait()调用期间能够使得所有进程都留在等待唤醒队列中，处于等待队列头部的进程仍然在队列头部，所以这个进程将取得所有互斥模式的fd。但是我们的目的是，多个进程轮询同一fd集合时，能够散开执行，而每次都唤醒的是同一个进程与此相悖。为解决这个问题，Jason添加了另一个flag，叫做 EPOLLROUNDROBIN，使得内核按顺序处理每个进程

一个新的等待队列用来支持实现这种方式

```cpp
    void add_wait_queue_rr(wait_queue_head_t *q, wait_queue_t *wait);
```
使用该函数后，当wait返回时，只有一个进程被唤醒，效果和add_wait_queue_exclusive()一样。但是，这个被唤醒的进程，将被从队列头移到队列尾，直到它前面的所有进程都得到唤醒机会后，才能再次被唤醒。

Jason的提交patch的同时也提交了一个用于压测的程序，压测结果显示，互斥模式使得执行时间降低了50%，当有大量的唤醒发生时，“惊群”效应带来的性能损耗就不会发生了。

Jason的patch得到了多次review和comments，并且自从1月份提出后进行了多次修改。编辑们对API相关的patch投入了越来越多的关注，这是对的，因为API将会长期有效，甚至是永远有效，以保持兼容，所以最好在向用户退出之前就搞定所有bug，以提供永久支持的态度提交。这些patch已经接近就绪，将会在下一个合并窗口中出现。
