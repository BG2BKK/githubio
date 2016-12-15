+++
date = "2016-11-07T15:22:50+08:00"
draft = true
title = "CacheMemory_读书笔记"

+++

Chapter 1
--------------

现代处理器系统中，Cache处于Memory子系统的最顶端，集成在CPU里；一般来说包含L1/L2和L3 Cache，访存时CPU能从Cache中获取数据的话，就不会进行下一级访问，以缩短访问时延。

随着工艺的提高，CPU的主频越来越高，而主存的访问时延却没有相应提高，差距越拉越大，亟需采用高效率Cache Memory来掩盖CPU-Memory之间的latency。

现代处理器的任务执行时间一般包括两部分：

```bash
	Execution Time = (CPU clock cycles + Memory stall cycles) * Clock cycle time
```

缩短访存时延是提高处理器执行效率的关键。

使用Cache提高访存性能的理论依据是程序执行的时间局部性（Temporal Locality）和空间局部性（Spatial Locality），合理安排Cache层次结构比简单增大Cache更有意义。

每次访存的平均时间AMAT计算方式为：

```bash
	Average memory access time = Hit time + Miss rate * Miss Penalty
```

计算AMAT只需要确定hit time、miss rate和miss penalty三个参数即可，而这也并不容易。

### 1.1 Cache 细细看

回到前文说的hit time、miss rate和miss penalty三个参数，探讨准确计算他们需要的注意的因素

* Hit Time

对于L1 Data Cache，原本认为Hit Time可以轻易从intel 手册得出，但是现代处理器大多使用了 Store-Load Forwarding 技术，存储器读时不是从L1 Cache开始的，而是在L1 Cache之前仍然有一段缓存，缓存没来得及提交都Store结果，这个缓存比L1还要接近CPU，也更快一些。

而对于L1 Instruction Cache，它之前也还有一个Line-Fill Buffer，以及其他微结构，计算这个Cache的延时也不容易。

总之，L1 Cache在计算机系统中不是第一级，也不是最快的，考虑到其他微结构的因素，Hit Time并不容易计算

考虑到多核处理器，存储器访问中在自己的L1 Cache没有命中，却在其他CPU的cache中命中，这里存在数据传递问题。如果再考虑到SMP系统间Cache的一致性，问题就更复杂了。

* Miss Rate

* Miss Penalty

此外，虚实地址转换也需要考虑，处理器使用的是Physical Address访问主存。

虚拟化技术的引入，带来了IOMMU和IO虚拟化技术。（IOMMU在驱动开发中也早有实现）

