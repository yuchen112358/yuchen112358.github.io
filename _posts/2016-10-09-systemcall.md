---
layout:     post
title:      "64位ARM Linux增加系统调用的方法"
subtitle:   "提供钩子函数（可外部修改系统调用的功能）"
date:       2016-10-12
author:     "yuchen"
header-img: "img/post-bg-java.jpg"
tags:
    - ARM Linux
    - System Call
---

## 1.在arch/arm/kernel目录下新建sys_onframe.c和sys_onframe.h文件，内容分别如下：

```cpp
//sys_onframe.c
#include <linux/linkage.h>
#include <linux/kernel.h>
#include <linux/random.h>
#include "sys_onframe.h"

asmlinkage long sys_original(unsigned long time_df){
  printk (KERN_EMERG "time_df = %lu\n",time_df);
  return get_random_int()*4;
}


static HOOK_FUNC sys_onframe_point = sys_original;//接收钩子的函数指针

//该函数主要供调频模块调用,获取原钩子函数地址
void* get_hook_func(void){
	return sys_onframe_point;
}
EXPORT_SYMBOL(get_hook_func);//设置该函数对模块可见


//该函数主要供调频模块调用实现钩子重置
void set_hook_func( void *funcp){
	sys_onframe_point = (HOOK_FUNC)funcp;
}
EXPORT_SYMBOL(set_hook_func);//设置该函数对模块可见

asmlinkage long sys_onframe(unsigned long time_df){
	return sys_onframe_point(time_df);
}
```

```cpp
//sys_onframe.h
#ifndef SYS_ONFRAME_H
#define SYS_ONFRAME_H
typedef long (*HOOK_FUNC)(unsigned long time_df);//定义钩子指针类型
asmlinkage long sys_original(unsigned long time_df);
void* get_hook_func(void);
void set_hook_func( void *funcp);
asmlinkage long sys_onframe(unsigned long time_df);
#endif
```

## 2.修改arch/arm/kernel/Makefile文件

在`obj-y :=...`后面加上sys_onframe.o，以将新建的.c文件编译。

## 3.修改arch/arm/include/uapi/asm/unistd.h文件

在`#define __NR_finit_module		(__NR_SYSCALL_BASE+379)`下方添加如下内容：

```cpp
......
#define __NR_process_vm_writev		(__NR_SYSCALL_BASE+377)
#define __NR_kcmp			(__NR_SYSCALL_BASE+378)
#define __NR_finit_module		(__NR_SYSCALL_BASE+379)
/* 下面是添加的内容 */
#define __NR_onframe                 (__NR_SYSCALL_BASE+380)
```

## 4.修改arch/arm/kernel/calls.S文件
在`CALL(sys_finit_module)`内容下面添加如下内容：

```cpp
......
/* 375 */	CALL(sys_setns)
		CALL(sys_process_vm_readv)
		CALL(sys_process_vm_writev)
		CALL(sys_kcmp)
		CALL(sys_finit_module)
/* 下面是添加的内容 */
/* 380 */     	CALL(sys_onframe) 	
```

## 5.修改arch/arm/include/asm/unistd.h文件

修改`#define __NR_syscalls  (380)`为`#define __NR_syscalls  (384)`  
注意，按照常理说应该改为381，但是由于padding的存在（（__NR_syscalls + 3) & ~3)，而((381 + 3) & ~3)=384   


`.equ syscalls_padding, ((NR_syscalls + 3) & ~3) - NR_syscalls`存在于arch/arm/kernel/calls.S文件中

## 6.修改系统调用功能的测试内核模块

```cpp
//内核模块代码
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/moduleparam.h>
//当前头文件的目录在$KDIR/include下,所以为../arch/arm/kernel/sys_onframe.h
#include <../arch/arm/kernel/sys_onframe.h>

// #ifndef __NR_onframe
// #define __NR_onframe                  (__NR_SYSCALL_BASE+380)
// #endif

static HOOK_FUNC original_call;

asmlinkage long new_sys_onframe(unsigned long time_df)
{
   printk("New onframe function:%lu\n",time_df+1);
   return original_call(time_df);
}

static int __init sys_onframe_init(void)
{
    original_call =(HOOK_FUNC)get_hook_func();
    set_hook_func((void*)new_sys_onframe);
    return 0;
}

static void __exit  sys_onframe_exit(void)
{
   // Restore the original call
   set_hook_func((void*)original_call);
}

module_init(sys_onframe_init);
module_exit(sys_onframe_exit);

MODULE_AUTHOR("Wang Zhen <wzzju@mail.ustc.edu.cn>");
MODULE_LICENSE("GPL");
```

```cpp
# Makefile
TARGET = changeSyscall
obj-m := $(TARGET).o
KDIR := /work/Odroid/AndroidSRC/kernel/linux
PWD := $(shell pwd)

CC = arm-eabi-gcc

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```



## 7.系统调用测试代码

1)用系统提供的 syscall() 函数：

```cpp
#include <linux/unistd.h>
#include <sys/syscall.h>
#include <stdio.h>
// #ifndef __NR_onframe
// #define __NR_onframe                  (__NR_SYSCALL_BASE+380)
int main() {
    long res;
    res = syscall(380,6789);
    printf("%ld\n",res );
    return 0;
}
```

2)嵌入汇编代码，用 r0 传参数：

```cpp
#include <stdio.h>
#define sys_call() {__asm__ __volatile__ ("swi 0x900000+380\n\t");} while(0)
int main(void) {
    sys_call();
    return 0;
}
```


