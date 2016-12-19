#### 1.设置老版本ubuntu软件源(10.10为例)
设置软件源如下：
* 打开sources.list：
```Bash
sudo gedit /etc/apt/sources.list
```
* 将其修改为：
```Bash
deb http://old-releases.ubuntu.com/ubuntu maverick main universe restricted multiverse
deb-src http://old-releases.ubuntu.com/ubuntu maverick main universe restricted multiverse
deb http://old-releases.ubuntu.com/ubuntu maverick-security universe main multiverse restricted
deb-src http://old-releases.ubuntu.com/ubuntu maverick-security universe main multiverse restricted
deb http://old-releases.ubuntu.com/ubuntu maverick-updates universe main multiverse restricted
deb http://old-releases.ubuntu.com/ubuntu maverick-proposed universe main multiverse restricted
deb-src http://old-releases.ubuntu.com/ubuntu maverick-proposed universe main multiverse restricted
deb http://old-releases.ubuntu.com/ubuntu maverick-backports universe main multiverse restricted
deb-src http://old-releases.ubuntu.com/ubuntu maverick-backports universe main multiverse restricted
deb-src http://old-releases.ubuntu.com/ubuntu maverick-updates universe main multiverse restricted
```
ubuntu每个版本都是有一个版本名的，10.10的版本名字叫maverick，你可以用文本编辑器将所有maverick换成你要用的版本的名字即可。  
查看版本名：
```Bash
lsb_release -a
```
输出：  
No LSB modules are available.  
Distributor ID:	Ubuntu  
Description:	Ubuntu 10.10  
Release:	10.10  
**Codename:	maverick**

#### 2.安装gcc-3.4.6
0. mkdir gcc-3.4
1. cd gcc-3.4
1. wget http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/cpp-3.4_3.4.6-6ubuntu5_i386.deb  
2. wget http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/gcc-3.4-base_3.4.6-6ubuntu5_i386.deb  
3. wget http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/gcc-3.4_3.4.6-6ubuntu5_i386.deb  
4. wget http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/g++-3.4_3.4.6-6ubuntu5_i386.deb  
5. wget http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/libstdc++6-dev_3.4.6-6ubuntu5_i386.deb  
6. sudo chmod 777 * 
7. sudo dpkg -i *
8. sudo ln -sf /usr/bin/gcc-3.4  /usr/bin/gcc
9. gcc -v 显示版本 gcc version 3.4.6 (Ubuntu 3.4.6-6ubuntu5)


#### 3.virtualbox的共享文件夹访问权限

virtualbox的共享文件夹一般都挂载在/media下面，用ll查看会发现文件夹的所有者是root，所有组是vboxsf，所以文件管理去无法访问是正常的，解决方法是把你自己加入到vboxsf组里面。
代码如下:

`sudo usermod -a -G vboxsf yourusernanme`

重启，即可。

#### 4.修改路径参数
```Bash
vim /home/sesc/.bashrc 
	+export PATH=$HOME/sescutils/install/bin:$HOME/sesc-build:$PATH
source /home/sesc/.bashrc
```
#### 5.编写sesc程序
```Bash
cd sesc-user
vim hellosesc.c
mipseb-linux-gcc -mips2 -mabi=32 -static -Wa,-non_shared -mno-abicalls -Wl,--script=$HOME/sescutils/src/mint.x,-static hellosesc.c -o hello
```
#### 6.运行
```Bash
cd ~/sesc-user
cp /home/sesc/sesc-build/../sesc/confs/mem.conf sesc.conf
cp /home/sesc/sesc-build/../sesc/confs/shared.conf .
../sesc-build/sesc.mem -h0x800000 -csesc.conf hello
../sesc-build/sesc.mem -h0x800000 -cpower.conf hello
```
sesc.conf 是运行时的配置 和前面的-c中间没有空格 若没有指定 则指定当前文件夹下的sesc.conf

若两者都没有 则运行错误

hello是之前编译的可执行文件

若有输入数据 写在文件中tt.in

```Bash
../sesc-build/sesc.mem -h0xC00000 -c ./power.conf ../sesc/tests/crafty < ../sesc/tests/tt.in

./sesc.mem -h0x800000 -cpower.conf /home/sesc/sesc-build/../sesc/tests/crafty < /home/sesc/sesc-build/../sesc/tests/tt.in
```

#### 7.获得可视化报告

 `../sesc/scripts/report.pl --last`

#### 8.参考文献:
[SESC模拟器的安装方法](http://blog.csdn.net/a8887396/article/details/8794378)