---
layout: post
title: Android.mk常用指南
categories: build
tags: makefile
published: true
comments: true
---
* content
{:toc}

> 本文基于Android O平台进行分析


Android.mk对于熟悉Android源码的人来说并不陌生，虽然Google开始逐步用Android.bp来替换Android.mk，但是其实质并没有发生什么变化，只是又在Android.mk的基础上又封装了一层。  







### 静态库与动态库
程序要运行一般会经过编译->链接->加载->运行的过程，在链接过程中连接器将从库文件取得所需的代码，复制到生成的可执行文件中，这样的库称为静态；

静态库会打包到目标代码中，而动态库只是放在环境变量所在的路径下，当应用程序需要的时候再加载，一般可以通过dlopen或者dlclose的方式调用

C/C++里面，静态库和动态库的形式不一样，一个是.a，另一个是.so；

Java库生成的是jar包，对应两种类型的库，一个是通过
那么，BUILD_JAVA_LIBRARY 与BUILD_STATIC_JAVA_LIBRARY的区别是什么？
1. BUILD_JAVA_LIBRARY编译出来的jar包，里面是DEX格式的文件，如果用户想用这个jar包放到Eclipse来做Android APP的开发，Eclipse是不认识这种格式的文件的，通常会报错：Conversion to Dalvik format failed with error 1；
2. 而BUILD_STATIC_JAVA_LIBRARY编译出来的jar包，里面每个java文件对应的class文件都单独存在，顾名思义，每个java文件里面用到的变量都被静态编译到了class内部，这种格式的jar包可以在Eclipse/AndroidStudio里面导入并正常使用，但是可能存在一定的兼容性隐患。
二者的区别在于静态JAVA库是由.class文件打包而成JAR包，它在任何一个JAVA虚拟机上都可以运行；而共享JAVA库则是在静态库的基础上进一步打包成的.dex文件，众所周知，dex是在android系统上所使用的文件格式。


### 实例分析
在OEM/ODM项目开发过程中，经常会遇到需要编译Native库，Jar文件，或者apk

#### Native库
输出so文件，根据32位、64位输出到system/lib(64)或者vendor/lib(64)下

有so的源码生成libmymodule.so
```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
#optional表示任何版本(eng,userdebug,user)都会编译
#LOCAL_MODULE_TAGS可选择eng,user,tests

#指定源码文件所在路径
LOCAL_SRC_FILES += \
     test.c \
#指定头文件所在路基
LOCAL_C_INCLUDES += \
    $(TOP)/system／core/include \
    $(LOCAL_PATH)/inclue


LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := mymodule #共享库的名字，生成的库会变成libmymodule.so
#LOCAL_MODULE_PATH := $(TARGET_OUT)/lib  #输出到指定的位置，target_out是 /system
#LOCAL_SDK_VERSION := current  #模块依赖的sdk的版本
#LOCAL_CERTIFICATE := platform   #构建需要平台签名的module
include $(BUILD_SHARED_LIBARY)  #构建C共享库，生成so库,输出到/out/target/product/../obj/SHARED_LIBRARIES/

```



如果需要预置没有源码的so
```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := mymodule
#LOCAL_MULTILIB := 32 如果预编译的是32位的so
LOCAL_SRC_FILES := prebuiltmodule.so #如果是32位的可以用LOCAL_SRC_FILES_32
LOCAL_MODULE_CLASS := SHARED_LIBRARIES
LOCAL_MODULE_SUFFIX := .so
include $(BUILD_PREBUILT)
```







#### Java库


Android.mk编译静态jar
```makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES := \
  $(call all-subdir-java-files)

LOCAL_MODULE := myjavalib
LOCAL_MODULE_CLASS := JAVA_LIBRARIES
LOCAL_DX_FLAGS := --core-library

LOCAL_JACK_ENABLED := disabled  #禁用jack
include $(BUILD_STATIC_JAVA_LIBRARY)
```
编译出来的jar包在
out/target/common/obj/JAVA_LIBRARIES/myjavalib_intermediates/javalib.jar

这个通过下面的方式被引用
   LOCAL_STATIC_JAVA_LIBRARIES := \


Android.mk编译动态jar
```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := myjni

#引用system/lib64中的so库，需要在out/target/product/xxx/obj/JAVA_LIBRARIES有生成对应的so
#LOCAL_JNI_SHARED_LIBRARIES := \
#                libmymodule.so \

#引用预编译好的so库
LOCAL_PREBUILT_JNI_LIBS := \
                @lib/libmymodule.so \

LOCAL_MULTILIB := 32 #来确定是否是32位第三方库

include $(BUILD_JAVA_LIBRARY)
```
编译出来的jar在out/target/product/xxx/system/framework/myjni.jar  

这个编译出来的jar可以通过LOCAL_JAVA_LIBRARIES := myjni 去引用

LOCAL_PACKAGENAME和LOCAL_MODULE不能同时存在



#### APK
使用Android.mk生成APK

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-java-files-under,java)
#如果需要依赖通过BUILD_STATIC_JAVA_LIBRARY编译出来的jar包，会输出到中间路径/out/target/common/object/JAVA_LIBRARIES/的jar，可以直接在下面输入module名称
LOLCAL_STATIC_JAVA_LIBRARIES := \
   android-support-v4
   android-support-v7-appcompat \
   android-support-v7-preference \
   android-support-v14-preference \
   libmyjavalib \
#如果需要依赖通过BUILD_JAVA_LIBRARY，这样编译出来的jar除了生成到/out/target/product/xxx/object/JAVA_LIBRARIES/ 还会copy到/out/target/product/../system/framework/目录下，可以使用下面的变量去指定
#LOCAL_JAVA_LIBRARIES := framework
LOCAL_PACKAGE_NAME := apkdemo

#是否需要平台签名
LOCAL_CERTIFICATE := platform

#对于外面的so,需要经过下面的prebuild，对于/out/target/pruduct/.../obj/SHARED_LIBRARIES下面的，可以直接应用
LOCAL_SHARED_LIBRARIES := \
            libmymodule.so

# 如果指向生成apk，不想生成odex可以启用
#LOCAL_DEX_PREOPT := false
include $(BUILD_PACKAGE)

################ 预制没有源码的jar和so库 ##################################
include $(CLEAR_VARS)

LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := libmyjavalib:libs/myjavalib.jar
LOCAL_PREBUILT_LIBS :=libmymodule:libs/libmymodule.so
LOCAL_MODULE_TAGS := optional
include $(BUILD_MULTI_PREBUILT)
```




#### 如何在native so中启用log?
 1. 在Android.mk中添加  
    LOCAL_LDLIBS := -llog  
 2. 在c文件中添加头文件  
   include "log/log.h"  
 3. 指定Log的tag  
     #define LOG_TAG "mytag"  
 4. 通过下面方式调用  
    ALOGE("this is log \n");    
