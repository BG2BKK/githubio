+++
date = "2016-07-06T12:51:05+08:00"
draft = false 
title = "stm32_eclipse_openocd_stlink"

+++



* 插件和eclipse环境配置：
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

* stlink 驱动
	* http://erika.tuxfamily.org/wiki/index.php?title=Tutorial:_STM32_-_Integrated_Debugging_in_Eclipse_using_GNU_toolchain&oldid=5474
	* http://www.st.com/content/st_com/en/products/embedded-software/development-tool-software/stsw-link004.html# 
	* Linux: https://github.com/texane/stlink
	* 用法：http://erika.tuxfamily.org/wiki/index.php?title=Tutorial:_STM32_-_Integrated_Debugging_in_Eclipse_using_GNU_toolchain&oldid=5474
