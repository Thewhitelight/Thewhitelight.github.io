---
title: React Native 版本升级
date: 2020-12-29 18:47:26
tags:
    - React Native
    - Android
categories: 技术
---

# 背景

React Native 版本迭代速度相对较快，所以在使用过程中就会遇到升级问题。我们也遇到了这个问题，但是由于我们的业务形态导致升级后还必须兼容低版本的 React Native 前端代码，所以就不能使用官方提供的[方案](https://react-native-community.github.io/upgrade-helper/)。

# 方案

对于 React Native Android 代码升级，最简单的就是将所有源码覆盖，然而事情并没有想的这么简单。

<!-- more -->
# 问题

1. React Native js 端版本校验
2. 当我们将源码升级至高版本后，再运行低版本的 js 代码时会出现如下错误
```
Error: `WebView` has no propType for native prop `RCTWebView.geolocationEnabled` of native type `boolean`
If you haven't changed this prop yourself, this usually means that your versions of the native code and JavaScript code are out of sync. Updating both should make this error go away
```
# 解决方案

1. 在[React Native 版本校验](https://libery.cn/2020/05/11/react-native-check-version/)篇文章中简析了版本校验过程，所以首先要去除校验逻辑，这个逻辑需要在 js 端去除，这样在开发阶段就不会出现相应的报错
2. 出现这个问题是因为 React Native 对于会对于 View 有属性校验，这段校验逻辑出现在 verifyPropTypes.js 

``` verifyPropTypes.js 
    // 核心判断代码如下，大概可以看出因为 js 的属性没有在 Native 源码里
    var nativeProps = viewConfig.NativeProps;
    if (
      !propTypes[prop] &&
      !ReactNativeStyleAttributes[prop] &&
      (!nativePropsToIgnore || !nativePropsToIgnore[prop])
    ) {
```

verifyPropTypes 通过这个类名就可以看出是校验 View 属性，通过搜索可以看出 requireNativeComponent 里调用了此方法

``` requireNativeComponent.js
    if (__DEV__) {
      componentInterface &&
        verifyPropTypes(
          componentInterface,
          viewConfig,
          extraConfig && extraConfig.nativeOnly,
        );
    }
```

这个判断条件看出只是在开发阶段会有此校验，当正式环境时则不会有校验，可以将源码中的此方法注释，从而避免出现校验问题。

遇到问题多读源码，才能了解问题的源头。

