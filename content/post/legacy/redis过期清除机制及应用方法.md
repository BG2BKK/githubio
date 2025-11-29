+++
date = '2016-08-01T13:26:07+08:00'
draft = false
title = 'redis过期清除机制及应用方法'

+++


redis过期清除和淘汰机制
---------------------------

* 过期时间设置
	* expire key seconds
	* 该命令设置指定key超时的秒数，超过该时间后，可以将被删除
	* 在超时之前，如果该key被修改，与之关联的超时将被移除
		* persist key 持久化该key，超时时间移除
		* set key newvalue 设置新值，会清除过期时间
		* del key	显然会清除过期时间
		* 例外情况：
			* lpush, zset, incr等操作，在高版本（2.1.3++）之后不会清除过期时间，毕竟修改的不是key本身
			* rename 也不会清除过期时间，只是改key名字

* 过期处理
	* redis对过期key采用lazy expiration方式，在访问key的时候才判定该key是否过期
	* 此外，每秒还会抽取volatile keys进行抽样，处理删除过期键

* [过期键删除策略种类](http://www.marser.cn/archives/87/)
	* 事件删除
		* 每个键都有一个定时器，到期时触发处理事件，在事件中删除
		* 缺点是需要为每个key维护定时器，key的量大时，cpu消耗较大
	* 惰性删除
		* 每次访问时才检查，如果没过期，正常返回，否则删除该键并返回空
	* 定期删除
		* 每隔一段时间，检查所有设置了过期时间的key，删除已过期的键
	
	* redis采用后两种结合的方式
		* 读写一个key时，触发惰性删除策略
		* 惰性删除策略不能及时处理冷数据，因此redis会定期主动淘汰一批已过期的key
		* 内存超过maxmemory时，触发主动清理

* http://blueswind8306.iteye.com/blog/2240088
* http://www.cnblogs.com/chenpingzhao/p/5022467.html
