+++
date = "2016-11-15T03:12:18+08:00"
draft = false
title = "ble_nrf51822"

+++


* [Segger RTT使用](http://blog.csdn.net/a369000753/article/details/51192707)
	* 在sdk的external/segger_rtt文件夹中，记得include和添加c文件
	* 编译方式：
		* armgcc: Makefile中记得使能CFLAGS += -DNRF_LOG_USES_RTT=1 
		* Keil MDK: option中添加 -DNRF_LOG_USES_RTT
	* 使用方式：
		* Keil MDK: 在新版的JLink软件(大于6.0版本)中的J-Link RTT Viewer
		* armgcc: 
			* JLinkRTTClient 打开连接
			* JLinkRTTLogger 连接nrf51822，并将log写入文件中
	* 代码如下

```cpp
#include "SEGGER_RTT.h"
#include "SEGGER_RTT_Conf.h"

main() {
	NRF_LOG_INIT();
	SEGGER_RTT_Init();
	// 两种方式
	SEGGER_RTT_printf(0, "\r\nello,rtt\r\n");
	NRF_LOG_PRINTF("ADC example\r\n");
}

```


* flash和ram布局
	* 在armgcc文件夹里的ld文件，应用程序的ld需要严格按照51822的内存布局来，
	* 我用的是256KB Flash和16KB的Ram，
		* 无协议栈
			* flash 	
				* start 0x00000 
				* length 0x40000 / 256KB
			* ram
				* start 0x20000000 
				* length 0x4000 / 16KB
		* 有协议栈
			* flash
				* start 0x18000 
					* 协议栈一般放在0x0位置，s110是80KB，即0x14000，所以应用程序的start比0x14000大就可以
				* length 0x28000
					* start+length=0x40000，注意不能越界
	* 如果你发现无协议栈的程序无法启动，那么你需要修改查看flash和ram的布局
	* 如果你发现有协议栈的程序无法启动，那么你还需要关注协议栈和应用程序是否和谐相处
		* 如果还无法启动或者不正常运行，那么你的程序可能出现了运行错误比如访问越界内存，但是单片机的话由于没有core dump所以机器崩溃但你并不知道。

* 例程要对应sdk的版本
	* 每个sdk里有很多例程，都是可以直接用keil、iar和armgcc编译的，后者就是我用的ubuntu
	* 目前我采用的是sdk10，由于sdk10和sdk11在twi，即I2C的驱动上有差别，sdk11不能一次写超过256B字节，导致我的12864的oled不能正常工作，所以采用sdk10
	* 但是我发现sdk11更加成熟些，目前只能暂时这样，解决方法要么是改造sdk11的twi驱动，要么是用GPIO模拟I2C控制oled，要么是等待sdk13发布

* ssd1306_128x64_oled
	* ssd1306是比较常用的oled驱动芯片了，各种例程很多，还便宜
	* 可以用I2C和SPI驱动，后者比前者多2根线，所以用I2C了；实际上很多oled都可以通过改led电阻来更改驱动方式
	* ssd1306的话最好能够将u8g的lib移植一下，这个目前还没有做

* [nRF5x-Command-Line-Tool](https://www.nordicsemi.com/eng/nordic/Products/nRF51822/nRF5x-Command-Line-Tools-Linux64/51386)
	* nrfjprog烧写芯片工具，应该是基于Jlink的
	* nrfjprog --eraseall 清空芯片
	* nrfjprog --program xyz.hex -f nrf51 --chiperase 烧写程序
	* nrfjprog -r 重启

* uart
	* nrf51822的uart驱动使用起来比较有意思
		* 一般我们采用115200波特率，所以需要修改
		* 一般我们不用流控，所以将APP_UART_FLOW_CONTROL_ENABLED改为APP_UART_FLOW_CONTROL_DISABLED
		* printf的话，Keil的microlib勾选上来；armgcc的话可以直接开搞。

```cpp
    uint32_t err_code;
    const app_uart_comm_params_t comm_params =
      {
          RX_PIN_NUMBER,
          TX_PIN_NUMBER,
          RTS_PIN_NUMBER,
          CTS_PIN_NUMBER,
          APP_UART_FLOW_CONTROL_DISABLED,
          false,
          UART_BAUDRATE_BAUDRATE_Baud115200
      };

    APP_UART_FIFO_INIT(&comm_params,
                         UART_RX_BUF_SIZE,
                         UART_TX_BUF_SIZE,
                         uart_error_handle,
                         APP_IRQ_PRIORITY_LOW,
                         err_code);

    APP_ERROR_CHECK(err_code);
```

* adc

