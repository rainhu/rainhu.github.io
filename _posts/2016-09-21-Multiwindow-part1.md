---
layout: post
title: Multiwindow in Android N (１)
---
##背景
　　Android N 引入了一个新的功能叫做多窗口支持(Multi-Window)，可以使多个Activity同时展现在屏幕上，并且可以在两个Activity之间通过拖拽的方式进行交互。具体包括三个功能点：屏幕分割(Split-screen)，自由模式（Freeform），画中画(Picture-in-picture)。
####屏幕分割(Split-screen)
这个功能是多窗口的基本体现方式，屏幕被均分同时显示两个Activity，用户无法调整Activity大小
####自由模式（Freeform）
自由模式可以认为是屏幕分割的加强版，用户可以自由地调整两个Activity显示区域的大小。这种模式在平板这样的大屏幕设备上非常有作用。
在运行 Android N 的手机和平板电脑上，用户可以并排运行两个APP，或者处于分屏模式时一个APP位于另一个APP之上。用户可以通过拖动两个APP之间的分隔线来调整 APP。
####画中画（Picture-in-picture）
此功能更多针对于TV。允许Android设备在用户操作其他应用的时候，仍旧播放video内容。在 Android TV 设备上，APP可以将自身置于画中画模式，从而让它们可以在用户浏览或与其他APP交互时继续显示内容。

##如何启用Multi-window?
- 任意打开一个应用，长按Recent
- 点击Recent，然后通过拖动其中一个Task到指定位置启动

##设备厂商如何开启Multi-window?
1. 启用屏幕分割(split-screen)模式
如果需要启用多窗口，需要在frameworks/base/core/res/res/values/config.xml 中设置参数config_supportsMultiWindow = true ，但是如果声明了low_ram为true，则这个flag无效。
```
ActivityManager.supportsMultiWindow(){  
  return !isLowRamDeviceStatic()  
          && Resources.getSystem().getBoolean(  
             com.android.internal.R.boolen.config_supportsMultiWindow);  
}  
```
2. 启用自由模式(Freeform)
除了要满足上面屏幕分割的条件，还需要启用
PackageManager#FEATURE_FREEFORM_WINDOW_MANAGEMENT并满足config_freeformWindowManagement 为true

3. 画中画(Picture-in-picture)
在满足屏幕分割的要求以后，还需要满足PackageManager#FEATURE_PICTURE_IN_PICTURE

