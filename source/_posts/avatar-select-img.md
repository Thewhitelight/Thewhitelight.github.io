---
title: 图片选择 Library
date: 2018-08-06 23:30:56
tags:
    - Android
categories: 技术
---

选择头像、图片时经常使用拍照和相册功能,但是由于 Android 碎片化问题,不同版本 API 有差异,并且在不同页面处理 onActivityResult 感觉太麻烦,所以就想着可以通过链式调用,将这些逻辑封装起来,让整个代码写起来简洁清晰明了,不至于好多类似代码重复,提高开发效率.

#### 选择图片存在的问题

##### Android 6.0 运行时权限

在 Android 6.0 中严格限制App的权限授予,读取 SD 卡、拍照、录音、定位、短信、通讯录等关键权限需要在代码中申请这些权限,否则会抛出异常导致程序奔溃,当 App 申请权限被拒绝后我们应该给用户弹出提示框,引导用户去设置页开启此权限,然后在 onRequestPermissionsResult 中处理回调结果.

##### Android 7.0 StrictMode 问题

在 Android 7.0 中强制启用了被称作 StrictMode 的策略，带来的影响就是你的App对外无法暴露 file:// 类型的 URI,否则会出现 FileUriExposedException 异常,进程间应使用 FileProvider 共享文件.

##### 三星机型图片选择问题

在三星机型中,当我们调用相机拍照后,获取的图片发现被旋转90度,所以需要获取图片的 ExifInterface 信息,然后根据 orientation 进行图片的角度旋转,使之方向正确.

#### 解决方式

针对以上问题写一个 library 就显得很有必要.方便在不用页面调用选择图片.在一个透明背景的 Activity 中处理以上问题,让调用方无感知以上问题,只用传入必要的设置条件最后拿到图片文件就可以,然后针对文件进行自己的业务逻辑操作即可.

<!-- more -->

```  java 
 //调用方式 
 Avatar.getInstance()
                    //选择模式 ALL 拍照和相册 CAMERA 拍照 GALLERY 相册
                    .setSelectMode(Avatar.ALL)
                    //文件目录名字 非必填 默认项目包名
                    .setImageFileDir("Tinder")
                    //是否剪裁
                    .setHasCrop(true)
                    //获取图片文件回调
                    .imageFile { imageFile ->
                        run {
                            Log.e("file:", imageFile.absolutePath)
                            img.setImageBitmap(BitmapFactory.decodeFile(imageFile.absolutePath))
                        }
                    }
                    .build(this)
```
在 Avatar 中使用使用静态内部类单例模式,减少对象创建,并方便 Activity 中传回图片文件目录,SelectAvatarActivity 使用透明主题,并在处理权限请求与照片回调处理.

``` java 
Avatar.java

    private Avatar() {
    }

    public static Avatar getInstance() {
        return AvatarHolder.INSTANCE;
    }

    public static class AvatarHolder {
        static final Avatar INSTANCE = new Avatar();
    }

SelectAvatarActivity.java

    //setImageFile 方法包级私有,故包外不可调用,保证封装
    Avatar.getInstance().setImageFile(mAvatarFile);

    //选择照片
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
            mOrigUri = Uri.fromFile(cameraFile);
        } else {
            ContentValues contentValues = new ContentValues(1);
            contentValues.put(MediaStore.Images.Media.DATA, cameraFile.getAbsolutePath());
            mOrigUri = getContentResolver().insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues);
        }
        intent.putExtra(MediaStore.EXTRA_OUTPUT, mOrigUri);
        startActivityForResult(intent, REQUEST_CODE_GET_IMAGE_CAMERA);

    //选择相册
    Intent intent;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            intent = new Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI);
        } else {
            intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("image/*");
        }
        startActivityForResult(intent, REQUEST_CODE_GET_IMAGE_SDCARD);

        //通过图片路径,获取图片ExifInterface信息得到旋转角度,使用Matrix旋转生成新的图片保存
        public static int readPictureDegree(String path) {
        int degree = 0;
        try {
            ExifInterface exifInterface = new ExifInterface(path);
            int orientation = exifInterface.getAttributeInt(
                    ExifInterface.TAG_ORIENTATION,
                    ExifInterface.ORIENTATION_NORMAL);
            switch (orientation) {
                case ExifInterface.ORIENTATION_ROTATE_90:
                    degree = 90;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_180:
                    degree = 180;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_270:
                    degree = 270;
                    break;
                default:
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return degree;
    }

ImageUtil.java 
    //旋转图片
    public static Bitmap rotateImageView(int angle, Bitmap bitmap) {
        Matrix matrix = new Matrix();
        matrix.postRotate(angle);
        if (bitmap != null) {
            return Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(), bitmap.getHeight(), matrix, true);
        } else {
            return null;
        }
    }

```

``` xml
    <!-- 透明主题 -->
    <style name="TransparentTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="colorPrimary">@android:color/transparent</item>
        <item name="colorPrimaryDark">@android:color/transparent</item>
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowAnimationStyle">@android:style/Animation.Translucent</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowContentOverlay">@null</item>
    </style>

    <!-- FileProvider 适配7.0-->
    <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="${applicationId}.fileProvider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/provider_paths" />
    </provider>

    <!-- provider_paths.xml -->
    <paths>
    <!-- 为了方便直接使用 root-path 对所有文件授权,当然安全起见考虑还是不要这么写,使用external-path范围越小越好-->
    <root-path
        name="root_path"
        path="."/>
    </paths>
```
在```<paths>``` 中可以添加``` <files-path>``` 分享app内部的存储; ```<external-path>``` 分享外部的存储; ```<cache-path> ```分享内部缓存目录.根据不同的需求对文件夹进行授权访问.

当封装好这些功能后,在调用页面获取图片的代码最多短短是几行,而不像原来需要上百行的处理各种问题,并且 API 变动时,只需要改动 library 就可以完成适配.
**[项目地址](https://github.com/Thewhitelight/Tinder)**