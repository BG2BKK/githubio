+++
date = '2016-10-14T18:18:41+08:00'
draft = true
title = 'page cache和buffer cache'

+++


* [page cache 和 buffer cache](http://www.cnblogs.com/mydomain/archive/2013/02/24/2924707.html)

* [磁盘的块大小(Block Size)和扇区大小(Sector Size)](http://www.xuebuyuan.com/2073816.html)
	* 查看分区 /dev/sda1 的块大小
		* blockdev --getbsz /dev/sda1
		* 一般是4096，也就是4KB
	* 查看硬盘的扇区大小
		* fdisk -l 可以查看，下表是512B

```bash
Disk /dev/sda1: 111.8 GiB, 120032591872 bytes, 234438656 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

碎碎念
-----------------

* inode结构体中包含文件对应的磁盘快号
* inode中包含对应的存储设备驱动


* page buffer和page cache都是为了处理块设备和内存交互时高速访问的

* page cache面向文件、面向内存，通过inode、address_space和page等数据结构，将一个文件映射到page级别，通过page+offset就可以定位到一个文件的具体位置。
	* kernel抽象了address_space来作为文件系统和页缓存的中间适配器，屏蔽了底层设备的细节问题。

* page buffer用在按块传输的场景

* page cache和page buffer可以集成在一起，属于一个page的块缓存使用buffer_head链表组织起来，page cache维护一个private指针指向buffer_head链表

* inode结构体包含文件的所有的block的块号，通过对文件偏移量offset取模可以定位出该偏移量的块号，然后是磁盘的扇区号；同时对offset取模可以算出其所在的页中的偏移量，
