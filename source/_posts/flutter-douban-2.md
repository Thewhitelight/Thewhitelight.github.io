---
title: Flutter 体验二(package开发)
date: 2018-08-31 14:06:20
tags:
    - Flutter
categories: 技术
---
Flutter 只是一个 UI 框架,对于一些系统调用或者各自平台的的特有实现需要开发者自行编写 package,通过在 package 内实现不同平台的调用方式,以这样的方式进行适配.今天记录下怎样在不同平台编写 package.由于自己不会 iOS,所以只是实现一个简单的 Android Toast.iOS 端使用定时的UIAlertView去模拟.
<!-- more -->

##### 新建 package
我的工具是 Android Studio,首先在项目根目录下new->module->选择 Fullter package,点击 next,然后看到这个界面![Flutter 目录结构](https://def-201655.cos.ap-shanghai.myqcloud.com/fultter_2_new_package.png)修改报名和描述,并且选择自己的 Flutter SDK 目录,最后点击 finish.一个新的 package 就新建完成了.当然如果使用 VSCode 的话也可以使用命令行新建 package.
package目录如下图所示![结构说明](https://def-201655.cos.ap-shanghai.myqcloud.com/flutter_2_package_module.png)

##### Android 端实现,即在这这里编写原生调用代码
``` java
public class FlutterToastPlugin implements MethodCallHandler {
    private static final String TAG = "FlutterToastPlugin";

    private static Context context;

    /**
     * Plugin registration.
     */
    public static void registerWith(Registrar registrar) {
        context = registrar.context();
        final MethodChannel channel = new MethodChannel(registrar.messenger(), "flutter_toast");//这里的  flutter_toast 包名不可修改,需要和iOS保持一直这样才可以调用生效.
        channel.setMethodCallHandler(new FlutterToastPlugin());
    }

    @Override
    public void onMethodCall(MethodCall call, Result result) {
        String message = call.argument("message");//获取 flutter 传入的参数 
        //获取传入的方法名,调用相应的不同逻辑
        switch (call.method) {
            case "showToast":
                Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
                break;
            case "showLongToast":
                Toast.makeText(context, message, Toast.LENGTH_LONG).show();
                break;
            case "getPlatformVersion":
                //默认实现
                result.success("Android " + android.os.Build.VERSION.RELEASE);
                break;
            default:
                result.notImplemented();
        }
    }
```

##### package 项目入口,我们可以在这里调用本身实现的功能,选择此文件 run 及时查看效果
``` dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_toast/flutter_toast.dart';

void main() => runApp(new MyApp());

class MyApp extends StatefulWidget {
  @override
  _MyAppState createState() => new _MyAppState();
}

class _MyAppState extends State<MyApp> {
  String _platformVersion = 'Unknown';

  @override
  void initState() {
    super.initState();
    initPlatformState();
  }

  Future<void> initPlatformState() async {
    String platformVersion;
    try {
        //默认实现调用手机系统版本号
      platformVersion = await FlutterToast.platformVersion;
    } on PlatformException {
      platformVersion = 'Failed to get platform version.';
    }

    if (!mounted) return;
    setState(() {
      _platformVersion = platformVersion;
    });
    //调用实现.showToast 即原生代码的的方法名 call.method, test toast 即原生代码的参数call.argument("message")
    FlutterToast.showToast("test toast");
  }

  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      home: new Scaffold(
        appBar: new AppBar(
          title: const Text('Plugin example app'),
        ),
        body: new Center(
          child: new Text('Running on: $_platformVersion\n'),
        ),
      ),
    );
  }
}
```

##### iOS 端实现 由于自己不会 Object-C 所以代码很简单,很容易看懂
{% codeblock lang:objc %}
#import "FlutterToastPlugin.h"

@implementation FlutterToastPlugin
+ (void)registerWithRegistrar:(NSObject<FlutterPluginRegistrar>*)registrar {
  FlutterMethodChannel* channel = [FlutterMethodChannel
      methodChannelWithName:@"flutter_toast"
            binaryMessenger:[registrar messenger]];
  FlutterToastPlugin* instance = [[FlutterToastPlugin alloc] init];
  [registrar addMethodCallDelegate:instance channel:channel];
}

- (void)handleMethodCall:(FlutterMethodCall*)call result:(FlutterResult)result {
  if ([@"getPlatformVersion" isEqualToString:call.method]) {
    //默认实现
    result([@"iOS " stringByAppendingString:[[UIDevice currentDevice] systemVersion]]);
  }else if([@"showToast" isEqualToString:call.method]) {//判断调用方法名
    //获取调用参数
    NSString *msg = call.arguments[@"message"];
    UIAlertView *alert = [[UIAlertView alloc]
                                 initWithTitle:nil
                                       message:msg
                                      delegate:nil
                             cancelButtonTitle:nil
                             otherButtonTitles:nil, nil];
    [alert show];
    double duration = 0.5;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [alert dismissWithClickedButtonIndex:0 animated:YES];
    });
  }else if([@"showLongToast" isEqualToString:call.method]) {
    NSString *msg = call.arguments[@"message"];
    UIAlertView *alert = [[UIAlertView alloc]
                                 initWithTitle:nil
                                       message:msg
                                      delegate:nil
                             cancelButtonTitle:nil
                             otherButtonTitles:nil, nil];
    [alert show];
    double duration = 1;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(duration * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [alert dismissWithClickedButtonIndex:0 animated:YES];
    });
  }else {
       result(FlutterMethodNotImplemented);
  }
}

@end
{% endcodeblock %}
若第一次把 package 运行在 iOS 端会提示错误,按照提示依次执行 `brew install cocoapods` 和 `pod setup` ,等待安装完成后,再次运行就可以在模拟器查看效果了.

##### Flutter 桥接代码实现
 ``` dart
 import 'dart:async';

import 'package:flutter/services.dart';

class FlutterToast {
  static const MethodChannel _channel = const MethodChannel('flutter_toast');
  //默认实现
  static Future<String> get platformVersion async {
    final String version = await _channel.invokeMethod('getPlatformVersion');
    return version;
  }
  //对外提供调用的方法
  static void showToast(String message) {
      //showToast 即调用原生的方法名, message 即调用原生的参数名,必须要一致否则无法调用
    _channel.invokeMethod("showToast", {
      "message": message,
    });
  }

  static void showLongToast(String message) {
    _channel.invokeMethod("showLongToast", {
      "message": message,
    });
  }
}

 ```

#### 总结
Flutter 插件编写逻辑还是很简单,具体的实现方式还没有具体研究,有时间需要大概理解原理,这样也又便于编程思维的开拓.通过这个 demo,熟悉了插件编写方式,对于一些 Flutter 不支持的功能就可以让原生实现,比如嵌入WebView就可以通过原生提供实现.
[GitHub地址](https://github.com/Thewhitelight/DouBanMovie)
[在线编写Flutter实时预览](https://flutterstudio.app/)