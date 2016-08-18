+++
date = "2016-08-15T08:00:14+08:00"
draft = true
title = "raspberry安装ffmpeg"

+++

ffmpeg和opencv是我的老朋友了，读研的时候犯轴，非要自己编译，每次编译好几个小时，在arm板子上。其实我需要的就是个库，先把应用写好，优化的事情可以以后再说，优先级不能变。因此现在我做应用，怎么省事怎么来，这是资源和时间的合理配置。能docker就都docker了，不能的话也要找apt-get，总之时间不能花费在编译这些库上，今天编译，下次还编译，这是很什么效率？

树莓派是有opencv的apt-get软件包的，所以可以直接安装；ffmpeg麻烦点，不过也找到[有人打包好的](https://github.com/ccrisan/motioneye/wiki/Install-On-Raspbian)

[树莓派model 1B+上手](http://www.pcworld.com/article/2598363/how-to-set-up-raspberry-pi-the-little-computer-you-can-cook-into-diy-tech-projects.html)，model 1B+是高于model 1低于model 2的奇葩版本，希望广大网民不要像我这样，买个奇葩的板子哈哈

https://ubuntu-pi-flavour-maker.org/download/
https://wiki.ubuntu.com/ARM/RaspberryPi
