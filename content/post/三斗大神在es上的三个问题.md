+++
date = "2016-08-17T20:00:25+08:00"
draft = true
title = "三斗大神在es上的三个问题"

+++

饶琛琳(251562081)  19:53:00
哈哈。一个是es打开lucene索引的内存消耗较大，所以无法保存较长时间索引。现在是close掉，但是close状态的如果宕机是不会自动恢复的，如果连着挂就彻底废了。所以修改方向是钻研lucene源码，用很小的内存维护索引的打开状态，不用的信息只在真正查询到的时候才动态加载
饶琛琳(251562081)  19:54:52
另一个方向是利用snapshot API导出索引到HDFS上，但是不用restore回来查，而是针对es的snapshot格式开发一个hadoop的inputformat，可以直接在hdfs上查询
饶琛琳(251562081)  19:57:18
第三个是es目前各线程是抢cpu的，如果搜一个特别大的范围的结果，可能这一个请求就hang住整个集群其他所有任务，读写都死了，甚至节点掉线啥的。修改方向两种：利用cgroup机制限制每个search thread的资源占用；或者自动切分大请求成多个小请求。
