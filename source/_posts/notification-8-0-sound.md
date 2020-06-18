---
title: Android 8.0 自定义通知铃声
date: 2018-12-04 14:10:16
tags:
    - Android
    - 版本适配
categories: 
    - 技术
---
## 问题
Android 系统不断的更新, targetVersion 一直定在 25 ,感觉差的有点多,如果还不升级更新,以后再改估计坑会更多,所以在新的迭代里,将 targetVersion 升级到 28 . 果然和预想的一样,有适配的问题出现了.
在 8.0 和 8.1 的手机上通知不能播放自定义铃声,5.0.2 的手机又是可以正常播放.自定义播放铃声使用` channel.setSound(notificationUri, Notification.AUDIO_ATTRIBUTES_DEFAULT); `,然后看到网上说调整`channel.setImportance()`发现还是无效,但是设置这个值会影响通知的显示和是否震动、显示指示灯、静音等情况.抱着试一试的心态使用 NotificationCompat.Builder 的 setSound() 方法,果然还是不行.
<!-- more -->
## 方法一
没折只能硬着头皮看下 api 然后猜猜哪里出问题![](https://def-201655.cos.ap-shanghai.myqcloud.com/notification_8.0_setSound.png)看到最后一句话,必须在 createNotificationChannel() 方法之前修改铃声.但是我的代码确实也是这样写的.这里其实对于通知渠道这个概念理解错了.单纯的以为对于不同的铃声只要在同一个渠道里修改铃声就可以了,其实这样的方法是错的.**对于不同的通知类型(铃声不同,指示灯颜色不同、震动模式不同),必须使用不同的渠道(channelId 不同)**,理解了这个就很容易写出真正适配的代码了.

``` java 

    @TargetApi(Build.VERSION_CODES.O)
    private static void createNotificationChannel(String channel_id, Uri voiceUri) {
        NotificationChannel channel = new NotificationChannel(channel_id, CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH);
        channel.setBypassDnd(true);//是否绕过请勿打扰模式
        channel.enableLights(true);//闪光灯
        channel.setLockscreenVisibility(VISIBILITY_PUBLIC);//锁屏显示通知
        channel.setLightColor(Color.RED);//闪关灯的灯光颜色
        channel.setShowBadge(true);//桌面launcher的消息角标
        channel.enableVibration(true);//是否允许震动
        channel.setVibrationPattern(new long[]{100, 100, 200});//设置震动模式
        if (voiceUri == null) {
            //当无特殊通知铃声时,使用系统默认
            Uri uri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION);
            channel.setSound(uri, Notification.AUDIO_ATTRIBUTES_DEFAULT);
        } else {
            channel.setSound(voiceUri, Notification.AUDIO_ATTRIBUTES_DEFAULT);
        }
        getManager().createNotificationChannel(channel);
    }


    private static NotificationManager getManager() {
        if (mManager == null) {
            mManager = (NotificationManager) MyApp.getInstance().getSystemService(Context.NOTIFICATION_SERVICE);
        }
        return mManager;
    }

    /**
     * @param intent       点击通知打开相应 Activity 的 Intent
     * @param title        通知标题
     * @param content      通知内容
     * @param pushId       pushId 不能重复
     * @param soundCommand 通知铃声类型
     */
    public static void sendNotify(Context context, Intent intent, String title, String content, int pushId, String soundCommand) {
        NotificationCompat.Builder builder;
        PendingIntent contentIntent = PendingIntent.getActivity(context, pushId, intent,
                PendingIntent.FLAG_UPDATE_CURRENT);
        Uri uri = notifyVoice(soundCommand);
        //使用 hashCode 保证不同类型渠道 channelId 不重复
        String id = uri == null ? String.valueOf("channelId".hashCode()) : String.valueOf(uri.hashCode());
        builder = new NotificationCompat.Builder(MyApp.getInstance(), id);
        builder.setSmallIcon(R.drawable.ic_notify)
                .setLargeIcon(BitmapFactory.decodeResource(context.getResources(), R.mipmap.ic_launcher))
                .setContentTitle(title)
                .setContentText(content)
                .setFullScreenIntent(null, true)
                .setContentIntent(contentIntent)
                .setPriority(NotificationCompat.PRIORITY_DEFAULT)
                .setTicker(content)
                .setDefaults(DEFAULT_LIGHTS)
                .setVisibility(VISIBILITY_PUBLIC)
                .setDefaults(DEFAULT_VIBRATE)
                .setWhen(System.currentTimeMillis())
                .setAutoCancel(true)
                .setOnlyAlertOnce(true)
                .setOngoing(false);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            createNotificationChannel(id, uri);
        }
        getManager().notify(pushId, builder.build());
    }

    private static Uri notifyVoice(String soundCommand) {
        Logger.e(soundCommand);
        int notifyVoice;
        switch (soundCommand) {
            case "N":
                notifyVoice = R.raw.notify_new_customer;
                break;
            case "O":
                notifyVoice = R.raw.notify_new_order;
                break;
            default:
                notifyVoice = 0;
        }
        return notifyVoice == 0 ? null : Uri.parse("android.resource://" + BuildConfig.APPLICATION_ID + "/" + notifyVoice);
    }

```
## 方法二
这是使用不同渠道方式实现了这个需求,如果感觉不想让用户对于这么多渠道做相应的设置(在 8.0 以上,用户可以针对不同的通知渠道设置相应的行为),或者希望每条通知的铃声都可以播放完整,那么可以使用 Ringtone 来播放音频文件,并且可以只设置一个通知渠道.
``` java 
        Uri uri = notifyVoice(soundCommand);
        Ringtone ringtone = RingtoneManager.getRingtone(MyApp.getInstance(), uri);
        if (ringtone != null) {
            ringtone.play();
        }
```
但是这样就需要通知取消时 stop 正在播放的音频.可以通过``` builder.setDeleteIntent()```配合 BroadcastReceiver 来取消播放.

## 参考
[Android notification setSound is not working](https://stackoverflow.com/questions/48986856/android-notification-setsound-is-not-working)
[Notification框架简介](https://www.jianshu.com/p/4b9fb73762df)