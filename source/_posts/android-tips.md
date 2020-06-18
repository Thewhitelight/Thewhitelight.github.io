---
title: Android 开发小技巧
date: 2018-12-07 11:37:10
tags:
    - Android
    - 开发Tips
categories: 
    - 技术
---
开发中的一些小技巧记录
## ConstraintLayout 布局文字显示宽度
当 TextView 左右都有 View 且文字显示不下时需要省略号,如下图所示
![](https://def-201655.cos.ap-shanghai.myqcloud.com/ic_android_tips_constaint_1.png)
**解决办法:**
如果`layout_width="match_parent"`或者`ayout_width="wrap_content"`,则文字显示有问题
需要设置为`layout_width="0dp" layout_constraintLeft_toLeftOf 为向左对其 layout_constraintRight_toLeftOf 为向右对其图片`,这样就可以正确显示内容了.
对于 ConstraintLayout 内所有内容都应该要有相对位置的设置,不然会出现意想不到的问题.
示例代码:
``` xml
 <TextView
        android:id="@+id/pin_tuan_title"
        android:layout_width="0dp" 
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/toolbar_expand_size"
        android:layout_marginRight="12dp"
        android:ellipsize="end"
        android:includeFontPadding="false"
        android:maxLines="1"
        android:textColor="@color/main_black"
        android:textSize="16sp"
        android:textStyle="bold"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toLeftOf="@id/pin_tuan_more"
        tools:text="商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品商品" />
```
<!-- more -->
## 全面屏适配 BottomSheet
我们的项目已经适配过全面屏了,适配方式也很简单,在 AndroidManifest 文件中添加`<meta-data 
	android:name="android.max_aspect"
	android:value="2.1"/>`,但是在使用 BottomSheetDialog 时出现了意想不到的结果,如下图所示
![](https://def-201655.cos.ap-shanghai.myqcloud.com/ic_android_tips_bottomsheet.png)在最屏幕地步出现了布局上移的情况,而移动的距离正好是屏幕圆角的开始地方.
**解决办法:**
使用 BottomSheetDialogFragment 可以避免这个问题.
## 快速添加简单动画
当我们设置 View 的可见与不可见时感觉太生硬，可以给父布局添加`android:animateLayoutChanges="true"`属性,当然这种适用于很简单的动画,这个属性开启了默认的`LayoutTransition`.它有五种类型对应不同的动作.