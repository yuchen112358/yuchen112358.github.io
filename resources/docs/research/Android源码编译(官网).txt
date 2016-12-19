##Android 4.4.4编译

###编译系统环境：

`java -version`：java version "1.6.0_45"  

`gcc -v`:gcc version 4.4.7 (Ubuntu/Linaro 4.4.7-8ubuntu1)  

`g++ -v`:gcc version 4.4.7 (Ubuntu/Linaro 4.4.7-8ubuntu1)  

####1.初始化环境变量

```
$ source build/envsetup.sh
或者$ . build/envsetup.sh
```

####2.选择编译目标

```
$ lunch aosp_arm-eng
```

####3.构建代码
```
$ make -j4
```

####4.启动所编译的系统
编译完之后会在out/target/product/generic目录下生成system.img ramdisk.img userdata.img三个镜像文件。  

在启动模拟器之前，需要先为模拟器系统设置环境变量，执行gedit ~/.bashrc，新增环境变量如下：
```
export ANDROID_PRODUCT_OUT=/work/Android/WORKING_DIRECTORY/out/target/product/generic
export ANDROID_PRODUCT_OUT_BIN=/work/Android/WORKING_DIRECTORY/out/host/linux-x86/bin
export PATH=${PATH}:${ANDROID_PRODUCT_OUT_BIN}:${ANDROID_PRODUCT_OUT}
```

最后，同步这些变化并启动模拟器
```
$ source ~/.bashrc
$ cd /work/Android/outAndroid/WORKING_DIRECTORY/target/product/generic/
$ emulator -system system.img -data userdata.img -ramdisk ramdisk.img
```

####5.备注：
>#####关于gcc和g++降级

######1\.首先安装gcc4.4和g++4.4

```
sudo apt-get install gcc-4.4

sudo apt-get install g++-4.4
```

######2\.gcc和g++的降级gcc降级：

```
sudo rm -rf /usr/bin/gcc  

sudo ln -s /usr/bin/gcc-4.4 /usr/bin/gcc  

gcc -v  
```

* g++降级

```
sudo rm -rf /usr/bin/g++  

sudo ln -s /usr/bin/g++-4.4 /usr/bin/g++  

g++ -v
```

>#####安装JDK 1.6

* [安装步骤链接](http://jingyan.baidu.com/article/2d5afd69f1f6b985a3e28e4f.html "百度经验")

####6\.解决编译错误参考链接
- [Android源码编译](http://www.xuebuyuan.com/1404326.html)
- [Ubuntu 13.04下载 编译Android 4.0](http://www.linuxidc.com/Linux/2014-03/97761p2.htm)
- [Android编译环境搭建](http://my.oschina.net/fanxiao/blog/41645?fromerr=1Nfq9HgP)
- [基于ubuntu10.04和ubuntu14.04搭建MTK6577+Android4.04开发环境的问题与解决](http://blog.csdn.net/loongembedded/article/details/38014841)
- [Android4.0编译 error](http://blog.csdn.net/z_guijin/article/details/7720841)
- [Ubuntu12.10编译Android 4.0.3的常见错误](http://blog.csdn.net/kangear/article/details/9565889)


