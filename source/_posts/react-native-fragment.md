---
title: React Native Fragment 容器化
date: 2021-03-15 22:29:38
tags:
    - React Native
    - Android
categories: 
    - 技术
---
# 背景

> 以下分析 React Native 版本均为 0.59.10

RN 实现 Fragment 加载因为想将其放置于首页，当放置在首页时就必须考虑其加载速度、so Crash、native module 及 js Crash 后如何显示等问题。对于容器化则是需要将现有 natvie 功能规范化后向上提供，还有 Android 和 iOS 双端开放能力的统一，方便上层业务调用。

# 容器化

![](https://def-201655.cos.ap-shanghai.myqcloud.com/rn_app_framework.png)
{% cq %}RN 容器架构图{% endcq %}

<!-- more -->
RN 容器框架大概思路

* ReactNative 层: 
jsc 替换为 v8 ，增强稳定性及缩短首屏渲染时间。
* 插件层: 
RN 框架已提供网络、文件等通用能力，但是其不足以适用我们的需求，我们可以通过复写对应插件，在其基础上接入我们现有的能力，这样对于js层面来讲可以不用新增任何新插件就可以无缝复用我们现有能力。
对于性能监控我们需要通过对 RN dev 工具进行改造，在非 debug 模式下可以打开，监控 js 与 native 互调用，监控页面的 FPS 及布局等。
* 业务层: 
Bundle 下发的版本可以通过各个插件版本去匹配，不同插件版本的 app 下发与之对应的 bundle，这样才能正常加载，否则可能会发生异常。
0.59 版本已经支持 metro 分包，在客户端只需要通过预加载 base bundle，使用预加载缩短首屏渲染时间，这个提示效果最为明显。

## 搭建

* 一致性
在跨端开发过程中由于需要原生层面的支持，所以我们需要开发大量的插件对 RN 提供我们现有能力，所以三端人员需要协调好方法及入参出参。我们可以提供一套脚手架，通过 dsl 生成三端代码。
* 稳定性
RN 加载过程中涉及大量的方法调用，我们需要对此进行全链路的监控，对于异常发生时也需要可以获取足够的信息，辅助判断问题原因。

### RN 框架
* RN 在 0.59 以下虽然无法使用 hermes，但是由于已经有 JSI 层，所以可以无缝替换 v8 引擎。由于 v8 本身较大，可以使用 Kudo 大佬提供的 nointl 版本，剔除了国际化的相关代码，jsc 本身不支持国际化，所以使用此版本没有影响，对于 v8 引擎的对比可以参考(react-native-js-benchmark)[https://github.com/Kudo/react-native-js-benchmark]
* 当在首页加载多个 RN Fragment，为了 jsbundle 避免全局变量被修改等问题，需要使用RN多引擎，但是插件里无法获取当前 Fragment 里的数据，我们需要将页面的信息放到 ReactContext 中，以便插件获取信息。
### 插件
* 对于插件的改造，可以根据自身的需要复写 RN 官方提供的插件。可以新增监控插件，在 App debug 模式下打开 dev 相关工具。
* 对于新增插件可以通过脚手架生成统一模版，通过这种方式可以约束三端代码。具体思路可以参考[Pigeon](https://mp.weixin.qq.com/s/E24bY7nt2HL0Pl-vEkECXg),这是腾讯音乐开源的 Flutter 的插件生成工具，但是其思路可以迁移到 RN 上来。
### 业务层
* 基于 metro 拆包获得 base.jsbundle,在应用初始化的时候加载base，当进入RN页面时再加载biz.bundle,具体逻辑可以参考[MetroExample](https://github.com/yxyhail/MetroExample)，这个项目有全面的介绍分包加载逻辑。
* RN 框架默认实现使用 Activity,但是了解其原理就可以发现是可以移植到 Fragment，因为本质就是生成一个自定义 ViewGroup，只要我们将其生成的 ViewGroup 放置到 Fragment 就可以实现。在处理 Fragment 返回时，我们需要提供给 Actvity 是否可以返回的方法。

## 总结
这个只是简单的容器方案，对于应对复杂业务还有很多改进的地方，如对 RN 加载的整个生命周期的监控能力还很薄弱，对齐 RN 官方版本问题修复。

# 参考
> https://tech.meituan.com/2019/12/19/meituan-mrn-practice.html
> https://tech.meituan.com/2020/09/30/waimai-mobile-architecture-evolution.html
> https://github.com/yxyhail/MetroExample
> https://mp.weixin.qq.com/s/E24bY7nt2HL0Pl-vEkECXg
> https://github.com/Kudo/react-native-v8