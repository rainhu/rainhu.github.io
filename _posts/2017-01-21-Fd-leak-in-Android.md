---
layout: post
title: Fd leak in Android
categoryies: framework
tags: FD stability
---

* content
{:toc}

FD（File Descriptor）文件描述符在形式上是非负整数，它是一个索引值，指向内核为每个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在Linux系统中，一切设备都视作文件，文件描述符为Linux平台设备相关的编程提供了一个统一的方法。





在stability测试的过程中，经常会出现许多FD泄漏导致的莫名其妙的FC，而crash的堆栈也是千奇百怪，
可能出现在应用层、framework层、Native层，其中以framework层居多。
所以当出现这个问题以后往往认为是framework出现问题了，
实际上从后面Debug的结果来看许多都是应用出现了问题。
同一个问题也会经常出现不同的堆栈。这就是FD泄漏的一个重要的特性，问题出现的不确定性。

FD作为文件句柄的实例，可以用来表示一个打开的文件，一个打开的网络流(socket)，管道或者资源（如内存块），输入输出(in/out/error)。 在Linux系统中，每个进程可以使用的FD数量是有上限的，在Android中这个上限为1024，表示每个进程可以创建的file descriptors 不能超多1024个。可以通过下面两种方式查看：

- ulimit -a
- cat /proc/sys/fs/file-max

通过使用ulimit –n 2048 可以临时提高进程可以拥有的file为2048个。

相比较传统的内存泄漏，FD泄漏在大部分情况下不会出现内存不足的情况，所以出现问题的时候会更加隐晦。由于发生FD泄漏的时候内存可能不会出现不足，所以不会出发系统的GC操作，导致只有通过crash进程的方式去自我恢复。事实上在很多情况下，就算触发系统GC，也不一定能够回收已经创建的句柄文件。

## Android中FD泄漏的几种类型

### 1. Resource related

Android应用可能会需要很多资源，像输入输出流，数据库资源Cursor， Binder设备。如果没能够很好的处理这些资源，不仅可能造成内存的泄漏，也可能会出现FD泄漏。

#### 输入输出

输入输出流的使用在任何程序中都会比较频繁，像FileInputStream，FileOutputStream，FileReader，FileWriter 等输入输出如果不断创建但是不及时关闭，不仅可能造成内存的泄露了也可能会造成FD的溢出。每次new一个FileInputStream、FileOutputStream 都会在进程中创建一个FD， 用来指向这个打开的文件，而如果反复执行下面的代码，FD文件会持续不断地增加，直至超过1024出现FC。

```java
String filename = prefix + "temp";
File file = new File(getCache(),fileName);
try{
    file.createNewFile();  
    FileOutputStream out = new FileOutputStream(file);
} catch (FileNotFoundException e){ 

} catch (IOException e){

}
```

在/proc/${进程id}/fd/ 目录下执行ls –l查看到增加的FD指向创建的文件，这里创建了不同的file，即使是对同一个文件，也会创建多个FD来指向这个打开的文件流。

> lr-x------ u0_a86   u0_a86            2015-06-20 01:25 80 -> /data/data/com.example.hu.memleakdemo/cache/48temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 81 -> /data/data/com.example.hu.memleakdemo/cache/49temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 82 -> /data/data/com.example.hu.memleakdemo/cache/50temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 83 -> /data/data/com.example.hu.memleakdemo/cache/51temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 84 -> /data/data/com.example.hu.memleakdemo/cache/52temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 85 -> /data/data/com.example.hu.memleakdemo/cache/53temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 86 -> /data/data/com.example.hu.memleakdemo/cache/54temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 87 -> /data/data/com.example.hu.memleakdemo/cache/55temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 88 -> /data/data/com.example.hu.memleakdemo/cache/56temp  
lr-x------ u0_a86   u0_a86            2015-06-20 01:25 89 -> /data/data/com.example.hu.memleakdemo/cache/57temp  

最终导致应用进程出现FC，并打出如下的Log， 表示这个进程FD数量已经到达了上限，无法再创建新的FD，只有终止进程。

> E/Parcel  ( 3601): dup failed in Parcel::read, fd 1 of 2  
> E/Parcel  ( 3601):   dup(1020) = -1 [errno: 24 (Too many open files)]  
> E/Parcel  ( 3601):   fcntl(1020, F_GETFD) = 1 [errno: 24 (Too many open files)]  
> E/Parcel  ( 3601):   flat 0x0 type 0  

正确的做法是能够在final中将流进行关闭,这样无论中途是否出现异常导致程序中断，都会将流顺利关闭。

```java
String filename = prefix + "temp";
File file = new File(getCache(),fileName);  
try{  
    file.createNewFile();  
    FileOutputStream out = new FileOutputStream(FileDescriptor. file );  
  }catch(Exception e){  
}
final{  
    if(out != null){  
      out.close();  
    }  
 }  
```

#### Cursor leak
与输入输出相似，数据库查询的Cursor如果没有及时进行Close，也会出现FD泄漏的情况。
如下面这段异常的Log:


> AndroidRuntime: FATAL EXCEPTION: IntentService[ContactSaveService]   
AndroidRuntime: Process: com.android.contacts,  
AndroidRuntime: android.database.sqlite.SQLiteException: unable to open database file (code 14)   
AndroidRuntime:     at android.database.DatabaseUtils.readExceptionFromParcel(DatabaseUtils.java:179)   
AndroidRuntime:     at android.database.DatabaseUtils.readExceptionFromParcel(DatabaseUtils.java:135)   
AndroidRuntime:     at android.content.ContentProviderProxy.delete(ContentProviderNative.java:544)   
AndroidRuntime:     at android.content.ContentResolver.delete(ContentResolver.java:1330)   
AndroidRuntime:     at com.android.contacts.ContactSaveService.saveContact(ContactSaveService.java:478)   
AndroidRuntime:     at com.android.contacts.ContactSaveService.onHandleIntent(ContactSaveService.java:222)   
AndroidRuntime:     at android.app.IntentService$ServiceHandler.handleMessage(IntentService.java:66)   
AndroidRuntime:     at android.os.Handler.dispatchMessage(Handler.java:102)   
AndroidRuntime:     at android.os.Looper.loop(Looper.java:148)   
AndroidRuntime:     at android.os.HandlerThread.run(HandlerThread.java:61)  

这个问题是在Stability测试环境下出现的，由于未能够在Cursor使用后及时进行关闭，最终出现了FD的溢出。
从上面FC的Log可以看到，异常出现在ContentProvider跨进程传递传递的时候，出现了异常，显示无法打开database文件。看到unable to open database file，大胆地猜测是否是出现了FD的泄漏。然后在Log中找到了下面这段，确定了是FD泄漏造成了这次的FC。

> E/JavaBinder( 3319): java.lang.RuntimeException: Could not write CursorWindow to Parcel due to error -24.  
E/JavaBinder( 3319): at android.database.CursorWindow.nativeWriteToParcel(Native Method)  
E/JavaBinder( 3319): at android.database.CursorWindow.writeToParcel(CursorWindow.java:705)  
E/JavaBinder( 3319): at android.database.BulkCursorDescriptor.writeToParcel(BulkCursorDescriptor.java:63)  
E/JavaBinder( 3319): at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:127)  
E/JavaBinder( 3319): at android.os.Binder.execTransact(Binder.java:453)  


而对于Cursor没有及时关闭这个问题，下面这种情况很容易造成开发者的疏忽，导致出现问题：

```java
public void problemMethod() {  
    Cursor cursor = query(); // 假设 query() 是一个查询数据库返回 Cursor 结果的函数   
 if (flag == false) {  // 出现了提前返回
        return;  
    }  
    cursor.close();  
} 
```

### 2.Thread related

#### HandlerThread
下面这段异常Log是在Monkey测试中出现的，所以没有明确的操作步骤，测试脚本提取的crash堆栈如下，显示的是无法分配JNI资源，看上去是一个典型了内存泄漏问题。


> 12-28 21:35:21.571 E/AndroidRuntime(11308): FATAL EXCEPTION: main  
12-28 21:35:21.571 E/AndroidRuntime(11308): Process: com.tct.fmradio, PID: 11308  
12-28 21:35:21.571 E/AndroidRuntime(11308): java.lang.OutOfMemoryError: Could not allocate JNI Env  
12-28 21:35:21.571 E/AndroidRuntime(11308): at java.lang.Thread.nativeCreate(Native Method)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at java.lang.Thread.start(Thread.java:1063)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.tct.fmradio.platform.QcomFMDeviceImpl.createRecordSinkThread(QcomFMDeviceImpl.java:391)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.tct.fmradio.platform.QcomFMDeviceImpl.&lt;init>(QcomFMDeviceImpl.java:295)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.tct.fmradio.device.FMDeviceImpl.createFMDevice(FMDeviceImpl.java:18)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.tct.fmradio.service.FMService.onCreate(FMService.java:1231)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.app.ActivityThread.handleCreateService(ActivityThread.java:2918)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.app.ActivityThread.access$1900(ActivityThread.java:157)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1457)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.os.Handler.dispatchMessage(Handler.java:102)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.os.Looper.loop(Looper.java:148)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at android.app.ActivityThread.main(ActivityThread.java:5509)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at java.lang.reflect.Method.invoke(Native Method)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)  
12-28 21:35:21.571 E/AndroidRuntime(11308): at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)  
12-28 21:35:21.572 I/ActivityManager( 6054): handleApplicationCrash  
12-28 21:35:21.572 I/JRDRecordService( 6054): jrdCrashHandler invoke ytf...  


但是在查看Log以后，在前面找到了下面的Log，发现原来是FD发生了泄漏。具体表现为打开FMRadio后，点击Recent按键500～600次， /proc/<FMRADIO所在进程>/fd/目录下fd文件数量不断增长，直到超过1024阀值发生FC.

> 12-28 21:35:21.569 E/art (11308): ashmem_create_region failed for 'indirect ref table': Too many open files
12-28 21:35:21.570 W/art (11308): Throwing OutOfMemoryError "Could not allocate JNI Env"
12-28 21:35:21.570 D/AndroidRuntime(11308): Shutting down VM
12-28 21:35:21.571 W/Adreno-GSL(11308): &lt;gsl_ldd_control:475>: ioctl fd 31 code 0xc0200933 (IOCTL_KGSL_TIMESTAMP_EVENT) failed: errno 24 Too many open files
12-28 21:35:21.571 W/Adreno-GSL(11308): &lt;ioctl_kgsl_syncobj_create:2979>: (20, 7, 46028) fail 24 Too many open files

此问题为FM有一个Service 在每次oncreate都会创建一个handlerthread，并且没有释放，而通过Recent方式会反复的调用这个service的oncreate代码，造成了泄漏。

```java
public void onCreate() {  
  Log.i(TAG, "onCreate()");  
  super.onCreate();  
  mDefaultName = getString(R.string.default_name_text);  
  // create a thread that messages will be processed on  
  new HandlerThread("FMService");  
  thread.start();  
  mMessenger = new FMMessenger(thread.getLooper(), mOnActionListener,null, null);  
  .... ...  
}  
```

通过在onDestroy添加下面语句，即可释放handlerthread所占用的句柄
```java
mHandlerThread.quitSafely(); 
```

在Android中使用线程，尤其是HandlerThread要尤其的谨慎，必须要确保创建HandlerThread的函数不会被反复的调用导致线程反复的被创建。
如果循环调用下面这段代码几百次，就会出现FD泄漏。

```java
HandlerThead handerThread = new HandlerThead("test");  
handlerThead.start();  
```

在不需要线程Loop的时候调用HandlerThead.quitSafely()或者HandlerThead.quit();销毁loop,释放句柄资源。

#### Thread
HandlerThread实际上是带有Loop的thread，而对于传统的Java Thread，需要声明Loop以后才会出现FD的增加。因为声明Loop相当于增加了一块缓冲区，需要有一个FD来标识。如果反复调用下面这段代码也会出现FD泄漏。
```java
Thread thread = new Thread (new Runnable(){  
    @Override  
    public void run(){  
    Looper.prepare();  
    // do things  
    Looper.loop();  
}  
});  
thread.start();  
```

如果确定不需要Looper，可以使用Looper.quit()或者Looper.quitSafely()来退出looper，避免出现FD泄漏。

### 3.Input channel file realted

#### WindowManager.addview
公司的同一个手机项目的的stability测试中，出现了两个crash的log，堆栈几乎完全不一样，但是后面发现原来都是没有及时调用WindowManager.removeView造成的。
第一份log如下：

<pre>
process:com.android.systemui  
Crash happen at 2016-07-07 19:15:48  
process:com.android.systemui  
pid:2438  
Classname:java.lang.RuntimeException  
Filename:InputChannel.java  
Methodname:nativeReadFromParcel  
LineNumber:-2  
Cause:Could not read input channel file descriptors from parcel.  
stackTrace:  
java.lang.RuntimeException: Could not read input channel file descriptors from parcel.  
    at android.view.InputChannel.nativeReadFromParcel(Native Method)  
    at android.view.InputChannel.readFromParcel(InputChannel.java:148)  
    at android.view.IWindowSession$Stub$Proxy.addToDisplay(IWindowSession.java:759)  
    at android.view.ViewRootImpl.setView(ViewRootImpl.java:550)  
    at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:310)  
    at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:86)  
    at com.android.systemui.assist.AssistManager.onConfigurationChanged(AssistManager.java:119)  
    at com.android.systemui.statusbar.phone.PhoneStatusBar$11.onVerticalChanged(PhoneStatusBar.java:962)  
    at com.android.systemui.statusbar.phone.NavigationBarView.notifyVerticalChangedListener(NavigationBarView.java:699)  
    at com.android.systemui.statusbar.phone.NavigationBarView.onSizeChanged(NavigationBarView.java:690)  
    at android.view.View.sizeChange(View.java:16892)  
    at android.view.View.setFrame(View.java:16854)  
    at android.view.View.layout(View.java:16770)  
    at android.view.ViewGroup.layout(ViewGroup.java:5488)  
    at android.view.ViewRootImpl.performLayout(ViewRootImpl.java:2190)  
    at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:1950)  
    at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:1126)  
    at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:6333)  
    at android.view.Choreographer$CallbackRecord.run(Choreographer.java:858)  
    at android.view.Choreographer.doCallbacks(Choreographer.java:670)  
    at android.view.Choreographer.doFrame(Choreographer.java:606)  
    at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:844)  
    at android.os.Handler.handleCallback(Handler.java:739)  
    at android.os.Handler.dispatchMessage(Handler.java:95)  
    at android.os.Looper.loop(Looper.java:148)  
    at android.app.ActivityThread.main(ActivityThread.java:5473)  
    at java.lang.reflect.Method.invoke(Native Method)  
    at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)  
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)  
</pre>


可以看到上面的Log是在Parcel传递的时候出现了异常。这个Bug问题出在没有能够处理好onConfigurationChanged中的代码。
onConfigurationChanged代码中出现了WindowManager.addView反复调用，却缺少对应的removeView匹配的情况。
WindowManager.addView每次调用，都会在server（WindowManagerService）和Client(用户进程)端创建fd文件来作为socket通信，如果不调用removeView这个fd将得不到释放。事实上，如果SystemServer所在的进程的FD数量超过1024个，还会造成Android的重启。


而第二份crash的log如下：

<pre>
pid:2494
Classname:java.lang.RuntimeException  
Filename:Bitmap.java  
Methodname:nativeCreateFromParcel  
LineNumber:-2  
Cause:Could not allocate dup blob fd.  
stackTrace:  
java.lang.RuntimeException: Could not allocate dup blob fd.  
    at android.graphics.Bitmap.nativeCreateFromParcel(Native Method)  
    at android.graphics.Bitmap.access$100(Bitmap.java:36)  
    at android.graphics.Bitmap$1.createFromParcel(Bitmap.java:1516)  
    at android.graphics.Bitmap$1.createFromParcel(Bitmap.java:1508)  
    at android.app.ActivityManager$TaskThumbnail.readFromParcel(ActivityManager.java:1390)  
    at android.app.ActivityManager$TaskThumbnail.&it;init>(ActivityManager.java:1411)  
    at android.app.ActivityManager$TaskThumbnail.&it;init>(ActivityManager.java:1359)  
    at android.app.ActivityManager$TaskThumbnail$1.createFromParcel(ActivityManager.java:1403)  
    at android.app.ActivityManager$TaskThumbnail$1.createFromParcel(ActivityManager.java:1401)  
    at android.app.ActivityManagerProxy.getTaskThumbnail(ActivityManagerNative.java:3367)  
    at android.app.ActivityManager.getTaskThumbnail(ActivityManager.java:1418)  
    at com.android.systemui.recents.misc.SystemServicesProxy.getThumbnail(SystemServicesProxy.java:357)  
    at com.android.systemui.recents.misc.SystemServicesProxy.getTaskThumbnail(SystemServicesProxy.java:336)  
    at com.android.systemui.recents.model.RecentsTaskLoader.getAndUpdateThumbnail(RecentsTaskLoader.java:430)  
    at com.android.systemui.recents.model.RecentsTaskLoadPlan.executePlan(RecentsTaskLoadPlan.java:250)  
    at com.android.systemui.recents.model.RecentsTaskLoader.loadTasks(RecentsTaskLoader.java:477)  
    at com.android.systemui.recents.Recents$TaskStackListenerImpl.run(Recents.java:143)  
    at android.os.Handler.handleCallback(Handler.java:739)  
    at android.os.Handler.dispatchMessage(Handler.java:95)  
    at android.os.Looper.loop(Looper.java:148)  
    at android.app.ActivityThread.main(ActivityThread.java:5473)  
    at java.lang.reflect.Method.invoke(Native Method)  
    at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)  
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)  
</pre>

这份Log很容易让人误以为是Binder传递了过大的Bitmap导致的异常。而这个Bug的问题出在没有处理好ActivityManagerProxy.getTaskThumbnail 的代码。
ActivityManagerProxy.getTaskThumbnail会在不断地点击Recents按键的时候被反复调用。而ActivityManagerProxy.getTaskThumbnail需要创建一个Parcel与ActivityManagerService通信，这时候就会创建一个FD指向Socket的端口，如果此时发现FD已经满了就会报出异常。

所以两个问题确实是属于同一个问题，都属于FD泄漏。我们在两份Log中到找到了如下的Log：

<pre>
2016-07-28 08:39:22,956 : 07-28 08:39:21.856 E/Parcel ( 2490): dup() failed in Parcel::read, i is 1, fds[i] is -1, fd_count is 2, error: Too many open files  
2016-07-28 08:39:22,956 : 07-28 08:39:21.856 E/Surface ( 2490): dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: -22  
2016-07-28 08:39:22,956 : 07-28 08:39:21.857 I/Adreno ( 2490): DequeueBuffer: dequeueBuffer failed  
2016-07-28 08:39:22,956 : 07-28 08:39:21.857 E/Parcel ( 2490): dup() failed in Parcel::read, i is 0, fds[i] is -1, fd_count is 2, error: Too many open files  
2016-07-28 08:39:22,956 : 07-28 08:39:21.857 E/Surface ( 2490): dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: -22  
2016-07-28 08:39:22,956 : 07-28 08:39:21.857 I/Adreno ( 2490): DequeueBuffer: dequeueBuffer failed  
2016-07-28 08:39:22,956 : 07-28 08:39:21.858 E/Parcel ( 2490): dup() failed in Parcel::read, i is 0, fds[i] is -1, fd_count is 2, error: Too many open files  
2016-07-28 08:39:22,956 : 07-28 08:39:21.858 E/Surface ( 2490): dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: -22  
2016-07-28 08:39:22,956 : 07-28 08:39:21.858 I/Adreno ( 2490): DequeueBuffer: dequeueBuffer failed  
2016-07-28 08:39:22,956 : 07-28 08:39:21.858 E/Parcel ( 2490): dup() failed in Parcel::read, i is 0, fds[i] is -1, fd_count is 2, error: Too many open files  
2016-07-28 08:39:22,956 : 07-28 08:39:21.858 E/Surface ( 2490): dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: -22  
2016-07-28 08:39:22,972 : 07-28 08:39:21.858 I/Adreno ( 2490): DequeueBuffer: dequeueBuffer failed  
2016-07-28 08:39:31,458 : 07-28 08:39:30.356 E/InputChannel-JNI( 2490): Error 24 dup channel fd 1015.  
</pre>

#### Multi-Task

一台手机在跑Monkey的时候出现的这个问题，错误的堆栈信息如下，看着堆栈，挂在了native层， abort message 发现原来是FD文件超了

<pre>
07-14 23:40:49.781 F/libc    (10511): Fatal signal 6 (SIGABRT), code -6 in tid 10511 (m.android.email)  
07-14 23:40:49.838 F/DEBUG   (  747): *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***  
07-14 23:40:49.838 F/DEBUG   (  747): Build fingerprint: 'Vertu/VM-08/tron:6.0.1/6.0.1_0.184.0.032/china_032:user/release-keys'  
07-14 23:40:49.838 F/DEBUG   (  747): Revision: '0'  
07-14 23:40:49.838 F/DEBUG   (  747): ABI: 'arm64'  
07-14 23:40:49.838 F/DEBUG   (  747): pid: 10511, tid: 10511, name: m.android.email  >>> com.android.email <<<  
07-14 23:40:49.838 F/DEBUG   (  747): signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------  
07-14 23:40:49.889 F/DEBUG   (  747): Abort message: 'FORTIFY: FD_SET: file descriptor >= FD_SETSIZE'  
07-14 23:40:49.889 F/DEBUG   (  747):     x0   0000000000000000  x1   000000000000290f  x2   0000000000000006  x3   0000000000000000  
07-14 23:40:49.889 F/DEBUG   (  747):     x4   0000000000000000  x5   0000000000000001  x6   0000000000000000  x7   0000000000000000  
07-14 23:40:49.889 F/DEBUG   (  747):     x8   0000000000000083  x9   525e43451f3c3d1f  x10  7f7f7f7f7f7f7f7f  x11  0101010101010101  
07-14 23:40:49.889 F/DEBUG   (  747):     x12  0000007f873b7888  x13  f98a40dee6834299  x14  f98a40dee6834299  x15  00267f6a359179b2  
07-14 23:40:49.889 F/DEBUG   (  747):     x16  0000007f873b1568  x17  0000007f873442fc  x18  0000007f84267000  x19  0000007f877cf088  
07-14 23:40:49.889 F/DEBUG   (  747):     x20  0000007f877cefc8  x21  0000000000000002  x22  0000000000000006  x23  0000007fd5d595d0  
07-14 23:40:49.889 F/DEBUG   (  747):     x24  0000000000000413  x25  0000007fd5d595cc  x26  0000000000000002  x27  0000007fd5d59bc0  
07-14 23:40:49.889 F/DEBUG   (  747):     x28  0000007f7ddc2394  x29  0000007fd5d59370  x30  0000007f87341a98  
07-14 23:40:49.889 F/DEBUG   (  747):     sp   0000007fd5d59370  pc   0000007f87344304  pstate 0000000020000000  
07-14 23:40:49.898 F/DEBUG   (  747):   
07-14 23:40:49.898 F/DEBUG   (  747): backtrace:  
07-14 23:40:49.898 F/DEBUG   (  747):     #00 pc 0000000000069304  /system/lib64/libc.so (tgkill+8)  
07-14 23:40:49.898 F/DEBUG   (  747):     #01 pc 0000000000066a94  /system/lib64/libc.so (pthread_kill+68)  
07-14 23:40:49.898 F/DEBUG   (  747):     #02 pc 00000000000239f8  /system/lib64/libc.so (raise+28)  
07-14 23:40:49.898 F/DEBUG   (  747):     #03 pc 000000000001e198  /system/lib64/libc.so (abort+60)  
07-14 23:40:49.898 F/DEBUG   (  747):     #04 pc 00000000000215e0  /system/lib64/libc.so (__libc_fatal+128)  
07-14 23:40:49.898 F/DEBUG   (  747):     #05 pc 0000000000021604  /system/lib64/libc.so (__fortify_chk_fail+32)  
07-14 23:40:49.898 F/DEBUG   (  747):     #06 pc 000000000006fb28  /system/lib64/libc.so (__FD_SET_chk+32)  
07-14 23:40:49.898 F/DEBUG   (  747):     #07 pc 0000000000000f74  /system/vendor/lib64/libqti-perfd-client.so (mpctl_send+1048)  
07-14 23:40:49.898 F/DEBUG   (  747):     #08 pc 0000000000000fec  /system/lib64/libqti_performance.so  
07-14 23:40:49.898 F/DEBUG   (  747):     #09 pc 0000000000004968  /system/framework/oat/arm64/QPerformance.odex (offset 0x4000)  
</pre>


我们发现问题出在如下的代码上：

```java
if (action == COMPOSE) {  
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_DOCUMENT | Intent.FLAG_ACTIVITY_MULTIPLE_TASK);  
} else if (message != null) { 
    
}
```

写一封新邮件,startactivity 使用的flag是multiTask，也就是说，每点击创建新的邮件，都会创建task。而Monkey在跑的时候创建了n个邮件的task， 而对应打开的ComposeActivityEmail.java 的 “插入快速语” 会创建很多个fd , 最终导致FD超过1024， 进程崩溃。
实际上，通过反复如下代码就会出现这个问题：

```java
Intent intent = new Intent();  
Intent.addFlags(Intent.FLAG_ACTIVITY_MULTIPLE_TASK);  
Intent.setClass(MainActivity.this, SecondActivity.class);  
startActivity(intent);  
```

应用的input event由WindowManagerService管理，WMS内部会创建一个InputManager，两者通过InputChannel来完成，WMS需要注册两个InputChannel与InputManager连接，其中Server端InputChannel注册在InputManager（SystemServer），Client端注册在应用程序主线程中。InputChannel使用Ashmem匿名共享内存来传递数据，它由一个fd文件描述符指向，同时read端和write端各占用一个fd。创建一个新的Task时， server(system_server)和client(app)都会构建FD。

所以设置为Intent.FLAG_ACTIVITY_MULTIPLE_TASK类似flag的时候，如果没有处理好Activity的生命周期，可能会出现system_server进程先于应用进程到达FD上限，造成 Android系统重启。


## 如何解决此类问题

FD会发生泄漏的根本原因是没有对资源进行有效的管理。无论是文件资源，设备资源，Socket资源，输入输出流还是线程等，如果被频繁的调用而没有即使释放，甚至根本就没有释放，将会使得FD越积越多，最终导致了泄漏的发生。

### 3.1. 确定问题
这类问题比较容易在Stabiliy、Monkey、JrdLog中出现，当在这些测试中遇到挂在不可思议的地方的时候就可以看看是否可能是FD泄漏引起的问题。

要确定这个问题是否由FD泄漏引起，一般情况下可以通过下面的关键字too many files open，ashmem_create_region failed for 'indirect ref table': Too many open files，leak , release ……

并不是说有上面的关键字，就一定是FD泄漏，JNI对象泄漏也会出现too many的字段；也并不是说没有上面的关键字就一定不是FD泄漏问题，具体到问题还是需要具体分析。

### 3.2 探索复现路径
每个进程创建的FD文件都在/proc/${问题应用所在进程}/fd/这个目录下,在这个目录下执行ls |wc –l 或者在任意adb shell目录下使用lsof ${问题应用所在进程} |wc –l, 都可以查看进程的FD文件数量，如果能够找到某个或者某些操作能够让FD数量稳定的增长，那么就容易解决。

一般可以尝试反复点击Recent， 反复打开应用或者Activity，然后按back返回等操作，反复横竖切换等。

由于存在1024这个阈值，普通用户在使用过程中一般不会出现这个问题，出现这个问题一般都是Stability测试或者Monkey出现的。

如果stability这样的测试中，可以通过编写脚本每1 min 抓取一次FD的数量，观察在什么时间段FD数量出现了稳定的增长过程，然后再对比期间所做的测试操作，通过分析尝试去找到手动的复现路径。

而Monkey测试就没有什么规律了，因为测试本身就是乱序的操作过程，要知道复现只能够通过联系Log上下文，查看操作了哪些Activity，做了哪些操作，哪些Log是反复打印的，哪些Log出现不正常的现象，然后一个个去尝试排除。

### 3.3定位问题代码段
一旦有了复现路径以后，要找到相应出问题的代码段的返回就会小了很多。

如果能定位到具体的文件，可以先查看一下最近的提交记录。如果这个问题之前没有后来才出现，一般都是改出来的问题，按照之前debug经验来看，这种情况还是占了大多数。但是有些问题之前没出现，可能只是没有被测试来而已，很多原生的代码也是会出现问题的。

如果运气不好遇到这些原生问题，可以重点查看哪些可能会被反复调用的代码段，在Java层面比如说onResume，onConfigurationchanged 之类函数，查看在这些代码段中是否由创建什么线程、资源，而在onPause或者onDestroy没有被释放掉的。
