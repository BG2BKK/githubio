+++
date = "2016-05-18T20:26:09+08:00"
draft = true
title = "HTTP协议学习过程"

+++

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
[Range字段](https://tools.ietf.org/html/rfc7233)


* HTTP的幂等性
	* [幂等性](http://www.cnblogs.com/weidagang2046/archive/2011/06/04/idempotence.html)
		* 比较 POST GET PUT DELETE等方法，指出各自的幂等性
	* [电商中的幂等性](http://www.cnblogs.com/zhengyun_ustc/archive/2012/11/22/topic6.html)


