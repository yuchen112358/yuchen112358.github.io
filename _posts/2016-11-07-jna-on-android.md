---
layout:     post
title:      "JNA On Android"
subtitle:   "——在AndroidStudio中使用JNA"
date:       2016-11-07
author:     "yuchen"
header-img: "img/post-bg-java.jpg"
tags:
    - Java
    - JNA
    - Android
---

## 一.编译Android平台上所需的JNA

使用[JNA官方](https://github.com/java-native-access/jna)提供的源码进行编译，得到的jna.jar在Android平台上运行时总会出错，这里使用的是pakoito修改后的jna源码。  

#### 1.下载jna源码

`git clone https://github.com/pakoito/jna.git`

#### 2.编译jna源码

1）在jna目录下创建jna.sh，该shell脚本内容如下：  

```bash
#!/bin/bash

#将SDK/NDK tools放到环境变量PATH中(native/Makefile用到此路径)  
export ANDROID_NDK=~/Android/Sdk/ndk-bundle
export ANDROID_SDK=~/Android/Sdk
PATH=$PATH:$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin:$ANDROID_SDK/tools  
#设置环境变量NDK_PLATFORM(native/Makefile用到此变量)  
export NDK_PLATFORM=~/Android/Sdk/ndk-bundle/platforms/android-21
ant -Dos.prefix=android-arm dist
```

2）使用jna.sh脚本进行编译  

`sh jna.sh`


#### 3.使用编译得到的jar文件

* 进入jna/dist,复制[jna-min.jar](/resources/jna-min.jar)到AndroidStudio项目的libs目录下;
* 在AndroidStudio项目的/src/main目录下创建jniLibs/armeabi、jniLibs/armeabi-v7a和jniLibs/x86三个目录；
* 解压[android-arm.jar](/resources/android-arm.jar)文件，将得到的libjnidispatch.so文件复制到jniLibs/armeabi和jniLibs/armeabi-v7a两个目录下（这里armeabi-v7a和armeabi共用android-arm.jar文件中的libjnidispatch.so）；
* 解压[android-x86.jar](/resources/android-x86.jar)文件，将得到的libjnidispatch.so文件复制到jniLibs/x86目录下。

## 二.在AndroidStudio中使用JNA

#### 1.在AndroidStudio中新建工程UsingJNA

* 新建工程名称为UsingJNA,并选择Basic Activity；
* 工程建立好后，进入1:Project的侧边栏，选择Project显示,然后将第一部分生成的jna/dist目录下的jna-min.jar复制到项目的app/libs目录下;
* 并在AS中右击jna-min.jar，选择 Add As Library...

#### 2.配置动态链接库使用环境

* 进入1.Project的侧边栏，选择Project显示；
* 在app/src/main下面右键，New->Directory新建文件夹jniLibs，然后再以同样的方式在jniLibs目录下创建armeabi、armeabi-v7a和x86三个目录；
* 解压android-arm.jar文件，将得到的libjnidispatch.so文件复制到jniLibs/armeabi和jniLibs/armeabi-v7a两个目录下（这里armeabi-v7a和armeabi共用android-arm.jar文件中的libjnidispatch.so）；
* 解压android-x86.jar文件，将得到的libjnidispatch.so文件复制到jniLibs/x86目录下。

#### 3.新建C/C++程序，编译生成.so动态链接库

* 1）在任意目录下创建一个名为jni的目录（此处创建的目录名必须为jni）；
* 2）在jni目录下创建目录src，将C++源码以及头文件放在src目录下；
* 3）在jni目录下创建Android.mk文件，其内容如下：

```
# 查找./src目录下的所有.cpp文件,作为编译源文件
LIBSRCS:=$(shell find src -name '*.cpp')
# 本地路径为当前目录
LOCAL_PATH := $(call my-dir)
# 清理变量，把CLEAR_VARS变量所指向的脚本文件包含进来
include $(CLEAR_VARS)

# 设置编译得到的动态链接库名称，实际得到的为libdemo.so
LOCAL_MODULE := demo
# 编译源文件为./src目录下的所有.cpp文件
LOCAL_SRC_FILES := $(LIBSRCS)

# 设置include目录
LOCAL_C_INCLUDES += ./src

# 设置编译选项
# LOCAL_CFLAGS :=
# LOCAL_CFLAGS +=
# 设置链接选项
# LOCAL_LDFLAGS +=

# 设置为编译动态链接库
include $(BUILD_SHARED_LIBRARY)

```

* 4）在jni目录下创建Application.mk文件，其内容如下：

```
# 通知构建系统为每个支持的ABI架构生成一个.so文件，
# 不写此句，默认只生成armeabi目录下的.so文件，即ARMv5的.so文件
APP_ABI := all
# 如果c++程序中使用了STL库，如string类型，需要设置下面两条语句
APP_CFLAGS += -fexceptions  
APP_STL := gnustl_static 
```

* 5）在jni目录下执行`ndk-build`命令进行编译；
* 6）执行`ndk-build`命令后，在jni父目录下将生成libs目录和obj目录
* 7）libs目录下的各个子目录中的.so文件即为所需的动态链接库
* 8）将生成的各个子目录下的.so动态链接库分别复制到对应的jniLibs/armeabi、jniLibs/armeabi-v7a和jniLibs/x86目录下（注意：如果此处只生成了armeabi目录下的.so动态链接库，最好将其复制到jniLibs/armeabi和jniLibs/armeabi-v7a两个目录中，因为你不知道系统执行时会选择armeabi目录还是armeabi-v7a目录）。


###### 3.1 补充说明
**I.在第1）步中所建目录也不必须为jni，若所建目录名为其他名称，则所执行的编译命令为`ndk-build V=1 NDK_PROJECT_PATH=. APP_BUILD_SCRIPT=Android.mk NDK_APPLICATION_MK=Application.mk`,并且此时生成的`libs`和`obj与src在同一个目录下。`**

**II.第2）步中src目录下的C++源码示例[^1]如下：**

a) 使用一些C++的基本语法创建JNIDemo类，作为真正的数据处理类

```cpp
//JNIDemo.h
#ifndef JNA_ON_AS_JNIDEMO_H
#define JNA_ON_AS_JNIDEMO_H

class JNIDemo {
private:
    int num;
public:
    void print();
    JNIDemo();
    JNIDemo(int a);
    int add(int b);
};


#endif //JNA_ON_AS_JNIDEMO_H
```

```cpp
//JNIDemo.cpp
#include "JNIDemo.h"
#include <stdio.h>
#include <stdlib.h>

void JNIDemo::print() {
    printf("JNIDemo from JNA.\n");
}

JNIDemo::JNIDemo() {
    num = 0;
}

JNIDemo::JNIDemo(int a) {
    num = a;
}

int JNIDemo::add(int b) {
    return num + b;
}
```

b) 使用STL库创建JNIStringDemo类，以对字符串进行处理

```cpp
//JNIStringDemo.h
#ifndef JNA_ON_AS_JNISTRINGDEMO_H
#define JNA_ON_AS_JNISTRINGDEMO_H

#include <string>
using namespace std;

class JNIStringDemo {
    string data;
public:
    JNIStringDemo(const char *);
    string substr(int begin, int end);
};


#endif //JNA_ON_AS_JNISTRINGDEMO_H
```

```cpp
//JNIStringDemo.cpp
#include "JNIStringDemo.h"

JNIStringDemo::JNIStringDemo(const char *str) {
    data = str;
}

string JNIStringDemo::substr(int begin, int end) {
    return data.substr(begin, end);
}
```

c) 创建JNIDemoInterface类，其作用是对以上两个类的操作进行封装，以作为之后与Java对接的接口

```cpp
//JNIDemoInterface.h
#ifndef JNA_ON_AS_JNIDEMOINTERFACE_H
#define JNA_ON_AS_JNIDEMOINTERFACE_H

#ifdef   __cplusplus   
extern   "C"{   
#endif  

void * initialize(int a);
void   run(void *);
void * finalize(void *);
int    add(void * demo, int b);


void * initializeString(char * str);
const char * substrString(void * demoString, int begin, int end);
void finalizeString(void * demoString);

#ifdef   __cplusplus   
}   
#endif

#endif //JNA_ON_AS_JNIDEMOINTERFACE_H
```

```cpp
//JNIDemoInterface.cpp
#include "JNIDemoInterface.h"
#include "JNIDemo.h"
#include "JNIStringDemo.h"


void *initialize(int a) {
    JNIDemo * demo = new JNIDemo(a);
    return (void *)demo;
}

void run(void *pVoid) {
    ((JNIDemo *)(pVoid)) -> print();
}

int add(void *pVoid, int b) {
    return ((JNIDemo *)(pVoid))->add(b);
}

void *finalize(void *pVoid) {
    delete ((JNIDemo *)(pVoid));
}

void *initializeString(char *str) {
    JNIStringDemo * strDemo = new JNIStringDemo(str);
    return strDemo;
}

const char *substrString(void *demoString, int begin, int end) {
    return ((JNIStringDemo*)demoString)->substr(begin, end).c_str();
}

void finalizeString(void *demoString) {
    delete ((JNIStringDemo *)demoString);
}
```

#### 4. 将.so动态链接库中的函数映射为对应的Java接口

在src/main/java/io/github/wzzju/usingjna下新建包CLibrary，并在该包中新建interface `JNACPPLibrary`，其内容如下：

```java
package io.github.wzzju.usingjna.CLibrary;

import com.sun.jna.Native;
import com.sun.jna.Pointer;

/**
 * Created by yuchen on 16-11-7.
 */

public interface JNACPPLibrary extends Library {
    JNACPPLibrary INSTANCE = (JNACPPLibrary)
            Native.loadLibrary("demo", JNACPPLibrary.class);

    Pointer initialize(int a);

    void run(Pointer demo);

    int add(Pointer demo, int b);

    void finalize(Pointer demo);

    Pointer initializeString(String str);

    String substrString(Pointer demoString, int begin, int end);

    void finalizeString(Pointer demoString);
}

```

当然，我们也可以建立仅使用.so动态链接库中部分函数的Java接口，示例如下：  

1）JNIDemoMapping接口对应于C++类JNIDemo的功能：

```java
package io.github.wzzju.usingjna.CLibrary;

import com.sun.jna.Library;
import com.sun.jna.Native;
import com.sun.jna.Pointer;

/**
 * Created by netiger on 12/23/15.
 */
public interface JNIDemoMapping extends Library {
    JNIDemoMapping INSTANCE = (JNIDemoMapping)
            Native.loadLibrary("demo", JNIDemoMapping.class);

    Pointer initialize(int a);

    void run(Pointer demo);

    int add(Pointer demo, int b);

    void finalize(Pointer demo);


}
```

2）JNIStringDemoMapping接口对应于C++类JNIStringDemo的功能：

```java
package io.github.wzzju.usingjna.CLibrary;

import com.sun.jna.Library;
import com.sun.jna.Native;
import com.sun.jna.Pointer;

/**
 * Created by netiger on 12/23/15.
 */
public interface JNIStringDemoMapping extends Library {
    JNIStringDemoMapping INSTANCE = (JNIStringDemoMapping)
            Native.loadLibrary("demo", JNIStringDemoMapping.class);

    Pointer initializeString(String str);

    String substrString(Pointer demoString, int begin, int end);

    void finalizeString(Pointer demoString);
}

```

#### 5. 通过Java接口使用.so动态链接库

网上有有资料说使用jna时需要在一个Activity的java文件里面加上以下代码，但是经过测试，以下代码并不需要添加。

```java
static{
	try{
		System.setProperty("jna.library.path", "/system/lib");
		System.loadLibrary("jnidispatch");
	}catch(Exception e){
		e.printStackTrace();
	}
}
```

**上述代码解释说明：**  

1）`System.setProperty("jna.library.path", "/system/lib");`表示从/system/lib目录下加载要调用的so文件，如果要调用的so文件在多个路径下，则多个路径用冒号分隔,如:`System.setProperty("jna.library.path", "/system/lib:/usr/lib");`表示jna会从/system/lib目录和/usr/lib目录下查找so文件。如果so文件是用户自己编译的，则只需要将so文件放在jniLibs/armeabi目录下就行了，jna会自动查找这个目录下的so文件，不在这个目录下的so文件需要将路径设置到jna.library.path属性里面，通过`System.setProperty("jna.library.path", "/system/lib:/usr/lib:so文件所在的路径");`设置即可。  

2)System.loadLibrary("jnidispatch");这条语句是加载jna运行所必需的so文件,经过测试这条语句现在不需要再写了。  


###### 5.1 示例如下：

在使用.so动态链接库的Java类（如MainActivity.java）中添加如下函数：

```java
//使用JNIDemoMapping接口示例
public static int JNACalculateFunction(int a, int b) {
	Pointer demo = JNIDemoMapping.INSTANCE.initialize(a);
	int c = JNIDemoMapping.INSTANCE.add(demo, b);
	JNIDemoMapping.INSTANCE.finalize(demo);
	return c;
}
//使用JNIStringDemoMapping接口示例
public static String JNASubstr(String oriString, int begin, int end) {
	Pointer demo = JNIStringDemoMapping.INSTANCE.initializeString(oriString);
	String ret = JNIStringDemoMapping.INSTANCE.substrString(demo, begin, end);
	JNIStringDemoMapping.INSTANCE.finalizeString(demo);
	return ret;
}
```

使用上述定义的函数（在FloatingActionButton被点击时触发调用）：

```java
FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
fab.setOnClickListener(new View.OnClickListener() {
	@Override
	public void onClick(View view) {
		try {
			//使用JNIDemoMapping接口示例
			String strDis = "6 + 6 = ";
			strDis += Integer.toString(JNACalculateFunction(6, 6));
			//使用JNIStringDemoMapping接口示例
			String strDisplay = JNASubstr("1234567890", 3, 5);
			Snackbar.make(view, strDis+" subString = " + strDisplay, Snackbar.LENGTH_LONG)
					.setAction("Action", null).show();
		} catch (Throwable t) {
			t.printStackTrace();
		}
	}
});
```

## 三、附录

#### 1.ndk环境的配置步骤：

* 1）.去官方网站http://developer.android.com/tools/sdk/ndk/index.html下载最新的NDK开发包;
* 2）.解压下载的开发包，放Android SDK所在路径的ndk-bundle目录下（如`~/Android/Sdk/ndk-bundle`）；
* 3）.编辑～/.bashrc文件，向其中添加内容如下：

```bash
#android-ndk-gcc
export NDK_ROOT=～/Android/Sdk/ndk-bundle
export NDKROOT=～/Android/Sdk/ndk-bundle
export SYSROOT=$NDKROOT/platforms/android-19/arch-arm
export NDKTOOL=$NDKROOT/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin
export PATH=$NDKROOT:$PATH
export PATH=～/Android/Sdk/tools:$PATH
export PATH=～/Android/Sdk/build-tools/19.1:$PATH
alias ndk-gcc='$NDKTOOL/arm-linux-androideabi-gcc-4.9 --sysroot=$SYSROOT'
alias ndk-g++='$NDKTOOL/arm-linux-androideabi-g++ --sysroot=$SYSROOT'
alias ndk-nm='$NDKTOOL/arm-linux-androideabi-nm'
alias ndk-objdump='$NDKTOOL/arm-linux-androideabi-objdump'
```

* 4）.保存～/.bashrc文件，并执行`. ~/.bashrc`命令。

#### 2.查看.so动态链接库中的函数信息

执行 `ndk-nm -D xxx.so`或`nm -D xxx.so`即可。

#### 3.源码

* [UsingJNA](https://github.com/yuchen112358/UsingJNA.git)  
* [AndroidSoMaker](https://github.com/yuchen112358/AdroidSoMaker.git)

#### 4.关于动态链接库的编译命令（ndk-build）

###### 4.1 构建命令(当父目录名为jni时只需执行如下命令)

`ndk-build`或`ndk-build V=1`

**附(清除命令)：**`ndk-build clean`

###### 4.2 构建命令(当父目录名不为jni时需要执行如下命令)

`ndk-build V=1 NDK_PROJECT_PATH=. APP_BUILD_SCRIPT=Android.mk NDK_APPLICATION_MK=Application.mk`

**附(清除命令)：**`ndk-build clean NDK_PROJECT_PATH=. APP_BUILD_SCRIPT=Android.mk NDK_APPLICATION_MK=Application.mk`

###### 4.3 ndk-build命令的参数选项

所有给ndk-build的选项都会直接传给GNU Make，由make运行NDK的编译脚本。几个常见调用方式如下：  

`ndk-build` —— 编译  

`ndk-build clean` —— 清掉二进制文件  

`ndk-build V=1` —— 执行ndk-build且打印出它所执行的详细编译命令  

`ndk-build -B` —— 强制重新编译  

`ndk-build -B V=1` —— -B 和 V=1 的组合  

`ndk-build NDK_DEBUG=1` —— 编译为可调试版的二进制文件  

`ndk-build NDK_DEBUG=0` —— 编译为release版  

`ndk-build NDK_LOG=1` —— 打印出内部的NDK日志信息（用于调试NDK自己）  

`ndk-build NDK_APPLICATION_MK=<文件路径>` —— 用这里指定的路径寻找Application.mk文件  

`ndk-build -C <project路径>` ——  先cd进入`<project路径>`，然后执行ndk-build  

**ndk-build的实质:**ndk-build 其实就是对GNU Make的封装，它的目的是调用正确的NDK编译脚本，它等价于 make -f $NDKROOT/build/core/build-local.mk [参数]

## 四、参考资料

* [Android Studio里使用JNA](http://blog.csdn.net/sadcup/article/details/50386478)
* [Android NDK编译带STL的 C/C++ 程序](http://blog.csdn.net/langeldep/article/details/6948374)
* [Android.mk详解](https://huangxiankui.github.io/blog/index.html?diaries/ARM/Android.mk%E8%AF%A6%E8%A7%A3.md)
* [NDK Build 用法（NDK Build）](http://blog.csdn.net/smfwuxiao/article/details/8523087)
* [JNA移植到android上](http://blog.csdn.net/simpsonwang/article/details/8612225)


[^1]: [示例代码出处](https://github.com/sadcup/AndroidDevelopment/tree/master/JNAonAS/app/src/main/jni)

























