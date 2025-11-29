+++
date = '2016-04-22T11:25:56+08:00'
draft = true
title = 'resume'

+++

黄振栋简历
================

联系方式
--------------------

- Email：bg2bkk # gmail.com 
- 微博：[@bg2bkk](http://weibo.com/BG2BKK)

个人信息
--------------------

 - 男/1990 
 - 硕士/哈工大深圳研究生院计算机系 
 - 工作年限：14个月
 - 英语水平：CET-6
 - 技术博客：http://bg2bkk.github.io
 - Github：http://github.com/bg2bkk

教育背景
--------------------

	- 2012.09 ~ 2015.01 哈尔滨工业大学 计算机 硕士
	- 2008.09 ~ 2012.06 哈尔滨工程大学 计算机 学士 

---

技能清单
-------------------

* c/ lua/ python/ golang/ shell/ vhdl
* nginx/ ngx_lua/ redis
* linux kernel/ performance profiling
* 算法、数据结构和设计模式
* linux系统编程、多线程编程、网络编程
* embedded system && IOT: ble/mqtt/elua
* microprocessor: stm32 51 avr FPGA

---

自我评价
-------------------

* 热爱计算机，热爱编程，热爱程序员这个工作

---

工作经历
-----------------

	- 2015.01 ~ NOW		 新浪微博	系统开发工程师 
	- 2013.09 ~ 2014.05	 极光推送	后台开发实习生 

项目经历
-----------------

#### 新浪微博 系统开发工程师 （ 2015年1月 ~ 2016年8月 ）

* 微博防抓站系统
	
<!--
	* 手机微博、PC微博以及Wap版微博的防抓站系统
		* 微博访问量日益增长，恶意抓站同时增多
		* 封禁非法访问的IP，阻止恶意抓站
	* 防抓站系统的松耦合架构
		* 管理节点
			* ngx_lua开发
			* 管理节点提供提交封禁IP、解禁IP、白名单等接口，微博安全组分析得出恶意IP向管理节点提交
			* 管理节点将封禁IP等信息写入redis中
		* 封禁节点
			* ngx_lua开发
			* 封禁节点的nginx worker启动定时任务，定期从远程redis拉取新的封禁信息，并将获取到的封禁IP加入到ngx_lua的shared dict共享内存中
			* 封禁模块工作在nginx的access阶段，获取用户请求的IP，从shared dict查询是否是封禁IP，如果是，则返回403终止用户访问，否则放行
		* 可靠性
			* 对于nginx的定时任务，由于脱离worker主进程，所以即使出错，不影响原有nginx业务
			* 对于封禁模块，采用lua的类try-catch机制的xpcall方法，保证即使access阶段的lua代码出错时，会将用户请求放行，所以最坏情况下是不封禁所有IP 
			* 日志监控，监控定时任务执行，实时报警；收集封禁数据，统计IP拦截量
		* 运行情况
			* 封禁模块嵌入在微博前端机的nginx中，承载全量访问
			* 封禁模块执行任务最轻，仅仅获取用户IP以及查询nginx的shm，用时低于1ms
-->

* 手机微博七层动态调度系统dygateway

<!--
	* 动态调度系统***dygateway*** 在手机微博7层实现
		* 系统资源的动态伸缩
		* 用户请求的动态分流
		* 产品服务的灰度发布
	* dygateway作为7层服务时：
		* 动态分流子系统 [ABTestingGateway](https://github.com/CNSRE/ABTestingGateway)
			* 根据用户请求的特征和分流策略将请求转发至不同的后端upstream；
			* 动态设置分流策略，实时生效；
			* 支持单级分流和多级分流。
			* 基于ngx_lua 和 redis 开发
		* 动态upstream模块[lua-upstream-nginx-module](https://github.com/CNSRE/lua-upstream-nginx-module)
			* 该模块可以实时修改nginx的upstream列表，增减upstream，增减upstream中的member成员，实现动态伸缩。
			* nginx模块开发
	* 目前应用于手机微博、微博头条等产品线

-->

<!--
修改nginx内部的upstream模块，在高并发压力下性能不下降，实时动态分流
灰度子系统在对用户请求做一定处理，比如添加uri参数、header头部等后，再转发至目标后端。可以动态设置分流策略，实时生效，无需重启
-->

------------

* 手机微博 HTTP DNS 服务端开发

<!--
	* 手机微博httpdns项目用于解决客户端恶意劫持的问题，并能够实现节点的智能调度。
	* 一期：nginx + edns server松耦合实现
		* 参与开发nginx的http dns模块
		* 对项目进行压测和评估，目前已灰度上线
	* 二期：golang + edns查询
		* 参与golang server的架构设计及系统开发
		* 开发golang 版本的edns查询模块和并发更新模块
			* 全量并发更新域名对应IP的DNS记录，每个域名对应26万段IP，更新时间低于3分钟
	* httpdns server端一期已上线，手机微博客户端采用渠道包灰度；二期已完成edns查询模块。
--> 

#### 极光推送(jpush.cn) 后台开发实习生（ 2013年9月 ~ 2014年5月 ）

* 大规模用户模拟系统

<!--
    * 实习项目，该系统是极光推送后台系统中用于功能和负载测试的子系统，通过模拟真实用户的功能，保持大规模用户在线，统计模拟用户的运行数据，以测试推送系统的推送速度、负载强度和其他情况。
    * 在mentor指导下独立完成。
    * 经过系统设计、功能实现、压力测试及系统调优，单机可实现最高200万TCP长连接，线上运行时单机模拟50万用户，长期稳定运行。

        * linux cpp tcp epoll 多线程及线程间通信
        * 单进程epoll和非阻塞IO实现高并发
        * 多线程分别实现配置上传下发模块、统计模块、上报模块和心跳模块
		* tornado实现管理界面和监控界面
-->

---

项目 演讲和讲义
----------------------

 - [ABTestingGateway](https://github.com/WEIBOMSRE/ABTestingGateway)：手机微博七层动态调度系统中关于灰度发布和动态分流的子项目，780+ stars。
 - 2015年OSC源创会运维专场：[基于动态策略的灰度发布系统](https://github.com/WEIBOMSRE/ABTestingGateway/blob/master/doc/%E5%9F%BA%E4%BA%8E%E5%8A%A8%E6%80%81%E7%AD%96%E7%95%A5%E7%9A%84%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83%E7%B3%BB%E7%BB%9F.pdf)


---

兴趣点
-------------------
* 兴趣广泛，从Linux服务端开发到安卓开发，从单片机开发到FPGA开发，从kernel源码到lua源码，我都有兴趣:)

* 对所使用的软件或服务，愿意并有能力进行深入理解，从设计思路到源码实现着手
	- TCP/ DNS/ HTTP
	- Linux/ Nginx/ Redis/ Lua/ 
* 对动态追踪和性能优化尤其有兴趣
	- linux/nginx的配置参数调优
	- 通过perf或者systemtap分析性能瓶颈
* Linux kernel
	- tcp/ip stack 
	- epoll
	- memory management

---

个人项目
-----------------------

* [smarthome](https://github.com/BG2BKK/smarthome)
	* 基于ESP8266单片机的家庭环境参数监测模块，目前支持温度、湿度、PM2.5等参数
	* 通过wifi上传监测参数和心跳到远程server
	* 通过mqtt进行远程推送和远程控制
	* server端采用nginx+nodejs+redis实现
	* 使用nginx的rtmp模块实现实时视频监控
	* TODO LIST
		* opencv抓取关键帧上报
		* android app简易开发
		* 使用dart 甲醛传感器监测室内甲醛气体

* [MyDict](https://github.com/BG2BKK/MyDict)
	* 一个离线词典
	* 原作者采用trie树对离线词库生成索引，我采用三向单词查找树(Ternary search tries TSTs)实现
	* 数据结构解决实际问题，体现程序员的价值

* [NFA_By_Python](https://github.com/BG2BKK/NFA_by_Python)
	* 《计算理论》专业课作业，内容是将使用NFA解析正则表达式，并将其可视化。
	* python分析正则表达式，自动生成dot代码，通过graphviz画图并生成pdf
	* 这恐怕是我离计算机科学最近的一次了:)
	* 其实想想，计算理论还是非常有趣的，而我也无比热爱这门科学

* [img_process_vhdl](https://github.com/BG2BKK/img_process_vhdl)
	* 纯VHDL写的通用图像处理框架，在FPGA进行图像卷积运算，进而可以实现滤波、开闭等操作
	* 考虑到github上好像逻辑工程师不多，我就不写README了。
	* 以后再也不碰FPGA了，VHDL这种逻辑语言其实没必要人肉来写



