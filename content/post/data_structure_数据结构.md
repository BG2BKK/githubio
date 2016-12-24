+++
date = "2016-05-27T11:39:18+08:00"
draft = true
title = "data_structure数据结构"

+++

* [B-tree](http://taop.marchtea.com/03.02.html)

	* B-Tree含有n个节点的话，其高度为lgN，可以在O(lgN)时间内实现插入和删除等操作
	* 一个内节点x如果含有n[x]个关键字，则x将含有n[x]+1个子女
	* 一颗m阶的B-tree
		* 每个节点最多含有m个孩子
		* 除根节点和叶节点外，每个节点最少含有[ceil(m/2)]个孩子
		* 根节点最少有2个孩子，除非该树只有根节点
		* 所有叶子节点都在同一层，叶子节点不包含任何关键字信息，这点与红黑树的nil的语义相同
		* 非叶子节点的信息：a、b、c

* RB-tree
	* http://www.cnblogs.com/CarpenterLee/p/5503882.html
	* http://www.cnblogs.com/CarpenterLee/p/5525688.html

* [lsm-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)
    * [中文](http://www.cnblogs.com/siegfang/archive/2013/01/12/lsm-tree.html)
    * [leveldb](http://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)
