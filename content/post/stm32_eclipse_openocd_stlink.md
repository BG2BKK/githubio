+++
date = "2016-07-06T12:51:05+08:00"
draft = false 
title = "stm32_eclipse_openocd_stlink"

+++


* eclipse下载
	* neon http://ftp.jaist.ac.jp/pub/eclipse/technology/epp/downloads/release/neon/R/eclipse-jee-neon-R-linux-gtk-x86_64.tar.gz
	* system workbench: 

* stlink 驱动
	* http://erika.tuxfamily.org/wiki/index.php?title=Tutorial:_STM32_-_Integrated_Debugging_in_Eclipse_using_GNU_toolchain&oldid=5474
	* http://www.st.com/content/st_com/en/products/embedded-software/development-tool-software/stsw-link004.html# 
	* Linux: https://github.com/texane/stlink
		* sudo apt-get install autoreconf
		* sudo apt-get install libusb-1.0-0 libusb-1.0-0-dev 
		* make && make -j && make install
		* sudo ./st-util
			* 之前还需要做udev.rules，现在发现不需要
	* 用法：http://erika.tuxfamily.org/wiki/index.php?title=Tutorial:_STM32_-_Integrated_Debugging_in_Eclipse_using_GNU_toolchain&oldid=5474

* arm gcc compiler
	*  sudo apt-get install gcc-arm-none-eabi


* 插件和eclipse环境配置：
	* eclipse cdt 插件
		* https://eclipse.org/cdt/downloads.php
	* http://gnuarmeclipse.sourceforge.net/updates
		* gnu arm eclipse plugins的几种安装方法 http://gnuarmeclipse.github.io/plugins/install/
		* http://gnuarmeclipse.github.io/eclipse/workspace/preferences/
		* http://gnuarmeclipse.github.io/plugins/packs-manager/

	* ac6 system workbench: http://www.ac6-tools.com/Eclipse-updates/org.openstm32.system-workbench.site/ 
		* 有了它就可以ac6 debugger了，但是没办法，neon不支持
		* http://www.emcu.it/STM32/What_should_I_use_to_develop_on_STM32/stm32f0_linux_dvlpt.pdf

* 如何使用eclipse新建工程
	* 可以安装以上两个插件后，从eclipse新建ac6工程，下载相应库即可，ac6保证这个好使；
	* 可以从cube新建工程sw4stm32类型的工程，然后引入SW4STM32工程

* 如何调试工程
	* debugger: AC6		普通的
		* 使用ac6调试, http://www.xlgps.com/article/387805.html

	* debugger: hardware debugger configuration 
		* http://stm32discovery.nano-age.co.uk/open-source-development-with-the-stm32-discovery/getting-hardware-debuging-working-with-eclipse-and-code-sourcey
		* neon还不支持ac6 debugger，所以只能用后者 http://www.openstm32.org/forumthread3023
	* create debugging configuration
		* http://www.openstm32.org/Creating+debug+configuration
		* 开始debug

* st-flash 烧录工具 https://www.youtube.com/watch?v=HKX12hJApZM
* openocd https://www.youtube.com/watch?v=ZeUQXjTg-8c
	* ./configure --enable-verbose --enable-verbose-jtag-io --enable-parport --enable-jlink --enable-ulink --enable-stlink --enable-ti-icdi 
	* make -j && sudo make install
	* openocd -f tcl/board/stm32f4discovery.cfg

* openocd是debug server，3333端口
* eclipse需要debug configuration

* eclipse的设置
	* 代码自动提示：http://blog.csdn.net/u012750578/article/details/16811227

* elua
 
```bash
openocd -f ../../openocd-0.9.0/tcl/board/stm32f429discovery.cfg   -c "init"    -c "reset halt"    -c "sleep 100"    -c "wait_halt 2"    -c "echo \"--- Writing elua_lua_stm32f4discovery.bin\""    -c "flash write_image erase elua_lua_stm32f4discovery.bin 0x08000000"    -c "sleep 100"    -c "echo \"--- Verifying\""    -c "verify_image elua_lua_stm32f4discovery.bin 0x08000000"    -c "sleep 100"    -c "echo \"--- Done\""    -c "resume"    -c "shutdown"

st-flash --reset write elua_lua_stm32f4discovery.bin 0x8000000
```
