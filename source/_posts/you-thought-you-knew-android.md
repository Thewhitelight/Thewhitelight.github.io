---
title: 你以为你懂得 Android (译)
date: 2019-08-21 20:23:16
tags:
    - Android
    - 译文
categories:
    - 技术
---
#### 译文
你知道 Android 里有个叫`blink`的 ViewGroup 吗?惊讶吗?我第一次知道它时和你是同样的表现.

当我正在阅读[LayoutInflater](https://developer.android.com/reference/android/view/LayoutInflater)的源码,我对 xml 如何转换成 object 特别感兴趣,所以我没有从头开始阅读这个类,而是直接阅读`inflate(layoutRes,parent`方法.

经过一段时间的阅读,我在`View createViewFromTag(View parent, String name, Context context, AttributeSet attrs, boolean ignoreThemeAttr)`方法前停了下来,这个方法签名没有特别之处,但是以下的情况很奇怪,并且注释也是.
![](https://miro.medium.com/max/1400/1*4Nb8L6s0VJN5QrGO5RSgbw.png)
`
Let's party like it's 1995!
`
<!-- more -->
看起来很有意思,但是接下来这有趣就要被打破了,我们确切的知道它会发生什么.**一个 ViewGroup 让 它的子 View 像节日灯一样的闪烁**.但这 ViewGroup 的名字是什么呢?是`tag1995`吗?
不是的,`blink`才是这个 ViewGroup 的名字,不像 Android 里的`LinearLayout`或者`ReltaiveLayout`或任何其他的 ViewGroup 的布局后缀.那么它对任何 Android 开发者都大有用处吗?大部分不是的,但是知道有这样一个彩蛋是不错的🙂.
想看它的实际效果?如下所示
![Usage in xml layout
](https://miro.medium.com/max/3348/1*6dHMO7KqOUsW9pXbZ7E4Cw.png)
![Party!!!
](https://miro.medium.com/max/1200/1*QawR47hKULpQqtwyYJUbZw.gif)
在 Android Framework 源码里尝试发现彩蛋.不知道我能发现多少,如果我后面没有更多的文章可能就意味着没有了.像🤞这样交叉手指发现更多的彩蛋吧.
参考
[Romain Guy](https://medium.com/u/c967b7e51f8b?source=post_page-----e46a556d0773----------------------) 向 Android  Framwork 里提交了`blink`布局源码.
#### 参考
[原文地址](https://medium.com/@anoopss/you-thought-you-knew-android-e46a556d0773)
[blink 源码](https://android.googlesource.com/platform/frameworks/base/+/9c1223a71397b565f38015c07cae57a5015a6500%5E%21)