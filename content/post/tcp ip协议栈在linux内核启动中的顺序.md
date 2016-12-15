+++
date = "2016-02-24T23:13:13+08:00"
draft = false
title = "tcp ip协议栈在linux内核启动中的顺序"

+++

    在我尝试从kernel中深入了解TCP IP协议栈时，遇到了难题。

    我选择的是linux-2.4.20 kernel，理由如下：
    
    1. 首先是<TCP/IP Architecture, Design, And Implementation In Linux>一书采用的是该版本，会有按图索骥的效果。
    2. 第二个理由，正如书中所说的，TCP/IP协议栈在2.4内核中就已经基本成型，而根据我实际对比，2.4.20内核与4.4.1内核在TCP/IP实现的框架上大体是相同的，区别是2.6 kernel以后完全将VFS中的各组件namespace化，另外是一些高版本内核引入的措施（比如对比net/socket.c中sock_create函数）。
    3. 第三个理由是，linux-2.4 kernel的代码还没有开始爆炸，适合初学者入门，也适合我这样学力不足的人。
    
    采用linux-2.4 kernel的不足之处在于，版本较老，想亲自动手实验，需要做一些兼容性的准备。


我搜到了[这样一篇帖子](http://www.skyfree.org/linux/kernel_network/startup.html)， 收益颇深。解决了我的疑问，用我最能接受的方式，先从kernel启动的函数说起，然后调用到我能看到的net/socket.c中的函数；然后又通过修改kernel源码添加标记，打印运行log来标志函数执行；然后通过讲解module_init注册的静态模块是如何加进内核可执行文件里的，然后编译出linux.map文件，进一步确定函数执行顺序。这个方式让我非常容易接受，也很感慨写博客的人功力之深，通篇干货没有废话；更感慨的是，这个帖子写于2001左右，当时进行kernel修改还是比较容易的事情，现在的kernel代码越来越庞大，初学者为此望而却步，很难入手；新人难以入门的问题，近年来也多有讨论。

那我就先把原作者的文章翻译过来，再继续下一步工作吧。

先说结论
--------------------

1. kernel启动时，第一个与network有关的函数是sock_init()，用来向kernel注册sock文件系统并挂载，以及加载其他模块，比如netfilter
1. loopback设备随后被初始化，因为该设备比较简单。***drivers/net/loopback.c***
1. dummy 和 Ethernet 设备随后被初始化
1. TCP/IP协议栈是在inet_init()中初始化的
1. Unix Domain Socket是在af_unix_init()中初始化的。1~5步按时间顺序排列。


Linux Kernel 2.4的入口函数
--------------------------

1.经过基本硬件设置后，启动代码(定义在head.S中)调用 /init/main.c 的 start_kernel()函数

```bash
#arch/i386/kernel/head.S
...
    call SYMBOL_NAME(start_kernel)
...
```

2.sock_init()调用过程，向系统注册sock文件系统并挂载

    sock_init()将向系统注册sock文件系统
    do_initcalls()中循环调用所有MODULE_INIT()的模块，包括系统中的inet_init和af_unix_init，至于如何关联起来的，稍后会有介绍。

```c
asmlinkage void __init start_kernel(void)
{
...
        printk(linux_banner);	// "linux_banner" is defined in init/version.c (W.N.).
...
...     // Dozens of initialize routines
...
        kernel_thread(init, NULL, CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
...
        cpu_idle();
}

static int init(void * unused)
{
...
        do_basic_setup();
...
        execve("/sbin/init",argv_init,envp_init);
...
}

static void __init do_basic_setup(void)
{
...
        sock_init();		// net/socket.c (SEE BELOW)
...
        do_initcalls();
...
}

static void __init do_initcalls(void)
{
        initcall_t *call;

        call = &__initcall_start;
        do {
                (*call)();
                call++;
        } while (call < &__initcall_end);
...
}
```
3.sock_init()的内容
    
    欢迎信息printk()
    清空协议栈数组，此时系统中没有任何协议

```cpp
...

/*
 *      The protocol list. Each protocol is registered in here.
 */

static struct net_proto_family *net_families[NPROTO];	// Current NPROTO is defined as 32
																// in <linux/net.h> (W.N.).
...

void __init sock_init(void)
{
        int i;

        printk(KERN_INFO "Linux NET4.0 for Linux 2.4\n");
        printk(KERN_INFO "Based upon Swansea University Computer Society NET3.039\n");

        /*
         *      Initialize all address (protocol) families.
         */

        #清空所有协议
        for (i = 0; i < NPROTO; i++)
                net_families[i] = NULL;
...
        /*
         *      Initialize the protocols module.
         */

        #注册文件系统并挂载，sock_fs_type之前被初始化
        register_filesystem(&sock_fs_type);
        sock_mnt = kern_mount(&sock_fs_type);

        /* The real protocol initialization is performed when
         *  do_initcalls is run.
         */
...
}
```

4.do_initcalls()中的调用函数

```cpp
#init/main.c

static void __init do_initcalls(void)
{
        initcall_t *call;

        call = &__initcall_start;
        do {
                #添加打印语句，dmesg命令可以输出启动结果
                printk(KERN_INFO "+++ do_initcall: %08X\n", call);	// Dump the entry address of initializer (W.N.).

                (*call)();
                call++;
        } while (call < &__initcall_end);

        /* Make sure there is no pending stuff from the initcall sequence */
        flush_scheduled_tasks();
}
```

同时也修改loopback_init函数
```cpp
#drivers/net/loopback.c

int __init loopback_init(struct net_device *dev)
{
        #添加打印语句
        printk(KERN_INFO "=== Executing loopback_init ===\n");

        dev->mtu                = PAGE_SIZE - LOOPBACK_OVERHEAD;
        dev->hard_start_xmit    = loopback_xmit;
        dev->hard_header        = eth_header;
        dev->hard_header_cache  = eth_header_cache;
        dev->header_cache_update= eth_header_cache_update;
        dev->hard_header_len    = ETH_HLEN;             /* 14                   */
        dev->addr_len           = ETH_ALEN;             /* 6                    */
...
};
```
重新编译内核，替换并重启，dmesg的输出结果为：

```cpp
Linux version 2.4.3 (root@mebius) (gcc version 2.95.3 20010315 (Debian release)) #9 Tue Apr 3 17:37:
44 JST 2001
BIOS-provided physical RAM map:
 BIOS-e820: 0000000000000000 - 000000000009f800 (usable)
 BIOS-e820: 000000000009f800 - 00000000000a0000 (reserved)
 BIOS-e820: 00000000000ebc00 - 0000000000100000 (reserved)
 BIOS-e820: 0000000000100000 - 0000000007ff0000 (usable)
 BIOS-e820: 0000000007ff0000 - 0000000007fffc00 (ACPI data)
 BIOS-e820: 0000000007fffc00 - 0000000008000000 (ACPI NVS)
 BIOS-e820: 00000000fff80000 - 0000000100000000 (reserved)
On node 0 totalpages: 32752
zone(0): 4096 pages.
zone(1): 28656 pages.
zone(2): 0 pages.
Kernel command line: root=/dev/hda1 mem=131008K
Initializing CPU#0
Detected 333.350 MHz processor.
Console: colour VGA+ 80x25
Calibrating delay loop... 665.19 BogoMIPS
Memory: 126564k/131008k available (1076k kernel code, 4056k reserved, 387k data, 184k init, 0k highm
em)
Dentry-cache hash table entries: 16384 (order: 5, 131072 bytes)
Buffer-cache hash table entries: 4096 (order: 2, 16384 bytes)
Page-cache hash table entries: 32768 (order: 5, 131072 bytes)
Inode-cache hash table entries: 8192 (order: 4, 65536 bytes)
CPU: Before vendor init, caps: 0183f9ff 00000000 00000000, vendor = 0
CPU: L1 I cache: 16K, L1 D cache: 16K
CPU: L2 cache: 256K
Intel machine check architecture supported.
Intel machine check reporting enabled on CPU#0.
CPU: After vendor init, caps: 0183f9ff 00000000 00000000 00000000
CPU: After generic, caps: 0183f9ff 00000000 00000000 00000000
CPU: Common caps: 0183f9ff 00000000 00000000 00000000
CPU: Intel Mobile Pentium II stepping 0a
Enabling fast FPU save and restore... done.
Checking 'hlt' instruction... OK.
POSIX conformance testing by UNIFIX
PCI: PCI BIOS revision 2.10 entry at 0xfd9be, last bus=0
PCI: Using configuration type 1
PCI: Probing PCI hardware
PCI: Using IRQ router PIIX [8086/7110] at 00:07.0
  got res[10000000:10000fff] for resource 0 of Ricoh Co Ltd RL5c475
Limiting direct PCI/PCI transfers.

#sock_init()的运行log
Linux NET4.0 for Linux 2.4				// Message from sock_init()
Based upon Swansea University Computer Society NET3.039
#sock_init()运行结束

#do_initcalls()中的每个initcall
+++ do_initcall: C029F4E8				// do_initcalls() START
+++ do_initcall: C029F4EC
+++ do_initcall: C029F4F0				// apm_init() in arch/i386/kernel/kernel.o
apm: BIOS version 1.2 Flags 0x03 (Driver version 1.14)
+++ do_initcall: C029F4F4
+++ do_initcall: C029F4F8
+++ do_initcall: C029F4FC				// kswapd_init() in mm/mm.o
Starting kswapd v1.8
+++ do_initcall: C029F500
+++ do_initcall: C029F504
+++ do_initcall: C029F508
+++ do_initcall: C029F50C
+++ do_initcall: C029F510
+++ do_initcall: C029F514
+++ do_initcall: C029F518
+++ do_initcall: C029F51C
+++ do_initcall: C029F520
+++ do_initcall: C029F524
+++ do_initcall: C029F528				// partition_setup() in fs/fs.o
pty: 256 Unix98 ptys configured
block: queued sectors max/low 84058kB/28019kB, 256 slots per queue
RAMDISK driver initialized: 16 RAM disks of 8000K size 1024 blocksize
Uniform Multi-Platform E-IDE driver Revision: 6.31
ide: Assuming 33MHz system bus speed for PIO modes; override with idebus=xx
PIIX4: IDE controller on PCI bus 00 dev 39
PIIX4: chipset revision 1
PIIX4: not 100% native mode: will probe irqs later
    ide0: BM-DMA at 0xfc90-0xfc97, BIOS settings: hda:DMA, hdb:pio
    ide1: BM-DMA at 0xfc98-0xfc9f, BIOS settings: hdc:pio, hdd:pio
hda: TOSHIBA MK8113MAT, ATA DISK drive
ide0 at 0x1f0-0x1f7,0x3f6 on irq 14
hda: 16006410 sectors (8195 MB), CHS=996/255/63, UDMA(33)
Partition check:
 hda: hda1 hda2 hda3 hda4 < hda5 hda6 hda7 hda8 hda9 hda10 >
Floppy drive(s): fd0 is 1.44M
FDC 0 is a National Semiconductor PC87306

#在loopback_init()中添加printk函数的结果
=== Executing loopback_init ===			// loopback initialization is here!

+++ do_initcall: C029F52C				// ext2_fs() in fs/fs.o
+++ do_initcall: C029F530
+++ do_initcall: C029F534
+++ do_initcall: C029F538
+++ do_initcall: C029F53C
+++ do_initcall: C029F540
loop: loaded (max 8 devices)
+++ do_initcall: C029F544
Serial driver version 5.05 (2000-12-13) with MANY_PORTS SHARE_IRQ SERIAL_PCI enabled
ttyS00 at 0x03f8 (irq = 4) is a 16550A
+++ do_initcall: C029F548				// dummy_init_module() in drivers/net/net.o
+++ do_initcall: C029F54C				// rtl8139_init_module() in drivers/net/net.o
8139too Fast Ethernet driver 0.9.15c loaded
PCI: Found IRQ 9 for device 00:03.0
PCI: The same IRQ used for device 00:07.2
eth0: RealTek RTL8139 Fast Ethernet at 0xc8800c00, 08:00:1f:06:79:20, IRQ 9
eth0:  Identified 8139 chip type 'RTL-8139B'
+++ do_initcall: C029F550
+++ do_initcall: C029F554
+++ do_initcall: C029F558
+++ do_initcall: C029F55C
+++ do_initcall: C029F560

#inet_init()的initcall结果
+++ do_initcall: C029F564				// inet_init() in net/network.o
NET4: Linux TCP/IP 1.0 for NET4.0
IP Protocols: ICMP, UDP, TCP
IP: routing cache hash table of 512 buckets, 4Kbytes
TCP: Hash tables configured (established 8192 bind 8192)

#af_unix_inet()的initcall结果
+++ do_initcall: C029F568				// af_unix_init() in net/network.o
NET4: Unix domain sockets 1.0/SMP for Linux NET4.0.
+++ do_initcall: C029F56C
+++ do_initcall: C029F570
+++ do_initcall: C029F574				// atalk_init() in net/network.o
NET4: AppleTalk 0.18a for Linux NET4.0	// do_initcalls() END
fatfs: bogus cluster size
reiserfs: checking transaction log (device 03:01) ...
Using r5 hash to sort names
ReiserFS version 3.6.25
VFS: Mounted root (reiserfs filesystem) readonly.
Freeing unused kernel memory: 184k freed
Adding Swap: 128516k swap-space (priority -1)
eth0: Setting half-duplex based on auto-negotiated partner ability 0000.
```

5.initcalls的实现机制

首先我们可以看到每个module都有使用module_init宏。

```cpp
#net/ipv4/af_inet.c

static int __init inet_init(void)
{
...
        printk(KERN_INFO "NET4: Linux TCP/IP 1.0 for NET4.0\n");
...
}

module_init(inet_init);
```
__init宏和 module_init宏在 *include/linux/init.h* 中定义

```cpp
#ifndef MODULE
#ifndef __ASSEMBLY__
...
typedef int (*initcall_t)(void);
...
extern initcall_t __initcall_start, __initcall_end;
#define __initcall(fn)                                                          \
        static initcall_t __initcall_##fn __init_call = fn
...
#endif /* __ASSEMBLY__ */

/*
 * Mark functions and data as being only used at initialization
 * or exit time.
 */
#define __init          __attribute__ ((__section__ (".text.init")))
...
#define __init_call     __attribute__ ((unused,__section__ (".initcall.init")))
...
/**
 * module_init() - driver initialization entry point
 * @x: function to be run at kernel boot time or module insertion
 *
 * module_init() will add the driver initialization routine in
 * the "__initcall.int" code segment if the driver is checked as
 * "y" or static, or else it will wrap the driver initialization
 * routine with init_module() which is used by insmod and
 * modprobe when the driver is used as a module.
 */
#define module_init(x)  __initcall(x);
...
#else // MODULE
...
#define __init
...
#define __initcall(fn)
...
#define module_init(x) \
        int init_module(void) __attribute__((alias(#x))); \
        extern inline __init_module_func_t __init_module_inline(void) \
        { return x; }
...
#endif // MODULE
```

    * init.h中的#define MODULE是在Makefile中的 -DMODULE 设置的，表示可以动态添加MODULE
    * 目前 CONFIG_INET (/arch/i386/defconfig) 不是 可选module （M），而是静态编译进内核的（y）。静态模块由init.h中的#ifndef MODULE块预处理，而可动态加载的模块（M）则会调用 #else //MODULE 后的初始化代码
    * 所以经过预编译后，inet_init()函数将由上述代码的#ifndef MODULE 预处理为

```cpp
    #include/linux/init.h
    static int __attribute__ ((__section__ (".text.init"))) inet_init(void)
    {
    ...
            printk(KERN_INFO "NET4: Linux TCP/IP 1.0 for NET4.0\n");
    ...
    }
    
    initcall_t __initcall_inet_init  __attribute__ ((unused,__section__ (".initcall.init"))) = inet_init;
```

* 这个扩展过程意味着：
    * inet_init()函数的代码段text code将编译进kernel可执行文件的***.text.init***段中，这种机制的目的是kernel启动，注册模块后能够释放所占用的内存
    * 预处理后的***__initcall_inet_init***作为inet_init()函数的入口，将被存储在kernel可执行文件的 ***.initcall.init*** 段中。
        * 注意这个宏定义是static类型的，所以我们并不能确定这个宏定义的结果是否在kernel的全局符号表中。（只有全局变量才在符号表中）
* 为了能够一探究竟，***移除该宏定义的static标志***，然后***__initcall_*** ***这些入口就是全局变量了，然后我们就能在内核编译后的符号表中看到这些入口函数。
    * 注意如果这些入口函数不是static作用域后，会导致一些链接错误，原因是命名冲突，比如netfilter中有类似命名
* Let's hack the kernel!!!


5.如何从内部观察linux kernel

    * linux kernel只是一个ELF可执行目标文件，和/bin/ls之类的可执行文件没有区别
    * 所以作为kernel ELF文件，vmlinux可以通过nm、objdump和readelf等工具观察
    * 默认情况下，linux kernel的顶层Makefile编译成功后将生成System.map文件，以方便调试，而这个文件不过是一个符号表。所以我向这个编译添加"--cref -Map linux.map"选项，可以生成一个包含更多信息的符号表

```cpp
#Makefile
#添加 --cref -Map linux.map 选项
vmlinux: $(CONFIGURATION) init/main.o init/version.o linuxsubdirs
        $(LD) $(LINKFLAGS) $(HEAD) init/main.o init/version.o \
                --start-group \
                $(CORE_FILES) \
                $(DRIVERS) \
                $(NETWORKS) \
                $(LIBS) \
                --end-group \
                --cref -Map linux.map \
                -o vmlinux
        $(NM) vmlinux | grep -v '\(compiled\)\|\(\.o$$\)\|\( [aUw] \)\|\(\.\.ng$$\)\|\(LASH[RL]DI\)'
 | sort > System.map
```

    * make vmlinux编译kernle源码，生成vmlinux和linux.map，通过objdump -h vmlinux查看各段信息
    * 可以看到***.text.init***段和***.initcall.init***段

```cpp
#objdump -h vmlinux
vmlinux:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0010bf68  c0100000  c0100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .text.lock    00001130  c020bf68  c020bf68  0010cf68  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .rodata       0004407c  c020d0a0  c020d0a0  0010e0a0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .kstrtab      000062fe  c0251120  c0251120  00152120  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 __ex_table    00001418  c0257420  c0257420  00158420  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 __ksymtab     00001d68  c0258838  c0258838  00159838  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .data         00013abc  c025a5a0  c025a5a0  0015b5a0  2**5
                  CONTENTS, ALLOC, LOAD, DATA
  7 .data.init_task 00002000  c0270000  c0270000  00170000  2**5
                  CONTENTS, ALLOC, LOAD, DATA
  8 .text.init    0000f56c  c0272000  c0272000  00172000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  9 .data.init    0001de60  c0281580  c0281580  00181580  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 10 .setup.init   00000108  c029f3e0  c029f3e0  0019f3e0  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 11 .initcall.init 00000090  c029f4e8  c029f4e8  0019f4e8  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 12 .data.page_aligned 00000800  c02a0000  c02a0000  001a0000  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 13 .data.cacheline_aligned 00001fe0  c02a0800  c02a0800  001a0800  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 14 .bss          0002b3d8  c02a27e0  c02a27e0  001a27e0  2**5
                  ALLOC
 15 .comment      00003bc9  00000000  00000000  001a27e0  2**0
                  CONTENTS, READONLY
 16 .note         00001a90  00000000  00000000  001a63a9  2**0
                  CONTENTS, READONLY
```

    * 如下是linux.map的文件内容
    * __initcall_start 和 __initcall_end 定义了***.initcall.init***段的起始和结束，并出现在do_initcalls()函数中
    * 之前将__initcall()宏的static关键字去掉了，所以__initcall_***这些入口函数地址，比如__initcall_inet_init就全局可见了，我们可以在文件中看到内核启动过程
    * 内核开发者们总是喜欢用grep等工具来找某个函数在哪个文件中，而我们在linux.map中可以看到每个函数在哪个模块中，只需要less linux.map就可以了

```cpp
     0xc029f4e8                __initcall_start=.

.initcall.init  0xc029f4e8       0x90
 *(.initcall.init)
 .initcall.init
                0xc029f4e8        0xc arch/i386/kernel/kernel.o
                0xc029f4e8                __initcall_dmi_scan_machine
                0xc029f4ec                __initcall_cpuid_init
                0xc029f4f0                __initcall_apm_init
 .initcall.init
                0xc029f4f4        0x4 kernel/kernel.o
                0xc029f4f4                __initcall_uid_cache_init
 .initcall.init
                0xc029f4f8        0xc mm/mm.o
                0xc029f4f8                __initcall_kmem_cpucache_init
                0xc029f4fc                __initcall_kswapd_init
                0xc029f500                __initcall_init_shmem_fs
 .initcall.init
                0xc029f504       0x3c fs/fs.o
                0xc029f504                __initcall_bdflush_init
                0xc029f508                __initcall_init_pipe_fs
                0xc029f50c                __initcall_fasync_init
                0xc029f510                __initcall_filelock_init
                0xc029f514                __initcall_dnotify_init
                0xc029f518                __initcall_init_misc_binfmt
                0xc029f51c                __initcall_init_script_binfmt
                0xc029f520                __initcall_init_elf_binfmt
                0xc029f524                __initcall_init_proc_fs
                0xc029f528                __initcall_partition_setup
                0xc029f52c                __initcall_init_ext2_fs
                0xc029f530                __initcall_init_fat_fs
                0xc029f534                __initcall_init_msdos_fs
                0xc029f538                __initcall_init_iso9660_fs
                0xc029f53c                __initcall_init_reiserfs_fs
 .initcall.init
                0xc029f540        0x4 drivers/block/block.o
                0xc029f540                __initcall_loop_init
 .initcall.init
                0xc029f544        0x4 drivers/char/char.o
                0xc029f544                __initcall_rs_init
 .initcall.init
                0xc029f548        0x8 drivers/net/net.o
                0xc029f548                __initcall_dummy_init_module
                0xc029f54c                __initcall_rtl8139_init_module
 .initcall.init
                0xc029f550        0x4 drivers/ide/idedriver.o
                0xc029f550                __initcall_ide_cdrom_init
 .initcall.init
                0xc029f554        0x4 drivers/cdrom/driver.o
                0xc029f554                __initcall_cdrom_init
 .initcall.init
                0xc029f558        0x4 drivers/pci/driver.o
                0xc029f558                __initcall_pci_proc_init
 .initcall.init
                0xc029f55c       0x1c net/network.o
                0xc029f55c                __initcall_p8022_init
                0xc029f560                __initcall_snap_init
                0xc029f564                __initcall_inet_init
                0xc029f568                __initcall_af_unix_init
                0xc029f56c                __initcall_netlink_proto_init
                0xc029f570                __initcall_packet_init
                0xc029f574                __initcall_atalk_init
                0xc029f578                __initcall_end=.
                0xc02a0000                .=ALIGN(0x1000)
                0xc029f578                __init_end=.
                0xc02a0000                .=ALIGN(0x1000)
```

linux.map中的信息可以帮助我们和dmesg的输出信息对照起来，可以看到内核中每个我们感兴趣的静态模块的加载顺序。   
以上工作都是基于linux-2.4内核实现的，新版本内核该如何实现呢？
