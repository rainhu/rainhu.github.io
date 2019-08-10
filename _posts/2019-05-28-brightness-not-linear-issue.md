---
layout: post
title: Android P中的一个用户体验的改进-LCD背光亮度不是线性变化
categories: Android
tags: framework lights lcd
published: false
comments: true
---

* content
{:toc}

> 最新调试项目，硬件的妹子报了一个Bug过来：在Android P系统的设置菜单中将系统背光值滑到百分之五十，但实际亮度却降到了最大亮度的百分之十三，这个对比之前Android O的项目中是不存在的。


### 原因定位

通过查看背光的节点sys/class/leds/back_ligtht/brightless，发现节点的值的设置就是有问题的，说明上层往节点设值的时候就有问题，排除是驱动的问题。

得到这个信息以后的第一个反应是，这个现象可能不是一个Issue而是一个Feature，因为上层的AOSP的代码，一般没人会去改这样的值。
问题的定位本身不难，如果熟悉Android LED的调用流程，从HAL层（Lights.c）跟踪到Framework（LightsService）再到最上层的Settings相关代码，会很快找到问题的原因所在：原来这个值在Settings这边设置下去的时候就已经做了转换处理！



Google提交的Patch(提交链接见参考链接)
Convert the BrightnessController to a log scale control.
Currently, the BrightnessController's UI is a linear scale control on
top of a linear backlight control, but humans perceive brightness on a
roughly logarithmic scale. By moving to a non-linear control, we both
give users more fine-grained control over the brightness of the display
as well as a UI that works more intuitively.
Test: manual
Bug: 73810208
Change-Id: I67090ad7c4ced0420314458473c9124cb9c61906
core/java/android/util/MathUtils.java[diff]
packages/SystemUI/src/com/android/systemui/settings/BrightnessController.java[diff]

核心的转换代码如下，通过查看代码注释，查阅资料了解到，这个转换引入了韦伯-费希纳定律（Fechner's Law）：感觉的大小同刺激强度的对数成正比。
所以设置值的时候，会进行对数换算，从而出现了当前不是正比的情况。原始的值在通过convertLinearToGamma转换之后会得到最终往底层设置的值，这个值不再是以最大亮度的线性变化值。

```JAVA
private static final int convertLinearToGamma(int val, int min, int max) {
       // For some reason, HLG normalizes to the range [0, 12] rather than [0, 1]
        final float normalizedVal = MathUtils.norm(min, max, val) * 12;
        final float ret;
        if (normalizedVal <= 1f) {
            ret = MathUtils.sqrt(normalizedVal) * R;
        } else {
            ret = A * MathUtils.log(normalizedVal - B) + C;
        }

        return Math.round(MathUtils.lerp(0, SLIDER_MAX, ret));
}
```

### 关于韦伯-费希纳定律（Fechner's Law）
这是由德国生理学家伟伯和德国物理学家费希纳发现的定律，用来表明心理量和物理量之间的关系，在很多领域都有应用。在图像显示方面，通过Gamma Space和Linear Space分别表示人类感官的值和实际的值。






### 思考
说起来，会有这样的一个提交真的一件非常神奇的事情。
#### Issue的提出
 按照正常的逻辑，线性地去调整背光亮度的大小，合情合理。所以非常好奇这个Issue是谁提的，属于什么职能部门，Issue的描述是如何的。
不管怎样，能够提出这样需求的人，

#### 专业领域之外的知识
从另一个方面来看，有时候多看看专业领域之外的东西，对于提升 如果这样的Issue如果在Android O的时候提到我这边，我肯定会毫不犹豫的Refuse掉。。。

#### 系统的优化方向
从中也可以看到，Android经过这么多年的发展，除了性能方面的提升和优化，用户体验的改进将



### 参考链接
https://docs.unity3d.com/Manual/LinearLighting.html
https://android.googlesource.com/platform/frameworks/base/+/585ff988bbc116f318741f3ed3916ed44558a3fb
