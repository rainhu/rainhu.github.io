---
layout: post
title: Adb device no permission的原因和解决方法
categories: common
tags: adb tools
published: true
---

* content
{:toc}

Ubuntu连上Android手机之后，调用adb devices出现如下的错误
![问题描述](https://github.com/rainhu/rainhu.github.io/raw/master/_assets/1.png)\

### 原因

lsusb查看usb设备是否已经连上，确定其已经连上ubuntu系统，且挂在在bus 001上且device Id = 095
那么通过ls –al /dev/bus/usb/001/095查看这个设备的使用权限。发现root用户才可以读写，所以在普通用户状态下调用adb devices会出现没有权限的情况





![原因分析](https://github.com/rainhu/rainhu.github.io/raw/master/_assets/2.png)

### 解决方案
#### 临时解决方案 
a) 将adb放到/user/bin/adb  
b) sudo –s 切换到root用户  
c) adb kill-server;  
   adb start-server;  
   adb devices  


#### 永久性解决方案
a）添加udev规则 
通过添加udev规则让普通用户也能访问。   
sudo vi /etc/udev/rules.d/51-android.rules  
添加下面   
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", MODE="0666", GROUP="plugdev"   
其中idVendor通过lsusb获取，下面黄色部分就是要填的值   
 
Bus 002 Device 002: ID 2109:8110 VIA Labs, Inc. Hub    
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub    
Bus 001 Device 003: ID 2109:2811 VIA Labs, Inc. Hub    
Bus 001 Device 014: ID <font color=yellow>0e8d</font>:201d MediaTek Inc.     
Bus 001 Device 005: ID 04b4:4042 Cypress Semiconductor Corp.    
Bus 001 Device 004: ID 0bda:0157 Realtek Semiconductor Corp. Mass Storage Device   
Bus 001 Device 002: ID 17ef:6019 Lenovo     
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub   
 
b)重启udev服务   
sudo udevadm control --reload-rules  
sudo service udev restart   
 
 
c)重启adb server   
sudo adb kill-server   
sudo adb start-server   
sudo adb devices
