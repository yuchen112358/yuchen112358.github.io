---
layout:     post
title:      "使用JNA"
subtitle:   "——Java调用C++/C"
date:       2016-11-04
author:     "yuchen"
header-img: "img/post-bg-java.jpg"
tags:
    - Java
    - JNA
    - CPP
---

## 一.编译Linux动态链接库so文件

如果是C++程序，注意在接口函数的前面加上extern "C"标记，在头文件加上如下标记：

```
#ifdef   __cplusplus   
extern   "C"{   
#endif   
    
头文件主体   
    
#ifdef   __cplusplus   
}   
#endif   
```

如果不加以上标记，经过编译后，so里的函数名并非你编写程序时设定的函数名，在开发环境左侧的工程文件列表中点开debug项里的PXXX.o可以看到so文件里的函数名都是在你设定的函数名后面加了一个__Fi标记，比如你用的设定的函数名称是Func(), 而so里的函数名则为Func__Fi()或者其他的名称（如： _Z4showv、_Z3addii）。


#### 1.编译c程序：
`gcc -shared -fPIC XXX.c -o libXXX.so`

#### 2.编译c++程序：
`g++ -shared -fPIC XXX.cpp -o libXXX.so`

#### 3.linux下查看动态链接库so文件  

nm libXXX.so，其中以T打头的是动态链接库里的函数的名称。

#### 4.示例：

###### 4.1 头文件

```
/*
 * hello.h
 *
 *  Created on: 2016年11月4日
 *      Author: yuchen
 */

#ifndef HELLO_H_
#define HELLO_H_

#ifdef   __cplusplus
extern   "C"{
#endif

void show();

int add(int a, int b);

#ifdef   __cplusplus
}
#endif
#endif /* HELLO_H_ */

```

###### 4.2 C文件

```
/*
 * helloC.c
 *
 *  Created on: 2016年11月4日
 *      Author: yuchen
 */
#include <stdio.h>
#include "hello.h"
void show()
{
	printf("hello,c!\n");
}

int add(int a, int b)
{
	return a + b;
}

```

###### 4.3 C++文件

```
/*
 * helloCPP.cpp
 *
 *  Created on: 2016年11月4日
 *      Author: yuchen
 */

#include <iostream>
#include "hello.h"

using namespace std;
void show()
{
    cout << "hello,cpp!" << endl;
}

int add(int a, int b)
{
	return a + b;
}
```

###### 4.4 编译C文件

`gcc -shared -fPIC helloC.c -o libhelloC.so`

###### 4.5 编译C++文件

`g++ -shared -fPIC helloCPP.cpp -o libhelloCPP.so`

###### 4.6 编译参数解析

最主要的是GCC命令行的一个选项:
-shared该选项指定生成动态连接库（让连接器生成T类型的导出符号表，有时候也生成弱连接W类型的导出符号），不用该标志外部程序无法连接。相当于一个可执行文件。   

-fPIC：表示编译为位置独立的代码，不用此选项的话编译后的代码是位置相关的所以动态载入时是通过代码拷贝的方式来满足不同进程的需要，而不能达到真正代码段共享的目的。  

-L.：表示要连接的库在当前目录中。  

-lXXX：编译器查找动态连接库时有隐含的命名规则，即在给出的名字前面加上lib，后面加上.so来确定库的名称。  

 使用so示例（当前目录下存在libtest.so文件）：`gcc test.c -L. -ltest -o test`

#### 5.使用Makefile

1）编译C语言编写的动态链接库所使用的Makefile

```makefile
#*****************************************************************************
# Copyright        :  
#
# Author           :   huochangjun
# Modificator  :    yuchen
# Date             :   2016-11-08
# Version           :   
# Description             :   使用C书写动态链接库
#
#****************************************************************************/

SHELL = /bin/sh
# 源码目录src放在${APP_DIR}下
APP_DIR = /work/SoMaker/LinuxSoMaker/C
LIB_DIR = ${APP_DIR}/lib/
BIN_DIR = ${APP_DIR}/bin/
OBJECT_DIR = ${APP_DIR}/obj/
SRC_DIR = ${APP_DIR}/src/

$(shell mkdir -p ${LIB_DIR})

$(shell mkdir -p ${BIN_DIR})

$(shell mkdir -p ${OBJECT_DIR})

RM = rm -fr

#****************************************************************************

CC = gcc

SHARED = -shared

FPIC = -fPIC -c

SRC_OBJECT = $(SRC_DIR)test_a.c $(SRC_DIR)test_b.c $(SRC_DIR)test_c.c

H_OBJECT = $(SRC_DIR)so_test.h

OBJECT = test_a.o test_b.o test_c.o

DY_SRC_OBJECT = $(SRC_DIR)test.c

DY_OBJECT=test.o

LIB_OBJECT = libtest.so

BIN_OBJECT = test

#****************************************************************************

.PHONY:all dlib clean

all:$(LIB_OBJECT) $(BIN_OBJECT)

dlib:$(LIB_OBJECT)

$(LIB_OBJECT):$(OBJECT)
	$(CC) $(OBJECT) $(SHARED) -fPIC -o $(LIB_OBJECT)
	mv $(OBJECT) $(OBJECT_DIR)
	mv $(LIB_OBJECT) $(LIB_DIR)

$(OBJECT):$(SRC_OBJECT) $(H_OBJECT)
	$(CC) $(FPIC) $(SRC_OBJECT)

$(BIN_OBJECT):$(DY_OBJECT)
	$(CC) $(OBJECT_DIR)$(DY_OBJECT) -L$(LIB_DIR) -ltest -o $(BIN_OBJECT)
	mv $(BIN_OBJECT) $(BIN_DIR)

$(DY_OBJECT):$(DY_SRC_OBJECT)
	$(CC) -c $(DY_SRC_OBJECT)
	mv $(DY_OBJECT) $(OBJECT_DIR)

clean:
	$(RM) $(LIB_DIR) $(BIN_DIR) $(OBJECT_DIR)
```

2）编译C++编写的动态链接库所使用的Makefile

```makefile
#*****************************************************************************
# Copyright        :  
#
# Author           :   huochangjun
# Modificator  :    yuchen
# Date             :   2016-11-08
# Version           :   
# Description             :   使用C++书写动态链接库
#
#****************************************************************************/

SHELL = /bin/sh
# 源码目录src放在${APP_DIR}下
APP_DIR = /work/SoMaker/LinuxSoMaker/CPP
LIB_DIR = ${APP_DIR}/lib/
BIN_DIR = ${APP_DIR}/bin/
OBJECT_DIR = ${APP_DIR}/obj/
SRC_DIR = ${APP_DIR}/src/

$(shell mkdir -p ${LIB_DIR})

$(shell mkdir -p ${BIN_DIR})

$(shell mkdir -p ${OBJECT_DIR})

RM = rm -fr

#****************************************************************************

CXX = g++

SHARED = -shared

FPIC = -fPIC -c

SRC_OBJECT = $(SRC_DIR)JNIDemo.cpp $(SRC_DIR)JNIStringDemo.cpp $(SRC_DIR)JNIDemoInterface.cpp

H_OBJECT = $(SRC_DIR)JNIDemo.h $(SRC_DIR)JNIStringDemo.h $(SRC_DIR)JNIDemoInterface.h

OBJECT = JNIDemo.o JNIStringDemo.o JNIDemoInterface.o

DY_SRC_OBJECT = $(SRC_DIR)JNIMain.cpp

DY_OBJECT=JNIMain.o

LIB_OBJECT = libjni.so

BIN_OBJECT = JNIMain

#****************************************************************************

.PHONY:all dlib clean

all:$(LIB_OBJECT) $(BIN_OBJECT)

dlib:$(LIB_OBJECT)

$(LIB_OBJECT):$(OBJECT)
	$(CXX) $(OBJECT) $(SHARED) -fPIC -o $(LIB_OBJECT)
	mv $(OBJECT) $(OBJECT_DIR)
	mv $(LIB_OBJECT) $(LIB_DIR)

$(OBJECT):$(SRC_OBJECT) $(H_OBJECT)
	$(CXX) $(FPIC) $(SRC_OBJECT)

$(BIN_OBJECT):$(DY_OBJECT)
	$(CXX) $(OBJECT_DIR)$(DY_OBJECT) -L$(LIB_DIR) -ljni -o $(BIN_OBJECT)
	mv $(BIN_OBJECT) $(BIN_DIR)

$(DY_OBJECT):$(DY_SRC_OBJECT)
	$(CXX) -c $(DY_SRC_OBJECT)
	mv $(DY_OBJECT) $(OBJECT_DIR)

clean:
	$(RM) $(LIB_DIR) $(BIN_DIR) $(OBJECT_DIR)
```

**注意**:  

1）编译选项说明如下：

* make ： 编译动态链接库和使用该链接库的可执行文件
* make dlib ： 仅编译动态链接库
* make clean ： 清除编译结果

2）运行使用动态链接库的程序

```bash
cd bin/
export LD_LIBRARY_PATH=../lib
./JNIMain
```

3)[查看源码](https://github.com/yuchen112358/LinuxSoMaker.git)



## 二.使用JNA调用动态链接库中的函数

为了使用.so文件，在项目下创建libs文件夹，将第一部分生成的.so文件以及[jna-x.x.x.jar](https://github.com/java-native-access/jna)文件放在该文件夹下，在Eclipse中右击libs文件夹，选择Build Path->Use As Source Folder。接着，右击jna-x.x.x.jar文件，选择Build Path->Add to Build Path。注意这里的.so文件一定要以lib为前缀，而在java代码中使用JNA寻找so库时使用的库名要去掉lib前缀和.so后缀！！！  

**补充：在IntelliJ IDEA中，新建工程，在该工程名上右击，选择New->Directory,新建文件夹名称为libs,然后将jan-xxx.jar和所需的.so文件均放在libs目录下，右击jan-xxx.jar，选择Add As Library...。最后，在使用动态链接库.so文件的Java类中加入如下代码：**

```java
static{
	try{
		System.setProperty("jna.library.path", "libs");
	}catch(Exception e){
		e.printStackTrace();
	}
}
```

**这样便可以在IntelliJ IDEA中通过JNA使用动态链接库了。**

使用JNA的java代码如下：

```
package com.zju;

import com.sun.jna.Library;
import com.sun.jna.Native;

/** Simple example of JNA interface mapping and usage. */
public class JNA4S {

    // This is the standard, stable way of mapping, which supports extensive
    // customization and mapping of Java to native types.

    public interface CLibrary extends Library {
        CLibrary INSTANCE = (CLibrary)
//            Native.loadLibrary((Platform.isWindows() ? "msvcrt" : "c"),
        		Native.loadLibrary("helloCPP",//寻找的动态链接库实际为libhelloCPP.so
                               CLibrary.class);

//        void printf(String format, Object... args);
        void show();
        int add(int a,int b);
    }

    public static void main(String[] args) {
    	CLibrary.INSTANCE.show();
    	System.out.println(CLibrary.INSTANCE.add(4, 6));
    }
}
```

## 三.参考资料

* [JNI的替代者—使用JNA访问Java外部功能接口](http://www.cnblogs.com/lanxuezaipiao/p/3635556.html)
* [linux下生成动态链接库so文件](http://blog.csdn.net/lwhsyit/article/details/2824589)
* [Linux下gcc编译生成动态链接库*.so文件并调用它](http://blog.sina.com.cn/s/blog_54f82cc20101153x.html)
* [Linux下编译链接动态库](http://hbprotoss.github.io/posts/linuxxia-bian-yi-lian-jie-dong-tai-ku.html)
* [makefile学习经验（三）----编译生成动态库文件（方式一）](http://www.cnblogs.com/huochangjun/archive/2012/09/04/2670539.html)
* [makefile学习经验（四）----编译生成动态库文件（方式二）](http://www.cnblogs.com/huochangjun/archive/2012/09/05/2671315.html)






