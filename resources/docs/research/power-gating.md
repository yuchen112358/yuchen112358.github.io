## Android平台下使用内核模块关闭某些CPU核
[TOC "float:right"]  

#### 1\. 查下CPU支持哪些governor的模式  
```Bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
```
Odroid-XU3平台下的输出：
```
userspace powersave interactive performance
```
注释：performance表示不降频，powersvae表示省电模式，通常是在最低频率下运行，userspace表示用户模式，在此模式下允许其他用户程序调节CPU频率。另外，也存在ondemand模式（Odroid-XU3平台下无此模式），表示使用内核提供的功能，可以动态调节频率。
#### 2\. 将模式调整为"powersvae"
```Bash
echo "powersvae" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
**此处必须要做，否则默认是`interactive`，该模式下，CPU虽然也可以被关闭，但是关闭后即立刻重新开启。猜测，这应该是`interactive` Governor导致的。**
#### 3\. cpu_up()和cpu_down()内核函数
cpu_up()和cpu_down()内核函数定义在kernel/cpu.c文件中，当在编译配置文件进行了如下配置时
```Bash
CONFIG_HOTPLUG_CPU=y
```
cpu_up()和cpu_down()内核函数会被导出到内核符号表中，如下所示：
```
linux$ nm vmlinux | grep "cpu_up"
c061e628 T __cpu_up
c061ef1c t _cpu_up
c061f080 T cpu_up
c001efb0 T exynos5422_cpu_up
c082bf87 r __kstrtab_cpu_up
c0826adc r __ksymtab_cpu_up
c061f668 t workqueue_cpu_up_callback
c099f5dc d workqueue_cpu_up_callback_nb.26011
```
```
linux$ nm vmlinux | grep "cpu_down"
c061d164 t _cpu_down
c061d3c4 T cpu_down
c001ece8 T exynos5422_cpu_down
c082bf8e r __kstrtab_cpu_down
c081fffc r __ksymtab_cpu_down
c061d10c t take_cpu_down
c061f9a8 t workqueue_cpu_down_callback
c099f5e8 d workqueue_cpu_down_callback_nb.26012
```

#### 4\. 使用cpu_up()和cpu_down()内核函数打开和关闭CPU核
内核模块代码如下所示：
```Swift
#include <linux/module.h>
#include <linux/kallsyms.h>
#include <linux/init.h>
#define cpu_up 0xc061f080
#define cpu_down 0xc061d3c4

typedef int (*CpuFun)(unsigned int);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Open/Close CPU");
MODULE_AUTHOR("Wang Zhen");

int init(void)
{
	int res0 = -1000;
	int res1 = -1000;
	int res2 = -1000;
	int res3 = -1000;
	int res4 = -1000;
	int res5 = -1000;
	int res6 = -1000;
	int res7 = -1000;
	printk(KERN_INFO "init module\n");
	res0 = ((CpuFun)cpu_down)(0);
	printk(KERN_INFO "result0=%d\n",res0);//res0=-1,因为cpu0是不能关闭的
	res1 = ((CpuFun)cpu_down)(1);
	printk(KERN_INFO "result1=%d\n",res1);
	res2 = ((CpuFun)cpu_down)(2);
	printk(KERN_INFO "result2=%d\n",res2);
	res3 = ((CpuFun)cpu_down)(3);
	printk(KERN_INFO "result3=%d\n",res3);
	res4 = ((CpuFun)cpu_down)(4);
	printk(KERN_INFO "result4=%d\n",res4);
	res5 = ((CpuFun)cpu_down)(5);
	printk(KERN_INFO "result5=%d\n",res5);
	res6 = ((CpuFun)cpu_down)(6);
	printk(KERN_INFO "result6=%d\n",res6);
	res7 = ((CpuFun)cpu_down)(7);
	printk(KERN_INFO "result7=%d\n",res7);
	return 0;
}

void cleanup(void)
{
	int res0 = -1000;
	int res1 = -1000;
	int res2 = -1000;
	int res3 = -1000;
	int res4 = -1000;
	int res5 = -1000;
	int res6 = -1000;
	int res7 = -1000;
	printk(KERN_INFO "cleanup module\n");
	res0 = ((CpuFun)cpu_up)(0);
	printk(KERN_INFO "result0=%d\n",res0);
	res1 = ((CpuFun)cpu_up)(1);
	printk(KERN_INFO "result1=%d\n",res1);
	res2 = ((CpuFun)cpu_up)(2);
	printk(KERN_INFO "result2=%d\n",res2);
	res3 = ((CpuFun)cpu_up)(3);
	printk(KERN_INFO "result3=%d\n",res3);
	res4 = ((CpuFun)cpu_up)(4);
	printk(KERN_INFO "result4=%d\n",res4);
	res5 = ((CpuFun)cpu_up)(5);
	printk(KERN_INFO "result5=%d\n",res5);
	res6 = ((CpuFun)cpu_up)(6);
	printk(KERN_INFO "result6=%d\n",res6);
	res7 = ((CpuFun)cpu_up)(7);
	printk(KERN_INFO "result7=%d\n",res7);
}
module_init(init);
module_exit(cleanup);
```
Makefile文件如下所示：
```Makefile
TARGET := power_gating
obj-m := $(TARGET).o
KDIR := /work/Odroid/AndroidSRC/kernel/linux
PWD := $(shell pwd)

CC = arm-eabi-gcc

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	rm -f *.o *.ko *.mod.c *.mod.o modules.order Module.symvers
```
#### 5\. 将生成的内核模块power_gating.ko导入的ODROIDXU3
```
adb push power_gating.ko /sdcard/power_gating.ko
```

#### 6\. 在ODROIDXU3平台下，使用`insmod power_gating.ko`命令插入内核模块
运行结果如下：
```
[ 2155.550251] [c6] init module
[ 2155.551698] [c6] MobiCore mcd: Cpu 0 is going to die
[ 2155.560815] [c6] result0=-1
[ 2155.562146] [c6] MobiCore mcd: Cpu 1 is going to die
[ 2155.568505] [c1] IRQ153 no longer affine to CPU1
[ 2155.569553] [c6] CPU1: shutdown
[ 2155.576465] [c6] MobiCore mcd: Cpu 1 is dead
[ 2155.580176] [c6] result1=0
[ 2155.582858] [c6] MobiCore mcd: Cpu 2 is going to die
[ 2155.588851] [c2] IRQ154 no longer affine to CPU2
[ 2155.589855] [c6] CPU2: shutdown
[ 2155.596891] [c6] MobiCore mcd: Cpu 2 is dead
[ 2155.600442] [c6] result2=0
[ 2155.603155] [c6] MobiCore mcd: Cpu 3 is going to die
[ 2155.609042] [c3] IRQ155 no longer affine to CPU3
[ 2155.610056] [c6] CPU3: shutdown
[ 2155.616819] [c6] MobiCore mcd: Cpu 3 is dead
[ 2155.620669] [c6] result3=0
[ 2155.623424] [c6] MobiCore mcd: Cpu 4 is going to die
[ 2155.631223] [c4] IRQ160 no longer affine to CPU4
[ 2155.631623] [c7] CPU4: shutdown
[ 2155.638372] [c7] MobiCore mcd: Cpu 4 is dead
[ 2155.642167] [c7] result4=0
[ 2155.644875] [c7] MobiCore mcd: Cpu 5 is going to die                         
[ 2155.650948] [c5] IRQ161 no longer affine to CPU5                             
[ 2155.651377] [c7] CPU5: shutdown                                              
[ 2155.658215] [c7] MobiCore mcd: Cpu 5 is dead                                 
[ 2155.661934] [c7] result5=0                                                   
[ 2155.664642] [c7] MobiCore mcd: Cpu 6 is going to die                         
[ 2155.675395] [c6] IRQ162 no longer affine to CPU6                             
[ 2155.675717] [c7] CPU6: shutdown                                              
[ 2155.682434] [c7] MobiCore mcd: Cpu 6 is dead                                 
[ 2155.686257] [c7] result6=0                                                   
[ 2155.689024] [c7] MobiCore mcd: Cpu 7 is going to die                         
[ 2155.717820] [c7] IRQ163 no longer affine to CPU7                             
[ 2155.718239] [c0] CPU7: shutdown                                              
[ 2155.731991] [c0] MobiCore mcd: Cpu 7 is dead                                 
[ 2155.739733] [c0] result7=0  
```
使用`cat /sys/devices/system/cpu/online`命令查看当前运行的CPU核编号，结果如下：
```
# cat /sys/devices/system/cpu/online
0
```
可见，1-7核已经被关闭了，cpu0是不能被关闭的。
#### 7\. 使用`rmmod power_gating`命令卸载内核模块
运行结果如下：
```
[ 2466.152384] [c0] cleanup module                                              
[ 2466.154093] [c0] result0=-22                                                 
[ 2466.158325] [c1] CPU1: Booted secondary processor                            
[ 2466.167427] [c0] result1=0                                                   
[ 2466.170052] [c2] CPU2: Booted secondary processor                            
[ 2466.174446] [c0] result2=0                                                   
[ 2466.178259] [c3] CPU3: Booted secondary processor                            
[ 2466.182384] [c0] result3=0                                                   
[ 2466.185718] [c4] CPU4: Booted secondary processor                            
[ 2466.192509] [c4] result4=0                                                   
[ 2466.194006] [c5] CPU5: Booted secondary processor                            
[ 2466.201218] [c4] result5=0                                                   
[ 2466.203939] [c6] CPU6: Booted secondary processor                            
[ 2466.208741] [c4] result6=0                                                   
[ 2466.211407] [c7] CPU7: Booted secondary processor                            
[ 2466.216316] [c4] result7=0                             
```
使用`cat /sys/devices/system/cpu/online`命令查看当前运行的CPU核编号，结果如下：
```
# cat /sys/devices/system/cpu/online                          
0-7
```
可见，1-7核已经被打开了。因为cpu0在初始化内核模块时并没有被关闭（((CpuFun)cpu_down)(0)返回值为-1，其他返回值均为0）.退出模块时，打开cpu0会出错，((CpuFun)cpu_up)(0)的返回值为-22，其他返回值均为0。
#### 8\. 总结
* 使用cpu_up()和cpu_down()内核函数可以打开和关闭CPU核，并且除cpu0外，其他的核均可以被关闭。
* 对cpu0-cpu3的governor设置不会影响cpu4-cpu7的governor，反之，亦然。
* 若想关闭cpu的某些核，必须首先将cpu0的governor设置为`powersave`或`userspace`，不能为`interactive`(odroidxu3默认的设置,在.config文件中有`CONFIG_CPU_FREQ_DEFAULT_GOV_INTERACTIVE=y`,可以将其修改为其他的governor)。只需要改cpu0的governor即可，就算cpu4-cpu7的governor仍为`interactive`也不会出现关闭某个cpu核又立刻被打开的情况（若不设置cpu0的governor，则会出现此种问题）。

#### 参考资料
* [Android下设置CPU核心数和频率](http://ju.outofmemory.cn/entry/140884)
* [Linux CPU core的电源管理(5)_cpu control及cpu hotplug](http://www.wowotech.net/pm_subsystem/cpu_hotplug.html)
* [Android下设置CPU核心数和频率](http://www.jianshu.com/p/111f79ab2106)

























