+++
date = "2016-07-15T03:12:18+08:00"
draft = false
title = "BLE 和 NRF51822 开发"

+++

* [nrf51822 sdk document](http://developer.nordicsemi.com/nRF51_SDK/doc/)

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
			* ram
				* start 0x20002000
				* length 0x2000 16KB的话协议栈和应用程序各占8KB，比较寒酸
	* 如果你发现无协议栈的程序无法启动，那么你需要修改查看flash和ram的布局
	* 如果你发现有协议栈的程序无法启动，那么你还需要关注协议栈和应用程序是否和谐相处
		* 如果还无法启动或者不正常运行，那么你的程序可能出现了运行错误比如访问越界内存，但是单片机的话由于没有core dump所以机器崩溃但你并不知道。

* [Jlink](https://www.segger.com/downloads/jlink/JLink_Linux_V610d_x86_64.deb)
	* JLink的Linux版非常方便，感觉比[st-link](https://github.com/texane/stlink)好用一些
	* 如何在一台电脑上搞多个JLink，还需要再学习
	* JLink在Ubuntu上进行带协议栈的调试还需要学习，带协议栈真是个蛋疼的事情

* [nRF5x-Command-Line-Tool](https://www.nordicsemi.com/eng/nordic/Products/nRF51822/nRF5x-Command-Line-Tools-Linux64/51386)
	* nrfjprog烧写芯片工具，应该是基于Jlink的
	* nrfjprog --eraseall 清空芯片
	* nrfjprog --program xyz.hex -f nrf51 --chiperase 烧写程序
	* nrfjprog -r 重启

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



* 例程要对应sdk的版本
	* 每个sdk里有很多例程，都是可以直接用keil、iar和armgcc编译的，后者就是我用的ubuntu
	* 目前我采用的是sdk10，由于sdk10和sdk11在twi，即I2C的驱动上有差别，sdk11不能一次写超过256B字节，导致我的12864的oled不能正常工作，所以采用sdk10
	* 但是我发现sdk11更加成熟些，目前只能暂时这样，解决方法要么是改造sdk11的twi驱动，要么是用GPIO模拟I2C控制oled，要么是等待sdk13发布

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

* twi / I2C
	* ssd1306_128x64_oled
		* ssd1306是比较常用的oled驱动芯片了，各种例程很多，还便宜
		* 可以用I2C和SPI驱动，后者比前者多2根线，所以用I2C了；实际上很多oled都可以通过改led电阻来更改驱动方式
		* ssd1306的话最好能够将u8g的lib移植一下，这个目前还没有做
	* nrf有两个twi，需要在例程中config文件夹里的头文件中将设备使能，置为1

* [adc](http://www.doc00.com/doc/100100a28)
	* 选对管脚
<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/nrf51822_adc_pin.png" ><p>nrf51822 adc的通道和输入管脚的对应</p></div>
		* P0.01到P0.06分别对应通道2至通道7，P0.26和P0.27对应通道0和通道1
		* 我选择通道3，P0.02
	* 选对参考电压
		* 可选择：
			* #define ADC_CONFIG_REFSEL_VBG (0x00UL) 
				* 选择内部稳定的1.2V作为参考电压
			* #define ADC_CONFIG_REFSEL_External (0x01UL) 
				* 采用外部参考电压
			* #define ADC_CONFIG_REFSEL_SupplyOneHalfPrescaling (0x02UL) 
				* 采用外部电压的1/2作为参考电压，一般用于测量1.7V到2.6V
			* #define ADC_CONFIG_REFSEL_SupplyOneThirdPrescaling (0x03UL) 
				* 采用外部电压的1/3作为参考电压，一般用于测量2.5V到3.6V

		* 一般来说内部稳定的1.2V是比较好的选择，我选择ADC_CONFIG_REFSEL_VBG

	* 选对输入比例
		* 可选择：
			* #define ADC_CONFIG_INPSEL_AnalogInputNoPrescaling (0x00UL) 
				* 直接将待检测电压输入
			* #define ADC_CONFIG_INPSEL_AnalogInputTwoThirdsPrescaling (0x01UL) 
				* 测量带检测电压的1/2
			* #define ADC_CONFIG_INPSEL_AnalogInputOneThirdPrescaling (0x02UL) 
				* 测量带检测电压的1/3
			* #define ADC_CONFIG_INPSEL_SupplyTwoThirdsPrescaling (0x05UL) 
			* #define ADC_CONFIG_INPSEL_SupplyOneThirdPrescaling (0x06UL) 
		* 测量电压不能大于参考电压，因此采用1.2V作为参考电压时，只使用测量电压的1/3比较安全
	* adc转换的结果在adc中断里取

	* [ble利用adc实现电量检测服务](http://blog.chinaunix.net/uid-28852942-id-5711152.html)

<div align="center"><img src="https://raw.githubusercontent.com/BG2BKK/githubio/master/static/nrf51822_adc_config.png" ><p>nrf51822 adc的配置寄存器</p></div>

* 疑难杂症
	* [armgcc: region RAM overflowed with stack](https://devzone.nordicsemi.com/question/3771/ld-region-ram-overflowed-with-stack/)
		* 原因是sd+app的ram布局超过了芯片整体的ram大小
		* 其实设置app start=0x20002800 length=0x1800 是刚够0x4000即16KB的，应该不会有这个问题；所以在注释掉<SDK_PATH>/components/toolchain/gcc/nrf5x_common.ld的最后一句ASSERT(__StackLimit >= __HeapLimit, "region RAM overflowed with stack")并编译、烧写是可以正常工作的
		* 内存超过布局是因为还留有一定ram给heap内存了，但是如果没有malloc等动态内存分配函数的话，[是用不到heap内存的，所以将HEAP_SIZE设置为0或者512](https://devzone.nordicsemi.com/question/3771/ld-region-ram-overflowed-with-stack/?answer=3778#post-id-3778)，可以编译通过
			* <SDK_PATH>/components/toolchain/gcc/gcc_startup_nrf51.s
			*     .equ    Heap_Size, 2048	改为 512
		* [另一个方法更优雅一点](https://devzone.nordicsemi.com/question/64564/having-problem-of-region-ram-overflowed-with-stack/)
			* 在Makefile中添加ASMFLAGS += -D__HEAP_SIZE=0
			* 如果是keil的话，Target >> C/C++ >> Preproccesor Symbols 添加 __HEAP_SIZE = 0
		* 参考链接
			* [课外阅读1](https://devzone.nordicsemi.com/question/38781/region-ram-overflowed-with-stack/)
			* [课外阅读2](https://devzone.nordicsemi.com/question/1088/putting-my-app-on-a-ram-memory-diet/)
			* [region ram overflow after upgrading with Segger RTT](https://devzone.nordicsemi.com/question/79873/region-ram-overflowed-with-stack-when-upgrading-segger-rtt/)

```bash	
	Linking target: nrf51422_xxac_s130.out
	/home/huang/workspace/nrf51822/gcc-arm-none-eabi-4_9-2015q3/bin/../lib/gcc/arm-none-eabi/4.9.3/../../../../arm-none-eabi/bin/ld: region RAM overflowed with stack
	collect2: error: ld returned 1 exit status
	Makefile:174: recipe for target 'nrf51422_xxac_s130' failed
	make: *** [nrf51422_xxac_s130] Error 1
```

* [dfu](https://devzone.nordicsemi.com/documentation/nrf51/4.4.1/html/group__bootloader__dfu__description.html)
	* 如何根据官方例程进行DFU升级，可以参考我的[博客 nrf51822 dfu 升级](https://bg2bkk.github.io/post/ble_nrf51822_dfu_%E7%A9%BA%E4%B8%AD%E5%8D%87%E7%BA%A7/)

* [nordic tutorials of bluetooth low energy](https://devzone.nordicsemi.com/tutorials/)
	* [advertising](https://devzone.nordicsemi.com/tutorials/5/)
		* advertising address
			* [gap address type](https://devzone.nordicsemi.com/question/6496/gap-address-types/)
				* Public Address
					* 公共地址
					* 在IEEE 注册
					* 设备地址终身不变
				* [Random Static Address](https://devzone.nordicsemi.com/question/43670/how-to-distinguish-between-random-and-public-gap-addresses/)
					* 可变静态地址
					* 在设备启动时动态设置
					* 设备断电前地址不变
					* 默认设置
				* Private Resolvable Address
					* 这种地址由identity resolving key(IRK)和一个随机数生成
					* 可以更改，甚至可以在一个连接的生命周期内更改，从而避免被位置设备标定和跟踪
					* 设备只能被拥有其下发IRK的设备解析，允许他们识别自己
				* Private Non-Resolvable Address
					* 不常用
		* advertising type
			* non-connectable
				* ibeacon
			* connectable
		* advertising data
			* [限制31bytes](https://devzone.nordicsemi.com/tutorials/5/)
			* 可以在scan response data中再加31bytes，这样在被扫描的时候可以总共发送62bytes的负载数据
	* [service](https://devzone.nordicsemi.com/tutorials/8/ble-services-a-beginners-tutorial/)
		* [the Generic Attribute Profile, GATT](https://devzone.nordicsemi.com/tutorials/8/ble-services-a-beginners-tutorial/)
			* he GATT Profile specifies the structure in which profile data is exchanged. This structure defines basic elements such as services and characteristics, used in a profile.
				* [GATT](https://www.bluetooth.com/specifications/adopted-specifications)指定了描述数据传输的结构体，这个结构体定义描述了基本元素：services服务和characteristics特征。
				* 通俗来讲，GATT是一组描述使用BLE进行数据的绑定、展现和传输的规则
		* Services
			* A service is a collection of data and associated behaviors to accomplish a particular function or feature. [...] A service definition may contain […] mandatory characteristics and optional characteristics.
				* 服务，***The Bluetooth Core Specification*** 定义为，一个服务是数据及数据的关联操作的集合，用于实现特定功能。一个service包括一个必须要有的characteristics和几个可选的characteritics。
				* 换句话说，一个service是信息的集合，比如传感器的值。Bluetooth SIG[预先定义了一些服务](https://developer.bluetooth.org/gatt/services/Pages/ServicesHome.aspx)，比如battery service，比如Heart Rate service，你也可以自定义服务。
		* Characteristics
			* A characteristic is a value used in a service along with properties and configuration information about how the value is accessed and information about how the value is displayed or represented.
				* 特性，官方定义是在一个服务里，所包含的属性值和配置信息，用于确定数据值如何被访问，如何呈现。
				* 换句话说，特征决定了值和信息如何被呈现和访问。
				* Security parameters，units和其他相关元数据也封装在特征中
		* 三者的关系相当于是，在房间里有柜子，柜子里有抽屉。GATT是房间，services是柜子，而特征就是存东西的抽屉。
		* Universally Unique ID, UUID
			* UUID是我们在BLE中经常看到的一个缩写，这是一个独一无二的数字，用来表示一个service、特征或者描述符。通过传输这些ID，peripheral设备可以向centeral设备通知自己提供的服务。
			* 有两类UUID
				* 16-bit UUID
					* 比如预先定义的Heart rate service是0x180D，Heart Rate Measurement characteristic是0x2A37.
					* 16-bit UUID是出于节省内存和能量来设计的，你只能用来表示已经定义好的这些UUID
				* 128-bit UUID
					* 128-bit UUID可以表示自定义的UUID，一般来说可以自定义UUID作为base UUID，比如4A98xxxx-1CC4-E7C1-C757-F1267DD021E8，中间的x表示你可以将service和特征的UUID填充进去
					* nRFgo Studio可以生成UUID
					* 该类随机生成的UUID只有3e~-39的概率是一样的，所以如果是随机生成的，一般不会冲突
	
	* [ble connection parameters](https://devzone.nordicsemi.com/question/60/what-is-connection-parameters/)
		* Connection Interval
		* Slave latency
		* Connection supervision timeout
	* [changing Gatt Characteristic Properties](https://devzone.nordicsemi.com/question/15099/changing-gatt-characteristic-properties/)

