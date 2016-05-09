+++
date = "2016-05-06T11:39:48+08:00"
draft = true
title = "effective tips in daily learning"

+++

http
--------------------------

Lua的学习、使用和源码精读
--------------------------

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


* [golang多协程channel同步](https://www.chenwang.net/2015/12/12/%E5%AE%8C%E6%95%B4%E7%9A%84golang-%E5%A4%9A%E5%8D%8F%E7%A8%8B%E4%BF%A1%E9%81%93-%E4%BB%BB%E5%8A%A1%E5%A4%84%E7%90%86%E7%A4%BA%E4%BE%8B/)	
	* sync.WaitGroup
	* defer
	* recover
* [golang闭包与协程使用](http://blog.csdn.net/gophers/article/details/40505683)