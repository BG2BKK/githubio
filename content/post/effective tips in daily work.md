+++
date = "2016-07-24T16:36:33+08:00"
draft = false
title = "effective tips in daily work"

+++

免密登录配置脚本
---------------------

```bash

#!/bin/bash

passwd='xxxxxxx'

pub_key=`cat ~/.ssh/id_rsa.pub`

for i in `cat $1`;do
/usr/bin/expect <<-EOF
set time 30
spawn ssh zhendong.hzd@$i "mkdir -p .ssh/;touch .ssh/authorized_keys;chmod 700 .ssh/authorized_keys;echo \"$pub_key\" >> .ssh/authorized_keys"
expect {
    "*yes/no" { send "yes\r"; exp_continue }
    "*password:" { send "$passwd\r" }
}
interact
expect eof
EOF
done

```

将待登录IP列表写入文件中，更改相应密码配置和用户名，然后执行


ubuntu命令行连接wifi
----------------------
[wifi设置](https://i.cmgine.net/archives/11053.html)

```bash
 cat /etc/network/interfaces

 # interfaces(5) file used by ifup(8) and ifdown(8)
 auto lo
 iface lo inet loopback

# 添加如下，启动时自动启动wlp2s0网卡，dhcp模式，采用wpa_conf配置
 auto wlp2s0
 iface wlp2s0 inet dhcp
 wpa_conf /etc/wpa_supplicant/wpa_supplicant.conf
```

wpa_supplicant.conf内容

```bash
network={
    ssid="wills"
    psk="1234567890"
    priority=1
}
```

连接wifi和密码，priority为优先级，数字越大级别越高

vim和shell的配置方式
--------------------

作为一个懒人，现在自然期待有一键脚本能够将我的shell和vim配置好。github上有很多repo帮助我们实现。

[知乎帮助一](https://www.zhihu.com/question/19989337)
[知乎帮助二](https://www.zhihu.com/question/20151659)
[vim一键配置](https://github.com/spf13/spf13-vim)
[shell && zsh](https://github.com/robbyrussell/oh-my-zsh)
[vim一键配置二](https://github.com/ma6174/vim)

linux下进程的有效用户ID和实际用户ID
-----------------------------------

见[博客](http://www.cpplive.com/html/1352.html)，getuid()

hexdump查看文件指定位置
------------------------

[方法为](http://crazyof.me/blog/archives/254.html)所示，


```bash
hexdump -s OFFSET -l LENGTH FILENAME
```

mac os下virtual box与虚拟机进行网络互通
---------------------------------------

方法见[链接](https://dplord.com/2016/03/10/some-exploration-of-the-virtualbox-on-mac-os/)，需要在virtual box添加host only网卡，并在虚拟机中添加网卡。

普通用户获取linux的root权限
------------------------

* 方法一：sudo sh -c "su"
* 方法二：sudo strace su

改变默认登录的shell
------------------------

使用不同linux发行版或者上不同机器时，发现shell不是我喜欢的bash，而是sh等，这个时候需要修改

* 修改登录shell
	* 方法一：chsh -s /bin/bash
	* 方法二：修改/etc/passwd中的shell设置

[解决shell不显示路径的方法](http://blog.sina.com.cn/s/blog_56aa61930101773n.html)

sar在ubuntu中的配置
-------------------------

```bash
sudo apt-get install sysstat

sudo vim /etc/default/sysstat
" change ENABLED="false" to "true"

sudo vim /etc/cron.d/sysstat
" change 
" 5-55/10 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
" to 
" */2 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
" change the collection interval from every 10 minutes to every 2 minutes.

sudo /etc/init.d/sysstat restart
```

全世界最愚蠢的事情就是，重复做相同的事情，却期待有不同的结果发生
-----------------------------------------------------------------

	Insanity: doing the same thing over and over again and expecting different results.

说实话，在写代码、调代码的时候，一但出现非预期结果，首先检查之前自己的编码和输入，确定后不应该再二次重试，毕竟相同条件下不可能产生不同结果，此时应该将思维跳脱出来，另辟蹊径为好。

ubuntu下解压zip文件出现乱码的解决办法
------------------------------------

[参考链接](https://www.zhihu.com/question/20523036)

由于zip格式中并没有指定编码格式，Windows下生成的zip文件中的编码是GBK/GB2312等，因此，导致这些zip文件在Linux下解压时出现乱码问题，因为Linux下的默认编码是UTF8。

目前网上流传一种unzip -O cp936的方法，但一些unzip是没有-O这个选项的。

亲测好用，不好用的看链接

redis设置和清除密码
---------------------------------

正规项目终于要对redis设置密码了，[如何加密码呢](http://blog.csdn.net/zyz511919766/article/details/42268219)

* 配置文件中添加密码
	* 配置文件中的requirepass配置指令用于配置密码
	* 配置文件中的masterauth用于配置从机登陆主机的密码

* 运行中添加密码
	* config set requirepass PASSWORD

* 运行中删除密码
	* config set requirepass ""

* 获取密码
	* config get requirepass

* 配置从机
	* config set masterauth MASTER_PASSWORD
	* config set requirepass SLAVE_AUTH

epoll是同步非阻塞的
---------------------------------

epoll、select等多路服用IO，将fd加入等待时间的队列中，每隔一段时间去轮询一次，因此是同步的；优点是能够在等待任务的时间里去做别的任务；缺点是任务完成的响应延迟增大，因为每隔一段时间去轮询他们，在时间间隔内任务可能已经完成而等待处理等待了一段时间了。

[参考链接](https://ring0.me/2014/11/sync-async-blocked/)

同步/异步指的是被调用方的通知方式，被调用方完成后，主动通知调用方，还是等待调用方发现。前者是异步，后者是同步。从这里也可以看出，[异步IO通知调用方时，数据已经就绪](https://segmentfault.com/a/1190000003063859)，对于网络IO来说，异步IO已经将数据从内核复制到用户空间了。

阻塞/非阻塞是调用方的等待方式，是一直等待在做的事件完成，还是去做别的事情，等到在做的事件完成后再接着进行处理。前者是阻塞，后者是非阻塞

因此epoll是同步和非阻塞的。


sed合并相邻两行
--------------------------------------------

从redis中取出一个键的所有内容时，比如hgetall，得到的结果并不是排序好的，类似于这样

```bash
11485) "1470347460"
11486) "69"
11487) "1470262350"
11488) "34"
11489) "1470262170"
11490) "68"
11491) "1470242400"
11492) "21"
11493) "1470288030"
11494) "65"
11495) "1470303390"
11496) "54"
11497) "1470205320"
11498) "85"
11499) "1470318330"
11500) "92"
11501) "1470167040"
11502) "1"
11503) "1470281880"
11504) "14"
11505) "1470298140"
11506) "113"

```

该hash的key为unix时间戳，val为数值，如果想手动看分布的话，需要将相邻两行合并然后排序，在此我们[借助sed](http://blog.chinaunix.net/uid-9950859-id-98222.html)

```bash
sed '$!N;s/\n/\t/' filename 
```

redis分析实例中所有key和单个key的内存占用情况
--------------------------------------------

采用[rdb工具](https://github.com/sripathikrishnan/redis-rdb-tools)

```bash
rdb -c memory /path/to/ab-dump.rdb > memory.csv
```

sort进行多重排序
-------------------------

sort和uniq在文本处理，尤其是日志处理中用的较多的工具，记得当年校招时候准备面试，用到这两个命令，惊为天人，非常shock。在日常工作中，用的也非常多。

目前有这样的需求，拿到两列数据，第一列是ip，第二列是访问计数，想看一下分布，要求ip要按文本排序，访问计数按数值排序。

```bash
221.179.175.109	97
221.179.175.164	101
221.179.175.46	8
221.179.175.164	102
221.179.175.178	78
221.179.175.46	9
221.179.175.109	98
221.179.175.178	79
120.239.141.197	96
221.179.175.70	114
218.202.7.121	70
221.179.175.178	80
218.202.7.121	71
221.179.175.134	83
120.239.141.197	97
221.179.175.46	10
221.179.175.178	81
221.179.175.47	24
120.239.141.197	98
221.179.175.70	115
221.179.175.70	116
120.239.141.197	99
221.179.175.134	84

```

[解决方法](http://blog.chinaunix.net/uid-25504049-id-3075909.html)

```bash
sort -t ' ' -k1,1 -k2n,2 data


其中 -t ' ' 指定使用空格分列
-k1,1  指定以第一列为关键字排序
-k2n,2 指定以第二列为关键字做数据排序
```

[也可以](http://blog.csdn.net/dabingxiaozu/article/details/6213851)

```bash
sort -k1 -k2n out
```

shell脚本批量处理文本
-------------------------

善用awk、grep、xargs、bash、sed等工具，可以提高生产力

```bash
grep 'config.config' . -r | awk -F':' '{print $1}' | grep 'lua$' | xargs sed "s/config.config')/config.config').ab/g" -i
```

shell循环

```bash
for j in {a..z}; do echo $j; done
```

redis批量删除key
----------------------

手动清理redis中的key时，很想通过 ***del keys**** 的方式实现批量删除，而redis却没有提供这样的选项，因此需要借助外部工具

* 1、sehll 命令行
	*  redis-cli keys ip:* | xargs redis-cli del

* 2、lua脚本
	* redis-cli eval "redis.call('del', unpack(redis.call('keys','ip*')))" 0  
	* 这种方式受限于lua的unpack函数，一次删除的key不能太多

* 3、借助客户端
	* [php](http://www.redisfans.com/?p=71)
	* [jedis](http://blog.csdn.net/tuposky/article/details/45364107)

unix获取时间戳
----------------------

```bash
date '+%s'

export timestamp=`date '+%s'`; echo $timestamp

# 标准时间格式转unix时间戳
date -d "2011-03-02 15:00" +%s

# unix时间戳转为标准格式
date -d '1970-01-01 UTC 1299049200 seconds'

# 或者

date -d "@1279592730" 

```
[链接](http://blog.csdn.net/hittata/article/details/7640974)中提到了各种各样的格式，以后写shell脚本就不担心时间戳问题了

docker 设置代理下载镜像
--------------------------


[在systemd中设置](http://stackoverflow.com/questions/23111631/cannot-download-docker-images-behind-a-proxy)

1. mkdir /etc/systemd/system/docker.service.d<p>
2. touch /etc/systemd/system/docker.service.d/http-proxy.conf<p>
3. 在文件中添加：<p>[Service]<p>Environment="HTTP_PROXY=http://proxy.example.com:80/"<p>
4. 重启daemon：<p>sudo systemctl daemon-reload<p>
5. 查看设置状态：<p>sudo systemctl show docker --property Environment<p>
6. 重启docker：<p>sudo systemctl restart docker<p>

tested on Ubuntu 16.04

用ps查看进程的执行时间
----------------------------

```bash
ps -eo pid,tty,user,comm,stime,etime | grep main
```

可以打印进程的开始时间和执行时间

vim下以16进制查看文本文件
----------------------------

vim的功能实在是太强大了，[可以以16进制查看文本信息](http://blog.sina.com.cn/s/blog_4c451e0e0100h4nl.html)：

	十六进制显示
	:%!xxd
	正常显示
	:%!xxd -r

Linux获取系统调度时间片长度
---------------------------

```cpp
#define _GNU_SOURCE
#include <sched.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <assert.h>

	int
main(int argc, char *argv[])
{
	int ret, i;  
	struct timespec tp;  
	ret = sched_rr_get_interval(0, &tp);  
	if(ret == -1)  
		printf("sched_rr_get_interval error.\n");  
	printf("The time is %ds:%ldns.\n", (int)tp.tv_sec, tp.tv_nsec);  
	return 0;  
}

```

```bash
$ ./cpu_time_slice.o 
The time is 0s:16000000ns.
```

可见Ubuntu-16.04 64bit的系统进程时间片是16ms


C语言中short、int、long内存占用
--------------------------------

随着工作年限的增加，很多基本功反而落了下来，甚至开始怀疑short等类型的内存占用问题了呵呵。印象里一直记得int和long类型都是4字节大小啊

```cpp
#include <stdio.h>
#include <stdlib.h>

struct test{
	char *ch1;
	int i;
	long ch2;
} aa;

int main()
{
	printf("Linux-64Bit-platform\n");
	printf("	%lu\n", (unsigned long)sizeof(aa));
	printf("	sizeof short : %d\n", sizeof(short));
	printf("	sizeof int   : %d\n", sizeof(int));
	printf("	sizeof long  : %d\n", sizeof(long));
	
}

```

```bash

Linux-64Bit-platform
	24
	sizeof short : 2
	sizeof int   : 4
	sizeof long  : 8

```

```bash

Linux-32Bit-platform
	24
	sizeof short : 2
	sizeof int   : 4
	sizeof long  : 4

```

看来，long和int大小一样已经是32位机器的老黄历了，基础知识还是应该常用常新啊

文件操作的线程安全相关（待续）
------------------------

http://stackoverflow.com/questions/29981050/concurrent-writing-to-a-file

ubuntu关闭键盘和触摸板的方法
---------------------------

家里的猫就是喜欢趴在笔记本键盘上看你干活，我只能再买一个键盘，然后笔记本键盘留给猫大爷了。

然而它还喜欢在键盘上跳舞，这样太影响输入了，只能想办法把笔记本键盘关掉。

在ubuntu下，键盘鼠标触控板都属于xinput设备，可以通过以下命令查看：

```bash

$ xinput  --list
⎡ Virtual core pointer                    	id=2	[master pointer  (3)]
⎜   ↳ Virtual core XTEST pointer              	id=4	[slave  pointer  (2)]
⎜   ↳ SynPS/2 Synaptics TouchPad              	id=16	[slave  pointer  (2)]
⎜   ↳ Rapoo Rapoo Gaming Keyboard             	id=11	[slave  pointer  (2)]
⎜   ↳ RAPOO Rapoo 2.4G Wireless Device        	id=12	[slave  pointer  (2)]
⎜   ↳ Wacom ISDv4 E6 Pen stylus               	id=13	[slave  pointer  (2)]
⎜   ↳ Wacom ISDv4 E6 Finger touch             	id=14	[slave  pointer  (2)]
⎜   ↳ Wacom ISDv4 E6 Pen eraser               	id=18	[slave  pointer  (2)]
⎜   ↳ TPPS/2 IBM TrackPoint                   	id=19	[slave  pointer  (2)]
⎣ Virtual core keyboard                   	id=3	[master keyboard (2)]
    ↳ Virtual core XTEST keyboard             	id=5	[slave  keyboard (3)]
    ↳ Power Button                            	id=6	[slave  keyboard (3)]
    ↳ Video Bus                               	id=7	[slave  keyboard (3)]
    ↳ Sleep Button                            	id=8	[slave  keyboard (3)]
    ↳ Integrated Camera                       	id=9	[slave  keyboard (3)]
    ↳ Rapoo Rapoo Gaming Keyboard             	id=10	[slave  keyboard (3)]
    ↳ AT Translated Set 2 keyboard            	id=15	[slave  keyboard (3)]
    ↳ ThinkPad Extra Buttons                  	id=17	[slave  keyboard (3)]
```

可以看到笔记本键盘是

```bash
    ↳ AT Translated Set 2 keyboard            	id=15	[slave  keyboard (3)]
```

而触控板是

```bash
⎜   ↳ SynPS/2 Synaptics TouchPad              	id=16	[slave  pointer  (2)]
```

他们的id分别是 15和 16，所以采用以下命令关掉就可以

```bash
sudo sudo xinput set-prop 15 "Device Enabled" 0
sudo sudo xinput set-prop 16 "Device Enabled" 0
```

附送shell脚本

```bash
#!/bin/bash

keyboard=`xinput --list | grep AT | awk -F'=' '{print $2}' | awk '{print $1}'`
touchpad=`xinput --list | grep Synaptics | awk -F'=' '{print $2}' | awk '{print $1}'`

function doit() {
	echo '关闭  笔记本键盘'
	`sudo xinput set-prop $keyboard "Device Enabled" 0`			
	echo '关闭  笔记本触摸板'
	`sudo xinput set-prop $touchpad "Device Enabled" 0`
}

doit 
```


小于1024的保留端口都有哪些
--------------------------

我们会遇到如下情况：

```bash
$ sudo tcpdump -i any port 1080
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
15:08:42.421693 IP localhost.55092 > localhost.socks: Flags [.], ack 1960200857, win 342, options [nop,nop,TS val 4687328 ecr 4676064], length 0
```

我想监听1080端口，tcpdump为什么不乖乖显示1080，而是出现个socks呢？（可以通过***-n***参数解决）为什么1080是socks，而不是别的呢？

这是因为低于1024的保留端口大多有自己的名字，他们[由IANA分配](http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)，通常用于系统进程，而我们可以在***/etc/services***文件中找到：

```bash
#
# From ``Assigned Numbers'':
#
#> The Registered Ports are not controlled by the IANA and on most systems
#> can be used by ordinary user processes or programs executed by ordinary
#> users.
#
#> Ports are used in the TCP [45,106] to name the ends of logical
#> connections which carry long term conversations.  For the purpose of
#> providing services to unknown callers, a service contact port is
#> defined.  This list specifies the port used by the server process as its
#> contact port.  While the IANA can not control uses of these ports it
#> does register or list uses of these ports as a convienence to the
#> community.
#
socks		1080/tcp			# socks proxy server
socks		1080/udp
proofd		1093/tcp
proofd		1093/udp
rootd		1094/tcp
rootd		1094/udp
openvpn		1194/tcp
openvpn		1194/udp
rmiregistry	1099/tcp			# Java RMI Registry
rmiregistry	1099/udp
kazaa		1214/tcp
kazaa		1214/udp
nessus		1241/tcp			# Nessus vulnerability
nessus		1241/udp			#  assessment scanner
lotusnote	1352/tcp	lotusnotes	# Lotus Note
lotusnote	1352/udp	lotusnotes
ms-sql-s	1433/tcp			# Microsoft SQL Server
ms-sql-s	1433/udp
ms-sql-m	1434/tcp			# Microsoft SQL Monitor
ms-sql-m	1434/udp
ingreslock	1524/tcp
ingreslock	1524/udp
prospero-np	1525/tcp			# Prospero non-privileged

```

git修改默认分支名
--------------------------
在develop分支改动太大了，导致merge 到master分支时非常被动，这个时候我想，干脆将develop分支作为分支好了。还好碰到[stackoverflow的一个帖子](http://stackoverflow.com/questions/1485578/change-a-git-remote-head-to-point-to-something-besides-master)

* git branch -m master oldmaster
* git branch -m develop master
* git push -f origin master

另一个方法是从github的[项目主页上更改](https://help.github.com/articles/setting-the-default-branch/)

编译openssl 1.0.2g
-----------------------------------------

```bash
./config shared -fPIC zlib-dynamic && make depend -j   && make -j
```

编译nginx/tengine: CPP模块
-----------------------------

```bash
./configure --add-module=../cpp_module  --with-ld-opt="-lstdc++"
```

curl -i 和 -I的区别
----------------------------------

man page:

```bash
	-i, --include
		(HTTP) Include the HTTP-header in the output. The HTTP-header includes things like server-name, date of the document, HTTP-version and more...
		
	-I, --head
		(HTTP/FTP/FILE) Fetch the HTTP-header only! HTTP-servers feature the command HEAD which this uses to get nothing but the header of a document. When used on an FTP or FILE file, curl displays the file size and last modification time only.
```

-i选项会打印出HTTP头部的一些信息，这个选项是curl软件的选项，这些信息本来就是存在的

-I选项会发送HEAD请求，获取信息

linux系统如何将父子进程一起kill掉
---------------------------------

对于普通进程而言，kill掉父进程将会连带着把子进程kill掉；而对于daemon等类型进程而言，kill掉父进程，子进程会被daemon接管，所以如果想父子一起kill掉的话，不能直接kill父进程。

有[两种方法](http://blog.csdn.net/lalaguozhe/article/details/11142855)

* kill -- -PPID
	* PPID前面有***-***号，可以将父子进程kill掉

* 使用exec或者xargs来kill掉他们

dns查询中，域名是否可以有多个cname呢？
-----------------------------------------

不可以
	* http://serverfault.com/questions/574072/can-we-have-multiple-cnames-for-a-single-name

git代理访问
--------------------
git config --global http.proxy 10.8.0.1:8118

ubuntu操作、挂载、格式化SD卡
-----------------------------------

玩树莓派等板子的时候，需要从host机器将os镜像烧进sd卡，然后启动。那么ubuntu如何操作呢？

fdisk -l命令可以用来查看系统中的存储硬件

```bash


Disk /dev/sda: 111.8 GiB, 120034123776 bytes, 234441648 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C27256BB-CE04-48C2-96F4-8F79FAE2AE87

Device     Start       End   Sectors   Size Type
/dev/sda1   2048 234440703 234438656 111.8G Linux filesystem


Disk /dev/sdb: 167.7 GiB, 180045766656 bytes, 351651888 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x42b438a2

Device     Boot     Start       End   Sectors  Size Id Type
/dev/sdb1  *         2048 105887743 105885696 50.5G  7 HPFS/NTFS/exFAT
/dev/sdb2       105887744 187807665  81919922 39.1G 83 Linux
/dev/sdb3       187807744 228767743  40960000 19.5G  7 HPFS/NTFS/exFAT
/dev/sdb4       228769790 351649791 122880002 58.6G  f W95 Ext'd (LBA)
/dev/sdb5       228769792 351649791 122880000 58.6G  7 HPFS/NTFS/exFAT


Disk /dev/sdc: 14.9 GiB, 16021192704 bytes, 31291392 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdc1        8192 31291391 31283200 14.9G  c W95 FAT32 (LBA)

```

如果sd卡（tf卡）通过usb 读卡器接入电脑，则会显示为 /dev/sdc

如果是标准sd卡（大卡），则会显示为 /dev/mmblck0

```bash

Disk /dev/mmcblk0: 14.9 GiB, 16021192704 bytes, 31291392 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device         Boot Start      End  Sectors  Size Id Type
/dev/mmcblk0p1       8192 31291391 31283200 14.9G  c W95 FAT32 (LBA)

```

推荐使用USB读卡器，速度较为快一些。


Lua库文件的加载路径
---------------------------

Lua 提供一个名为 [require](http://www.lua.org/manual/5.1/manual.html#pdf-require) 的函数来加载模块，使用也很简单，它只有一个参数，这个参数就是要指定加载的模块名，[例如](http://dhq.me/lua-learning-notes-package-and-module)：

```lua
require("<模块名>")
-- 或者是
-- require "<模块名>"
```

然后会返回一个由模块常量或函数组成的 table，并且还会定义一个包含该 table 的全局变量。

或者给加载的模块定义一个别名变量，方便调用：

```lua
local m = require("module")
print(m.constant)
m.func3()
```

对于自定义的模块，模块文件不是放在哪个文件目录都行，函数 require 有它自己的文件路径加载策略，它会尝试从 Lua 文件或 C 程序库中加载模块。

require 用于搜索 Lua 文件的路径是存放在全局变量 package.path 中，当 Lua 启动后，会以环境变量 LUA_PATH 的值来初始这个环境变量。如果没有找到该环境变量，则使用一个编译时定义的默认路径来初始化。

```bash

Lua 5.1.5  Copyright (C) 1994-2012 Lua.org, PUC-Rio
>  print(package.path)
~/lua/?.lua;/usr/local/share/lua/5.1/?.lua;/home/huang/workspace/luactor/?.lua;./?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/share/lua/5.1/?/init.lua;/usr/local/lib/lua/5.1/?.lua;/usr/local/lib/lua/5.1/?/init.lua;

```

如果没有 LUA_PATH 这个环境变量，也可以自定义设置

```bash
huang@ThinkPad-X220:~/workspace/luapkg/luasocket-2.0.2$ export LUA_PATH="4;;"
huang@ThinkPad-X220:~/workspace/luapkg/luasocket-2.0.2$ lua
Lua 5.1.5  Copyright (C) 1994-2012 Lua.org, PUC-Rio
>  print(package.path)
4;./?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/share/lua/5.1/?/init.lua;/usr/local/lib/lua/5.1/?.lua;/usr/local/lib/lua/5.1/?/init.lua;
> 
```
可以看到，随便加的环境变量"4;"写在了package.path中。

而为什么4需要两个'；'号呢：文件路径以 ";" 号分隔，最后的 2 个 ";;" 表示新加的路径后面加上原来的默认路径。

```bash
huang@ThinkPad-X220:~/workspace/luapkg/luasocket-2.0.2$ export LUA_PATH="4;"
huang@ThinkPad-X220:~/workspace/luapkg/luasocket-2.0.2$ lua
Lua 5.1.5  Copyright (C) 1994-2012 Lua.org, PUC-Rio
> print(package.path)
4;
> 
```
可见如果只有一个；号，将只采用这个分号。

如果找过目标文件，则会调用 package.loadfile 来加载模块。否则，就会去找 C 程序库。搜索的文件路径是从全局变量 package.cpath 获取，而这个变量则是通过环境变量 LUA_CPATH 来初始。搜索的策略跟上面的一样，只不过现在换成搜索的是 so 或 dll 类型的文件。如果找得到，那么 require 就会通过 package.loadlib 来加载它。


我们也可以在lua代码中[动态修改package.path变量](https://github.com/rtsisyk/luafun)，

```bash
package.path = "../?.lua;"..package.path
require "fun"

```

这点对于我们自己的lua project的设置来说无疑是很方便的。
[参考链接](http://www.runoob.com/lua/lua-modules-packages.html)

cpp调用c函数
--------------------------
由于CPP在链接时与C不太一样，因此在调用C函数时，[需要做一定处理。](http://www.cnblogs.com/skynet/archive/2010/07/10/1774964.html)

将C函数的声明房子 ***#ifdef __cplusplus*** 块中

```bash
#ifdef __cplusplus
extern "C" {
#endif
 
/*.
 * c functions declarations
..*/

#ifdef __cplusplus
}
#endif
```

多少人在猜你机器的密码呢
-----------------------------

VPS在公网就是个待宰的肥肉，都想去登陆，那[都谁猜我的IP了呢？](https://plus.google.com/+AlbertSu2015/posts/Uu1vbeJY1Hw)

```bash
sudo grep "Failed password for root" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr | more
```

grep的简单使用，与 或 非
------------------------------

* 或操作

```bash
grep -E '123|abc' filename  // 找出文件（filename）中包含123或者包含abc的行
egrep '123|abc' filename    // 用egrep同样可以实现
awk '/123|abc/' filename   // awk 的实现方式
```

* 与操作

```bash
grep pattern1 files | grep pattern2 ：显示既匹配 pattern1 又匹配 pattern2 的行。
```

* 其他操作

```bash

grep -i pattern files ：不区分大小写地搜索。默认情况区分大小写，
grep -l pattern files ：只列出匹配的文件名，
grep -L pattern files ：列出不匹配的文件名，
grep -w pattern files ：只匹配整个单词，而不是字符串的一部分（如匹配‘magic’，而不是‘magical’），
grep -v pattern files ：不匹配pattern
grep -C number pattern files ：匹配的上下文分别显示[number]行，
```

iptables的简单使用
-----------------------

其实并不想写iptables相关的内容，因为用的不熟，但是一些常用的命令还是记一下吧

[iptables的详细解释](https://linux.cn/article-1586-1.html)

    Linux系统中,防火墙(Firewall),网址转换(NAT),数据包(package)记录,流量统计,这些功能是由Netfilter子系统所提供的，而iptables是控制Netfilter的工具。iptables将许多复杂的规则组织成成容易控制的方式，以便管理员可以进行分组测试，或关闭、启动某组规则。

```bash
https://blog.phpgao.com/vps_iptables.html
http://www.tabyouto.com/bandwagon-vps-for-shadowsocks-was-hacked.html
http://my.oschina.net/yqc/blog/82111?fromerr=VxVIazGW
http://www.vpser.net/security/linux-iptables.html
```

```bash
# 列出所有规则
iptables -L -n

# 更新iptables规则，规则写在/etc/iptables.rules
iptables-restore < /etc/iptables.rules

# 保存iptables规则，规则写在/etc/iptables.rules
iptables-save > /etc/iptables.rules

```

需要注意的是Debian/Ubuntu上iptables是不会保存规则的。

需要按如下步骤进行，让网卡关闭是保存iptables规则，启动时加载iptables规则：

创建/etc/network/if-post-down.d/iptables 文件，添加如下内容：

```bash
#!/bin/bash
iptables-save > /etc/iptables.rules
```
执行：chmod +x /etc/network/if-post-down.d/iptables 添加执行权限。

创建/etc/network/if-pre-up.d/iptables 文件，添加如下内容：

```bash
#!/bin/bash
iptables-restore < /etc/iptables.rules
```
执行：chmod +x /etc/network/if-pre-up.d/iptables 添加执行权限。

iptables的一些常用规则：

```bash
#允许ping
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
```

如果想清空的话，先执行

	/sbin/iptables -P INPUT ACCEPT

然后执行

	/sbin/iptables -F

VPS简单的ssh登陆设置
-----------------------

初次使用VPS，不懂得安全的重要性，直到扣款时候才心疼，这个时候，弱口令，密码登陆什么的，还是都放弃吧，只用ssh登陆，并且换一个自己的端口。[参考链接](https://imququ.com/post/bandwagon-vps-and-basicly-usage.html)

简单来说，任何一台主机想登陆VPS的主机都需要有本身的ssh公钥私钥

```bash
cd ~/.ssh/
ssh-keygen -t rsa -C "username@gmail.com"
```
然后复制~/.ssh/id_rsa.pub中的内容，就是本机的公钥。

将公钥添加到VPS服务器的/home/username/.ssh/authorized_keys中，本机就能以username用户名登陆VPS了

然后在/etc/ssh/sshd_config中禁用禁用 VPS 的密码登录和 root 帐号登录，将以下两项改为no

```bash
PasswordAuthentication no
PermitRootLogin no

Port 11111

```
随后重启SSH服务

```bash
sudo service ssh restart
```

vim删除空行
------------------------

* 从网页上copy下代码后，发现很多情况下有不想要的空行，非常影响阅读，通过[vim的正则](http://bbs.chinaunix.net/thread-510754-1-1.html)可以解决
    * Delete all blank lines (^ is start of line; \s* is zero or more whitespace characters; $ is end of line) 
    * 删除所有空白行(^是行的开始，\s*是零个或者多个空白字符；$是行尾)

```bash
:g/^\s*$/d
```

ubuntu通过命令设置系统时间
--------------------------------

在嵌入式开发中，在pcduino或者rpi板子上安装好linux后，系统时间是UTC时间1970年，对于有些软件来说可能影响安装，所以需要命令行修改date

```bash
sudo date -s "13 DEC 2015 20:43"
```

ubuntu终端下中文设置
----------------------------

在安装完ubuntu系统后，我们发现中文支持的不好，主要体现在locale的错误，[解决方法：](http://www.linuxidc.com/Linux/2015-08/122501.htm)

```bash
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_PAPER = "zh_CN.UTF-8",
	LC_ADDRESS = "zh_CN.UTF-8",
	LC_MONETARY = "zh_CN.UTF-8",
	LC_NUMERIC = "zh_CN.UTF-8",
	LC_TELEPHONE = "zh_CN.UTF-8",
	LC_IDENTIFICATION = "zh_CN.UTF-8",
	LC_MEASUREMENT = "zh_CN.UTF-8",
	LC_TIME = "zh_CN.UTF-8",
	LC_NAME = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").

```

这是因为中文包没有安装好的缘故，如下命令就可以解决：

```bash
添加简体中文支持
sudo apt-get -y install language-pack-zh-hans language-pack-zh-hans-base

添加繁体中文支持
sudo apt-get -y install language-pack-zh-hant language-pack-zh-hant-base

```

如果还不行，先观察下locale的配置


```bash
huang@localhost:~$ locale
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_MESSAGES to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
LANG=en_US.UTF-8
LANGUAGE=
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC=zh_CN.UTF-8
LC_TIME=zh_CN.UTF-8
LC_COLLATE="en_US.UTF-8"
LC_MONETARY=zh_CN.UTF-8
LC_MESSAGES="en_US.UTF-8"
LC_PAPER=zh_CN.UTF-8
LC_NAME=zh_CN.UTF-8
LC_ADDRESS=zh_CN.UTF-8
LC_TELEPHONE=zh_CN.UTF-8
LC_MEASUREMENT=zh_CN.UTF-8
LC_IDENTIFICATION=zh_CN.UTF-8
LC_ALL=
```
再重新配置下语言包

```bash
huang@localhost:~$  sudo locale-gen "en_US.UTF-8"
Generating locales...
  en_US.UTF-8... done
Generation complete.
huang@localhost:~$ sudo  pip install shadowsocks^C
huang@localhost:~$  sudo locale-gen "zh_CN.UTF-8"
Generating locales...
  zh_CN.UTF-8... done
Generation complete.
huang@localhost:~$ sudo dpkg-reconfigure locales
Generating locales...
  en_US.UTF-8... done
  zh_CN.UTF-8... up-to-date
  zh_HK.UTF-8... done
  zh_SG.UTF-8... done
  zh_TW.UTF-8... done
Generation complete.
```

一般就都能解决

Linux终端下的颜色设置输出
------------------------------

Linux终端下，如果有一个彩色的终端，可以明显提升人的阅读兴趣，通过printf的简单设置即可[实现彩色输出](http://www.w2bc.com/Article/39141)

```bash
\033[显示方式;前景色;背景色m

    显示方式、前景色、背景色至少一个存在即可。
    格式：\033[显示方式;前景色;背景色m
```

```bash
前景色  背景色  颜色
30  40  黑色
31  41  红色
32  42  绿色
33  43  黃色
34  44  蓝色
35  45  紫红色
36  46  青蓝色
37  47  白色


显示方式    意义
0   终端默认设置
1   高亮显示
4   使用下划线
5   闪烁
7   反白显示
8   不可见

```

```bash
\033[1;31;40m    <!--1-高亮显示 31-前景色红色  40-背景色黑色-->
\033[0m          <!--采用终端默认设置，即取消颜色设置-->

printf("\033[1;31;40m");
printf("\033[0m");
```

tsar监控系统负载和nginx运行情况
------------------------------------------
[tsar](https://github.com/alibaba/tsar)是阿里巴巴发布的一款能够实时监控系统状态的命令行工具，并且支持第三方模块扩展，其中比较注明的是nginx模块。使用tsar时，可以将系统负载和nginx运行情况同步同时打出，可以用来定位系统瓶颈，所以广受好评。

***tsar -li1*** 是其最经典的用法，可以将一般我们感兴趣的监控项每秒更新一次并输出

```bash
Time              ---cpu-- ---mem-- ---tcp-- -----traffic---- --sda---  ---load- 
Time                util     util   retran    bytin  bytout     util     load1   
25/03/16-19:03:30   0.08    10.22     0.00     1.4K    1.2K     0.00     0.33  
25/03/16-19:03:31   0.08    10.21     0.00   424.00  468.00     0.00     0.33   
```

如果想使能nginx模块，需要对其进行配置

```bash
1. mkdir /etc/tsar/conf.d
2. touch /etc/tsar/conf.d/nginx.conf

3. 写入如下内容并保存
mod_nginx on

####add it to tsar default output
output_stdio_mod mod_nginx

####add it to center db
#output_db_mod mod_nginx

####add it to nagios send
####set nagios threshold for alert
#output_nagios_mod mod_nginx

#threshold nginx.value1;N;N;N;N;
#threshold nginx.value2;N;N;N;N;
#threshold nginx.value3;N;N;N;N;

表示使能nginx模块，并使用stdio输出

4. tsar -li1

Time              ---cpu-- ---mem-- ---tcp-- -----traffic---- --sda---  ---load- ------------------nginx----------------- 
Time                util     util   retran    bytin  bytout     util     load1      qps      rt  sslqps  spdyps  sslhst   
25/03/16-19:06:19   0.08    11.40     7.14   302.00  546.00     0.00     0.02     1.00    0.00    0.00    0.00    0.00   

```

wrk在CentOS系统上的编译方法
----------------------------------

[wrk](https://github.com/wg/wrk)作为一款可以内嵌lua脚本的，支持多线程的压测工具，受到了广泛欢迎。在高版本CentOS 7上，直接在wrk目录下执行make，可以首先编译deps/luajit，得到deps/luajit/libluajit.a，然而在低版本上，CentOS 6.5系统中，会报一些莫名奇妙的错误。 

解决方法是，查看wrk的Makefile，发现wrk依赖于luajit，那么首先进入deps/luajit编译它，并且是静态编译

```bash
cd wrk
cd deps/luajit
make -j24 BUILDMODE=static

cd ../..
make -j24

```

rpmbuild环境的快速初始化
-----------------------------------

需要将代码打包为CentOS的RPM包时，可以先自己在本地新建一个环境

```bash
1. mkdir -p ~/rpmbuild/{SOURCES,BUILD,BUILDROOT,RPMS,SRPMS,SPECS}
2. 将代打包的代码压缩包 software.tar.gz 放入SOURCES文件夹
3. 将 software.spec 放入SPECS文件夹
4. rpmbuild -ba path/to/software.spec 即可
```

git记住密码，不用每次都输密码才登入
--------------------------------------

git有两种方式，一种是ssh方式，配置公钥私钥，对于新手而言还是比较麻烦的；另一种是http方式，这里有一个办法可以让git记住密码，避免每次都需要输入密码

```bash
1. touch ~/.git-credentials
2. 将  https://{username}:{password}@github.com  写入该文件
3. git config --global credential.helper store  就可以使得git记住密码了
4. 此时查看 ~/.gitconfig，发现多了一项
    
    [credential] 
    helper = store 
```

centos系统上某些软件，比如gcc、python等版本过低的解决方案
----------------------------------------------------------

在CentOS Server上，经常会遇到某些软件依赖版本过低的问题，比如CentOS 6.5的python是2.7版本的，gcc是4.2版本的，那么我们如何获得一个干净的、与原版本无冲突的运行环境呢。CentOS系提供了一个叫SCL的工具，可以帮我们实现目的

```bash
$ sudo wget http://people.centos.org/tru/devtools-1.1/devtools-1.1.repo -P /etc/yum.repos.d
$ sudo sh -c 'echo "enabled=1" >> /etc/yum.repos.d/devtools-1.1.repo'
$ sudo yum install devtoolset-1.1
$ scl enable devtoolset-1.1 bash
$ gcc --version
# 通过devtoolset工具可以暂时提高gcc版本，而不更改之前服务器的配置，这个很有效果，高版本的gcc会智能保留symbol。
```

```bash
# CentOS 6.5
sudo yum install centos-release-SCL
sudo yum install python27
scl enable python27 bash
python --version
```
ubuntu系统上某些软件，比如gcc等版本过高的解决方案
---------------------------------------------------

与CentOS相反，debian系发行版的软件版本都很高，Ubuntu 16.04的gcc 版本已经到了5.2，然而编译一些早期linux内核的话，需要gcc-4.7左右的版本，这时候我们怎么办呢，有两个方法：
* 通过apt安装低版本gcc
    * sudo apt-get install gcc-4.7
    * 在编译linux 内核时， make CC=gcc-4.7 即可
* update-alternatives可以帮忙更改符号链接，指向不同版本的gcc
    * [参考链接1](http://www.metsky.com/archives/607.html)
    * [参考链接2](http://blog.csdn.net/zyxlinux888/article/details/6708775) [附赠](http://blog.csdn.net/zyxlinux888/article/details/6709036)
    


python的matplotlib库实现绘制图标
---------------------------------------

* sudo apt-get install python-matplotlib

[参考链接](http://matplotlib.org/index.html)
[example](http://matplotlib.org/examples/index.html)

python使用requests库发送http请求
--------------------------------------

[参考链接](http://cn.python-requests.org/zh_CN/latest/user/quickstart.html#json)

python解析命令行参数：argparse
-----------------------------------

[参考链接](http://blog.xiayf.cn/2013/03/30/argparse/)

git比较两次commit的差异
----------------------------

通过比较两次commit的代码差异，能够快速理解此次commit的目的，理解作者意图

* git log
    * 查看commit历史

```bash
commit 2279c3f4a8a42e696a0f34e6e9b6289487da92c1
Author: bg2bkk <bg2bkk@gmail.com>
Date:   Sun Mar 13 09:12:26 2016 +0800

    add SO_REUSEADDR和SO_REUSEPORT.md

commit 2b9d85f8427c5ca9e4f9c128c22acd280eb94405
Author: bg2bkk <bg2bkk@gmail.com>
Date:   Sat Mar 12 01:16:00 2016 +0800

    add 采用二级指针实现单链表操作 单链表翻转 删除单链表结点
```

* git diff commit 2279c3f4a8a42e696a0f34e6e9b6289487da92c1 2b9d85f8427c5ca9e4f9c128c22acd280eb94405


git返回强制返回某次提交
----------------------------

* git log
* git reset 5f4769a98985b5acfea45462df27830e51a75145 --hard
    * 可见commit号很重要

iptables允许端口被外网访问
------------------------------

防火墙设置，配置1985端口可以被外网访问 

* sudo iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 1985 -j ACCEPT


tcpdump过滤指定标志的packet
------------------------------

```bash
# tcp包里有个flags字段表示包的类型，tcpdump可以根据该字段抓取相应类型的包：
# tcp[13] 就是 TCP flags (URG,ACK,PSH,RST,SYN,FIN)
# Unskilled 32
# Attackers 16
# Pester     8
# Real       4
# Security   2
# Folks      1

#抓取fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 1 != 0 ' -s0  -w fin.cap -vvv
#抓取syn+fin包：
tcpdump -ni any port 9001 and 'tcp[13] & 3 != 0 ' -s0  -w syn_fin.cap -vvv
#抓取rst包：
tcpdump -ni any port 9001 and 'tcp[13] & 4 != 0 ' -s0  -w rst.cap -vvv
```
[参考链接](http://babyhe.blog.51cto.com/1104064/1395489)


查看进程的内存占用情况
--------------------------

用Ternary Search Tree代替Trie Tree后，我想知道我的进程内存占用有多大区别。

* ps -e -o 'pid,comm,args,pcpu,rsz,vsz,stime,user,uid' | grep MyDict
    * rsz是实际占用内存，单位是KB

* pmap -d pid




