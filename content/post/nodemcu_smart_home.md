+++
date = '2016-07-13T17:03:29+08:00'
draft = true
title = 'nodemcu_smart_home'

+++


* [智能门锁](http://www.myelectronicslab.com/tutorial/door-sensor-with-push-notification-using-esp8266-nodemcu/)
	* [推送服务器instapush](https://instapush.im/)
	* [智能门锁内含推送服务器端php代码](http://www.myelectronicslab.com/wp-content/uploads/2016/04/my-electronics-lab-InstaPushCode.zip)

* [blynk.cc](http://www.blynk.cc/)
	* First drag-n-drop IoT app builder for Arduino, Raspberry Pi, ESP8266, SparkFun boards, and others

* [marsTechHan的天气检测](https://github.com/MarsTechHAN/Weplaio)

* [esp8266开发板的对比](http://frightanic.com/iot/comparison-of-esp8266-nodemcu-development-boards/)

* [mqtt简介](http://dataguild.org/?p=6817)

* [nodemcu的wifi配置经](https://iotbytes.wordpress.com/wifi-configuration-on-nodemcu/)
	* [src](https://github.com/pradeesi/NodeMCU-WiFi) 
	* [topic](https://www.reddit.com/r/esp8266/comments/3ydxnx/cant_get_esp8266_access_point_to_work_with/)

* [esp8266 user guide](https://nurdspace.nl/ESP8266#Technical_Overview)

* [服务器和智能硬件的互相通信方案](http://www.xue163.com/588880/39187/391875076.html)
	* google 'server push message to  nodemcu'

* [物联网的协议和标准](http://postscapes.com/internet-of-things-protocols)
* [websocket官网](http://websocket.org/)

* [mqtt](http://blog.csdn.net/leytton)

* [esp8266的smartconfig](http://nodemcu.readthedocs.io/en/master/en/modules/wifi/#wifistagetap)
	* [示例](http://blog.csdn.net/txf1984/article/details/51188561)
* [例子](http://www.jianshu.com/p/a852d5ca6a44)

* [sming hub](https://github.com/SmingHub)
* [Rock solid esp8266 wifi mqtt, restful client for arduino](http://tuanpm.net/rock-solid-esp8266-wifi-mqtt-restful-client-for-arduino/)
* [8266 mcu](http://blog.nyl.io/esp8266-led-arduino/)

* [esp8266的GPIO](http://www.instructables.com/id/ESP8266-Using-GPIO0-GPIO2-as-inputs/)
	* esp8266模块有17管脚，记做GPIO0 ~ GPIO16
	* 启动时，所有IO口都是输入模式，GPIO0~GPIO15可以配置为INPUT、OUTPUT和INPUT_PULLUP，GPIO16可以配置为INPUT、OUTPUT和INPUT_PULLDOWN_16，GPIO16是一个比较特殊的管脚，可用于睡眠唤醒
	* 小模块，比如esp-01只会留出GPIO0和GPIO2；除了esp-01，其他模块也还留了GPIO15供使用；用户可以通过这些IO来控制模块的使用方式
		* GPIO0和GPIO2共同用于决定启动方式（加上拉电阻）,GPIO0-GPIO2值为H-H时，从spi flash启动，为L-H时，从串口UART启动；而当GPIO15为高时，模块从SD卡启动
	* GPIO6到GPIO11用于esp8266读写flash存储，所以一般不要用他们；写flash的方式，如果是Dual IO，用2根数据线，总共4根线实现读写flash，如果是Quad IO，用4根数据线，总共6根线读写flash；因此一般来说还是不要用GPIO6到GPIO11之间的管脚了。[esp8266有17个管脚，其中6个用于spi flash](http://blog.falafel.com/programming-gpio-on-the-esp8266-with-nodemcu/),如果想使用HSPI，还得借助于GPIO12到GPIO15；
	* GPIO1和GPIO3分别是UART0的Tx和Rx
	* GPIO4、GPIO5、GPIO12、GPIO13、GPIO14可以用作普通IO口
	* GPIO15和GPIO13可以复用为UART0的管脚，用于读取其他设备数据；当SPI flash采用DIO方式时，GPIO9和GPIO10可以作为普通IO口使用；GPIO2可以作为UART1的输出口，一般不怎么用；[其他复用信息可以参考](http://www.esp8266.com/wiki/doku.php?id=esp8266_gpio_pin_allocations)
	* 对于nodemcu而言，它对IO口还有一种编号方式，nodemcu使用Dx，比如D0来标记IO口，这和esp8266的GPIO不一致，具体见表格

	<table>
	<tbody>
	<tr>
	<td width="47"><strong>GPIO</strong></td>
	<td width="47"><strong>ID</strong></td>
	<td width="54"><strong>INPUT</strong></td>
	<td width="66"><strong>OUTPUT</strong></td>
	<td width="68"><strong>PULL-UP</strong></td>
	<td width="72"><strong>PULL-DN</strong></td>
	<td width="270"><strong>Note</strong></td>
	</tr>
	<tr>
	<td width="47">0</td>
	<td width="47">3</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270"></td>
	</tr>
	<tr>
	<td width="47">1</td>
	<td width="47">10</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">The is the TXD0 pin by default</td>
	</tr>
	<tr>
	<td width="47">2</td>
	<td width="47">4</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270"></td>
	</tr>
	<tr>
	<td width="47">3</td>
	<td width="47">9</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">This is the RXD0 pin by default</td>
	</tr>
	<tr>
	<td width="47">4</td>
	<td width="47">2</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">Caution: Sometimes misidentified as #5</td>
	</tr>
	<tr>
	<td width="47">5</td>
	<td width="47">1</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">Caution: Sometimes misidentified as #4</td>
	</tr>
	<tr>
	<td width="47">12</td>
	<td width="47">6</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">This is the HSPI MISO pin</td>
	</tr>
	<tr>
	<td width="47">13</td>
	<td width="47">7</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">This is the HSPI MOSI pin</td>
	</tr>
	<tr>
	<td width="47">14</td>
	<td width="47">5</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">This is the HSPI CLK pin</td>
	</tr>
	<tr>
	<td width="47">15</td>
	<td width="47">8</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68">X</td>
	<td width="72"></td>
	<td width="270">This is the HSPI CSn pin</td>
	</tr>
	<tr>
	<td width="47">16</td>
	<td width="47">0</td>
	<td width="54">X</td>
	<td width="66">X</td>
	<td width="68"></td>
	<td width="72">X</td>
	<td width="270">Belongs to the RTC module, not the general GPIO module, so behaves differently</td>
	</tr>
	</tbody>
	</table>
