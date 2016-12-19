##Odroid软件开发环境搭建及烧录
[TOC "float:right"]
###一.ODROID-XU3开发板图示
<div align="center">![图片](http://img.blog.csdn.net/20151130215340982 "Odroid开发板简介" "width:440px;height: 540px")</div>  
<div align="center">开发板硬件介绍[^1]</div>
###二.软件开发环境搭建及烧录
#####1.系统配置信息
&nbsp;&nbsp;&nbsp;&nbsp;1)操作系统：Ubuntu 14.04.3 LTS  
&nbsp;&nbsp;&nbsp;&nbsp;2)gcc版本：gcc version 4.4.7  
&nbsp;&nbsp;&nbsp;&nbsp;3)g++版本:gcc version 4.4.7  
&nbsp;&nbsp;&nbsp;&nbsp;4)JDK版本:java version "1.6.0_45"  
#####2.编译说明
建议在编译Odroid的Android版本之前先编译谷歌官方发布的Android源码，按照谷歌官方的方法进行android源码同步及环境配置，参见[Android官网](http://source.android.com/source/initializing.html)，简易方法请点击[编译Android4.4.4](/2016/03/02/android.html)。
#####3.烧录官方release的镜像
为了验证开发版功能，也为了后面烧录内核镜像、Android系统镜像方便，我们首先烧录官方提供的安卓release镜像，这样我们就不需要自己制作[分区表](http://odroid.com/dokuwiki/doku.php?id=en:xu3_partition_table)（如，u-boot区、kernel区、system区等，注意：自己划分分区表也很容易出错）。这样，在我们将官方release镜像烧录到SD卡后，我们就搭建好了一个分区框架，之后就可以使用fastboot命令向对应分区烧录镜像。  
==烧录官方release镜像步骤：==  
a.准备一个读卡器和一个8G以上的SD卡；  
b.下载在windows平台上进行烧写的[工具](http://com.odroid.com/sigong/nf_file_board/nfile_board_view.php?bid=199and),用管理员权限打开，方法：选择镜像->选择U盘符->点击Write->点击Verify。  
c.下载官方release版本镜像（[Download](http://dn.odroid.com/5422/ODROID-XU3/Android/)）；  
d.烧写到SD卡后，即可将SD卡插到板载SD卡槽中，并将启动模式选择为左下右上的模式，即SD Card启动方式，之后Android系统便成功启动。  
#####4.编译并安装U-boot
1）下载U-boot交叉编译工具链（[Download](http://dn.odroid.com/toolchains/gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2)）。  
2）将工具链解压到/opt/toolchains/目录下:

```
$ sudo mkdir -p /opt/toolchains
$ sudo tar jxf gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2 -C /opt/toolchains/
```
3）配置.bashrc文件
```
$ vim ~/.bashrc
export PATH=/opt/toolchains/gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux/bin:$PATH
export CROSS_COMPILE=arm-linux-gnueabihf-
$ source ~/.bashrc
```

4)测试工具链是否配置正确：
```
$ arm-linux-gnueabihf-gcc -v
```

5)下载U-boot源码:
```
$ git clone https://github.com/hardkernel/u-boot.git -b odroidxu3-v2012.07
```

6）编译U-boot：
```
$ cd u-boot
$ make odroid_config
$ make -j4
```
7)因为我们利用了官方的release版本镜像，所以如果需要安装新的u-boot（一般不需要的），可以使用如下命令：
```
a)u-boot.bin 安装

$ sudo fastboot flash bootloader u-boot.bin

b)bl1.bin 安装

$ sudo fastboot flash fwbl1 sd_fuse/hardkernel/bl1.bin.hardKernel

c)bl2.bin 安装

$ sudo fastboot flash bl2 sd_fuse/hardkernel/bl2.bin.hardKernel

d)tzsw.bin 安装

$ sudo fastboot flash tzsw sd_fuse/hardkernel/tzsw.bin.hardKernel

e)安装完成，使用fastboot重启 ODROID-XU3

$ sudo fastboot reboot
```
#####5.编译Android Kernel和Android Platform所需工具链的配置  
1）下载交叉编译工具链（[Download](http://dn.odroid.com/ODROID-XU/compiler/arm-eabi-4.6.tar.gz)）。  
2）将工具链解压到/opt/toolchains/目录下:  
```
$ sudo mkdir -p /opt/toolchains
$ sudo tar xvfz arm-eabi-4.6.tar.gz -C /opt/toolchains/
```
3）配置.bashrc文件：  
```
$ vim ~/.bashrc
export ARCH=arm
export CROSS_COMPILE=arm-eabi-
export PATH=/opt/toolchains/arm-eabi-4.6/bin:$PATH
$ source ~/.bashrc
```
4)测试工具链是否配置正确：
```
$ arm-eabi-gcc -v
```
#####6.Android Kernel下载
```
$ git clone --depth 1 https://github.com/hardkernel/linux.git -b odroidxu3-3.10.y-android
$ cd linux
```
#####7.Android Platform下载
```
$ mkdir <Android Platform Folder Name>
$ cd <Android Platform Folder Name>
$ repo init -u https://github.com/hardkernel/android.git -b 5422_4.4.4_master
$ repo sync
$ repo start 5422_4.4.4_master --all
```

#####8.Android Kernel编译
```
$ make odroidxu3_defconfig
$ make -j8
```

编译好的Linux kernel image名称为zImage-dtb，位置为：arch/arm/boot/zImage-dtb。
#####9.Android Platform编译
```
$ ./build.sh odroidxu3
```
编译好的镜像文件位置为：自己设置的输出目录（默认为out）/target/product/odroidxu3/update。

#####10.安装
在进行安装内核和安卓平台之前，需要先安装minicom（串口通信工具）、fastboot(烧录工具)和adb（安卓调试工具）。工具安装完成后，即可连接板子和电脑（板子和电脑之间的通信需要使用minicom）。一接通电源就迅速点击“Enter”或者“space”两次，进入fastboot模式，等待烧录。重新在电脑上打开一个Linux shell终端，进行如下操作：
1）内核镜像安装：
```
$ sudo fastboot flash kernel <path/of/your/zImage-dtb>
```
2）system.img, userdata.img, cache.img,ramdisk.img安装
```
$ sudo fastboot flash system <path/of/your/system.img>
$ sudo fastboot flash userdata <path/of/your/userdata.img>
$ sudo fastboot flash cache <path/of/your/cache.img>
$ sudo fastboot flash ramdisk <path/of/your/ramdisk.img>
```

#####11.初始化分区表并重新启动系统
```
$ sudo fastboot erase fat
$ sudo fastboot reboot
```
#####12.附录
1. [minicom的安装方法](http://www.oschina.net/question/565065_115186?sort=time)
2. [adb和fastboot的安装方法](http://www.metsky.com/archives/740.html)
3. [adb shell error: device not found 的解决方法](http://blog.csdn.net/aganlengzi/article/details/47185571 )

#####==注意==  
1)ODROID-XU3将各种镜像烧录到SD卡以及主机运行的adb shell命令都是通过板载USB3.0进行的，而ODROID-XU3运行时的数据输出通过串口转USB传到电脑上（接通电源前运行`sudo minicom`命令即可完成ODROID-XU3与电脑主机的通信，ODROID-XU3上系统正常运行后会在主机终端上也显示一个adb shell命令终端，但是这个终端是在ODROID-XU3上的安卓系统运行并传送到主机shell终端上的）。  
2)ODROID-XU3上的USB3.0可以用移动硬盘的数据线，但是必须插在主机的USB2.0端口上，否则会导致adb shell error: device not found错误（插在主机的USB3.0端口上就会发生该错误，这一点很是奇怪）。  
3）不管哪部分镜像烧写后，只需要利用`$ sudo fastboot reboot`命令重启开发板即可。  

[^1]:图片引自http://blog.csdn.net/aganlengzi/article/details/50119419
