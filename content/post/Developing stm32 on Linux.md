+++
date = '2016-04-12T10:20:13+08:00'
draft = true
title = 'Developing stm32 on Linux'

+++

* stlink
    * https://github.com/texane/stlink
    * 用于下载调试
    * 将windows keil编译出的elf、bin等文件烧进stm32，正常工作
    * todo: stflash

* https://github.com/rowol/stm32_discovery_arm_gcc
    * gcc to make stm32
    * gcc makefile

* stm32cubemx
    * 用于生成代码
    * 生成代码后报 Gtk-Message: Failed to load module "overlay-scrollbar"
        * 原因：The Message lines mean you are missing the overlay-scrollbar-gtk2 and unity-gtk2-module packages.
        * 解决办法：http://askubuntu.com/questions/453124/gtk-message-and-warnings-in-ubuntu-14-04
        * apt-get install overlay-scrollbar-gtk2
