---
title: Android 内存优化实践(上)
date: 2018-09-03 15:13:44
tags:
    - Android
    - 性能调优
categories: 
    - 技术
---
一直在写代码,感觉总结反思比较少,自己的表达能力很欠缺,所以想把自己的一些经验记录下来,顺便理清思路.
公司项目经历了一两年多的迭代,业务越来越复杂,由于是电商项目,有大量的图片需要加载,页面布局多样化,对 WebView 使用也很多,所以在内存方面开始出现较大的压力.在历经几个迭代处理相关问题后,现在首页 Tab 全部加载后内存比优化前内存使用减少40%,当打开十多个页面后,内存减少有30%以上,从 bugly 的日志来看已经没有 OOM 相关日志了,改进效果还是比较明显.
<!-- more -->
# 针对应用进行分析,优化前主要存在下面几个问题:
1. 图片问题
    1. 图片内存回收不及时
    2. 只有单份本地图片,但放置在 xhdpi 中,当高分辨率手机加载时,会占用较大内存
2. 布局优化
    1. 自定义 View 时,在 xml 根布局使用 LinearLayout 之类,并且在代码里又继承相应的 ViewGroup,造成重复嵌套
    2. 在复杂布局中嵌套过的多,RelativeLayout 使用也较多,影响布局测量绘制性能
    3. 页面中未经常使用的 View 使用 Gone 隐藏,而未使用 ViewStub,影响绘制性能
3. 某些 ViewPager 页面为使用懒加载
4. 某些页面可以递归调用,而未做任何限制,存在 OOM 风险
5. 复杂长页面使用 ScrollView,造成单页面内存消耗过大
6. onDestroy 中为将全局变量置空,导致内存回收缓慢
7. WebView
   1. 非 Web 页面嵌套 WebView,没做好内存释放处理,造成内存占用不能释放
   2. Web未使用独立进程,而 WebView 中 VR 展示,内存消耗严重

# 反思
有的问题是编码习惯问题,平时没有养成良好的习惯,导致代码维护越来越难做,补丁越打越多,对性能开始产生了影响.
有的问题是思考太少,着急写代码而没有考虑好应用场景,对后来的扩展和性能产生了影响.
有的则是自己水平不够,没有好好学习,意识不到存在问题.

现在又让我想起了代码整洁之道里的那句话.&#123;% note danger %&#1235; 让营地比你来时更干净 &#123; endnote %&#125;以后在写代码时一定要默念,时刻警惕让其成为一种习惯,这样自己所犯的错误也就少一点.

# 解决方案
 ## 图片内存及时释放
项目目前使用的是图片框架是 Glide3.7.0 版本,由于其他 library 也在依赖这个版本,所以目前还没有迁移到最新版本,打算抽空更新下,毕竟图片加载框架对于加载大量图片的应用还是很重要的,直接关乎体验和性能.Fresco 本身对于图片的请求、内存的优化和各种图形变换,有很好的支持和体验,但是由于其必须指定宽度或者高度,对于目前项目的迁移存在较大的障碍,目前不予考虑.

在整合进项目中,我们使用自定义 ImageView,将 Glide 各种默认策略封装在 View 内,对外直提供默认的方法就可以使用,消除有多余配置,对于特殊的需求提供统一方法,进行自定义配置.简化调用逻辑.

当在页面内滑动时,页面滑出屏幕,这时候 View 已经看不到,但是内存依旧在占用,而没有及时释放,而这样的的情况在打开页面较少时看起来无所谓,但是一旦 Activity 栈中有大量历史 Activity 而未销毁,就会给内存带来很大压力,**雪崩时,没有一片雪花是无辜的**.所以要防范这种情况的发生.如果首页就加载了大量的图片,占用过多内存,那么对于整个程序可使用的内存就会减少很多,用户随便进入多级页面,内存就快预警.

经过上面的分析,及时释放图片占用的最好的时机就是 View 滑出时回收,显示出来时再加载.于是`onAttachedToWindow`和`onDetachedFromWindow`这两个 View 生命周期里的方法就起作用了.Activity 中的 onAttachedToWindow调用是在 onResume 之后调用,而 onDetachedFromWindow 调用是在 onDestroy 之后,所以当 Activity 增加时,历史 Activity 屏幕中的图片内存释放不掉,也是一个隐患点,但是配合业务逻辑不会出现特别多的 Activity 叠加的情况,还是可以接受,如果有必要这个点可以以后优化.
``` java
public class SmartImageView extends AppCompatImageView {

    ··· 省略其他变量
    private String mUrl;

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        onAttach();
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        onDetach();
    }

    //当Viwe可见时,将图片重新加载
    private void onAttach() {
        setImageUrl(mUrl);
    }

    //将图片不可见时,及时清除View的图片
    private void onDetach() {
        setImageDrawable(null);
    }
    
    //只是示例方法,所以很简单的描述清楚原理,对于服务器返回图片的尺寸后,根据View真是大小,计算缩放比例,使用override去加载在屏幕实际尺寸,减少不必要的内存消耗
    public void setImageUrl(String url){
        DrawableRequestBuilder<String> builder = Glide.with(getContext().getApplicationContext())
                    .load(url)
                    .dontAnimate()//有的机型出现图片后出现偏移,故去掉加载动画,便恢复正常
                    .fitCenter();
            //不能使用builder.into(this) 以为Context使用ApplicationContext,所以有可能此时view销毁,而图片加载成功后,导致内存泄漏
            builder.into(new SimpleTarget<GlideDrawable>() {
                @Override
                public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> glideAnimation) {
                    if (resource != null && getContext() != null) {
                        setImageDrawable(resource);
                    }
                }
            });
    }
}
```
这样就可以保证图片在滑出屏幕后内存得到及时的回收,而不用等到界面销毁后才能回收图片内存.最大限度的降低图片对于内存的压力.
对于 Android 8.0 以上图片不再使用堆存放,而是分配在 Native 中,并且 **不需要用户手动回收、不受应用内存分配限制**,对于多应用来说真是莫大的好消息,虽然不受最大堆限制,但是当 Native 内存过大也会造成系统层面的 OOM,而且应用本身不会捕获此异常. 在 Android 2.3 以下图片内存分配也是这样的逻辑,由于不能回收不及时所以废弃不使用,但是在 Android 8.0 加入一种自动回收 Native 內存的 NativeAllocationRegistry机制,对开发者而言是透明的.不用手动回收所分配的内存.
## 本地图片处理
* 对于本地资源图片需要尽量压缩,设计师给过来就直接放进项目了,再次检查时发现有的图片较大,当载入时会占用更多内存.
* 对于图片格式可以转换 webp,这种编码格式在大小和质量有很好的平衡,在 Android4.2 版本以上解决了有关bug,可以放心使用,当然可以使用第三方支持到 Android4.1 版本.
* 对于尺寸较大的图片可以放入 xxhdpi,如果将高分辨率图片放入低密度目录，将会造成低端机占用较大内存可能造成 OOM.
## 布局优化
* 开发中经常有多个 View 组合,简化 Activity 中代码的逻辑,可以在多个页面复用,比如有时会继承 LinearLayout 然后在构造方法里用 LayouInfflate r将 xml 布局代码加载进来,这个方式看似没有毛病,原来也经常这么写,但是在使用 Layout Inspector 分析布局时发现造成了不必要的嵌套,所以需要改变方式.
    * 继承各种 Layout 后直接new出所需要的 widget,但是这样不利于直观的分析布局,并且存在大量的 java 代码,很费力.
    * 继承各种 Layout 后并且使用 LayoutInflater 加载 xml 布局,但是 xml 布局里使用 merge,这样就可以减少一层布局嵌套.
      相比较而言第二种方式更方便、更直观和符合习惯,我们也采用第二种方式.
``` xml
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:layout_height="match_parent"
    tools:layout_width="match_parent"
    tools:parentTag="android.widget.FrameLayout">
</merge>
```
使用 parentTag 指定当前布局的是哪种类型.这样做可以实时预览效果,但是根布局的属性都要在 java 代码指定.
* Google 在 2016 I/O 大会上推出 ConstrainLayout 布局.解决了复杂布局使用各种 Layouts 互相嵌套的问题,可以让布局更加扁平化,对于现在开发使用 LinearLayout, FrameLayout 和 ConstrainLayout 完全足以.具体的用法可以参考[ConstrainLayout Guide](https://developer.android.com/training/constraint-layout/).
* 打开一个新Activity并且请求网络时,就会出现四种状态:加载中、显示内容、显示错误、显示空页面,对于不经常显示的错误和空页面我们在 Activity 根布局中却一直解析,只是调用 setVisible(GONE),消耗了不必要的性能和内存.因此使用 ViewStub 可以很好的解决这个问题,这个 ViewStub 相当于懒加载,只有记载它时才会初始化,否则不会解析. ViewStub 不能应用 merge 标签.这个特性很适合在首页弹出促销的 WebView,平时不需要加载节省内存.
``` java
((ViewStub) findViewById(R.id.my_layout)).setVisibility(View.VISIBLE);
// 或者
View importPanel = ((ViewStub) findViewById(R.id.my_layout)).inflate();
```
    ``` xml
<!--  android:layout 指定所要加载的布局-->
<ViewStub
        android:id="@+id/stub_home_web"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@android:color/transparent"
        android:layout="@layout/layout_web" />
    ```
# Android Profiler
内存优化是个任重道远的的过程,需要不断的验证观察,Android Studio 提供了强大的性能分析工具,在 Android Profiler 里可以很方便的观察和分析 CPU、内存和网络的状态.[Android Profiler 使用参考](https://developer.android.google.cn/studio/profile/memory-profiler)里面具体用法,可以帮助我们发现各种潜在问题.