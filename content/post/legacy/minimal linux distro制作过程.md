+++
date = '2016-03-30T15:13:13+08:00'
draft = true
title = 'minimal linux distro制作过程'

+++


github:     https://github.com/ivandavidov/minimal
blog:   http://minimal.linux-bg.org/

 sudo apt-get install isolinux
  sudo apt-get install syslinux syslinux-common syslinux-efi syslinux-utils
  sudo apt-get install syslinux-themes-ubuntu-xenial

7gen iso img时报找不到 isolinux.bin错误，需要事先copy到这里
sudo cp /usr/share/syslinux/themes/ubuntu-xenial/isolinux-live/isolinux.bin /usr/lib/syslinux/isolinux.bin


sh ./qemu64.sh时，报can't load ldlinux.c32, 首先将ldlinux.c32copy到该目录下

sudo cp /usr/share/syslinux/themes/ubuntu-xenial/isolinux-live/ldlinux.c32 /usr/lib/syslinux/

修改kernel的 [arch/x86/boot/Makefile](https://github.com/mhiramat/boot2minc/blob/master/src/patches/kernel/x86-copy-linux-c32-for-newer.patch)
将如下信息

```bash
           if [ -f /usr/$$i/syslinux/ldlinux.c32 ] ; then \
               cp /usr/$$i/syslinux/ldlinux.c32 $(obj)/isoimage ; \
           fi ; \
```

写在

```bash
        if [ -f /usr/$$i/syslinux/isolinux.bin ] ; then \
                    cp /usr/$$i/syslinux/isolinux.bin $(obj)/isoimage ; \
```
之后

在

```bash
break ; \
```

之后

http://www.ruanyifeng.com/blog/2009/10/5_ways_to_search_for_files_using_the_terminal.html


另外一个版本和方法

http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html

https://github.com/SunliyMonkey/tiny_linux
