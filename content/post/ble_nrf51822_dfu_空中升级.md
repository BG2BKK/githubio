+++
date = "2016-07-20T15:31:44+08:00"
draft = false
title = "ble_nrf51822_dfu_空中升级"

+++

nrf51822的dfu与别的芯片的空中升级几乎没有差别，以ESP8266为例，本次启动是从启动点bank0启动，空中升级的过程是，从远程网络服务器获取固件并写在bank1中，校验成功后设置下次启动点为bank1，重启后升级成功；nrf51822有点不同的是，空中升级通过ble获取固件并写入flash的bank1后，校验成功会擦除bank0，将bank1内容复制到bank0中，然后重启。

我的芯片是nrf51822 QFAA，16KB Ram，256KB Flash，使用SDK10。nrf51822启动时，bootloader检查是否进入dfu流程中。bootloader采用 SDK_10_PATH/examples/dfu/bootloader/pca10028/dual_bank_ble_s110，该例程启动时读取button4状态，如果被按下，说明需要进入dfu状态，在讯连的板子上没有button4，所以改用button0。

* [Keil MDK实现空中升级](http://blog.csdn.net/a369000753/article/details/51153882)
	* 如上链接所示的过程写的比较详细
		* 准备工作
			* softdevice: SDK_10_PATH/components/softdevice/s110/hex/s110_nrf51_8.0.0_softdevice.hex
			* bootloader: SDK_10_PATH/examples/dfu/bootloader/pca10028/dual_bank_ble_s110
			* ble application: SDK_10_PATH/examples/ble_peripheral/ble_app_uart
			* [nrf-Connect](https://github.com/NordicSemiconductor/Android-nRF-Connect)，主要使用hex2bin.exe和[怎样制作DFU初始化包](https://raw.githubusercontent.com/BG2BKK/githubio/master/static/How%20to%20generate%20the%20INIT%20file%20for%20DFU.pdf)
			* Keil MDK4/5
			* Master Control Panel，采用最新版3.10.0.14
				* 制作升级包

	* 制作bootloader，采用dual_bank_ble_s110
		* dual_bank_ble_s110的flash和ram布局
			* nrf51822 QFAC 32KB Ram 256 KB Flash
				* flash
					* start 0x3C000
					* length 0x3C00
				* ram
					* ram1
						* start 0x20002C00
						* length 0x5380
					* ram2
						* start 0x20007F80
						* length 0x80
						* NoInit
			* nrf51822 QFAA 16KB Ram 256 KB Flash
				* flash
					* start 0x3C000
					* length 0x3C00
				* ram
					* ram1
						* start 0x20002C00 
						* length 0x1380
					* ram2
						* start 0x20001F80
						* length 0x80
						* NoInit
		* 烧写bootloader
			* 首先烧写　softdevice s110，采用nrfgo studio写入s110.hex
				* SDK_10_PATH/components/softdevice/s110/hex/s110_nrf51_8.0.0_softdevice.hex
			* 烧写bootloader
			* 重启，可以在手机的nrf Connect或者nrf Beacon看到DfuTarg，说明bootloader写成功；由于目前只有bootloader没有app，所以bootloader默认进入DFU状态
	
	* 开发应用程序和升级包
		* 应用程序可以在ble_peripheral中找，我习惯用ble_app_uart
		* ble_app_uart的flash和ram布局
			* nrf51822 QFAC 32KB Ram 256 KB Flash
				* flash
					* start 0x18000
					* length 0x28000
				* ram
					* start 0x20002000
					* length 0x4000
			* nrf51822 QFAA 16KB Ram 256 KB Flash
				* flash
					* start 0x18000
					* length 0x28000
				* ram
					* start 0x20002000
					* length 0x2000
		* 生成应用程序bin文件
			* MDK编译生成hex文件，伴有axf文件
			* 生成bin文件的两种方式
				* keil mdk fromelf.exe
					* PATH: ``` C:\Keil\ARM\ARMCC\bin\fromelf.exe```
					* cmd: ``` fromelf.exe --bin --output outfile.bin infile.axf```
				* Android-nRF-Connect的hex2bin方式
					* ``` git clone https://github.com/NordicSemiconductor/Android-nRF-Connect```
					* ``` path/Android-nRF-Connect/init packet handling/hex2bin.exe _build/nrf51422_xxac_s110.hex``` 就可以生成bin文件
				* 推荐方式为后者，并借助keil的设置
					* 在``` option->User->Run User Programs After Buidl/Rebuild``` 的``` run #1``` 中
					* 填写``` "D:\workspace\android\Android-nRF-Connect-master\init packet handling\hex2bin.exe" _build/nrf51422_xxac_s110.hex``` 
					* 每次编译后，生成hex之后会自动调用hex2bin工具生成bin文件
		* 制作升级包
			* 安装Master Control Panel，我采用最新版3.10.0.14
			* 工具： ``` C:\Program Files (x86)\Nordic Semiconductor\Master Control Panel\3.10.0.14\nrf\nrfutil.exe``` 
			* 命令： ``` nrfutil.exe dfu genpkg --application nrf51422_xxac_s110.bin --application-version 0xFFFFFFFF --dev-revision 0xFFFF --dev-type 0xFFFF --sd-req 100 nrf51422_xxac_s110.zip``` 
				* [参数讲解](http://creeus.blogspot.com/2015/09/nrf51822-bootloader.html)
					* ``` --application-version version``` : the version of the application image, for example, 0xff
					* ``` --dev-revision version``` : the revision of the device that should accept the image, for example, 1
					* ``` --dev-type type``` : the type of the device that should accept the image, for example, 1
					* ``` --sd-req sd_list``` : a comma-separated list of FWID values of SoftDevices that are valid to be used with the new image, for example, 0x4f,0x5a
		* 空中升级
			* 将zip文件传到手机
			* 打开nrf Beacon，在dfu tab下select file，选择zip文件
			* 点击select device选择DfuTarg
			* 点击upload开始上传更新包
			* 传输完成后，nrf51822重启，进入ble_app_uart应用中
		* 手动方式
			* 按住button0然后重新启动nrf51822，进入dfu模式
			* 重复空中升级的手机操作

* [Linux下使用armgcc和nrf工具实现DFU升级]
	* 准备工作
		* softdevice: SDK_10_PATH/components/softdevice/s130/hex/s130_nrf51_8.0.0_softdevice.hex
			* 采用s130，而非Keil中用的s110
		* bootloader: SDK_10_PATH/examples/dfu/bootloader/pca10028/dual_bank_ble_s130
		* ble application: SDK_10_PATH/examples/ble_peripheral/ble_app_uart
		* [nRF5x-Command-Line-Tool](https://www.nordicsemi.com/eng/nordic/Products/nRF51822/nRF5x-Command-Line-Tools-Linux64/51386)
		* [nrfutil 0.5.2](https://github.com/NordicSemiconductor/pc-nrfutil/tree/0_5_2)
			* [nrfutil master分支](https://github.com/NordicSemiconductor/pc-nrfutil)介绍sdk11和sdk11之前的版本用nrfutil-0.5.2，之后的sdk用nrfutil高版本，目前是2.0.0版本。
	* 制作bootloader
		* ``` SDK_10_PATH/examples/dfu/bootloader/pca10028/dual_bank_ble_s130/armgcc ```
		* 由于nrf51822 QFAA蛋疼的16KB Ram，所以内存和flash布局要做相应修改，修改 ```SDK_10_PATH/examples/dfu/bootloader/dfu_gcc_nrf51.ld```的相应布局

```bash
MEMORY
{
  /** Flash start address for the bootloader. This setting will also be stored in UICR to allow the
   *  MBR to init the bootloader when starting the system. This value must correspond to 
   *  BOOTLOADER_REGION_START found in dfu_types.h. The system is prevented from starting up if 
   *  those values do not match. The check is performed in main.c, see
   *  APP_ERROR_CHECK_BOOL(*((uint32_t *)NRF_UICR_BOOT_START_ADDRESS) == BOOTLOADER_REGION_START);
   */
  FLASH (rx) : ORIGIN = 0x3C000, LENGTH = 0x3C00

  /** RAM Region for bootloader. This setting is suitable when used with s110, s120, s130, s310. */
  RAM (rwx) :  ORIGIN = 0x20002C00, LENGTH = 0x1380

  /** Location of non initialized RAM. Non initialized RAM is used for exchanging bond information
   *  from application to bootloader when using buttonluss DFU OTA. 
   */
  NOINIT (rwx) :  ORIGIN = 0x20001F80, LENGTH = 0x80

  /** Location of bootloader setting in at the last flash page. */
  BOOTLOADER_SETTINGS (rw) : ORIGIN = 0x0003FC00, LENGTH = 0x0400

  /** Location in UICR where bootloader start address is stored. */
  UICR_BOOTLOADER (r) : ORIGIN = 0x10001014, LENGTH = 0x04
}

```

* <br>
	* <br>
		* 烧写bootloader
			* 方式与Keil烧写顺序一样，由于sdk10提供的bootloader中Makefile缺少``` make flash_softdevice```，因此修改Makefile，添加如下配置
			* make; make flash_softdevice; make flash
			* 可以在nrf Beacon中看到DfuTarg设备

```bash
flash: nrf51422_xxac
	@echo Flashing: $(OUTPUT_BINARY_DIRECTORY)/$<.hex
	nrfjprog --program $(OUTPUT_BINARY_DIRECTORY)/$<.hex -f nrf51  --sectorerase
	nrfjprog --reset

## Flash softdevice
flash_softdevice:
	@echo Flashing: s130_nrf51_1.0.0_softdevice.hex
	nrfjprog --program ../../../../../../components/softdevice/s130/hex/s130_nrf51_1.0.0_softdevice.hex -f nrf51 --chiperase
	nrfjprog --reset

```

* <br>
	* 开发应用程序和制作升级包
		* 其实我觉得，采用armgcc方式开发，比采用Keil方式要简便很多，Keil还要钱。
		* nrfutil-0.5.2
			* git clone https://github.com/NordicSemiconductor/pc-nrfutil -b 0_5_2
			* python setup.py install
		* ble_app_uart
			* /home/huang/workspace/nrf51822/sdk10/examples/ble_peripheral/ble_app_oled/pca10028/s130/armgcc
			* make
			* 在_build/ 中生成nrf51422_xxac_s130.hex和nrf51422_xxac_s130.bin，但是在linux下不需要bin文件，使用hex作为应用程序
		* 制作升级包
			* ``` nrfutil dfu genpkg --application _build/nrf51422_xxac_s130.hex --application-version 0xffffffff --dev-revision 0xffff --dev-type 0xffff --sd-req 0xfffe _build/nrf51422_xxac_s130.zip```
			* 注意观察```--sd-req 0xfff3```，在[文档How to generate the INIT file for DFU.pdf](https://raw.githubusercontent.com/BG2BKK/githubio/master/static/How%20to%20generate%20the%20INIT%20file%20for%20DFU.pdf)中讲到softdevice版本对应的0x5A代表SD 7.1.0,0x4F代表 SD 7.0.0，0x64代表SD 8.0.0，而SDK10中的SD正是8.0.0，但是我怎么试都不好使，只能使用0xFFFE，它可以接受任何版本的softdevice
			* 注意观察```--application _build/nrf51422_xxac_s130.hex```，application采用的是hex文件
			* 可能是nrfutil版本和SDK版本较老的原因吧，很多参数都很不正规，比如application-version，采用的全是FF
	* 空中升级
		* 将zip文件传到手机
		* bootloader烧写进nrf51822后，打开nrf Beacon
		* select file选择zip文件，select device选择DfuTarg设备
		* upload更新程序
	

* 以上就是根据nordic的sdk10提供的官方bootloader例程和ble_app_uart应用程序实现的空中升级，只是一个最简单的实现，还需要多学习和总结
