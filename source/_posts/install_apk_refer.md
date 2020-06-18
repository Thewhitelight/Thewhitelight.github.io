---
title: 应用安装版本适配问题
date: 2019-02-21 15:34:41
tags:
    - Android
    - 版本适配
categories: 
    - 技术
---
使用 DownloadManger 进行应用内更新然后当下载成功时调起安装界面,感觉挺简单调用系统 api 就可以完成,但是里面适配的坑真的是很多.
<!-- more -->
## 踩坑
### 6.0以下

几乎没什么坑按照常规代码的下载安装即可
``` java
        //downloadId 使用 DownloadManager 时指定的 id
        Uri uri = downloadManager.getUriForDownloadedFile(downloadId);
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setDataAndType(uri, "application/vnd.android.package-archive");
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        mContext.startActivity(intent);
```
### 6.0

{% note info %} 首先要申请运行时读写权限 {% endnote %}
获得授权后,如果使用上述代码打开安装页面则会抛出下面的错误
{% note danger %} Caused by: android.content.ActivityNotFoundException: No Activity found to handle Intent { act=android.intent.action.VIEW dat=content: typ=application/vnd.android.package-archive flg=0x10000000 } {% endnote %}
应为在 6.0 中 `downloadManager.getUriForDownloadedFile(downloadId)` 获取的 uri 为 content://downloads/my_downloads/109 这种格式,显然这不是个文件地址路径导致上面的报错*,**但这个问题只存在于 6.0**,所以需要针对 6.0 版本进行判断处理
``` java
 if (Build.VERSION_CODES.M == Build.VERSION.SDK_INT) {
            DownloadManager dManager = (DownloadManager) mContext.getSystemService(Context.DOWNLOAD_SERVICE);
            DownloadManager.Query query = new DownloadManager.Query().setFilterById(downloadId);
            Cursor c = null;
            if (dManager != null) {
                c = dManager.query(query);
            }
            if (c != null) {
                if (c.moveToFirst()) {
                    int columnIndex = c.getColumnIndex(DownloadManager.COLUMN_STATUS);
                    if (DownloadManager.STATUS_SUCCESSFUL == c.getInt(columnIndex)) {
                        String downloadFileUrl = c.getString(c.getColumnIndex(DownloadManager.COLUMN_LOCAL_URI));
                        //获取 apk 地址安装
                        installAPK(Uri.parse(downloadFileUrl));
                    }
                }
                c.close();
            }
        } else {
            Uri downloadFileUri = downloadManager.getUriForDownloadedFile(downloadId);
            installAPK(downloadFileUri);
 }
```
获取 apk 地址后便可以进行安装

### 7.0

{% note info %} 对于面向 Android 7.0 的应用,Android 框架执行的 StrictMode API 政策禁止在您的应用外部公开 file:// URI。如果一项包含文件 URI 的 intent 离开您的应用，则应用出现故障，并出现 FileUriExposedException 异常。 {% endnote %}
具体做法 可以参考 [关于 Android 7.0 适配中 FileProvider 部分的总结](http://yifeng.studio/2017/05/03/android-7-0-compat-fileprovider/)

### 8.0以上

由于权限变更 需要在 AndroidManifest.xml 加入
``` xml
    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
```
否则会 RuntimeExcetption
在 GitHub 中看到有人反映,在 Android 8.0 Oreo 中,Google 移除掉了容易被滥用的“允许位置来源”应用的开关,在安装 Play Store 之外的第三方来源的 Android 应用的时候,竟然没有了“允许未知来源”的检查框,但是我手上的 8.1 和 9.0 系统则都有这个提示框. 但是不能排除有手机没有或者用户关闭过,所以还是加入权限判断及申请会比较好,这个 issues 正好提供了解决方案
[申请“允许位置来源”](https://github.com/yjfnypeu/UpdatePlugin/issues/51)

## 参考
[关于 Android 7.0 适配中 FileProvider 部分的总结](http://yifeng.studio/2017/05/03/android-7-0-compat-fileprovider/)
[申请“允许位置来源”](https://github.com/yjfnypeu/UpdatePlugin/issues/51)