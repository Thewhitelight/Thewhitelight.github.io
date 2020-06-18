---
title: React Native 的 NativeMoudle 生命周期
date: 2020-06-15 20:47:37
tags:
    - React Native
    - Android
---
# 前言
在混合开发时经常需要调用 Native 方法获取资源，此时需要以 NativeMoudle 的方式供前端调用。当继承ReactContextBaseJavaModule时可以重写的方法只有 onCatalystInstanceDestroy() 可以看出来是容器销毁时会调用的方法，但是其他的生命周期都没有相关方法。通过查询api发现`LifecycleEventListener`·这个listener可以监听主要的生命周期如`onHostResume() onHostPause() onHostDestroy()`通过这几个方法可以监听页面的生命周期，方便释放资源。

# 问题
但是有个很特殊的场景就是从锁屏界面调用 ReactNative 界面，这时 ReactNative 界面会在调用`onResume()`后直接调用`onPause()`,这个是由于系统机制决定的，无法更改这个逻辑。由于快速进入 pause 状态导致 NativeMoudle 的`onHostResume()`没有调用，造成无法触发这个方法里的逻辑。

# 分析
那这样的生命周期是从哪里开始调用，以什么方式开始调用呢。下面做个简单的分析。
当看到`onHostResume`时一定会发现和`Activity`的生命周期很相似，所有当需要从 Activity 中入手查看。本文忽略`ReactFragment`相关内容。
<!-- more -->
## NativeMoudle 加载流程
ReactActivity 是 ReactNative 的入口承载页面，ReactActivity 中实际是交给ReactActivityDelegate来完成。
1. ReactActivityDelegate 中初始化 ReactInstanceManager ，并将所有的 ReactPackage 添加进 ReactInstanceManager
2. ReactInstanceManager 中创建出所有的 ReactPackage  并将其设置进 CatalystInstanceImpl
3. CatalystInstanceImpl 中将所有 ReactPackage 传入 C++ 层，通过C++实现与js 通信

## NativeMoudle 生命周期
在 js 层触发 componentWillMount 时，NativeMoudle 已经初始化完成。已经可以调用相关的 ReactMethod 方法。
前面说 NativeMoudle 的加载流程，当 ReactInstanceManager 加载后，ReactRootView 调用 startReactApplication 时便开始在主线程上异步初始化 ReactContext、CatalystInstanceImpl。
``` java ReactInstanceManager.java
  //简化过的代码
  private void setupReactContext(final ReactApplicationContext reactContext) {
    synchronized (mAttachedReactRoots) {
      synchronized (mReactContextLock) {
        mCurrentReactContext = Assertions.assertNotNull(reactContext);
      }

      CatalystInstance catalystInstance =
        Assertions.assertNotNull(reactContext.getCatalystInstance());

      catalystInstance.initialize();
      //调用 NativeMoudle 生命周期方法
      moveReactContextToCurrentLifecycleState();

      for (ReactRoot reactRoot : mAttachedReactRoots) {
        attachRootViewToInstance(reactRoot);
      }
    }
  }

  private synchronized void moveReactContextToCurrentLifecycleState() {
    if (mLifecycleState == LifecycleState.RESUMED) {
      moveToResumedLifecycleState(true);
    }
  }

    //此方法会调用两次，在 onHostResume 里 force=false 调用，也会在 moveReactContextToCurrentLifecycleState force=true 时调用
    private synchronized void moveToResumedLifecycleState(boolean force) {
    ReactContext currentContext = getCurrentReactContext();
    if (currentContext != null) {
      //当 currentContext 不为空且未调用过
      // we currently don't have an onCreate callback so we call onResume for both transitions
      if (force
        || mLifecycleState == LifecycleState.BEFORE_RESUME
        || mLifecycleState == LifecycleState.BEFORE_CREATE) {
        currentContext.onHostResume(mCurrentActivity);
      }
    }
    mLifecycleState = LifecycleState.RESUMED;
  }

```
当 LifecycleState 为 RESUMED 且 ReactContext 不为空 时便会调用 NativeMoudle 的生命周期方法 `onHostResume()`。
当退出 Activity 时调用 ReactInstanceManager 的 onHostPause 方法进而 调用`moveToBeforeResumeLifecycleState()`
``` java ReactInstanceManager.java
  private synchronized void moveToBeforeResumeLifecycleState() {
    ReactContext currentContext = getCurrentReactContext();
    if (currentContext != null) {
      if (mLifecycleState == LifecycleState.BEFORE_CREATE) {
        currentContext.onHostResume(mCurrentActivity);
        currentContext.onHostPause();
      } else if (mLifecycleState == LifecycleState.RESUMED) {
        //因为 CatalystInstanceImpl 初始化后 mLifecycleState 会变为 RESUMED 所以会进入此分支
        currentContext.onHostPause();
      }
    }
    mLifecycleState = LifecycleState.BEFORE_RESUME;
  }
```
## 锁屏引起的异常流程
由于锁屏调用 ReactNative 界面会在页面未显示前快速调用`onPause`从而调用到 ReactInstanceManager `moveToBeforeResumeLifecycleState()`方法,使得 `mLifecycleState` 为 `LifecycleState.BEFORE_RESUME`
``` java ReactInstanceManager.java
  private synchronized void moveToBeforeResumeLifecycleState() {
    ReactContext currentContext = getCurrentReactContext();
    if (currentContext != null) {
      if (mLifecycleState == LifecycleState.BEFORE_CREATE) {
        currentContext.onHostResume(mCurrentActivity);
        currentContext.onHostPause();
      } else if (mLifecycleState == LifecycleState.RESUMED) {
        currentContext.onHostPause();
      }
    }
    //此处更改了 mLifecycleState 状态
    mLifecycleState = LifecycleState.BEFORE_RESUME;
  }
```
在`moveReactContextToCurrentLifecycleState()`中`mLifecycleState==LifecycleState.RESUMED`时才会调用 NativeMoudle 的`onHostResume()`,但现在已经不会调用到`moveToBeforeResumeLifecycleState`,导致在 NativeMoudle 的`onHostResume()`代码都不会调用。
# 解决办法
将`moveReactContextToCurrentLifecycleState()`修改下
``` java ReactInstanceManager.java
  private synchronized void moveReactContextToCurrentLifecycleState() {
      //注释掉原有的状态判断，可以在任意状态下调用    
    //if (mLifecycleState == LifecycleState.RESUMED) {
      moveToResumedLifecycleState(true);
    //}
  }
  ```
  而在`moveToResumedLifecycleState()`里含有关于状态的判断，所以不用担心会重复调用`onHostResume()`引起其他问题。
