---
title: 简析 Android ReactView 加载逻辑，修改 ReactView 尺寸
date: 2020-09-17 20:51:07
tags:
    - React Native
    - Android
---

> ReactView 相对是所有 RN 创建出的 View 统称

# 背景
在开发过程中遇到个比较特别的需求，在现有的 RN 源码不改动的情况下，以竖屏的方式在横屏的 Activity 上显示 RN 页面。
听起来比较绕其实效果很简单，如下图
```
 _____________________________________________
|             |                 |             |
|             |                 |             |
|             |                 |             |
|             |                 |             |
|             |                 |             |
|    Empty    |    Activity     |    Empty    |
|             |      View       |             |
|             |                 |             |
|             |                 |             |
|             |                 |             |
|             |                 |             |
|_____________|_________________|_____________|
                      平板
```
如果是纯原生开发的页面，我们最简单的方式就是修改 RootView 的宽高或者 Window 的 宽高。但是经过尝试当我们改变宽高后 RN 页面里的 View 并未随着改变，还是按照横屏模式下屏幕的宽高显示。

遇到这种问题肯定是 RN 源码有自身的逻辑，所以需要撸一遍源码才能搞清楚是哪里有了和我们预想的差异。
<!-- more -->
# ReactView 加载流程
## 创建 ReactRootView
当我们创建一个新的 RN 工程的时候，会看到对应的一个继承于 ReactActivity 的 Activity，ReactActivity 的具体实现则是在 ReactActivityDelegate，通过 ReactActivityDelegate 代理 Activity 主要的生命周期(onResume,onPause,onDestroy),让我们写的各种 Module 有了生命周期，在 onCreate 的调用方法里能看到到初始化了 ReactRootView，并且也看到了熟悉的 setContentView 方法，大概就能猜出来 ReactRootView 是所有 React View 的根布局，它是继承了 FrameLayout，在 ReactRootView 里绑定了 ReactInstanceManager、RN 入口组件名和启动参数。
## ReactInstanceManager
通过构造方法我们大概了解到它指定初始化各种组件及指定 js 引擎、jsbundle、UI实现类、生命周期等。我们这次重点看 ui 的加载，所以其他忽略。
## NativeViewHierarchyManager 
它是把所有 ViewManger 缓存起来，然后根据队列的变化操作 ViewManager 的新增删除从而更新界面
## UIImplementation
从名字就可以看出是 ui 加载的实现类，通过它将各种 js 布局转化成原生布局。RN 使用 yoga 引擎将 CSS 各种属性转化成原生可识别的属性，ShadowNodeRegistry 则记录了 ReactShadowNode 所有 view 的位置、层次之类的信息，UIViewOperationQueue 则是 ui 各种操作队列，
在 ReactShadowNode 的最终实现类 LayoutShadowNode 中可以看到各种被`@ReactProp`注解标记的方法，如
```
  @ReactProp(name = ViewProps.WIDTH)
  public void setWidth(Dynamic width) {
    if (isVirtual()) {
      return;
    }
    // *** 忽略各种实现细节
  }
```
这样 js 就可以使用这个方法设置宽高了。
## UIManagerModule
这个类则是 js 直接会调用到的类，它则通过调用 UIImplementation 去操作不同的 view

## 总结
通过上面的分析过程，发现其实通过倒叙来看这个逻辑会更清晰。
js 
-> 经过 C++ 层 调用到 java
-> 调用 UIManagerModule 各种的 ReactProp 方法，设置布局属性和自身属性
-> UIImplementation 里创建各个 View 的 ReactShadowNode,在 NativeViewHierarchyOptimizer 优化 UIViewOperationQueue，并不是每个 js view 都需要被转化成 Native View，在 ViewProps 类可以查到相关属性
-> UIViewOperationQueue 各种 View 的待操作队列
-> NativeViewHierarchyManager 里进行对应的 ViewManager 操作，从而改变 Native View

# 修改 ReactView 尺寸

前端对于屏幕适配时会通过 Dimensions 获取屏幕的宽度，并通过宽度适配 View，所以这是突破口。当我们在源码里搜索这个关键字时，发现是在 DeviceInfoModule 里的常量将屏幕信息传给前端。下面代码即是传给前端的值
``` java
  private WritableMap getDimensionsConstants() {
    // 此方法已经废弃 不建议给前端使用
    DisplayMetrics windowDisplayMetrics = DisplayMetricsHolder.getWindowDisplayMetrics();

    DisplayMetrics screenDisplayMetrics = DisplayMetricsHolder.getScreenDisplayMetrics();
    WritableMap screenDisplayMetricsMap = Arguments.createMap();
    //当我们修改这里的宽高，变可以完成适配上述的宽度
    screenDisplayMetricsMap.putInt("width", screenDisplayMetrics.widthPixels);
    screenDisplayMetricsMap.putInt("height", screenDisplayMetrics.heightPixels);
   //....
    WritableMap dimensionsMap = Arguments.createMap();
    dimensionsMap.putMap("screenPhysicalPixels", screenDisplayMetricsMap);

    return dimensionsMap;
  }
```
getScreenDisplayMetrics 则是通过
``` java 
  WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    Display display = wm.getDefaultDisplay();
      display.getRealMetrics(screenDisplayMetrics);
```
最终获取屏幕真正的宽高，这个宽高包含虚拟按键。
## 注意
当 RN 使用 RCTModalHostView 这个组件时，这个组件调用了原生 Dialog，所以需要将 Dialog 的尺寸重新设置。

# 参考
[ReactNative是如何让JS代码『变成』Android控件的](https://zhuanlan.zhihu.com/p/32263682)
[ReactNative设计与实现之四：Android端源码分析](https://zhuanlan.zhihu.com/p/45837390)