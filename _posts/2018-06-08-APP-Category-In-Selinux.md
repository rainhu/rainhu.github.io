---
layout: post
title: Selinux中的APP分类
categories: framework
tags: selinux app domain
published: true
comments: true
---
* content
{:toc}

> 本文基于MTK平台的Android8.1进行分析
Selinux在Android M之后被要求强制开启，主要用来描述一些domain对系统中的资源，如文件、socket之类的访问的限制。而在Android系统中，Application中的应用根据一些条件被分为了好几类domain。
在开发的过程中会遇到很多APP的权限问题，有些app能够访问节点的节点却不能被某些app却不能，就是因为这些app所属于的SELinux domain不同。
本文将会对这些进行介绍。


### Selinux中APP的domain
abd shell之后通过ps -AZ查看进程  
![进程详情](https://github.com/rainhu/rainhu.github.io/blob/master/_assets/2018-06-08/process.png)





第一列为Selinux中的SContext, 第二列为UID，uid=system为system_app
U0_xxx要么属于platform_app，priv-app,untrusted_app  
一个uid可以属于多个进程

可以看到APP进程在Selinux中的domain大概分为下面几类  
1）untrusted_app 第三方app，没有android平台签名，没有system权限  
2）platform_app 有android平台签名，没有system权限，且安装在system/app目录下的应用  
3）priv_app 有平台签名，没有system权限，且安装在system/priv-app下的应用  
4）system_app 有android平台签名和system权限  


他们是在如下seapp_contexts文件中进行确定的。  
其中：  
user对应UID: 由AndroidManifest.xml中的shareduserI来决定  
einfo对应签名信息: system/sepolicy/private/mac_perissions.xml  
domain就是我们说的app所所属于的具体的domain，由上面的user和einfo确定  
type就是这个domain的文件类型  

```
system/sepolicy/private/seapp_contexts

isSystemServer=true domain=system_server
user=system seinfo=platform domain=system_app type=system_app_data_file
user=bluetooth seinfo=platform domain=bluetooth type=bluetooth_data_file
user=nfc seinfo=platform domain=nfc type=nfc_data_file
user=radio seinfo=platform domain=radio type=radio_data_file
user=shared_relro domain=shared_relro
user=shell seinfo=platform domain=shell type=shell_data_file
user=_isolated domain=isolated_app levelFrom=user
user=_app seinfo=media domain=mediaprovider name=android.process.media type=app_data_file levelFrom=user
user=_app seinfo=platform domain=platform_app type=app_data_file levelFrom=user
user=_app isV2App=true isEphemeralApp=true domain=ephemeral_app type=app_data_file levelFrom=user
user=_app isPrivApp=true domain=priv_app type=app_data_file levelFrom=user
user=_app minTargetSdkVersion=26 domain=untrusted_app type=app_data_file levelFrom=user
user=_app domain=untrusted_app_25 type=app_data_file levelFrom=user
```

> 那么问题来了，UID和einfo是由什么来决定的呢？

## UID（用户ID）与进程的关系
UID是PackageManagerService在启动的时候从AndroidManifest中的shareduserId中进行获取，系统默认的有android.uid.system对应userid和所属的进程为system,android.uid.phone对应userid和所属进程为phone  
其中uid为0的进程为root进程，uid >= Process.FIRST_APPLICATION_UID的进程所属的进程为u0_xx  

下面是shareduserId初始化的过程：  


```java
1--> 将android.uid.system,android.uid.phone等，system.,phone,log,nfc,bluetooth,shell这几个uid都会添加package的Flag为ApplicationInfo.FLAG_SYSTEM和private的flag为ApplicationInfo.PRIVATE_FLAG_PRIVILEGED
这两个flag会被用来判断是否是属于系统应用以及高优先级的应用，之前讲过的不能引用非公开so库的普通应用就不会带有FLAG_SYSTEM的标志，且会安装到data分区而不是system分区
        public boolean isSystemApp() {
             return (flags & ApplicationInfo.FLAG_SYSTEM) != 0;
         }

         public boolean isPrivilegedApp() {
             return (privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
         }

framework/base/services/core/java/com/android/oserver/pm/PackageManagerService.java         
public PackageManagerService(Context context, Installer installer,
          boolean factoryTest, boolean onlyCore) {
         ... ...
mSettings = new Settings(mPackages);
mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
         ... ...


1.1-->继续上面的上面的addSharedUserLPw的流程,SharedUser的信息会存在mSharedUsers这个map中，其中key   为"android.uid.system"之类，value则为SharedUserSetting对象

/frameworks/base/services/core/java/com/android/server/pm/Settings.java
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
    SharedUserSetting s = mSharedUsers.get(name);
    ... ...
    s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
    s.userId = uid;
    if (addUserIdLPw(uid, s, name)) {
        mSharedUsers.put(name, s);
        return s;
    }
    return null;
}

1.1.1-> 继续上面addUserIdLPw的流程，最终将系统级别的uid信息存储在mOtherUserIds中，而普通应用级别的uid存储在mUserIds中
private boolean addUserIdLPw(int uid, Object obj, Object name) {
     ... ...
     /*
       Process.FIRST_APPLICATION_UID开始到Process.LAST_APPLICATION_UID为普通应用的uid,保存到mUserIds中
     */
    if (uid >= Process.FIRST_APPLICATION_UID) {
        int N = mUserIds.size();
        final int index = uid - Process.FIRST_APPLICATION_UID;
        while (index >= N) {
            mUserIds.add(null);
            N++;
        }
        ... ...
        mUserIds.set(index, obj);
    } else {  //将system,nfc等系统预留的uid保存到mOtherUserIds
         ... ...
        mOtherUserIds.put(uid, obj);
    }
    return true;
}
```



```java
2-->遍历Android中所安装的app，按照vendor/overlay, framwork, system/priv-app, system/app,vendor/app的顺序进行遍历

framework/base/services/core/java/com/android/oserver/pm/PackageManagerService.java
      ... ...
      // Collected privileged system packages.
      final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
      scanDirTracedLI(privilegedAppDir, mDefParseFlags
              | PackageParser.PARSE_IS_SYSTEM
              | PackageParser.PARSE_IS_SYSTEM_DIR
              | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

      // Collect ordinary system packages.
      final File systemAppDir = new File(Environment.getRootDirectory(), "app");
      scanDirTracedLI(systemAppDir, mDefParseFlags
              | PackageParser.PARSE_IS_SYSTEM
              | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

      // Collect all vendor packages.
      File vendorAppDir = new File("/vendor/app");
       ... ...
      scanDirTracedLI(vendorAppDir, mDefParseFlags
              | PackageParser.PARSE_IS_SYSTEM
              | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

      // Collect all OEM packages.
      final File oemAppDir = new File(Environment.getOemDirectory(), "app");
      scanDirTracedLI(oemAppDir, mDefParseFlags
              | PackageParser.PARSE_IS_SYSTEM
              | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);


2.1--> scanDirTracedLI
2.1.1 --> scanDirLI
2.1.1.1 --> scanPackageLI (六个参数，其中userhandler可以为null)
2.1.1.1.1 --> scanPackageInternalLI
2.1.1.1.2 --> scanPackageLI (六个参数，其中userHandler != null)
2.1.1.1.2.1 -->scanPackageDirtyLI()



```

## 关于seinfo的读取
PackageManagerService会在系统初始化或者应用安装的时候通过调用scanPackageDirtyLi将seinfo从/etc/selinux/plat_mac_permissions.xml和/etc/selinux/nonplat_mac_permissions.xml读取出来，然后比对解析每个package的签名进行相应的seinfo的赋值，将数据存储在每个应用的Application#seInfo中  

```java
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
         final int policyFlags, final int scanFlags, long currentTime, @Nullable UserHandle user)throws PackageManagerException {
          ... ...
        /* PackageManager会从下面两个路径中国年读取mac_permissions.xml文件，分为平台相关的和平台不相关的
         private static final File[] MAC_PERMISSIONS =
    { new File(Environment.getRootDirectory(), "/etc/selinux/plat_mac_permissions.xml"),
      new File(Environment.getVendorDirectory(), "/etc/selinux/nonplat_mac_permissions.xml") };
      */
          mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();
          ... ...
         /*
           根据package的签名与前面读取出的mac_permission.xml文件信息将对应的seinfo
         */
          if (mFoundPolicyFile) {
              SELinuxMMAC.assignSeInfoValue(pkg);
          }
          ... ...
}

```
>那么/etc/selinux/plat_mac_permissions.xml和/etc/selinux/nonplat_mac_permissions.xml是如何生成的呢？

通过查看/system/sepolicy/Android.mk
```
##################################
#system/sepolicy/private/mac_permissions.xml输出到/etc/selinux/plat_mac_permissions.xml作为system相关的policy规则，对于Vendor，可以在下面目录下定义mac_permissions.xml：$(PLAT_VENDOR_POLICY) $(BOARD_SEPOLICY_DIRS) $(REQD_MASK_POLICY) ，自定义的mac_permissions.xml会输出到/etc/selinux/nonplat_mac_permissions.xml作为vendor相关的policy规则
##################################
/system/sepolicy/Android.mk

include $(CLEAR_VARS)

LOCAL_MODULE := plat_mac_permissions.xml
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_OUT)/etc/selinux
... ...
$(plat_mac_perms_keys.tmp): $(call build_policy, keys.conf, $(PLAT_PRIVATE_POLICY))
	@mkdir -p $(dir $@)
	$(hide) m4 -s $(PRIVATE_ADDITIONAL_M4DEFS) $^ > $@

all_plat_mac_perms_files := $(call build_policy, mac_permissions.xml, $(PLAT_PRIVATE_POLICY))
... ...
```
所以在aosp上，seinfo由决定system/sepolicy/private/mac_permissions.xml，如果签名是platform的(LOCAL_CERTIFICATE=platform)，则seinfo=platform  

```xml
<policy>   
 <!-- Platform dev key in AOSP -->
    <signer signature="@PLATFORM" >
      <seinfo value="platform" />
    </signer>

 <!-- Media key in AOSP -->
    <signer signature="@MEDIA" >
      <seinfo value="media" />
    </signer>

</policy>
```



### 思考: systemapp，PRIVILEGED app相比较其他应用会有那些特殊的权限？  
从Selinux角度来说，App可以分为
1.priv_app  
2.system_app  
3.platform_app  
4.untrusted_app  
他们的权限也就是访问特定文件的权限都是定义在对应的**.te下的，如priv_app.te。不同selinux domain类别的应用限制了对文件节点、socket之类的访问。


而对于Android Framework 来说，应用的分类可以从/frameworks/base/core/java/android/content/pm/ApplicationInfo.java的一些方法看出来
1. SystemApp -> isSystemApp -> 带有ApplicationInfo.FLAG_SYSTEM
2. PrivilegedApp -> isPrivilegedApp -> 带有ApplicationInfo.PRIVATE_FLAG_PRIVILEGED
3. UpdatedSystemApp -> isUpdatedSystemApp -> 带有ApplicationInfo.FLAG_UPDATED_SYSTEM_APP
4. instantApp -> isInstantApp -> 带有ApplicationInfo.PRIVATE_FLAG_INSTANT
他们的权限控制更多的是对于某些API的控制，so的控制。
