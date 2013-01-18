# 原生模式编程 #

从[gihub](https://github.com/RT-Thread/ART)上[下载基本工程](https://github.com/RT-Thread/ART/tree/master/software/basic)，这是一个实现最基本功能的工程，对于想接触RT-Thread编程的爱好者，这个是最基本也是最容易入门的工程。

如果喜欢使用Keil MDK，可以通过如下方式转换出Keil MDK的工程文件。

	scons --target=mdk -s

如果喜欢iar，可以通过如下方式获得工程文件

	scons --target=iar -s

这个工程包括的功能：

- 基本的内核
- 基本的命令行
- 如果使用USB连接电脑，可以使用USB虚拟的串口
- 如果使用真实串口，可以使用TTL线连接RX0, TX0

这里推荐使用USB线直接连接方式（但使用USB方式会忽略掉启动时信息。以及在系统当机时也不能输出出错信息）。

ART板上也包括一个SWD调试接口，可用于连接支持swd的仿真器，例如常用的ulink2, jlink等仿真器。当使用仿真器连接swd时，可以配合Keil MDK或IAR等工具进行源代码级别的单步调试。

ART swd连接20 pin常规JTAG如下图所示:

![SWD](figures/swd.png)

请注意图中的20 pin JTAG缺口是朝上的。

## 工程 ##

假设使用Keil MDK来进行工程的编译和调试，请在rtconfig.py中修改交叉工具链为keil，如下所示：

	PLATFORM 	= 'armcc'

然后在命令行下执行：

	scons --target=mdk -s

正确的情况下，会在basic目录下生成project.uvproj文件。可以用Keil MDK直接打开这个工程文件。打开后工程会有几个默认的组（Group）：

![SWD](figures/keil_group.png)

其中每个组的对应情况：

* Applications - 用户自己的应用程序；
* Drivers - RT-Thread在ART上基本的驱动程序；
* STM32_StdPeriph - ST的STM32固件库；
* Kernel - RT-Thread内核代码；
* CORTEX-M4 - RT-Thread在ARM Cortex-M4上的移植；
* DeviceDrivers - RT-Thread的设备驱动框架；
* finsh - RT-Thread的finsh shell；
* Components - RT-Thread的组件管理器，当前主要用于RT-Thread组件的初始化；

## 用户应用 ##

在RT-Thread中，通常会提供一个applications.c文件，在这个文件中初始化用户自己的应用（组件初始化也会在这里调用），例如这个工程中的applications.c文件：

	#include "stm32f4xx.h"
	#include <board.h>
	#include <rtthread.h>

	#include <components.h>
	static void thread_entry(void* parameter)
	{
		rt_components_init();

	#ifdef RT_USING_USB_DEVICE
		/* usb device controller driver initilize */
		rt_hw_usbd_init();

		rt_usb_device_init("usbd");

		rt_usb_vcom_init();

	#ifdef RT_USING_CONSOLE
		rt_console_set_device("vcom");
	#endif
	#ifdef RT_USING_FINSH
		finsh_set_device("vcom");
	#endif
	#endif
	}

	int rt_application_init(void)
	{
		rt_thread_t tid;

		tid = rt_thread_create("init",
				thread_entry, RT_NULL,
				2048, RT_THREAD_PRIORITY_MAX/3, 20);

		if (tid != RT_NULL)
		    rt_thread_startup(tid);

		return 0;
	}

函数：rt_application_init是用户任务的入口点，在这里创建了一个初始化任务用于多任务级别的初始化，其入口函数是：thread_entry。在这个任务中，它调用了rt_components_init函数以初始化系统的组件（在这个工程中用到了finsh shell以及usb device stack，所以这两个组件会分别在rt_components_init函数中进行初始化）。

## 编译运行 ##

使用生成的Keil MDK工程，我们可以使用Keil MDK自己携带的编译器进行编译、下载：

![Build](figures/keil_build.jpg)

编译完成后，应该会在Keil MDK的输出窗口中显示成功：

![BuildOk](figures/keil_buildok.jpg)

编译完成后可以下载到ART板子上，同时可以使用PuTTY工具打开ART的虚拟USB串口，进入到RT-Thread的命令行状态。
