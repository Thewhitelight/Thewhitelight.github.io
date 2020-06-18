---
title: Android 内存优化实践(中)
date: 2018-09-10 15:02:46
tags:
    - Android
    - 性能调优
categories: 
    - 技术
---
上一篇从图片加载和布局方面分析怎样优化内存,这样可以解决大部分的 OOM 问题了,但是为了尽可能小的占用内存,我们还需要从其他方面入手,不光是从编程角度,还需要在业务方面配合.

# ViewPager 页面使用懒加载
在首页、搜索等页面中需要使用 ViewPager 承载不同 Fragment,并且有的需要 ViewPager 嵌套 ViewPager,如果进入 Activity 就加载 Fragment 会消耗大量资源,如网络请求、内存、布局绘制耗时耗内存,对用户而言需要等待造成不好的体验.所以需要使用懒加载方式以减少网络请求、布局绘制等不必要的消耗.
懒加载的思路就是首次Fragment加载进Activity不可见的时候不初始化布局和请求网络,让Fragment切换为可见时,再加载布局、请求网络,这样就避免首次打开Activity卡顿和消耗不必要内存.在 Fragment 重写`setUserVisibleHint(boolean isVisibleToUser)`方法便可以获取到,Fragment 是否用户可见.
<!-- more -->
``` java
public abstract class LazyFrgment extends Fragment {

    /**
     * 标志位，标志已经初始化完成，因为setUserVisibleHint是在onCreateView之前调用的，
     */
    private boolean isPrepared;
    /**
     * 标志当前页面是否可见
     */
    private boolean isVisible;

    /**
     * 不需要懒加载，需要在onAttach时设置
     */
    private boolean hasNotLazyLoad;

    public void setHasNotLazyLoad(boolean hasNotLazyLoad) {
        this.hasNotLazyLoad = hasNotLazyLoad;
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        isPrepared = true;
        return inflater.inflate(getContentView(), container, false);
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        initView(view);
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        if (hasNotLazyLoad) {
            initData();
            loadData();
        }
    }

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        isVisible = getUserVisibleHint();
        if (isVisible) {
            onVisible();
        } else {
            onInvisible();
        }
    }

    protected void onVisible() {
        lazyLoad();
    }

    protected void onInvisible() {

    }

    protected void lazyLoad() {
        if (isVisible && isPrepared && !hasNotLazyLoad) {
            isPrepared = false;
            initData();
            loadData();
        }
    }

    /**
     * 设置Fragment布局
     *
     * @return 布局Id
     */
    protected abstract int getContentView();

    /**
     * 初始化View
     *
     * @param view ContentView
     */
    protected abstract void initView(View view);


    /**
     * 初始化View属性、加载基本数据
     */
    protected abstract void initData();

    /**
     * 加载网络或者数据库数据
     */
    protected abstract void loadData();

}
```
这样当继承 LazyFragment 后,只需实现四个抽象方法后,就可以完成懒加载.这样的方式也有助于减少网络并发导致 Token 失效的问题.当 ViewPager 中 Fragment 个数过多时,对内存也会造成很大压力,所以需要用`setOffscreenPageLimit`给 ViewPager 设置一个合适的 limit 值,当 ViewPager 加载 Fragment 超过这个值时,会回收栈底的 Fragment 缓解内存压力.
# 页面递归调用限制
当进入商品详情页后又可以进入店铺详情页,店铺详情页就可以进入商品详情页,这样的递归调用,理论情况一定会出现 OOM,就算不出现 OOM 也会让内存压力过大,有可能使应用卡顿.所以需要设置一个阈值,当页面递归调用超过这个阈值后,杀死这个阈值前所以的商品详情页.当然也可以有更复杂的规则,目前我们业务规则这样指定,但是总体思路不变.
在 API 14以后,Application 中提供了一个应用生命周期回调的注册方法，用来对应用的生命周期进行集中管理.这样可以免去手动维护 Activity 栈.
``` java
    /**
     * 在 Application 的 onCreate 调用
     */
    private void registerMainProcessActivityLife() {
        //记录商品详情页,Activity 也是堆栈结构,所以采用此数据结构
        final Stack<Activity> activities = new Stack<>();
        //商品详情页最多加载个数
        final int maxGoodDetailActivityNum = 3;
        registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                if (activity instanceof GoodsDetailActivity) {
                    if (activities.size() >= maxGoodDetailActivityNum) {
                        activities.peek().finish(); //移除栈底的详情页
                    }
                    activities.add(activity);
                }
            }

            @Override
            public void onActivityStarted(Activity activity) {

            }

            @Override
            public void onActivityResumed(Activity activity) {

            }

            @Override
            public void onActivityPaused(Activity activity) {

            }

            @Override
            public void onActivityStopped(Activity activity) {

            }

            @Override
            public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

            }

            @Override
            public void onActivityDestroyed(Activity activity) {
                activities.remove(activity);
            }
        });
    }
```
# 复杂长页面使用 vlayout
对于电商首页而言,页面承载内容需要十分丰富,并且数量也很多,如果使用 ScrollView 根本无法满足,如果使用 RecyclerView 需要编写大量的布局,对数据结构要求很高.十分感谢阿里在的开源项目[vlayout](https://github.com/alibaba/vlayout),简直是编写复杂页面之光.我们目前应用于首页、商品详情页、店铺详情页等结构样式较多,且内容较多的页面.vlayout 支持固定布局、浮动布局、浮动布局、瀑布流布局等复杂布局.`vlayout 是一个针对 RecyclerView 的 LayoutManager 扩展, 主要提供一整套布局方案和布局间的组件复用的问题。通过定制化的LayoutManager，接管整个RecyclerView的布局逻辑；LayoutManager管理了一系列LayoutHelper，LayoutHelper负责具体布局逻辑实现的地方；每一个LayoutHelper负责页面某一个范围内的组件布局；不同的LayoutHelper可以做不同的布局逻辑，因此可以在一个RecyclerView页面里提供异构的布局结构，这就能比系统自带的LinearLayoutManager、GridLayoutManager等提供更加丰富的能力。同时支持扩展LayoutHelper来提供更多的布局能力.`
# onDestroy 置空全局变量
正常情况下,Activity finish 后系统会销毁 Activity 的实例,当该 Actiivty 对象引用没有指向原先分配给该Activity对象的内存时，该内存便成为垃圾.JVM的一个系统级线程会自动释放该内存块.但是这个有可能不及时,我们手动置空后会更加积极的回收内存.
原来并没有在意这个操作会带来多大变化,但是在优化过程发现还挺有效.