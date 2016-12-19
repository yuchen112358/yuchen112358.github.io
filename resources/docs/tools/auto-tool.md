## autoconf和automake的使用
***
##### 1.安装autoconf和automake
```Bash
 sudo apt-get install autoconf
```
==注意：==运行上述命令，`automake` `autotools-dev` `m4`三个工具也会被安装。
##### 2.创建一个源程序，命名为helloworld.c
程序内容如下：
```C
#include <stdio.h>
int main(int argc, char** argv){
     printf("%s", 'Hello, Linux World!\n");
     return 0;
}
```
##### 3.使用autoscan工具生成configure.scan
```Bash
ns@ubuntu:~/workplace/studyauto$ ls
helloworld.c
ns@ubuntu:~/workplace/studyauto$ autoscan ./
ns@ubuntu:~/workplace/studyauto$ ls
autoscan.log  configure.scan  helloworld.c
```
##### 4.将configure.scan重新命名为configure.ac,并修改相关内容
```Bash
mv configure.scan configure.ac
```
原内容如下：
```Makefile
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.68])
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
AC_CONFIG_SRCDIR([helloworld.c])
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CC

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT
```
修改后如下：
```Makefile
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.68])
AC_INIT([helloworld], [1.0], [wzzju@mail.ustc.edu.cn])
AM_INIT_AUTOMAKE
AC_CONFIG_SRCDIR([helloworld.c])
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CC

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT(Makefile)
```
==文件内容解释：==
* AC_PREREQQ宏声明本文件要求的autoconf版本，这里是2.68

* AC_INIT定义软件的名称和信息。（FULL-PACKAGE-NAME为软件名，VERSION为软件的版本号，BUG-REPORT-ADDRESS为bug的报告地址，一般为软件作者的邮箱）

* AC_CONFIG_SRCDIR用来侦测指定的源码文件是否存在，确定源码目录的有效性。此处为当前目录下helloworld.c

* AC_CONFIG_HEADER用于生成config.h文件，以便autoheader使用

* AC_PROG_CC用来指定编译器，以便不指定的时候默认为gcc

* AC_OUTPUT用来设定config要产生的文件。如果是Makefile,config会把它检查出来的结果带入Makefile.in文件产生合适的Makefile.
* **AM_INIT_AUTOMAKE宏需要自己进行添加，它是automake所必备的宏。新版本中该宏不需要任何参数。**

##### 5.使用aclocal工具生成aclocal.m4
```Bash
ns@ubuntu:~/workplace/studyauto$ ls
autoscan.log  configure.ac  helloworld.c
ns@ubuntu:~/workplace/studyauto$ aclocal
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4  autom4te.cache  autoscan.log  configure.ac  helloworld.c
```
##### 6.使用autoconf工具生成configure文件
```Bash
ns@ubuntu:~/workplace/studyauto$ autoconf
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4  autom4te.cache  autoscan.log  configure  configure.ac  helloworld.c
```
##### 7.使用autoheader工具生成config.h.in
```Bash
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4  autom4te.cache  autoscan.log  configure  configure.ac  helloworld.c
ns@ubuntu:~/workplace/studyauto$ autoheader
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4      autoscan.log  configure     helloworld.c
autom4te.cache  config.h.in   configure.ac
```
##### 8.创建Makefile.am文件
automake工具会根据config.in中的参量把Makefile.am转换成Makefile.in文件。在使用Automake之前，要先手动建立Makefile.am文件。
Makefile.am的内容如下：
```Makefile
AUTOMAKE_OPTIONS=foreign
bin_PROGRAMS=helloworld
helloworld_SOURCES=helloworld.c
```
```Bash
ns@ubuntu:~/workplace/studyauto$ vim Makefile.am
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4      autoscan.log  configure     helloworld.c
autom4te.cache  config.h.in   configure.ac  Makefile.am
```
Makefile.am的内容解释：
* AUTOMAKE_OPTIONS为设置的Automake选项。它有三种等级提供给用户选择：foreign，gnu,gnits,默认等级为gnu.在此使用foreign，它只检测必须的文件。
* bin_PROGRAMS定义要产生的执行文件名。如果要产生多个可执行文件，则每个文件名用空格隔开。
* hello_SOURCES定义为产生helloworld这个可执行程序所需要的原始文件。如果其是由多个文件组成的，则必须用空格进行隔开。

##### 9.使用automake工具生成Makefile.in文件
要使用选项“--add-missing”可以让Automake自动添加一些必要的脚本文件。
```Bash
ns@ubuntu:~/workplace/studyauto$ automake --add-missing
configure.ac:6: installing `./install-sh'
configure.ac:6: installing `./missing'
Makefile.am: installing `./depcomp'
```
##### 10.运行自动配置设置文件configure，根据Makefile.in生成最终的Makefile
```Bash
ns@ubuntu:~/workplace/studyauto$ ./configure 
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /bin/mkdir -p
checking for gawk... no
checking for mawk... mawk
checking whether make sets $(MAKE)... yes
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables... 
checking whether we are cross compiling... no
checking for suffix of object files... o
checking whether we are using the GNU C compiler... yes
checking whether gcc accepts -g... yes
checking for gcc option to accept ISO C89... none needed
checking for style of include used by make... GNU
checking dependency style of gcc... gcc3
configure: creating ./config.status
config.status: creating Makefile
config.status: creating config.h
config.status: executing depfiles commands
```
##### 11.运行make命令进行编译、测试
```Bash
ns@ubuntu:~/workplace/studyauto$ make
make  all-am
make[1]: Entering directory `/home/ns/workplace/studyauto'
gcc -DHAVE_CONFIG_H -I.     -g -O2 -MT helloworld.o -MD -MP -MF .deps/helloworld.Tpo -c -o helloworld.o helloworld.c
mv -f .deps/helloworld.Tpo .deps/helloworld.Po
gcc  -g -O2   -o helloworld helloworld.o  
make[1]: Leaving directory `/home/ns/workplace/studyauto'
```
```Bash
ns@ubuntu:~/workplace/studyauto$ ls
aclocal.m4      config.h.in    configure.ac  helloworld.o  Makefile.in
autom4te.cache  config.log     depcomp       install-sh    missing
autoscan.log    config.status  helloworld    Makefile      stamp-h1
config.h        configure      helloworld.c  Makefile.am
```
运行生成的可执行程序helloworld。
```Bash
ns@ubuntu:~/workplace/studyauto$ ./helloworld 
Hello, Linux World!
```
##### 12.参考资料

* [Autoconf/Automake工具简介](http://www.cnblogs.com/xf-linux-arm-java-android/p/3590770.html)
* [automake-GNU](http://www.gnu.org/software/automake/manual/automake.html#Modernize-AM_005fINIT_005fAUTOMAKE-invocation)
* [automake,autoconf使用详解](http://www.laruence.com/2009/11/18/1154.html)

#### 附录——解释说明
```Bash
1. autoscan
　　autoscan是用来扫描源代码目录生成configure.scan文件的.autoscan可以用目录名做为参数,但如果你不使用参数的话,那么autoscan将认为使用的是当前目录.autoscan将扫描你所指定目录中的源文件,并创建configure.scan文件.
2. configure.scan
　　configure.scan包含了系统配置的基本选项,里面都是一些宏定义.我们需要将它改名为configure.ac。
3. aclocal
　　aclocal是一个perl脚本程序.aclocal根据configure.ac文件的内容,自动生成aclocal.m4文件.aclocal的定义是 ："aclocal - createaclocal.m4 by scanning configure.ac"。
4. autoconf
　　autoconf是用来产生configure文件的.configure是一个脚本,它能设置源程序来适应各种不同的操作系统平台,并且根据不同的系统来产生合适的Makefile,从而可以使你的源代码能在不同的操作系统平台上被编译出来.
　　configure.ac文件的内容是一些宏,这些宏经过autoconf处理后会变成检查系统特性、环境变量、软件必须的参数的shell脚本.configure.ac文件中的宏的顺序并没有规定,但是你必须在所有宏的最前面和最后面分别加上AC_INIT宏和AC_OUTPUT宏.
　　在 configure.ac中：
　　#号表示注释,这个宏后面的内容将被忽略.
　　AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])这个宏定义了软件的名称和信息。（FULL-PACKAGE-NAME为软件名，VERSION为软件的版本号，BUG-REPORT-ADDRESS为bug的报告地址，一般为软件作者的邮箱）。当你使用make dist命令时,它会给你生成一个类似helloworld-1.0.tar.gz的软件发行包,其中就有对应的软件包的名字和版本号.
   AM_INIT_AUTOMAKE这个宏需要自己进行添加，它是automake所必备的宏。新版本中该宏不需要任何参数。
   AC_PROG_CC这个宏将检查系统所用的C编译器.
   AC_OUTPUT(FILE)这个宏是我们要输出的Makefile的名字.
　 我们在使用automake时,实际上还需要用到其他的一些宏,但我们可以用aclocal来帮我们自动产生.执行aclocal后我们会得到aclocal.m4文件.
　 产生了configure.ac和aclocal.m4两个宏文件后,我们就可以使用autoconf来产生configure文件了。
5. Makefile.am
　　Makefile.am是用来生成Makefile.in的,需要你手工书写.Makefile.am中定义了一些内容：
AUTOMAKE_OPTIONS
　　这个是automake的选项.在执行automake时,它会检查目录下是否存在标准GNU软件包中应具备的各种文件,例如AUTHORS.ChangeLog.NEWS等文件.我们将其设置成foreign时,automake会改用一般软件包的标准来检查.
bin_PROGRAMS
　　这个是指定我们所要产生的可执行文件的文件名.如果你要产生多个可执行文件,那么在各个名字间用空格隔开.
helloworld_SOURCES
　　这个是指定产生"helloworld"时所需要的源代码.如果它用到了多个源文件,那么请使用空格符号将它们隔开.比如需要helloworld.h,helloworld.c那么请写成:helloworld_SOURCES= helloworld.h helloworld.c.
　　如果你在 bin_PROGRAMS定义了多个可执行文件,则对应每个可执行文件都要定义相对的filename_SOURCES.
6. automake
　　我们使用automake --add-missing来产生Makefile.in文件。
　　选项--add-missing的定义是 "add missing standard files to package",它会让automake加入一个标准的软件包所必须的一些文件.
　　我们用automake产生出来的Makefile.in文件是符合GNU Makefile惯例的 ,接下来我们只要执行configure这个shell脚本就可以产生合适的Makefile文件了.
7. Makefile
　　在符合GNU Makefiel惯例的Makefile中,包含了一些基本的预先定义的操作：
make
　　根据Makefile编译源代码,连接,生成目标文件,可执行文件.
make clean
　　清除上次的make命令所产生的object文件（后缀为".o"的 文件）及可执行文件.
make install
　　将编译成功的可执行文件安装到系统目录中,一般为/usr/local/bin目录.
make dist
　　产生发布软件包文件（即distribution package）.这个命令将会将可执行文件及相关文件打包成一个tar.gz压缩的文件用来作为发布软件的软件包.
　　它会在当前目录下生成一个名字类似"PACKAGE-VERSION.tar.gz"的文件.PACKAGE和VERSION,是我们在configure.ac中定义的AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS]).
make distcheck
　　生成发布软件包并对其进行测试检查,以确定发布包的正确性.
```



































