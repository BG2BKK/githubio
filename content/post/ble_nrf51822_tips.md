+++
draft = true
date = "2017-01-30T22:53:22+08:00"
title = "ble_nrf51822_tips"

+++


* 设置广播
    * The advertising module have two different modes you can use for this. ADV_MODE_FAST and ADV_MODE_SLOW, and you can specify the advertising interval you want for each mode. 
    * 可以设置fast模式和slow模式，在fast模式超时后转向slow模式，省电
    * [参考链接](https://devzone.nordicsemi.com/question/111448/can-i-change-nrf5-bluetooth-advertising-interval-dynamically/)
