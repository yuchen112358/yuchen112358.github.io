## 基于ODROID-XU3平台移植SystemTap至Android平台
[TOC "float:right"]  

#### 1\. 重新编译内核  
执行如下命令：
```Bash
cd /work/Odroid/AndroidSRC/kernel/linux
make odroidxu3_defconfig
```
之后，生成.config文件。
#### 2\. 用gedit打开.config文件，进行如下设置：
```Bash
CONFIG_DEBUG_INFO=y
CONFIG_KPROBES=y
CONFIG_RELAY=y
CONFIG_DEBUG_FS=y
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y
```
**下面还要进行一项重要的设置：**
```Bash
CONFIG_ARM_UNWIND=y
```
在odroidxu3_defconfig的默认配置中CONFIG_ARM_UNWIND是未设置的，这样会在编译.stp文件时，报出一个警告，即`WARNING: "__aeabi_unwind_cpp_pr1" [/tmp/stapo1UQuV/monitor_fopen.ko] undefined!
hello_monitor.ko`,而在插入内核模块时，之后会因为这个警告导致内核模块运行错误。  
** 另外一个可能需要的步骤: **  
在内核的Makefile(位于内核源码的根目录中)中添加下面一句：
```Makefile
CFLAGS_MODULE   = -fno-pic
```
配置文件设置完毕，即可进行编译，命令如下：
```Bash
make clean -j8
make -j8
```
#### 3\. 烧录 zImage-dtb 到Odroid-XU3
```Bash
sudo fastboot flash kernel /work/Odroid/AndroidSRC/kernel/linux/arch/arm/boot/zImage-dtb
sudo fastboot reboot
```
#### 4\. 在PC上安装systemtap
```Bash
cd /work/Odroid/
mkdir SystemTap-New
cd SystemTap-New
git clone https://github.com/flipreverse/systemtap-android.git
cd systemtap-android/
# 可能需要此步：git remote add origin https://github.com/flipreverse/systemtap-android.git
git submodule init
git submodule update
sh build.sh
```
#### 5\. 创建一个.stp文件
```	Bash
cd /work/Odroid/SystemTap-New/systemtap-android/scripts/
vi hello_monitor.stp
```
```Java
#! /usr/bin/stap


probe begin
{
        printf("start monitoring");
}

probe end
{
        printf("end monitoring");
}
```
#### 6\. 创建设备配置文件
在/work/Odroid/SystemTap-New/systemtap-android/config目录下创建一个名为odroidxu3.conf的文件，文件内容如下：
```makefile
KERNEL_ARCH=arm
KERNEL_SRC=/work/Odroid/AndroidSRC/kernel/linux
```
#### 7\. 构建内核模块
利用hello_monitor.stp脚本文件构建内核模块hello_monitor.ko
命令如下:
```Bash
cd /work/Odroid/SystemTap-New/systemtap-android
./build-module hello_monitor odroidxu3
```
**说明**：build-module脚本后面有跟两个参数，它们分别为.stp脚本文件的名称和第六步中所创建的配置文件名（.stp文件必须放在scripts目录下，并且.stp文件和.conf文件都不要带后缀名）。
或直接使用下面的命令：
```Bash
/work/Odroid/SystemTap-New/systemtap-android/installed/bin/stap -p 4 -v -a arm -B CROSS_COMPILE=/opt/toolchains/arm-eabi-4.6/bin/arm-eabi- -r /work/Odroid/AndroidSRC/kernel/linux -I /work/Odroid/SystemTap-New/systemtap-android/installed/share/systemtap/tapset/ -R /work/Odroid/SystemTap-New/systemtap-android/installed/share/systemtap/runtime/ -t -g -m monitor_fopen /work/Odroid/SystemTap-New/systemtap-android/scripts/monitor_fopen.stp
```
编译完成后，生成的内核模块(.ko文件)在/work/Odroid/SystemTap-New/systemtap-android/modules目录下。  
#### 8\. 安装Systemtap Android App到Odroid-XU3。
APP的下载路径为：[https://github.com/flipreverse/systemtap-android-app](https://github.com/flipreverse/systemtap-android-app)  

#### 9\.启动APP并赋予APP root权限
#### 10\. 将第7步生成的内核模块(.ko文件)push到设备的sdcard目录下
命令如下:
```Bash
adb push hello_monitor.ko /sdcard/systemtap/modules/hello_monitor.ko
```
#### 11\.加载内核模块
命令如下：
```Bash
adb shell
su
cd /data/data/com.systemtap.android
# dos2unix可以将windows下的文本格式转换为linux下的文本格式
# 若APP是在windows下编译的必须使用下面的两条语句
dos2unix start_stap.sh
dos2unix kill.sh
sh start_stap.sh
```
之后，输入：
```
modulename=hello_monitor
moduledir=/sdcard/systemtap/modules
outputname=hello_monitor_2016.mm.dd_sss
outputdir=/sdcard/systemtap/stap_output
logdir=/sdcard/systemtap/stap_log
rundir=/sdcard/systemtap/stap_run
stapdir=/data/data/com.systemtap.android
:q!
```
若要，停止模块的运行并移除模块，需要使用如下命令：
```Bash
sh kill.sh
```
之后，输入：
```
pid=-1        
busyboxdir=/system/xbin/
:q!
```
`pid=-1`是移除所有内核模块，另外一种格式是`pid=内核模块对应的PID`,但是此法一直未尝试成功。
当然，在第10步将内核模块push到/sdcard/systemtap/modules/目录下后，在SystemTap 安卓APP的MODULES界面下便会显示出来，可以点击该模块的名称，选择运行或停止该模块。此时，可以单独停止运行某个内核模块。

#### 12\. 读取所加载内核模块的输出结果
```Bash
adb shell
cd /sdcard/systemtap/stap_log/
cat hello_monitor_2016.mm.dd_sss.txt
cd /sdcard/systemtap/stap_output
cat hello_monitor_2016.mm.dd_sss.0
```
当然，这些输出也可以分别在APP中的LOG和OUTPUT界面中直接看到。

#### 附录——心得
* 1）在研一上学期临近期末，我也试图将systemtap移植到odroid-xu3，但是已失败告终，原因是在执行第11步时，脚本执行错误，报出了许多莫名奇怪的错误，让人感觉脚本好像写错了似的。究其原因，发现是因为我的SystemTap安卓APP是在Windows下编译的，所以生成的start_stap.sh和kill.sh脚本文件是Windows格式，Linux下执行会出错，需要使用如下命令进行转换：
```
dos2unix start_stap.sh
dos2unix kill.sh
```
该命令可以在安装了Git工具的windows下，在systemtap-android-app工程目录下的res/raw/目录下打开Git Bash终端，在终端中执行如下命令完成：
```
dos2unix.exe start_stap.sh
dos2unix.exe kill.sh
```
这样在第11步中便不再需要执行dos2unix命令了。
* 2）在本次移植中也遇到了一个棘手的问题，即所以配置都成功了，但是在第7步构建内核模块时老是报**WARNING: "__aeabi_unwind_cpp_pr1" [/tmp/stapo1UQuV/hello_monitor.ko] undefined!
hello_monitor.ko**，并且在第11步加载进内核时，加载出错，使用dmesg命令（dmesg -c清除历史日志，dmesg显示日志），发现出错的原因就是来源于这个警告，即__aeabi_unwind_cpp_pr1符号未定义。
解决问题的思路，符号未定义，原因应该是在内核符号表中没有导出该符号，使用如下命令：
```Bash
cd /work/Odroid/AndroidSRC/kernel/linux
nm vmlinux | grep "__aeabi_unwind_cpp_pr1"
```
发现确实没有输出信息。
从网上搜索资料后发现在内核源码/work/Odroid/AndroidSRC/kernel/linux/arch/arm/kernel/unwind.c文件中有如下语句：

```swift
/* Dummy functions to avoid linker complaints */
void __aeabi_unwind_cpp_pr0(void)
{
};
EXPORT_SYMBOL(__aeabi_unwind_cpp_pr0);

void __aeabi_unwind_cpp_pr1(void)
{
};
EXPORT_SYMBOL(__aeabi_unwind_cpp_pr1);

void __aeabi_unwind_cpp_pr2(void)
{
};
EXPORT_SYMBOL(__aeabi_unwind_cpp_pr2);
```
很奇怪，这不是已经使用EXPORT_SYMBOL(__aeabi_unwind_cpp_pr1）语句将__aeabi_unwind_cpp_pr1符号导出了吗？猜测，可能是在由`make odroidxu3_defconfig`命令生成.config文件中并没有设置某个选项，报着试试看的心态，打开.config文件，搜索关键字"unwind"，发现有一个CONFIG_ARM_UNWIND选项处于未设置状态。对.config文件进行如下改动：
```Bash
CONFIG_ARM_UNWIND=y
```
重新编译内核(`make clean -j8`和`make -j8`),重新烧录内核，重复之前的步骤，最终，移植成功！  
另外，再次使用如下命令：
```Bash
cd /work/Odroid/AndroidSRC/kernel/linux
nm vmlinux | grep "__aeabi_unwind_cpp_pr1"
```
这次的输出如下：
```
c0014598 T __aeabi_unwind_cpp_pr1
c082b8a1 r __kstrtab___aeabi_unwind_cpp_pr1
c081eb7c r __ksymtab___aeabi_unwind_cpp_pr1
```
这也说明了之前的猜想是正确的。
#### 参考资料
* [Android Systemtap can not load module](http://stackoverflow.com/questions/23535968/android-systemtap-can-not-load-module)

























