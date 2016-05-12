+++
date = "2016-05-06T11:39:48+08:00"
draft = true
title = "effective tips in daily learning"

+++

一致性哈希 consistent hashing
----------------------------------

* http://www.codeproject.com/Articles/56138/Consistent-hashing
* http://blog.huanghao.me/?p=14
* http://blog.csdn.net/sparkliang/article/details/5279393
并发编程
----------------
* [volatile](http://hedengcheng.com/?p=725)
* [并发编程](http://blogread.cn/it/article/7282?f=wb)

nginx配置文件
---------------------

* rewrite: 
	* http://www.xiehaichao.com/articles/428.html
	* http://seanlook.com/2015/05/17/nginx-location-rewrite/

HTTP
--------------------------
* HTTP/0.9
* HTTP/1.0
	* 引入POST方法，允许client发送表单
	* 引入Header头部，能够返回错误码、以及文字或图片等其他格式
	* 头部的Connection: keep-alive可以保持长连接

* HTTP/1.1
	* 增加HOST字段，然后GET 后面只需要跟相对路径即可，由HOST字段分辨访问主机的哪个域；之前是不能有多个域的，因为多个域指向同一IP会造成混淆
	* 引入Range头，可以只下载一部分内容
	* 默认长链接 keepalive

* HTTP/2:
	* 革命性的更新，压缩头部，stream和frame等机制

* HTTP request 请求
	* 请求行 eg. GET www.cnblogs.com HTTP/1.1
	* HTTP头部 eg. Content-Type: text/html; charset=utf-8
	* 内容	只在post请求存在

* HTTP respond 响应
	* 状态行 eg. HTTP/1.1 200 OK
	* HTTP头部
		* 响应头 response header
		* 通用头 general header eg. Date  ETAG Connection: Keep-Alive

```bash
通用头即可以包含在HTTP请求中，也可以包含在HTTP响应中。通用头的作用是描述HTTP协议本身。比如描述HTTP是否持久连接的Connection头，HTTP发送日期的Date头，描述HTTP所在TCP连接时间的Keep-Alive头，用于缓存控制的Cache-Control头等。
```

		* 实体头 entity header eg. Content-Type: Content-Length

```bash
实体头是那些描述HTTP信息的头。既可以出现在HTTP POST方法的请求中，也可以出现在HTTP响应中。比如图5和图6中的Content-Type和Content-length都是描述实体的类型和大小的头都属于实体头。其它还有用于描述实体的Content-Language,Content-MD5,Content-Encoding以及控制实体缓存的Expires和Last-Modifies头等。
```
	* 响应数据

	* 浏览器靠响应头部的Content-Type来确定如何处理收到的信息，类型由IANA分配，包括[8大类媒体类型](http://www.iana.org/assignments/media-types/media-types.xhtml)

* HTTP头部
	* 请求头 request header

```bash
请求头是那些由客户端发往服务端以便帮助服务端更好的满足客户端请求的头。请求头只能出现在HTTP请求中。比如告诉服务器只接收某种响应内容的Accept头，发送Cookies的Cookie头，显示请求主机域的HOST头，用于缓存的If-Match，If-Match-Since,If-None-Match头，用于只取HTTP响应信息中部分信息的Range头，用于附属HTML相关请求引用的Referer头等。
```
	* 响应头 response header

```bash
HTTP响应头是那些描述HTTP响应本身的头，这里面并不包含描述HTTP响应中第三部分也就是HTTP信息的头（这部分由实体头负责）。比如说定时刷新的Refresh头，当遇到503错误时自动重试的Retry-After头，显示服务器信息的Server头，设置COOKIE的Set-Cookie头，告诉客户端可以部分请求的Accept-Ranges头等。
```

[一次http请求的过程](http://www.linux178.com/web/httprequest.html)
[http请求头部keepalive](https://www.byvoid.com/blog/http-keep-alive-header)
[content-encoding transfer-encoding content-length transfer-length](https://imququ.com/post/transfer-encoding-header-in-http.html)

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
