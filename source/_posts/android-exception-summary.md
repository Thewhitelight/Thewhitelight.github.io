---
title: Android 开发异常总结
date: 2018-12-07 09:24:46
tags:
    - Android
    - 异常回顾
    - 版本适配
    - 兼容性问题
categories: 
    - 技术
---
在开发过程中遇到各种不太常见的问题,在此记录.
# 运行时异常
## Android 8.1 apk 安装权限问题 android.permission.ACCESS_ALL_DOWNLOADS
targetVersion 28 应用内更新安装 apk 如果没有添加flag将会引起以下错误,在华为 8.1.0(27) 系统遇到此问题,但是三星 8.0(26) 系统则没有,估计是内部处理逻辑不同吧.
```
    Writing exception to parceljava.lang.SecurityException: Permission Denial: reading com.android.providers.downloads.DownloadProvider uri content://downloads/all_downloads/7 from pid=3633, uid=10037 requires android.permission.ACCESS_ALL_DOWNLOADS, or grantUriPermission()
        at android.content.ContentProvider.enforceReadPermissionInner(ContentProvider.java:631)
        at android.content.ContentProvider$Transport.enforceReadPermission(ContentProvider.java:501)
        at android.content.ContentProvider$Transport.enforceFilePermission(ContentProvider.java:492)
        at android.content.ContentProvider$Transport.openTypedAssetFile(ContentProvider.java:420)
        at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:302)
        at android.os.Binder.execTransact(Binder.java:698)
```
解决办法: 添加`intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);`
完整安装代码
```  java 
 Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setDataAndType(downloadFileUri, "application/vnd.android.package-archive");
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        mContext.startActivity(intent);
```
<!-- more -->
## TimeoutException
在 bugly 中看到个别这种错误
![](https://def-201655.cos.ap-shanghai.myqcloud.com/exception_timeout.png)但是机型和页面等具体日志都没有,所以只能猜测问题.
查看源码,这个错误在这里被抛出来[finalizerTimedOut()](http://androidxref.com/9.0.0_r3/xref/libcore/libart/src/main/java/java/lang/Daemons.java#396),根据报错和源码查看可以得知是由于 Finalizer 对象创建过多导致这个报错.120s 后抛这个错,搜索了下发现是 oppo 的系统规定的120s,所以估计这个问题在 oppo 手机比较多.
解决办法:
1. 重写 该对象的 finalize 方法并手动 GC,但是不会可靠,只能尽量减少此类问题
2. 尽可能少的创建该对象
3. 通过反射置空 FinalizerWatchdogDaemon 中的 thread ,这样不会执行此线程,便不会有异常(https://stackoverflow.com/questions/24021609/how-to-handle-java-util-concurrent-timeoutexception-android-os-binderproxy-fin),这个里有个回答介绍了此做法
[再谈Finalizer对象--大型App中内存与性能的隐性杀手](https://yq.aliyun.com/articles/225755) 这篇文章对这个问题分析的很透彻,并且天猫对 Finalizer 有自己的监控体系,这个方法真是羡慕.

## java.lang.IllegalStateException: Only fullscreen opaque activities can request orientation
在 bugly 发现微信分享 WXEntryActivity 出现了这个 bug,并且都是8.0的设备
查看 8.0 源码发现在[这里](https://github.com/aosp-mirror/platform_frameworks_base/blob/39791594560b2326625b663ed6796882900c220f/core/java/android/app/Activity.java#L984)抛异常,但是我在 9.0 源码中又找不到这个方法,应该是被去掉了,不太理解这么做的原因是什么.
根据`ActivityInfo.isTranslucentOrFloating`方法得知,凡是在 Activity 的 theme 中设置了 `android:windowIsFloating 或 android:windowSwipeToDismiss 或 android:windowIsTranslucent`的值为真且指定了屏幕方向,都会抛出这个异常.
在 AndroidManifest 中查看发现,原来当时配置 WXEntryActivity 时设置了`android:screenOrientation="portrait"
            android:theme="@android:style/Theme.Translucent.NoTitleBar"`
由于这个 Activity 不对用户展示只是做了分发功能,所以去除`android:screenOrientation="portrait"`即可.
这种兼容性适配坑真是太多了,都是泪啊.

## 自定义相机拍照时出现 RuntimeException: takePicture failed
在线上看到这个 bug 时一头雾水,看源码也看不到一场抛出点,因为直接调用了 native 方法,自己对于 jni 不太了解,只能通过手动测试复现问题了,可是在原来的测试中从来没出现过这样的问题.以为是权限被拒绝导致相机没启动就点击拍照抛出这个错,在各种尝试后还是不能复现.最后在无意中将拍照键快速连击了之后抛出了这个一场,终于知道问题所在了.
解决起来很简单,只需要在点击之后将拍照按钮设置为不可用,拍完照的回调里设置为可用就可以了.

## RemoteException:Remote stack trace: at com.android.server.am.ActivityManagerService.enforcePermission()
测试环境里当 App 在后台时,有时会弹出异常崩溃提示框,然后再 bugly 中发现异常只在 9.0 设备里出现,看了下日志发现是 Leakcanary 库引起的.
在 9.0 源码里 AMS 的`enforcePermission`方法检查了`android.permission.FOREGROUND_SERVICE`权限,由于 AndroidManifest 文件没有配置这个权限,所以导致异常.在 9.0 里[创建前台服务](https://developer.android.com/reference/android/app/Service.html#startForeground(int,%20android.app.Notification))都必须申请这个权限.
对于应用 targetSdkVersion 的迁移一定要阅读 Android 官方的[迁移提示](https://developer.android.com/about/versions/pie/android-9.0-migration)以免出现重大问题.

* 对于 9.0 多进程使用 WebView 也作出了重大改变,可以参考[行为变更](https://developer.android.com/about/versions/pie/android-9.0-changes-28#web-data-dirs)里的建议修改

## 资源文件统一存放,不能随意申明否则会导致 FileNotFoundException
项目图片资源文件都在 xhdpi 下,但是由于多模块间引用,有时候不去依赖模块而是直接在自己模块里申明`<item name="test"  type="drawable"/>`一下,在屏幕分辨率高于 xhdpi 的手机上是正常运行,但是在低于这个屏幕分辨率下,就会直接读取了空的申明资源,导致会抛出`java.io.FileNotFoundException`.

# 编译异常
## SSL异常

电脑开着代理后,AS编译时有新的三方库更新时,会提示SSL错误
解决办法:关闭代理,重启AS(必须 command+q 退出AS)
## app:transformNativeLibsWithStripDebugSymbolForProduceRelease, not found mips64el-linux-android-strip
项目环境 AS 3.2.1, Gradle 3.0.1, NDK 17, 配置 NDK 后出现如下异常![](https://def-201655.cos.ap-shanghai.myqcloud.com/ic_android_exception_ndk.png)打开异常里抛出的文件路径发现没有找到这个文件,原因是 17 里删除了这个配置.
解决办法:
1. Gradle 版本升级到最新,现在是 3.2.1,但是这样做太麻烦了,如果版本较低要做大量的适配工作.
2. 下载 NDK 16,将 ndk-bundle/ 文件夹下的文件全部替换掉
3. 下载 NDK 16,将 mips64el-linux-android-strip 复制到抛错提示的文件路径里

## PKIX path building failed

报错信息 `sun.security.validator.ValidatorException: PKIX path building failed:
sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target`
解决办法:
关闭代理,点击 File->invalidate Caches/Restart 重启 AS

Error occurred while communicating with CMake server. Check log /Users/***/Company/react-native/ReactAndroid/.externalNativeBuild/cmake/debug/armeabi/cmake_server_log.txt for additional information.

## 打混淆包后,有包名被改为类似`a.a.a`

```
    buildTypes {
        release {
            minifyEnabled true
            //需要加入 Android默认混淆规则 proguard-android.txt 
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```