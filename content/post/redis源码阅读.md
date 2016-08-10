+++
date = "2016-08-10T15:10:37+08:00"
draft = true
title = "redis源码阅读"

+++


有序集
-----------------------------

[参考链接](http://redisbook.readthedocs.io/en/latest/datatype/sorted_set.html#sorted-set-chapter)

* 编码方式
	* ziplist 编码方式
	* skiplist 编码方式
	* 在redis配置文件中，有如下配置
		* zset-max-ziplist-entries和zset-max-ziplist-value不满足其中之一的时候，采用ziplist编码
		* 否则采用skiplist方式编码

```bash
# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

```

* ziplist编码方式
	* ziplist将节点KV按顺序压缩排序在一块内存，类型为ZIPLIST，查找特定元素时按序查找，时间复杂度为O(N)，增删查等更新操作的时间复杂度都会大于O(N)

* skiplist编码方式
	* 采用skiplist编码方式时，zset定义如下：

```c
/*
 * 有序集
 */
typedef struct zset {

    // 字典
    dict *dict;

    // 跳跃表
    zskiplist *zsl;

} zset;
```
	* zset采用跳表和哈希表共同维护，通过将skiplist和hash set的指针指向同一对象来共享数据
		* 查找时，从hash set中以O(1)的时间复杂度读取数据
		* 查找区间时，从skip list中以O(N)的复杂度获取区间
		* 通过score对key进行定位时，skip list可以以最坏O(N)，期望O(log N)的时间复杂实现
