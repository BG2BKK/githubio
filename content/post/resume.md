+++
date = "2016-04-22T11:25:56+08:00"
draft = false 
title = "resume"

+++

黄振栋简历
================

联系方式
--------------------

- 手机：15010359288
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

工作经历
-----------------

	- 2015.01 ~ NOW		 新浪微博				系统开发工程师 
	- 2013.09 ~ 2014.05	 极光推送				后台开发实习生 

项目经历
-----------------

#### 新浪微博 系统开发工程师 （ 2015年1月 ~ 2016年3月 ）

* 手机微博七层动态调度系统dygateway

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

<!--
修改nginx内部的upstream模块，在高并发压力下性能不下降，实时动态分流
灰度子系统在对用户请求做一定处理，比如添加uri参数、header头部等后，再转发至目标后端。可以动态设置分流策略，实时生效，无需重启
-->

------------

* 手机微博 HTTP DNS 服务端开发
	* 手机微博httpdns项目用于解决客户端恶意劫持的问题，并能够实现节点的智能调度。
	* 一期：nginx + edns server松耦合实现
		* 参与开发nginx的http dns模块
		* 对项目进行压测和评估，目前已灰度上线
	* 二期：golang + edns查询
		* 参与golang server的架构设计及系统开发
		* 开发golang 版本的edns查询模块
	* httpdns server端一期已上线，手机微博客户端采用渠道包灰度；二期已完成edns查询模块。
 
#### 极光推送(jpush.cn) 后台开发实习生（ 2013年9月 ~ 2014年5月 ）

* 大规模用户模拟系统

    * 实习项目，该系统是极光推送后台系统中用于功能和负载测试的子系统，通过模拟真实用户的功能，保持大规模用户在线，统计模拟用户的运行数据，以测试推送系统的推送速度、负载强度和其他情况。
    * 在mentor指导下独立完成。
    * 经过系统设计、功能实现、压力测试及系统调优，单机可实现最高200万TCP长连接，线上运行时单机模拟50万用户，长期稳定运行。

        * linux cpp tcp epoll 多线程及线程间通信
        * 单进程epoll和非阻塞IO实现高并发
        * 多线程分别实现配置上传下发模块、统计模块、上报模块和心跳模块
		* tornado实现管理界面和监控界面

---

技能清单
-------------------
* c/lua/python/golang/shell/vhdl
* nginx/ngx_lua/redis
* linux kernel/ performance profiling
* embedded system && IOT: ble/mqtt/elua
* microprocessor: stm32 51 avr FPGA

---

项目 演讲和讲义
----------------------

 - [ABTestingGateway](https://github.com/WEIBOMSRE/ABTestingGateway)：手机微博七层动态调度系统中关于灰度发布和动态分流的子项目，660+ stars。
 - 2015年OSC源创会运维专场：[基于动态策略的灰度发布系统](https://github.com/WEIBOMSRE/ABTestingGateway/blob/master/doc/%E5%9F%BA%E4%BA%8E%E5%8A%A8%E6%80%81%E7%AD%96%E7%95%A5%E7%9A%84%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83%E7%B3%BB%E7%BB%9F.pdf)

