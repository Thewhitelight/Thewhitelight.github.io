---
title: Fultter 体验一(豆瓣电影 Api 简单使用)
date: 2018-08-26 20:03:23
tags:
    - Flutter
categories: 技术
---

#### Flutter

Flutter 是 Google 开发的一套全新的跨平台、开源UI框架，支持 iOS、Android 系统开发，并且是未来新操作系统 Fuchsia 的默认开发套件。物联网是未来趋势,所以还是很看好 Flutter 项目,看到跨平台就感觉很熟悉了,有 Cordova、ReactNative、Weex 等前辈,但是Flutter的思路与上述的框架有本质却别.Flutter 使用全新的 Dart 语言开发,分别在 iOS、Android 实现了各自的 UI 控件、渲染逻辑,所以相比其他框架其性能更接近原生,美团的实际对比也确实说明了这个问题.记得前两年 Google 推出过基于 Dart 语言的 Sky 项目,当时写过 Demo 体验了下,然而确实昙花一现再也没有人讨论了,不过现在我估计是 Sky 项目孵化出了 Flutter 项目.

<!-- more -->

Dart 语言的使用相对 Java 开发者还是很友好的,看看官方的 Demo 和 Api 就可以上手开发了,既有静态语言的特性也有动态语言的特性,妈妈再也不用担心项目重构了.Dart 的页面刷新有点像 ReactNative 改变 State 则页面自动对比刷新.相同页面效果相比原生可以少写很多代码,并且官方支持Material Design 组件,很多交互逻辑已经默认实现,并且社区也有很多开源组件供开发着调用原生功能.Dart 支持 Hot Reload 功能,对于开发确实方便很多,美团文章中介绍了热刷新的限制,但是在我实际使用的时候还是发生了意想不到情况,我改变变量类型,但是保存后没有刷新的问题,导致我以为是代码写的有点问题改了几次,想到试着重新运行下,发现就正常识别了.

#### Flutter 实践

Flutter 的安装参考中文网一步一步走下去,使用国内镜像几乎不会遇到什么坑,Android Studio 对其开发支持也十分友好,只需要下载 Flutter 和 Dart 插件后变可以编写代码了.
使用豆瓣 api,实现一个可以查看最近上映、top250 电影的简单App![Flutter 包管理](http://7qndg9.com1.z0.glb.clouddn.com/flutter_douban_yaml.png)上图是我项目的依赖包,其中 flutter_toast 是本地 Flutter package 桥接原生的 Toast,目前还没有适配 iOS,所以iOS端不能运行此项目.Dio 是一个网络请求框架,方便添加 header intercept 等.cached_network_image 是一个图片加载框架,可以设置缓存、加载动画之类.
![项目目录结构](http://7qndg9.com1.z0.glb.clouddn.com/flutter_douban_dir.png)这是项目的目录结构,main.dart即项目入口.
``` dart
//导包
import 'package:dou_ban_movie/douban/douban_hot_movies.dart';
import 'package:dou_ban_movie/douban/douban_top_movies.dart';
import 'package:flutter/material.dart';

//程序入口
void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
      //使用 Material 组件,构建符合 Material Design 规范的 app
    return new MaterialApp(
      title: 'Welcome to Flutter',
      //Android 中style.xml中的属性
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  @override
  _MyHomePageState createState() => new _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int mCurrentIndex = 0;

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: AppBar(
        title: Text("Douban Movies"),
      ),
      bottomNavigationBar: new BottomNavigationBar(
        items: [
          new BottomNavigationBarItem(
              icon: new Icon(Icons.all_inclusive), title: new Text('HOT')),
          new BottomNavigationBarItem(
              icon: new Icon(Icons.favorite), title: new Text('TOP'))
        ],
        currentIndex: mCurrentIndex,
        onTap: (index) {
            //点击刷新tab
          setState(() {
            mCurrentIndex = index;
          });
        },
      ),
      body: _setupCurrentPage(mCurrentIndex),
    );
  }

  Widget _setupCurrentPage(int mCurrentIndex) {
    switch (mCurrentIndex) {
      case 0:
        return new HotMoviesPage();
      case 1:
        return new TopMoviesPage();
      default:
        return new HotMoviesPage();
    }
  }
}

```
这是首页的 Dart 代码,他不同于 Android 原生将页面样式和控制代码分离的逻辑,他是全都揉和在一起,更多的通过组件的组合来实现代码的复用.并且代码量大幅减少.可以加快开发速度.

#### 总结

Flutter对于提升开发效率确实有一定帮助,并且有 Google 加持,前景感觉也很不错,有新系统对其的支持,应该比其他跨平台方案更有优势,但是目前而言还是在成长期,还是不够稳定.跨平台开发确实很吸引人,感觉一个前端可以扛起很多原生开发的任务,对于 mvp 阶段的的项目确实很适合,但是对于中大型项目全部使用这种技术,其中的风险很大,前方各种坑等着趟过去,最后发现还是得走混合开发的路子.所以打算试试将 Flutter 作为 module 整合进原生里面,这样更加灵活.在原来ReactNative实践中也试过这样的做法,发现还是很不错,对于一些简单的功能用此开发很方便,对于复杂的功能可以直接切换到原生去做.

#### 截图
![首页](http://7qndg9.com1.z0.glb.clouddn.com/Screenshot_20180826-225527.jpg)
![详情页](http://7qndg9.com1.z0.glb.clouddn.com/Screenshot_20180826-225535.jpg)
![查看图片](http://7qndg9.com1.z0.glb.clouddn.com/Screenshot_20180826-225548.jpg)


#### 项目地址
[Github 地址](https://github.com/Thewhitelight/DouBanMovie)
#### 参考
[Flutter 中国官网](https://flutter-io.cn/)
[Flutter 中文网](https://flutterchina.club/)
[Flutter 的原理及美团的实践](https://mp.weixin.qq.com/s/cJjKZCqc8UuzvEtxK1BJCw)