---
title: OKHttp动态切换Host
date: 2018-07-21 01:54:50
tags:
    - Android
    - OKHttp
categories: 技术
---

在日常开发中会分测试、预发、生产等环境,不同环境下请求 Host 不一致,原来的解决方案是通过定制不同的 Flavors,通过添加 applicationIdSuffix 可以让多个 App 同时安装在手机方便测试,但是又产生新的问题,每次都要打包上传不同的 App 让测试安装,这样感觉略显麻烦,所以就想到能不能 App 内部做环境切换,这样切换自如,无需重新打包.

<!-- more -->

#### 使用官方提供的@url修改Host

官方提供@url方法来动态改变请求的url,但是不符合我们的需求,使用@url只适合修改个别接口的情况,我们这样支持全部接口的修改显然工作量太大,几乎无法维护,如果App中使用第三方图片或者文件上传可以使用这个方法,只在个别接口修改url使用起来比较方便.

``` java
public interface ApiService {  
    @GET
    public Call<ResponseBody> uploadImg(@Url String url);
}
```

#### 封装不同 Host 的 OKHttpClient

由于 OKHttp 不支持动态切换h ost 并且 api 请求为单例模式,然后想到封装3个不同Host的OkHttpClient,然后通过策略模式去实现不同的请求调用,但是这样总感觉不太优雅,又占用内存.

#### 利用 Interceptor 切换 Host

继续想会不会有通过改变 api 的方式去实现这个功能,然后突然想到我们在添加 Cookie 时用到的 Interceptor,既然它每次都会拦截请求,那么我们可以在拦截到请求时换掉 Host,实现我们想要的功能.有了思路就可以动手开始看源码改造了.

在 Intercept 方法中通过 Chain获取 Request,修改 Request 中的 HttpUrl 来改变 Host.HttpUrl类用于获取 Host、端口,URL解析,和处理URL字符串等,但是发现HttpUrl,URL,Request类中的变量都是私有并且没有修改方法,只能通过反射方法去修改.不多说直接上代码.

``` java 
public class DynamicInterceptor implements Interceptor {

    /**
     * 在Api定义好不同的环境的Host
     */
    private static String mHost = Api.getHost();
    private static final String HOST = "host";

    /**
     * 对外提供动态修改host
     *
     * @param host 设置的新host
     */
    public static void setHost(String host) {
        mHost = host;
    }

    public static String getHost() {
        return mHost;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        //根据自己的判断逻辑 是否修改host
        if (AppUtil.canCheckoutHost() && !mHost.equals(Api.getHost())) {
            request = replaceHost(request);
        }
        return chain.proceed(request);
    }

    private static Request replaceHost(Request request) {
        HttpUrl httpUrl = request.url();
        URL url = httpUrl.url();
        try {
            
            //反射修改URL中的Host
            Class<?> urlClz = url.getClass();
            Field urlHost = urlClz.getDeclaredField(HOST);
            urlHost.setAccessible(true);
            urlHost.set(url, mHost);

            //反射修改HttpUrl中的Host
            Class<?> httpUrlClz = httpUrl.getClass();
            Field httpUrlHost = httpUrlClz.≈getDeclaredField(HOST);
            httpUrlHost.setAccessible(true);
            httpUrlHost.set(httpUrl, mHost);

            //反射修改Request中的HttpUrl
            Class<?> requestUrlClz = request.getClass();
            Field requestUrl = requestUrlClz.getDeclaredField("url");
            requestUrl.setAccessible(true);
            requestUrl.set(request, httpUrl);
            
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
        //用修改后的HttpUrl构造新的HttpUrl
        request = request.newBuilder().url(httpUrl).build();
        return request;
    }

}
```

``` java
//在OkHttpClient.Builder中加入即可完成动态切换
builder.addInterceptor(new DynamicInterceptor());
```

#### 总结

[项目地址](https://github.com/Thewhitelight/DynamicRetrofitHost.git) 目前这个方法已经满足需求,在MainActivity中使用悬浮按钮去动态切换Host很实用.
