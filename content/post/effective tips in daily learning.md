+++
date = "2016-05-06T11:39:48+08:00"
draft = false
title = "effective tips in daily learning"

+++

C语言中的宏
--------------

在看代码的过程中我看到了 ## 和 # 两个符号，前一个我知道，后一个就不清楚了。在[文档](http://blog.csdn.net/dotphoenix/article/details/4345174)中我才知道C语言有这么多的宏，##表示连接符concat，而#EXP表示将代码中的表达式EXP转为字符串，这样写在错误日志中我们就知道是那句代码导致错误的了。

C语言中的宏以及用法还有很多，上述文档总结的非常好，除了##和#外，还有...、宏展开时多次求值问题都有讲解，值得一读一试。

[这篇文章](http://hbprotoss.github.io/posts/cyu-yan-hong-de-te-shu-yong-fa-he-ji-ge-keng.html)也不错，他们都源自于[gnu的官方文档](http://gcc.gnu.org/onlinedocs/cpp/Macros.html)

OmniGraffle
--------------

* [OmniGraffle简书](http://www.jianshu.com/p/52f3ecbe8f2d)
* [UX基础 - OmniGraffle新手指南](http://beforweb.com/node/202)

markdown
------------
	* [markdown边角料](http://blog.csdn.net/phunxm/article/details/49565427)

kernel
---------------

* [kernel工程导论](http://blog.csdn.net/column/details/linux-kernel-no-code.html)

内存管理
--------------------

* http://blog.jobbole.com/103993/
* http://blog.jobbole.com/88673/

前端开发
-------------------

* 如何使textarea的大小随着其内容增加而变化呢？

* 解决方法
	* 1、http://audi.tw/Blog/Javascript/javascript.textarea.autogrow.asp
	* 2、http://www.cnblogs.com/xmmcn/archive/2012/12/18/2822968.html
	* 3、jquery 插件: https://bobscript.com/archives/419/


awk 
-------------------------------
* http://blog.sina.com.cn/s/blog_4a033b090100xo2b.html
* http://blog.csdn.net/junjieguo/article/details/7525794

postgre
-------------------------------

* [修改表结构](http://blog.51yip.com/pgsql/1528.html)，[postgre_wiki](https://wiki.postgresql.org/wiki/ALTER_TABLE)
* [插入数据](http://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html)、[postgre_wiki](https://wiki.postgresql.org/wiki/9.1%E7%AC%AC%E5%85%AD%E7%AB%A0)
* [清楚pg_xlog](http://my.oschina.net/Kenyon/blog/101432)


python
-------------------------------

* python魔术方法
	* [指南](http://pycoders-weekly-chinese.readthedocs.io/en/latest/issue6/a-guide-to-pythons-magic-methods.html#id19)
	* [入门](http://blog.csdn.net/wklken/article/details/8126381)

[http协议]
----------------------
http协议301和302的区别

* http://www.cnblogs.com/caiyuanzai/archive/2012/04/24/2469013.html
* http://blog.csdn.net/qmhball/article/details/7838989

linux socket编程样例
---------------------------

* http://alas.matf.bg.ac.rs/manuals/lspe/snode=106.html

* [进程间传递fd](http://blog.csdn.net/win_lin/article/details/7760951)

linux apue编程
--------------------------------

* epoll是同步非阻塞的

epoll、select等多路服用IO，将fd加入等待时间的队列中，每隔一段时间去轮询一次，因此是同步的；优点是能够在等待任务的时间里去做别的任务；缺点是任务完成的响应延迟增大，因为每隔一段时间去轮询他们，在时间间隔内任务可能已经完成而等待处理等待了一段时间了。

[参考链接](https://ring0.me/2014/11/sync-async-blocked/)

同步/异步指的是被调用方的通知方式，被调用方完成后，主动通知调用方，还是等待调用方发现。前者是异步，后者是同步。从这里也可以看出，[异步IO通知调用方时，数据已经就绪](https://segmentfault.com/a/1190000003063859)，对于网络IO来说，异步IO已经将数据从内核复制到用户空间了。

阻塞/非阻塞是调用方的等待方式，是一直等待在做的事件完成，还是去做别的事情，等到在做的事件完成后再接着进行处理。前者是阻塞，后者是非阻塞

因此epoll是同步和非阻塞的。

性能分析
------------------------

* [有用的systemtap脚本](https://sourceware.org/systemtap/examples/keyword-index.html#FUTEX)

分布式存储
--------------------------

* [知识体系](http://wuchong.me/blog/2014/08/07/distributed-storage-system-knowledge/)
	* [B-Tree](http://taop.marchtea.com/03.02.html)

分布式ID生成
----------------------

* [ID](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=403837240&idx=1&sn=ae9f2bf0cc5b0f68f9a2213485313127&scene=21#wechat_redirect)
	* 方案一： 常规数据库的auto increment服务
		* 改进：可以将ID划均分给若干数据库，每个数据库自增的起点不一样，可以保证各库生成ID不同
			* 缺点是非强一致性
	* 方案二： 单点批量生成；ID生成服务每次从数据库预定一定容量的ID，然后派发；可以成倍降低数据库压力；
		* 缺点：单点服务、可能造成空洞；
		* 改进：找备胎，一旦主ID服务挂掉，备胎立刻备上；通过vip+keepalived实现
	* 方案三：uuid
	* 方案四：当前毫秒数；缺点是每个毫秒容量有限，也可能重复
	* 方案五：将64bit数字作为ID，分别包含字段：毫秒数、业务线、机房、机器、以及毫秒内序列号；根据业务来规划容量；毫秒数可以保证ID是趋势自增的
* [ID](http://www.cnblogs.com/heyuquan/archive/2013/08/16/global-guid-identity-maxId.html)

分布式锁
-------------------

* http://www.cnblogs.com/zhengyun_ustc/archive/2012/11/17/topic2.html
* http://www.jeffkit.info/2011/07/1000/

一致性哈希 consistent hashing
----------------------------------

* http://www.codeproject.com/Articles/56138/Consistent-hashing
* http://blog.huanghao.me/?p=14
* http://blog.csdn.net/sparkliang/article/details/5279393

2PC、3PC和Paxos算法
------------------------

* [coolshell](http://coolshell.cn/articles/10910.html)


并发编程
----------------
* [volatile](http://hedengcheng.com/?p=725)
* [并发编程](http://blogread.cn/it/article/7282?f=wb)

* [聊聊并发](http://www.infoq.com/cn/articles/producers-and-consumers-mode)

nginx配置文件
---------------------

* rewrite: 
	* http://www.xiehaichao.com/articles/428.html
	* http://seanlook.com/2015/05/17/nginx-location-rewrite/

* [nginx配置使用用户自定义错误页面](http://stackoverflow.com/questions/13621915/nginx-error-pages-one-location-rule-to-fit-them-all)

Lua的学习、使用和源码精读
--------------------------

## lua的元表

* http://lua-users.org/wiki/MetamethodsTutorial
	* 元表用来扩展lua对象的功能，元表的含义在于生来就有，lua对象本身会带有这个表，所以称为元表
	* metatable也是普通的lua table，包含一系列元方法metamethods，每个元方法有对应的events触发；比如[算术运算符、\_\_index 等操作](http://lua-users.org/wiki/MetatableEvents)
	* \_\_index



* http://lua-users.org/wiki/MetatableEvents
	<ul>
	<li> <strong>__index</strong> - Control 'prototype' inheritance. When accessing "myTable[key]" and the key does not appear in the table, but the metatable has an __index property:
	<ul>
	<li> if the value is a function, the function is called, passing in the table and the key; the return value of that function is returned as the result.
	</li><li> if the value is another table, the value of the key in that table is asked for and returned
	<ul>
	<li> <em>(and if it doesn't exist in <strong>that</strong> table, but that table's metatable has an __index property, then it continues on up)</em>
	</li></ul>
	</li><li> <em>Use "rawget(myTable,key)" to skip this metamethod.</em>
	</li></ul>
	</li></ul>

	* 控制类型继承；当访问myTable[key]，而table中没有key域时，如果元表有\_\_index项：
		* 如果\_\_index是函数，则调用该函数
		* 如果\_\_index是table，返回这个table中key域的值；如果\_\_index是table并且该table没有key域，但是该table有\_\_index，则继续查找(calls fallback function or fallback table)
		* 使用rawget(myTable, key)可以跳过metatable

	* \_\_index是一个应用广泛并用处很大的元方法metamethod
		* 如果想获取table中的元素key，而key在table中没有找到，该方法可以定义为函数或者一个table，来寻找key。
			* 如果\_\_index是函数，该函数的第一个参数是没有找到key的这个table，第二个参数是key； 
			* 如果\_\_index是table，那么将在该table中寻找key，如果没有找到，可以继续从这个table的\_\_index寻找，因此你可以通过\_\_index进行一整个链条的查找。


	* \_\_metatable
		* 用于隐藏metatable，当调用getmetatable(myTable)时，如果该域不为空，则返回这个域的值，而不是metatable
	
	* [sample code](http://lua-users.org/wiki/LuaClassesWithMetatable)

## Lua面向对象

* http://dabing1022.github.io/2014/03/18/multiple-inheritance-understand-lua/

## Lua源码精读

* Lua的全局和状态，以及初始化
	* [Lua的全局和状态](http://blog.csdn.net/maximuszhou/article/details/46277695)
		* 在调用lua_newstate 初始化Lua虚拟机时，会创建一个全局状态和一个线程（或称为调用栈），这个全局状态在整个虚拟机中是唯一的，供其他线程共享。一个Lua虚拟机中可以包括多个线程，这些线程共享一个全局状态，线程之间也可以调用lua_xmove函数来交换数据。
	* [LuaVM 初始化](http://www.cnblogs.com/ringofthec/archive/2010/11/09/lua_State.html)
	* [lua_State](https://yq.aliyun.com/articles/1756?spm=5176.100240.searchblog.8.YbMjAK)

## Lua与C的交互
	* [深入理解Lua与C用于数据交互的栈](http://blog.csdn.net/maximuszhou/article/details/21331819)
	* 为啥要通过 栈 来通信呢？
		* Lua是动态类型语言，在Lua语言中没有类型定义的语法，每个值都携带了它自身的类型信息，而C语言是静态类型语言
		* Lua使用垃圾收集，可以自动管理内存，而C语言要求程序自己释放分配的内存，需应用程序自身管理内存
	* 压栈的影响
		* C将值压入栈中后，Lua将会生成相应类型的结构，存储和管理这个值
		* Lua不会持有指向VM外部的指针，指向的都是自己的结构和栈上的结构
		* 比如压入字符串，Lua生成Lua_TTSTRING类型的对象，C可以随意释放这个字符串


* http://if-yu.info/lua-notes.html

## list和nil
	* https://techsingular.org/2012/12/22/programming-in-lua%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%8D-nil-%E5%92%8C-list/
	* nil不但不是无的意思，反而在list中起到占位和有的意思。

## [lua的first class](https://techsingular.org/2012/12/22/programming-in-lua%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%8D-nil-%E5%92%8C-list/)
	
* 可以被赋值给变量；
* 可以作为参数；
* 可以作为返回值；
* 可以作为数据结构的构成部分。( 注意 nil 并不完全符合这个要求，但是可以通过某个 field 的缺失来表示 nil。)

## Lua的GC
	* lua的[gc](https://techsingular.org/2013/10/27/lua-%E7%9A%84%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/)

## lua的table
	* https://facepunch.com/showthread.php?t=1306348

> I don't believe this is correct. It was my understanding that Lua tables are implemented as a two-part data structure whereby dense non-negative integer keys are stored as a simple array and are therefore indexed by a simple pointer addition and dereference, which is O(1). All other keys (non-integer, negative and sparse integer) are stored in a hashmap as a chained scatter table which stores key-values pairs as a flat array indexed by the hash of the key. Collisions are resolved by storing pointers to the next element with the same hash alongside this (essentially a linked list of colliding elements). At worst case, where all elements collide, the complexity of this implementation is O(n), however it is expected that on average the number of collisions per element is 1 and the maximum is 2; this means that the average complexity is O(1).

> The implementation of Lua tables is such that, even with a huge number of elements, lookup is as quick as possible.

## lua函数使用

* setmetatable
	* https://www.lua.org/manual/5.2/manual.html
	* setmetatable(table, metatable)
		* 将metatable设置为table的元表，（在Lua中只能设置table的元表，其他类型的对象不行，除非使用C）。如果metatable为nil，则参数cable的元表被清除；如果该table的__metatable不为空，则抛出异常
	* 该函数返回table

* collectgarbage("count")
	* 垃圾回收函数

* [lua yield 和 resume](http://www.lua.org/manual/5.2/manual.html#2.6)
	* http://www.lua.org/manual/5.2/manual.html#pdf-coroutine.resume

```lua
-- http://my.oschina.net/wangxuanyihaha/blog/186401

function foo(a)
    print("foo", a)
    return coroutine.yield(2 * a)
end

co = coroutine.create(function ( a, b )
    print("co-body", a, b)
    local r = foo(a + 1)
    print("co-body", r)
    local r, s = coroutine.yield(a + b, a - b)
    print("co-body", r, s)
    return b, "end"
end)

print("main", coroutine.resume(co, 1, 10))
print("main", coroutine.resume(co, "m"))	-- resume的参数 'm' 是在调用yield传入的，所以本次是在第5行 return m
print("main", coroutine.resume(co, "x", "y"))
print("main", coroutine.resume(co, "x", "y"))
```

```bash
co-body	1	10
foo	2
main	true	4
co-body	m
main	true	11	-9
co-body	x	y
main	true	10	end
main	false	cannot resume dead coroutine
```


## lua编码的陷阱
	* [字符串拼接导致垃圾产生](http://tech.uc.cn/?p=1131)

golang
--------------------------

### golang的并发、协程和channel

* [golang并发编程初探](http://ibillxia.github.io/blog/2014/03/16/go-concurrent-programming-first-try/)

```lua
	for i := 0; i < 10; i++ {
		go func() {
			arr[i] = i + i*i
			chs[i] <- arr[i]
		}()
	}
```
类似于如上形式的for循环中启动go协程，在for循环结束时协程才会开执行，所以每个协程用到的i都是最大值10

可以将i加入到协程的参数中，如下：

```lua
	for i := 0; i < 10; i++ {
		
		chs[i] = make(chan int)
		go func(i) {
			arr[i] = i + i*i
			chs[i] <- arr[i]
		}(i)
	}
```

可以用局部变量保存i值

```lua
	for i := 0; i < 10; i++ {
		i := i
		chs[i] = make(chan int)
		go func() {
			arr[i] = i + i*i
			chs[i] <- arr[i]
		}()
	}
```

* [golang的底层数据结构](http://my.oschina.net/goal/blog/196891)


* [golang多协程channel同步](https://www.chenwang.net/2015/12/12/%E5%AE%8C%E6%95%B4%E7%9A%84golang-%E5%A4%9A%E5%8D%8F%E7%A8%8B%E4%BF%A1%E9%81%93-%E4%BB%BB%E5%8A%A1%E5%A4%84%E7%90%86%E7%A4%BA%E4%BE%8B/)	
	* sync.WaitGroup
	* defer
	* recover
* [golang闭包与协程使用](http://blog.csdn.net/gophers/article/details/40505683)

### golang设计模式
	* [单例模式](http://marcio.io/2015/07/singleton-pattern-in-go/)
	* [golang map reduce](https://gist.github.com/mcastilho)


### golang接口
	* https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/02.6.md
	* http://xiaorui.cc/2016/03/11/%E5%85%B3%E4%BA%8Egolang-struct-interface%E7%9A%84%E7%90%86%E8%A7%A3%E4%BD%BF%E7%94%A8/
	* http://blog.csdn.net/zhangzhebjut/article/details/24974315

* UML图
	* 实现关系、泛化关系、关联关系、聚合关系 http://design-patterns.readthedocs.io/zh_CN/latest/read_uml.html


* 设计模式
	* http://tengj.top/2016/04/04/sjms3abstractfactory/
