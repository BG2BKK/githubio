+++
date = "2016-05-18T18:42:22+08:00"
draft = true
title = "kvm qemu Virtual Machine实践"

+++


* [kvm使用](ttps://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/qemu1_%25e4%25bd%25bf%25e7%2594%25a8qemu%25e5%2588%259b%25e5%25bb%25ba%25e8%2599%259a%25e6%258b%259f%25e6%259c%25ba?lang=en)
	* 创建磁盘：qemu-img create -f qcow2 server_01.img 10G
	* 启动虚拟机：qemu-system-x86_64 -m 512 -enable-kvm server_01.img -cdrom ./mini.iso
* [克隆虚拟机](http://blog.csdn.net/csfreebird/article/details/8878808)

* http://blog.csdn.net/csfreebird/article/details/8878808

* http://www.ilanni.com/?p=6173
