---
layout: post
title: Android中public的So库的引用问题
categories: framework
tags: cts debug so
published: true
comments: true
---
* content
{:toc}

> 本文记录项目中遇到的一个CTS的问题，然后解决此问题的过程  

### 问题背景

CtsJniTestCases模块android.jni.cts.JniStaticTest#test_linker_namespaces测试失败  
通过run cts -m android.jni.cts.JniStaticTest -t android.jni.cts.JniStaticTest#test_linker_namespaces 运行


```
失败的log如下：
junit.framework.AssertionFailedError: The library "/vendor/lib64/libdirect-coredump.so" is not a public library but it loaded.
at junit.framework.Assert.fail(Assert.java:50)
at android.jni.cts.JniStaticTest.test_linker_namespaces(JniStaticTest.java:41)
at java.lang.reflect.Method.invoke(Native Method)
at junit.framework.TestCase.runTest(TestCase.java:168)
```





测试的结果是/vendor/lib64/libdirect-coredump.so不是public库但是却允许被应用使用，这个违反了GoogleCDD中的违反CDD-3.3.1  
>[C-0-9] MUST list additional non-AOSP libraries exposed directly to third-party apps in /vendor/etc/public.libraries.txt.

### 问题分析
查看CTS测试代码发现
>cts/tests/tests/jni/src/android/jni/cts/JniStaticTest.java
cts/tests/tests/jni/src/android/jni/cts/LinkerNamespacesHelper.java
cts/tests/tests/jni/libjnitest/android_jni_cts_LinkerNamespacesTest.cpp

->test_linker_names[cts/tests/tests/jni/src/android/jni/cts/JniStaticTest.java]  
-> LinkerNamespacesHelper.runAccessibilityTest [LinkerNamespacesHelper.java]  
-> runAccessibilityTestImpl [LinkerNamespacesHelper.java]  
将system/etc/public.libraries.txt和vendor/etc/public.libraries.txt以及公开的systemlib加入对应的venderlib和systemlib数组中
```java
private final static String VENDOR_CONFIG_FILE = "/vendor/etc/public.libraries.txt";
private final static String[] PUBLIC_SYSTEM_LIBRARIES = {
    "libaaudio.so",
    "libandroid.so",
    "libc.so",
    "libcamera2ndk.so",
    "libdl.so",
    "libEGL.so",
    "libGLESv1_CM.so",
    "libGLESv2.so",
    "libGLESv3.so",
    "libicui18n.so",
    "libicuuc.so",
    "libjnigraphics.so",
    "liblog.so",
    "libmediandk.so",
    "libm.so",
    "libnativewindow.so",
    "libOpenMAXAL.so",
    "libOpenSLES.so",
    "libRS.so",
    "libstdc++.so",
    "libsync.so",
    "libvulkan.so",
    "libz.so"
};

public static String runAccessibilityTest() throws IOException {
    ...
    return runAccessibilityTestImpl(systemLibs.toArray(new String[systemLibs.size()]),
                                    vendorLibs.toArray(new String[vendorLibs.size()]));
}
```

->Java_android_jni_cts_LinkerNamespacesHelper_runAccessibilityTestImpl [android_jni_cts_LinkerNamespacesTest.cpp]  
通过遍历system/lib(64)?和vendor/lib(64)?然后比对上面传进来的属于public的so的列表，如果发现so库能被APP访问但是不在对应的public.libraries.txt列表中的，就会Fail


### 问题解决
所以这边参考system/core/rootdir/etc/public.libraries.txt中创建了含有libdirect-coredump.so的device/xx/xx/etc/public.libraries.txt  

通过在device/xx/xx/device.mk中添加copy语句，使得这个文件在编译时候能够copy到/vendor/etc/public.libraries.txt  
```
PRODUCT_COPY_FILES += $(LOCAL_PATH)/etc/public.libraries.txt:$(TARGET_COPY_OUT_VENDOR)/etc/public.libraries.txt
```



### 思考-问题1
>public.libraries.txt是如何生效的？

参考LinkerNamespacesHelper中的实现，满足下面两个条件的都算是public的so，能够被普通app访问  
1)在/etc/public.libraries.txt出现  
2)在/vender/etc/public.libraries.txt出现  

那么这两个文件是什么时候被加载的呢？
>/system/core/libnativeloader/native_loader.cpp
/art/runtime/java_vm_ext.cc
/system/core/libnativeloader/native_loader.cpp

系统在初始化的时候就会通过LibraryNamespaces#Initialize去加载这两个文件
```c++
native_loader.cpp

static constexpr const char* kPublicNativeLibrariesSystemConfigPathFromRoot =
                                  "/etc/public.libraries.txt";
static constexpr const char* kPublicNativeLibrariesVendorConfig =
                                  "/vendor/etc/public.libraries.txt";
void Initialize() {
  ...
  std::vector<std::string> sonames;
  const char* android_root_env = getenv("ANDROID_ROOT");
  std::string root_dir = android_root_env != nullptr ? android_root_env : "/system";
  std::string public_native_libraries_system_config =
          root_dir + kPublicNativeLibrariesSystemConfigPathFromRoot;

  std::string error_msg;
  LOG_ALWAYS_FATAL_IF(!ReadConfig(public_native_libraries_system_config, &sonames, &error_msg),
                      "Error reading public native library list from \"%s\": %s",
                      public_native_libraries_system_config.c_str(), error_msg.c_str());

   ...
  ReadConfig(kPublicNativeLibrariesVendorConfig, &sonames);

  vendor_public_libraries_ = base::Join(sonames, ':');
}
```
而这个方法是在创建一个ART虚拟机的时候（也就是应用进程被创建的时候），初始化NativeLoader，这个NativeLoader就是用来装载so库的
```c++
java_vm_ext.cc

extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
    ......
    // Initialize native loader. This step makes sure we have
    // everything set up before we start using JNI.
    android::InitializeNativeLoader();
     .......
```
下面的g_namespaces->Initialize调用上面介绍的LibraryNamespaces#Initialize完成公共so库的赋值
```c++
native_loader.cpp

void InitializeNativeLoader() {
     #if defined(__ANDROID__)
     std::lock_guard<std::mutex> guard(g_namespaces_mutex);
    g_namespaces->Initialize();
  #endif
}
```


### 思考-问题2
>为什么普通应用无法引用非public的库？

普通的应用不能直接引用系统的一些so库，这是在Andrid N之后作出的一下变化  
https://source.android.com/devices/tech/config/namespaces_libraries

问题1介绍到public的so库会在APP进程创建的时候将访问权限加入到虚拟机可访问的环境中，但是非public的so库却只有systemapp才有权限访问。

通过下面的代码可以获知，如果是系统应用且不是更新过的App，就会把system/lib(64)?路径加上去，所以系统应用有权限访问/system/lib(64)?下的so库，普通的应用无法访问.
```java
frameworks/base/core/java/android/app/LoadedApk.java

final boolean isBundledApp = mApplicationInfo.isSystemApp()
             && !mApplicationInfo.isUpdatedSystemApp();

     makePaths(mActivityThread, isBundledApp, mApplicationInfo, zipPaths, libPaths);

     String libraryPermittedPath = mDataDir;
     if (isBundledApp) {
         // This is necessary to grant bundled apps access to
         // libraries located in subdirectories of /system/lib
         libraryPermittedPath += File.pathSeparator +
                                 System.getProperty("java.library.path");
     }
```
