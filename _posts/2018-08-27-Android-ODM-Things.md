---
layout: post
title: Android ODM 常见定制化需求
categories: odm
tags: odm
published: false
comments: true
---
* content
{:toc}

> 本文基于Android 8.1平台进行分析

### 设置系统默认语言
device\公司名字\项目名字\full_项目名字.mk

修改变量：
PRODUCT_LOCALES := en_US es_ES zh_CN zh_TW ru_RU pt_BR fr_FR de_DE tr_TR it_IT in_ID ms_MY vi_VN ar_EG hi_IN th_TH bn_IN pt_PT ur_PK fa_IR nl_NL el_GR hu_HU tl_PH ro_RO cs_CZ iw_IL my_MM km_KH ko_KR pl_PL es_US bg_BG hr_HR lv_LV lt_LT sk_SK uk_UA de_AT da_DK fi_FI nb_NO sv_SE en_GB ja_JP

把此段覆盖上面的语言代码
en_US es_ES ru_RU pt_BR fr_FR hi_IN ur_PK bn_IN ar_EG fa_IR

不过除了这个还有一个地方也可以修改系统默认语言
就是buildinfo.sh的ro.product.locale.language变量，修改默认的为自己的国家语言代码即可

默认的语言最终会存储在ro.product.locale中






### 序列号问题
   systemproperty中gsm.serial，通过getprop可以获取

### 如何获取默认的帧率
74          Display dpy = wm.getDefaultDisplay();
75          float claimedFps = dpy.getRefreshRate();



### 将Keyevent添加唤醒源
 framework/base/core/java/android/view/KeyEvent.java
  public static final isWakeKey(int keyCode){
        switch(keycode) {
            ...
            case KeyEvent.KEYCODE_STEM_3:
 ++      case KeyEvent.KEYCODE_
        }
  }


### 修改下拉通知栏的透明度
 packages/apps/SystemUI/res/values/dimens.xml
  <item name="scrim_behind_alpah" format="float" type="dimen">0.65</item>

### 修改蓝牙显示名称
 device/genetic/common/bluetooth/broid_buildcfg.h
 #define TBM_DEF_LOCAL_NAME "xxx"
