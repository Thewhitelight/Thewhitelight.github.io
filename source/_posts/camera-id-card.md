---
title: 使用 Camera Api 拍摄身份证
date: 2019-02-27 10:38:12
tags:
    - Android
    - Camera
categories: 
    - 技术
---
项目中需要核验身份信息,所以模拟支付宝的身份证 OCR 界面,做一个类似的功能.但是又有不同的地方,我们需要拍下照片而不是不断的扫描获取图像.
由于项目最低支持 4.0 系统为了方便起见使用 Camera 而不是 Camera2 接口,因为 Camera2 是 5.0 以后加入的 api.
<!-- more -->
### 请求相机权限
打开相机需要`<uses-permission android:name="android.permission.CAMERA" />`权限,并且相机权限也是运行时权限,需要在打开相机前动态请求.
### 请求存储权限
因为 sample 中使用了 cache 目录,所以不需要请求存储权限,如果使用外部存储,则需要请求运行时权限.
### 调用相机接口
1. 创建预览界面，使用 SurfaceView 绘制实时预览图像,现在推荐使用 TextureView,关于两者的特性和区别不再赘述,文末有参考文章,
由于相机镜头是横向的,所以需要设置相机预览为竖屏`camera.setDisplayOrientation(90);`,在拍照页面退出时也需要及时释放相机资源.
2. **调整预览尺寸、调整图片尺寸** 这个是拍照关键
我们所要实现的功能是全屏预览,相机预览尺寸是从它所支持的规格里面选择,而不是我们自身决定,而不同手机又有不同屏幕的比例,所以我们需要在手机预览尺寸匹配到一个最相近的尺寸,设置为相机的预览尺寸.
匹配算法:
``` java
@NonNull
    private Point machBestSize(Camera.Parameters parameters, Point screenResolution, List<Camera.Size> supportedSizes) {
        //降序排列
        List<Camera.Size> supportedPreviewSizes = new ArrayList<Camera.Size>(supportedSizes);
        Collections.sort(supportedPreviewSizes, new Comparator<Camera.Size>() {
            @Override
            public int compare(Camera.Size a, Camera.Size b) {
                int aPixels = a.height * a.width;
                int bPixels = b.height * b.width;
                if (bPixels < aPixels) {
                    return -1;
                }
                if (bPixels > aPixels) {
                    return 1;
                }
                return 0;
            }
        });
        //打印镜头支持的预览尺寸
        if (Log.isLoggable(TAG, Log.INFO)) {
            StringBuilder previewSizesString = new StringBuilder();
            for (Camera.Size supportedPreviewSize : supportedPreviewSizes) {
                previewSizesString.append(supportedPreviewSize.width).append('x').append(supportedPreviewSize.height).append(' ');
            }
            Log.i(TAG, "Supported preview sizes: " + previewSizesString);
        }

        double screenAspectRatio = (double) screenResolution.x / (double) screenResolution.y;

        Iterator<Camera.Size> it = supportedPreviewSizes.iterator();
        while (it.hasNext()) {
            Camera.Size supportedPreviewSize = it.next();
            int realWidth = supportedPreviewSize.width;
            int realHeight = supportedPreviewSize.height;
            //MIN_PREVIEW_PIXELS = 480 * 320 移除过小的尺寸
            if (realWidth * realHeight < MIN_PREVIEW_PIXELS) {
                it.remove();
                continue;
            }

            boolean isCandidatePortrait = realWidth < realHeight;
            int maybeFlippedWidth = isCandidatePortrait ? realHeight : realWidth;
            int maybeFlippedHeight = isCandidatePortrait ? realWidth : realHeight;

            double aspectRatio = (double) maybeFlippedWidth / (double) maybeFlippedHeight;
            double distortion = Math.abs(aspectRatio - screenAspectRatio);
            // MAX_ASPECT_DISTORTION = 1.5 异常纵横比偏差过大的尺寸
            if (distortion > MAX_ASPECT_DISTORTION) {
                it.remove();
                continue;
            }

            if (maybeFlippedWidth == screenResolution.x && maybeFlippedHeight == screenResolution.y) {
                Point exactPoint = new Point(realWidth, realHeight);
                Log.i(TAG, "Found preview size exactly matching screen size: " + exactPoint);
                return exactPoint;
            }
        }

        //在没有匹配到最优结果情况下的兜底策略
        if (!supportedPreviewSizes.isEmpty()) {
            Camera.Size largestPreview = supportedPreviewSizes.get(0);
            Point largestSize = new Point(largestPreview.width, largestPreview.height);
            Log.i(TAG, "Using largest suitable preview size: " + largestSize);
            return largestSize;
        }

        //在没有获取预览尺寸列表下的兜底策略
        Camera.Size defaultPreview = parameters.getPictureSize();
        Point defaultSize = new Point(defaultPreview.width, defaultPreview.height);
        Log.i(TAG, "No suitable preview sizes, using default: " + defaultSize);

        return defaultSize;
    }
```
通过这个算法我们就可以匹配到相近的相机预览尺寸,但是还必须匹配拍照尺寸,因为预览尺寸和照片尺寸是两个不同的逻辑,如果只匹配到预览尺寸,有的手机会匹配到一个过小的照片尺寸,这样就无法看清图片信息.
``` java
    Point pictureSize = findBestPictureSizeValue(parameters, cameraResolution);
    //设置拍照尺寸
    parameters.setPictureSize(pictureSize.x, pictureSize.y);
    camera.setParameters(parameters);
```
这样就可以和手机自带的相机全屏模式下预览效果一样了
### 拍照、剪裁、压缩、保存
为了方便OCR,所以我们将图片进行剪裁至身份证大小,对于过大的尺寸压缩至复合我们要求,保存成文件方便上传至服务器.
``` java 
//拍照
private fun takePhoto() {
    val camera = cameraManager.camera
    //只需要获取 jpeg 格式图片
    camera.takePicture(null, null, Camera.PictureCallback { data, _ ->
        var resource = BitmapFactory.decodeByteArray(data, 0, data.size)
        if (resource == null) {
            Toast.makeText(this@CameraActivity, "拍照失败",Toast.LENGTH_SHORT).show()
        } else {
            if (canCrop) {
                //剪裁图片至身份证大小
                resource = cropImage(resource)
            }
            //保存图片至文件
            val path = saveBitmap(resource)
            if (path.isNotEmpty()) {
                Log.e("photo saved in path:", path)
            } else {
                Toast.makeText(this@CameraActivity, "照片保存失败",Toast.LENGTH_SHORT).show()
            }
            val intent = Intent()
            intent.putExtra("imagePath", path)
            setResult(Activity.RESULT_OK, intent)
            finish()
        }
    })
}

// 剪裁图片 通过计算身份证框与屏幕的距离 确定图片剪裁的范围
private fun cropImage(resource: Bitmap): Bitmap {
    val ratio = resource.width / (height * 1.0)
    val pictureWidth = dp2px(450) * ratio
    val pictureHeight = dp2px(282) * ratio
    val x = (resource.width - pictureWidth) / 2
    val y = (resource.height - pictureHeight) / 2
    val bitmap = Bitmap.createBitmap(resource, x.toInt(), y.toInt(),pictureWidth.toInt(), pictureHeight.toInt())
    if (pictureWidth > maxWidth) {
        val originRatio = (pictureWidth / (pictureHeight * 1.0)).toFloat()
        val matrix = Matrix()
        matrix.postScale(originRatio, originRatio)
        return Bitmap.createScaledBitmap(bitmap, maxWidth, (maxWidth /originRatio).toInt(), true)
    }
    return bitmap
}
//保存图片至缓存目录,不需要请求存储权限
private fun saveBitmap(resource: Bitmap): String {
    val picturePathName = cacheDir.absolutePath + File.separator +System.currentTimeMillis() + ".jpg"
    val file = File(picturePathName)
    var result = ""
    var isSuccess = false
    try {
        val fos = FileOutputStream(file)
        isSuccess = resource.compress(Bitmap.CompressFormat.JPEG, 100, fos)
        fos.flush()
        fos.close()
        result = file.absolutePath
    } catch (e: Exception) {
        e.printStackTrace()
    }
    return if (isSuccess) result else ""
}
```
### 总结
有的手机相机存在预览时画面稍微变形,在相机里用全屏预览时也发现有同样的问题,应该是预览比例和屏幕比例有差别导致的.如果想要预览效果完美匹配,则需要改变预览尺寸而不是固定为屏幕尺寸大小.
### 截图
拍摄国徽页效果图(sample 效果) 如果为人像页则有人像框 **国徽页为正面 人像页为反面**
![](https://def-201655.cos.ap-shanghai.myqcloud.com/camera_id_card_front.png)
###  更新兼容性方案
经过线上验证存在匹配照片尺寸小于身份证尺寸的问题,导致拍出的图片太小模糊问题.
所以启用备用方案,点击拍照时直接通过`setOneShotPreviewCallback`实时预览回调获取图片数组,然后将数组转化为 Bitmap 供调用方使用.
这里需要注意的是 `onPreviewFrame`里返回的数组是`YUV420SP 格式`,不能直接通过`BitmapFactory`生成图片,需要用`YuvImage`进行转化才可以使用.
``` java 生成 Bitmap
private fun generateBitmap(camera: Camera, data: ByteArray): Bitmap? {
        val size = camera.parameters.previewSize
        val w = size.width 
        val h = size.height
        val image = YuvImage(data, ImageFormat.NV21,
                w, h, null)
        val os = ByteArrayOutputStream()
        if (!image.compressToJpeg(Rect(0, 0, w, h), 100, os)) {
            return null
        }
        val tmp = os.toByteArray()
        return BitmapFactory.decodeByteArray(tmp, 0, tmp.size, BitmapFactory.Options())
    }
```
其他逻辑都不用改变,这种方法通过界面上的所见即所得直接获取图片,以减少适配问题.
### 项目地址
[GitHub camera module为此 sample](https://github.com/Thewhitelight/Tinder/tree/master/camera)
### 参考文章
[Android 5.0(Lollipop)中的SurfaceTexture，TextureView, SurfaceView和GLSurfaceView](https://blog.csdn.net/jinzhuojun/article/details/44062175)