###GEM5在ubuntu14.04下安装

#####1.下载gem5_stable
```Bash
sudo apt-get install mercurial  #安装mercurial工具包
hg clone http://repo.gem5.org/gem5-stable  #下载gem5-stable版
```
#####2.安装Python
因为SCons（编译GEM5的工具）是用Python编写的，在使用SCons之前安装好Python。Python的版本应该为2.7.5+ 。
因为本人之前已经安装过python了，故在此无需安装。
```Bash
python -V
# Python 2.7.6
```
#####3.安装scons
```Bash
sudo apt-get install scons #安装scons
scons -v                   #查看版本
#SCons by Steven Knight et al.:
#	script: v2.3.0, 2013/03/03 09:48:35, by garyo on reepicheep
#	engine: v2.3.0, 2013/03/03 09:48:35, by garyo on reepicheep
#	engine path: ['/usr/lib/scons/SCons']
#Copyright (c) 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013 #The SCons Foundation
```
#####4.确保g++版本为4.6+
```Bash
sudo apt-get install gcc-4.6
sudo apt-get install g++-4.6
sudo rm -rf /usr/bin/gcc
sudo ln -s /usr/bin/gcc-4.6 /usr/bin/gcc
gcc -v
sudo rm -rf /usr/bin/g++
sudo ln -s /usr/bin/g++-4.6 /usr/bin/g++
gcc -v
```
#####5.安装swig
swig下载网址：[http://www.swig.org/download.html](https://sourceforge.net/projects/swig/files/swig/)
下载swig2.0.7安装包，不要输入命令sudo apt-get install swig否则gem5编译后，运行hello时会报错。
```Bash
tar -zxvf swig-2.0.7.tar.gz #解压
cd ./swig-2.0.7
./configure --without-pcre
make
sudo make install
```
可以用`swig -version`查看swig版本。
#####6、安装python-dev
```Bash
sudo apt-get install python-dev
```
#####7.安装zlib
安装zlib[最近的版本](http://www.zlib.net/)
将zlib-1.2.8.tar.gz解压到gem5_stable目录下
```Bash
cd gem5-stable
tar -zxvf ../tools-gem5/zlib-1.2.8.tar.gz
cd zlib-1.2.8
./configure
sudo make install
```
#####8.安装M4

m4：宏处理器。[http://www.gnu.org/software/m4/](http://www.gnu.org/software/m4/)

将m4-1.4.17.tar.gz到gem5_stable目录下  
==备注==：一不小心下载了m4-1.4.1.tar.gz，发现最后编译并没有出错orz。。。
```Bash
cd gem5-stable
tar -zxvf ../tools-gem5/m4-1.4.17.tar.gz
cd m4-1.4.17
./configure
sudo make install
```
#####9.安装protobuf
安装2.5版本,下载地址：[http://protobuf.googlecode.com/files/protobuf-2.5.0.zip](http://protobuf.googlecode.com/files/protobuf-2.5.0.zip)
```Bash
unzip protobuf-2.5.0.zip
cd protobuf-2.5.0
./configure
make
make check
sudo make install
```
#####10.安装libprotobuf-dev
```Bash
sudo apt-get install libprotobuf-dev
```
#####11.安装libgoogle-perftools-dev
```Bash
sudo apt-get install libgoogle-perftools-dev
```
#####12.编译gem5：
```Bash
cd gem5-stable
mkdir build
```
指定编译的选项，及目标文件:

```Bash
scons build/ALPHA/gem5.opt
```
==如果出现如下错误：==  
错误：can't find Python.h header in ['/usr/include/python2.7']  
解决：`sudo apt-get install python-dev`

#####13.测试
在gem5目录下输入命令：
```Bash
./build/ALPHA/gem5.opt ./configs/example/se.py -c tests/test-progs/hello/bin/alpha/linux/hello
```
运行结果如下：
```
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Mar 21 2016 16:31:22
gem5 started Mar 21 2016 16:33:42
gem5 executing on yuchen-PC
command line: ./build/ALPHA/gem5.opt ./configs/example/se.py -c tests/test-progs/hello/bin/alpha/linux/hello

/work/gem5/gem5-stable/configs/common/CacheConfig.py:48: SyntaxWarning: import * only allowed at module level
  def config_cache(options, system):
Global frequency set at 1000000000000 ticks per second
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
**** REAL SIMULATION ****
info: Entering event queue @ 0.  Starting simulation...
info: Increasing stack size by one page.
Hello world!
Exiting @ tick 3233000 because target called exit()
```
**安装成功！**
#####14.一些注意点
如果protobuf有错误，可以移除以前的版本。
```Bash
sudo apt-get autoremove libprotobuf-dev
```























