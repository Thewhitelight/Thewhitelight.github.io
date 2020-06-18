---
title: Android 内存优化实践(下)
date: 2018-09-13 22:35:04
tags:
    - Android
    - 性能调优
categories: 
    - 技术
---
# Web 独立进程
一般应用中会大量使用网页,我们项目由于有家装 VR 页面,如果都放在主进程则会造成大量的内存占用,因此将 WebView 拆分为独立进程运行,从而减轻主进程内存压力很有必要,当内存紧张时,系统则会自动杀死 web 进程.拆分为多进程后,主要问题在于进程间通讯与主进程保活.

## 拆分 Web 进程
独立进程分为两种模式,**私有独立进程** 和 **全局独立进程** 两种模式,开始方式也很简单.
``` xml
 <!-- 私有独立进程:与主进程同ShareUID,共享data目录、组件信息、共享内存数据  -->
 <activity
            android:name=".WebActivity"
            android:process=":web"/>
<!-- 全局独立进程:与主进程不同ShareUID -->
 <activity
            android:name=".WebActivity"
            android:process=".web"/>
```
本项目使用的是第一种方法,因为只与本应用通信,所以不需要为全局独立进程.
<!-- more -->
## 进程间通信
进程间的通信方式:
1. Bundle
2. 文件共享
3. AIDL
4. Messager
5. ContentProvider
6. Socket

当打开网页时,发送请求应该带用户信息,原来的信息存储方式是SharePreference,但是这种方式对于多进程调用时,容易出现不稳定的情况,并且它的多进程调用方式已经被标记为废弃,所以为了保证稳定性,使用ContentProvider将其封装,供不同的进程调用.
``` java
/**
 * 在 web 进程里只是获取用户信息,当web里返回要登录信息时,跳转至主进程里的登录页面,所以只实现了 query 方法,更新 SharePreference 依旧使用了单进程读写模式.
 */
public class UserProvider extends ContentProvider {

    private static String sAuthoriry = BuildConfig.APPLICATION_ID + ".UserProvider";

    @Override
    public boolean onCreate() {
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        //这个 name 获取的就是 xml 的文件名,默认取 uri 的 path 字段的第一个
        if (!sAuthoriry.equals(uri.getAuthority())) {
            return null;
        }
        Bundle bundle = new Bundle();
        if (getContext() != null) {
            bundle.putString("user", SharedPreferUtil.get(getContext(), Constants.EXTRA_USER_CACHE, ""));
        }
        return new BundleCursor(bundle);
    }

    private static final class BundleCursor extends MatrixCursor {
        private Bundle mBundle;

        public BundleCursor(Bundle extras) {
            super(new String[]{}, 0);
            mBundle = extras;
        }

        @Override
        public Bundle getExtras() {
            return mBundle;
        }

        @Override
        public Bundle respond(Bundle extras) {
            mBundle = extras;
            return mBundle;
        }
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        throw new UnsupportedOperationException("No external call");
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        throw new UnsupportedOperationException("No external call");
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
        throw new UnsupportedOperationException("No external call");
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
        throw new UnsupportedOperationException("No external call");
    }
}

```
AndroidManifest.xml 配置
``` xml
<!-- 由于只与本应用通信所以就没有配置权限 -->
 <provider
            android:name=".provider.UserProvider"
            android:authorities="${applicationId}.UserProvider"
            android:exported="true" />

```
这样就配置好了多进程读取 SharePreference 的 ContentProvider,凡是读取用户信息的地方都需替换为如下方式
``` java
    @Nullable
    public static User getUser() {
        String authority = "content://" + BuildConfig.APPLICATION_ID + ".UserProvider";
        Uri uri = Uri.parse(authority);
        //由于多进程模式,所以 Application 会多次初始化,MyApplication.getInstance() 的初始化不能写在某个进程里,如果这样则其他进程获取不到实例,导致这里会出现NPE
        Cursor cursor = MyApplication.getInstance().getContentResolver().query(uri, null, null, null, null);
        if (cursor != null) {
            Bundle args = cursor.getExtras();
            cursor.close();
            if (args != null) {
                return User.stringToUser(args.getString("user"));
            }
        }
        return null;
    }
```
通过以上配置,就可以在不同进程里获取User对象.
我们业务中有个逻辑是分享网页,当用户点击网页分享按钮,调起分享页面,然后分享至微信,分享成功返回后调用 js,所以需要在微信回调中通知网页,使用BroadcastReceiver通知网页执行 js 脚本.
## 多进程注意事项
1. 静态成员和单例模式会失效
2. 线程同步机制失效
3. SharePreference 稳定性不能保证,使用 ContentProvider 封装,对外提供数据服务
4. Application 会多次创建,需要注意多进程间使用的对象是否初始化
在 web 进程中调用主进程功能都需要注意 Context 和数据的读取,否则会出现空指针的问题.
# 主进程保活
在完成进程拆分后测试中发现,当主进程占用一百多 MB 时红米 Note3 机器打开网页进程,再消耗一百多 MB时,系统会自动杀死主进程,导致返回到主进程会再次加载,为了避免这种问题发生,只能在网页进程启动后,将主进程置为前台进程.
进程保活话题如果要展开谈,可以写好多东西,这里只介绍我们应用的方法.逻辑很简单就是在主进程中启动一个前台 Service,然后再启动一个相同 ID 的 Service,最后停止一个 Service,这样通知栏里便不会出现通知,而应用在前台 oom_adj 值较高,进程不会被杀死.
``` java 
public class KeepLiveService extends Service {
    public static final int NOTIFICATION_ID = 0x11;

    public KeepLiveService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public void onCreate() {
        super.onCreate();
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
            //API 18 以下，直接发送 Notification 并将其置为前台
            startForeground(NOTIFICATION_ID, new Notification());
        } else {
            //API 18 以上，发送 Notification 并将其置为前台后，启动 InnerService
            Notification.Builder builder = new Notification.Builder(this);
            builder.setSmallIcon(R.drawable.push);
            startForeground(NOTIFICATION_ID, builder.build());
            ContextCompat.startForegroundService(getApplicationContext(), new Intent(this, InnerService.class));
        }
    }

    public static class InnerService extends Service {

        public InnerService() {
        }

        @Override
        public IBinder onBind(Intent intent) {
            return null;
        }

        @Override
        public void onCreate() {
            super.onCreate();
            //发送与 KeepLiveService 中I D 相同的 Notification，然后取消自己的前台显示
            Notification.Builder builder = new Notification.Builder(this);
            builder.setSmallIcon(R.drawable.push);
            startForeground(NOTIFICATION_ID, builder.build());
            stopSelf();
        }

    }
}
```
当然 Service 也必须在 AndroidManifest 中注册.
## 注意事项
1. compleVerison 27 targetVersion 26 如果再为更高的版本则通知拦会显示出应用正在后台运行,给用户造成不好的体验
2. 为了在用户体验和内存消耗间平衡,在 Application 的 onTrimMemory中,当 level 值大于等于 TRIM_MEMORY_MODERATE 且 Web 进程在后台后,主动杀死 web 进程.