---
published: false
layout: post
categories: framework
tags: power libsuspend
---

* content
{:toc}

>
本文基于Android 7.1  
相关源码路径  
/framework/base/services/core/jni/com_android_server_PowerManagerService.cpp  
/system/core/libsuspend/*  

### 关于libsuspend
libsuspend的作用是接收JNI的调用，在合适的时候往/sys/power/state节点写mem来控制Android系统进入suspend的状态






### libsuspend的调试方法
最好去理解代码的方式是在代码中添加Log进行调试，可以使用下面的方法对libsuspend进行调试  
1）修改代码后，生成libsuspend库,输出到out/target/produc..._arm/STATIC_LIBRARIES/libsuspend_intermediates/libsuspend.a
mmm system/core/libsuspend/  
2）com_android_server_PowerManagerService.cpp需要依赖此静态库，生成service.jar
mmm framework/base/service/   
3）将变更push到手机端，是的修改生效
adb root  
adb remount  
adb sync system  









