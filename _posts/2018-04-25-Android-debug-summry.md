---
published: true
layout: post
categories: tools
tags: debug tools
comments: true
---
* content
{:toc}

> 本文记录在工作中所用到的一些常见的命令使用和Debug方式


### Logcat
mar
在eng下调试Camera时候使用adb logcat出现如下的错误  
read: unexpected EOF!  
通过使用  
$ adb logcat -G 1m  
 将logcat的缓存从默认256KB提升到1MB

通过-E来过滤多个标签  
$ adb logcat | grep -E "tag1|tag2"

既在Terminal输出Log同时保存到文件里面   
$ adb logcat | tee a.txt






### 代码调试生效
a)修改了Framework代码之后，快速刷入到手机的方式    
$ adb remount；  
$ cd <ProjectHome>;  
$ adb sync system;   
$ adb reboot;   

b)编译某个模块，快速输入到手机  
前提需要一次整编  
$ cd <ProjectHome>;   
$ . build/envsetup.sh; lunch <ProjectName>; (只要在开始的时候执行一次就可以了)  
$ mmm packages/apps/Calendar/   
$ adb remount;   
$ adb push out/target/producr/<ProjactName>system/app/Calendar/Calendat.apk system/app/Calendar/Calendat.apk   
$ adb reboot 生效  

### 截屏
$ adb shell screencap $(filename)   
$ adb shell screencap /sdcard/screen.png

### 录制视频
$ adb shell screenrecord [options] filename      
$ adb shell screen --verbose /sdcard/demo.mp4

### am命令
发送广播改变电池电量和温度  
$ adb shell am broadcast -a android.intent.action.BATTERY_CHANGED –ei temperature 300 –ei level 50 //新Android版本貌似已经不能用

可以用下面的方式设置电量
让手机电量显示百分百： adb shell dumpsys battery set level 100  
让手机电量显示1： adb shell dumpsys battery set level 1  

启动activity  
$ adb shell am start-activity com.android.calculator2/com.android.calculator2.Calculator .

### pm命令
$ adb shell pm list packages [PackageName] 列出系统安装所有应用的包名  
$ adb shell pm list packages -f [PackageName] 列出APK安装的位置  
$ adb shell pm dump [PackageName] dump APK的Activity等  
$ adb shell pm list permissions -g -d  查看所有danguerous的权限   
$ adb shell pm [grant| revoke] <permission-name> ... 授予权限   

### 电脑端和手机端文件传输 .
$ adb push /home/file.txt /tmp/file.txt 将PC端home路径下的file.txt复制到手机tmp目录下   
$ adb pull /tmp/file.txt /home/file.txt 将手机tmp目录下file.txt复制到PC端home目录下  


### dump系统服务
$ adb shell service list 列出所有系统服务
然后通过dumpsys media.camera 打印media.camera服务的信息  

$ adb shell dumpsys meminfo 打印内存信息
$ adb shell dumpsys SurfaceFlinger 显示当前应用的包名    
$ adb shell dumpsys activity  
$ adb shell dumpsys cpuinfo CPU  
$ adb shell dumpsys battery    
$ adb shell dumpsys window（最后部分可以看到分辨率的信息）
有些service能够接收额外的参数，我们可以使用-h查看帮助信息。   
$ adb shell dumpsys package -h  
$ adb shell dumpsys activity -h  
  如adb shell dumpsys activity o 能够输出oom的值
    adb shell dumpsys activity p 能够打印运行中的进程

### 查找文本内容
$ grep -r "adb"

### 查找文件
$ find . -name *.text

### git图形化
进入某个仓库  
$ cd framework/base/  
$ gitk  
如果提示需要安装gitk安装即可

### 阅读代码工具
Visual Studio Code轻量级的文本编辑、代码阅读工具   
$ code framework/base 就可以打开framework/base下的所有文件

### Selinux问题快速定位
通过下面命令暂时关闭selinux来确定此问题是否是由于selinux引起的   
adb shell setenforce 0

通过adb logcat | grep avc  来抓取selinux deny的log
将抓取出来的log访问avc.txt以后可以通过
audit2allow -i avc.txt 来输出推荐的selinux添加的规则  

### 当前运行进程ANR Log的获取
a)$ adb shell  
b)kill -3 <Pid>  
c)在/data/anr下面找到trace log
  
### Java源码中打印堆栈
Log.d(TAG,Log.getStackTraceString(new Throwable()));  

### C++源码中打印堆栈
#include <utils/CallStack.h>    
 ...  
 CallStack stack;    
 stack.update();  
 stack.dump();   

### C(Kernel)中打印堆栈
dump_stack();   

### adb wifi连接
可以对usb连接相关的问题进行调试，如OTG  
a)电脑和手机连接同一个局域网   
b)手机usb线连上电脑， adb devices   
c) adb usb   
d)adb  tcpip 5555   
e)找到手机的IP地址，然后 adb connect 192.168.0.2:5555  
