---
title: Android P 反射灰名单方法引起的问题
date: 2019-03-17 21:45:19
tags:
    - Android
    - 版本适配
categories: 
    - 技术
---
**本文中的灰名单指的是 Aandroid 规定的深灰名单**
我们应用使用了 [Pandora 1.3.2 版本](https://github.com/whataa/pandora) 库,方便测试环境里查看界面元素、网络、数据库之类的数据.在使用 Aandroid 9.0 时发现会弹出下面这样的提示框.需要探寻下是什么原因引起的,在那里调用的,能不能去掉这个强制提示.
![](https://def-201655.cos.ap-shanghai.myqcloud.com/android_p_invoke.jpg)
应用的 targetSdkVersion 是 27,compileSdkVersion 是 28
Pandora 2.0 版本目前已经规避了这个问题,都使用了浅灰名单里的方法或字段.
<!-- more -->
## 寻找问题
在打开 Pandora 之后,使用了其中一项功能发现 Logcat 中输出了` Accessing hidden field Landroid/view/ViewRootImpl;->mWindowAttributes:Landroid/view/WindowManager$LayoutParams; (dark greylist, reflection)`这段信息,应该是他反射调用了灰名单的字段,所以会有提示和弹框.
首先将这段提示复制到 AS 里通过全局搜索发现,是在 Activity 的 performStart 方法里调用了提示框
``` java Activity.java

    final void performStart(String reason) {
        //省略了其他代码
        //白灰黑名单维护在系统,对系统应用不做限制
        boolean isApiWarningEnabled = SystemProperties.getInt("ro.art.hiddenapi.warning", 0) == 1;
        if (isAppDebuggable || isApiWarningEnabled) {
            if (!mMainThread.mHiddenApiWarningShown && VMRuntime.getRuntime().hasUsedHiddenApi()) {
                // 仅提示一次
                mMainThread.mHiddenApiWarningShown = true;

                String appName = getApplicationInfo().loadLabel(getPackageManager())
                        .toString();
                String warning = "Detected problems with API compatibility\n"
                                 + "(visit g.co/dev/appcompat for more info)";
                if (isAppDebuggable) {
                    //弹窗提示
                    new AlertDialog.Builder(this)
                        .setTitle(appName)
                        .setMessage(warning)
                        .setPositiveButton(android.R.string.ok, null)
                        //竟然如此强制,生怕开发者不注意关闭
                        .setCancelable(false)
                        .show();
                } else {
                    //如果非 debug 版本只 Toast 提示
                    Toast.makeText(this, appName + "\n" + warning, Toast.LENGTH_LONG).show();
                }
            }
        }

    }
```
看过源码知道了调用地点,那么只改变调用条件就可以将它不提示了.这个变量便是我们的突破口
## 解决问题
### targetSdkVersion 27
* VMRuntime.getRuntime().hasUsedHiddenApi() 这个方法在AS里无法查看,但是在 Androidxref 查看后是个 Natvie 方法,反射修改起来困难有点大 
* mMainThread.mHiddenApiWarningShown 这个变量很明显反射容易点,并且根据注释和源码得知,也就只在这里使用
查看源码这个变量来自于 ActivityThread 那么我们直接反射修改这个变量初始值即可.
``` java
   try {
            Class cls = Class.forName("android.app.ActivityThread");
            Method declaredMethod = cls.getDeclaredMethod("currentActivityThread");
            declaredMethod.setAccessible(true);
            Object activityThread = declaredMethod.invoke(null);
            Field mHiddenApiWarningShown = cls.getDeclaredField("mHiddenApiWarningShown");
            mHiddenApiWarningShown.setAccessible(true);
            mHiddenApiWarningShown.setBoolean(activityThread, true);
            Log.e("reflection dark greylist is","success");
        } catch (Exception e) {
            e.printStackTrace();
            Log.e("reflection dark greylist is","fail");
        }
```
将反射修改变量方法放在 Application 初始化里便可以了.再也不用烦恼提示框的出现了.
### targetSdkVersion 28
当在 targetSdkVersion 为28时,系统是不允许反射灰名单中的 API,如果使用则会抛出这个错误`java.lang.NoSuchFieldException: No field mWindowAttributes in class Landroid/view/ViewRootImpl; (declaration of 'android.view.ViewRootImpl' appears in /system/framework/framework.jar!classes2.dex)`,而我们反射的 ActivityThread 类也在灰名单,这样就尴尬了陷入死循环里了.Google Play 对于 2019/5/1 以后上线的 APP 必须基于 28 版本开发,所以解决各种 hook 问题是很有必要和迫切的.
对于反射灰名单的字段或者方法都会抛出如下异常![hidden-api](http://gityuan.com/images/hidden-api/hidden-api-exp.png)
#### 方案一
但是天无绝人之路,万物皆可反射,有大佬给出了解决办法思路骚的不行,感觉自己真的思维限制了我的想象.通过**以系统身份去反射**拿到 API ,然后再去反射修改灰名单 API,这样系统就认为我们第二次修改的时候是系统类而豁免了检查.[另一种绕过 Android P以上非公开API限制的办法](http://weishu.me/2019/03/16/another-free-reflection-above-android-p/),这个是大佬的博文,详细阐述了怎么解决这个问题,并且在 GitHub 开源了代码.鉴于 Android P 版本已经发布,所以这个方式很稳定,可以用于生产环境.而且在 Android Q beta 1 里测试也可以使用该方式.
#### 方案二
Pandora 2.0 版本更新后查看了下源码是怎么实现绕过灰名单检查机制的,然后也发现了很巧妙的做法,将 Class 的 classloader 置空,从而系统自动调用了 BootCalssLoader 去加载,即成功伪造成了系统方法去反射灰名单字段,绕过了检查机制,思路也是很棒.[Reflect28Util](https://github.com/whataa/pandora/blob/master/pandora-core/src/main/java/tech/linjiang/pandora/util/Reflect28Util.java)这个是反射工具类,所有的反射都通过这个工具类进行操作.

## 参考文章
[Pandora 绕过检查](https://github.com/whataa/pandora/blob/master/pandora-core/src/main/java/tech/linjiang/pandora/util/Reflect28Util.java)
[FreeReflection](https://github.com/tiann/FreeReflection)
[另一种绕过 Android P以上非公开API限制的办法](http://weishu.me/2019/03/16/another-free-reflection-above-android-p/)
[一种绕过Android P对非SDK接口限制的简单方法](http://weishu.me/2018/06/07/free-reflection-above-android-p/)
[理解Android P内部API的限制调用机制](http://gityuan.com/2019/01/26/hidden_api/)