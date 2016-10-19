+++
date = "2016-10-14T18:18:41+08:00"
draft = true
title = "page cache和buffer cache"

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
