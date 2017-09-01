---
layout: post
title: Android O 后台限制的行为变更
category: android_o
---

为了提高设备允许的性能，Android O对应用进行了下面三个限制

- 后台服务限制（Background Service Limitations）：对处于后台运行的应用访问后台服务做了限制
- 广播限制（Broadcast Limitations）：应用不能在manifests中注册大部分的隐式广播
-  后台位置（Background Location）访问限制：限制location的更新频率

### 1. 后台服务限制（Background Service Limitations）
默认清空下，这些限制是针对target api为Android O平台的应用，但是用户也可以在Setting中对应用进行设置来开启上面的限制。

~~~~
如何认为App是处于后台状态？

App有下面的任一行为，可以认为是前台(forceground)的
- 有一个可见的Activity，无论这个Activity是处于started或者paused
- 有一个foreground service
- 有一个前台的App关联到这个App，无论是通过bindService的方式还是通过contentProvider的方式，如IME，Wallpaper service, Notification listener, Voice or text service

如果以上条件都不满足，就认为这个App是处于background.
~~~~

当App处于前台进程的时候，能够自由地创建前台和后台的服务，而当其处于后台的时候，App将有几分钟时间继续创建和使用服务。在这几分钟的时间窗口过去以后，App被将会处于idle状态，此时系统将会停止app的后台服务继续运行，效果相当于调用了其Service.stopSelf()方法。

~~~~
App过了多久会处于idle状态？
//TODO idle mode study
~~~~

在一些环境下，后台进程会有一段时间处于临时的白名单中，在处于白名单的这段时间，系统将不会对应用的后台进程采取限制，下面几种情况，会让其处于白名单状态：
- 处理一个高优先级的FCM信息（Google的推送系统）
- 正在接收广播，如SMS/MMS信息
- 正在执行一个来自notification的PendingIntent
- 在VPN App进入后台之前开启一个VpnService


#### 应用适配方案
如果你的应用需要在App有后台的service需要在应用空闲的时候去做一些操作，那么需要针对Android O进行适配。
Google建议采用下面两种方式
1. 采用JobScheduler来代替后台service执行周期性的工作
2. 如果app需要在处于后台之后创建前台的service，那么可以使用NotificationManager.startServiceInForeground()来代替目前采用的先创建一个后台service，再把它设为前台(android_o_r4没找到这个函数)
3. 使用FCM来唤醒应用（中国区不行）
4. 推迟后台的工作，直到应用回到前台
5. Android 8.0推荐使用新的方法startForegroundService来从后台创建一个新的服务，当服务创建的时候，App有5s的时间取调用服务的startForeground来显示服务的通知，如果系统不调用，系统将停止这个service的运行，并报ANR.


### 2. 广播限制（Broadcast Limitations）
如果一个app注册接收广播，那么每次这个广播发出的时候，这个app的recviver将会消耗系统的资源。当有过多的app注册基于系统事件的广播的时候，将可能在一瞬间造成系统资源不足，从而引起卡顿影响用户体验。

为了减轻这个问题，Android N针对广播做了很多的限制，具体如下
- 当targetSDkVersion >= 24时，manifests声明CONNECTIVITY_ACTION的广播将无法收到通知，但是在代码中动态注册不会受到影响
- 不能发送和接收 ACTION_NEW_PICTURE和ACTION_NEW_VIDEO这两种类型的广播，这个影响所有app, 与targetsdkversion多少无关。

而Android O在Android N的基础上，进一步对隐式广播进行了限制，
- 声明targetSdkVersion >= 26 的app将不能够在manifest中注册隐式广播
- 

~~~~
什么是隐式广播(implicit broadcast)？
这里我们说的广播，无论是显式的还是隐式的，都是Android系统发出来的广播。这些广播可以分为两种，一种是显式广播(explicit broadcast)，一种是隐式广播(implicit broadcast)。如果有 App 安装了新版本，那么ACTION_PACKAGE_REPLACED将会发送给所有注册了此广播的 App，而不是某一个指定的 App，所以我们称它为隐式广播；而像ACTION_MY_PACKAGE_REPLACED这样的只会发送到指定App的，称为显式广播。
~~~~

查看ACTION_MY_PACKAGE_REPLACED是在PackageManagerSevice里面发出来的广播， 可以看到broadcast在发出的时候设了一个setPackage，所以指定了targetPackage。除了此之外，发出广播时候通过setClass或者setConponent等方式设了target的系统广播都可以称为explict broadcast

```Java
PackageManagerService.java

->sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED,
            null, null, 0,
            packageName,
            null, updateUsers);
-> sendPackageBroadcast {
    ...
       if (targetPkg != null) {
            intent.setPackage(targetPkg);
        }

        am.broadcastIntent(null, intent, null,         finishedReceiver,0, null, null, null,android.app.AppOpsManager.OP_NONE,
        null, finishedReceiver != null, false, id);
}           


```
在Android O中，并非所有的隐式广播都不允许在manifest中注册，下面这些广播是必不可少被允许注册：  
- ACTION_LOCKED_BOOT_COMPLETED, ACTION_BOOT_COMPLETED
- ACTION_USER_INITIALIZE, "android.intent.action.USER_ADDED", "android.intent.action.USER_REMOVED"
- "android.intent.action.TIME_SET", ACTION_TIMEZONE_CHANGED, ACTION_NEXT_ALARM_CLOCK_CHANGED
- ACTION_LOCALE_CHANGED
- ACTION_USB_ACCESSORY_ATTACHED, ACTION_USB_ACCESSORY_DETACHED, ACTION_USB_DEVICE_ATTACHED, ACTION_USB_DEVICE_DETACHED
- ACTION_HEADSET_PLUG
- ACTION_CONNECTION_STATE_CHANGED, ACTION_CONNECTION_STATE_CHANGED, ACTION_ACL_CONNECTED, ACTION_ACL_DISCONNECTED
- ACTION_CARRIER_CONFIG_CHANGED, TelephonyIntents.ACTION_*_SUBSCRIPTION_CHANGED, "TelephonyIntents.SECRET_CODE_ACTION"
- LOGIN_ACCOUNTS_CHANGED_ACTION
- ACTION_PACKAGE_DATA_CLEARED
- ACTION_PACKAGE_FULLY_REMOVED
- ACTION_NEW_OUTGOING_CALL
- ACTION_DEVICE_OWNER_CHANGED
- ACTION_EVENT_REMINDER
- ACTION_MEDIA_MOUNTED, ACTION_MEDIA_CHECKING, ACTION_MEDIA_UNMOUNTED, ACTION_MEDIA_EJECT, ACTION_MEDIA_UNMOUNTABLE, ACTION_MEDIA_REMOVED, ACTION_MEDIA_BAD_REMOVAL
- SMS_RECEIVED_ACTION, WAP_PUSH_RECEIVED_ACTION

#### 应用适配方案
那么在Android O之后，应用需要用到这些受限的广播的时候该怎么办呢？

比如你有一个应用在收到系统“充电广播”的时候做备份数据等需要耗电的操作，但是 ACTION_POWER_CONNECTED 属于“implicit”类型广播，所以你无法在Android O中进行使用。

有下面两种方案可以处理：
1. 使用JobScheduler 来替代受限的广播，如手机充电广播 ACTION_POWER_CONNECTED
2. 使用Context.registerReceiver()来注册这些隐式广播，而不是在 AndroidManifest中注册


### 3. 后台位置（Background Location）访问限制
为了提高手机电池续航能力和用户的体验，减少功耗和资源浪费，Google修改了后台位置更新的的频率。如果应用接收来自后台服务的位置更新，则其在 Android O 上接收更新的频率要比旧版本 Android 低。具体地讲，后台服务接收位置更新的频率不能超过每小时几次。不过，当应用位于前台时，位置更新频率不变。

所有在Android O上跑的应用都会受到影响，无论应用的targetSdkVersion是多少。

下面相关的API会受到影响
- Fused Location Provider (FLP)
- Geofencing
- GNSS Measurements
- Location Manager
- Wi-Fi Manager: startScan每一个小时只会请求几次，如果后台服务访问，系统提供的是上次扫描留在cache中的数据

#### 应用适配方案
1. 重新check一下与location相关的代码和逻辑，确保使用最新的location APIs
2. 考虑使用Fused Location Provider (FLP)或者geofencing来处理需要依赖用户当前位置的case
3. 使用被动location监听器，如果有App在前台请求位置的话，会比较快的获取位置信息



参考文档  
1. https://developer.android.com/about/versions/oreo/background.html?hl=zh-cn
2. https://developer.android.com/about/versions/oreo/android-8.0-changes.html
3. https://developer.android.google.cn/guide/components/broadcast-exceptions.html
4. https://developer.android.com/about/versions/oreo/background-location-limits.html