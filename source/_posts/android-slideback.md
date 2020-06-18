---
title: 仿小米返回交互
date: 2018-09-28 23:37:35
tags:
    - Android
categories: 
    - 技术
---
进入全面屏时代,iPhone 的交互是业界标杆,所以Android阵营都向其靠拢.小米全面屏手势是我挺喜欢的交互.所以模仿下小米的返回操作.其返回逻辑远不如微信返回动画复杂,几乎没有机型适配问题的存在.
<!-- more -->
动画效果如图
![](https://def-201655.cos.ap-shanghai.myqcloud.com/7g4t2-ihykn.gif)

看到动画效果我们需要达到的效果是在 Activity 的 DecorView 上绘制此 View,所以就像当于我们所有页面的的根布局需要添加此 View,如果每个页面都手动添加,则显得太过麻烦,显然在抽象的 BaseActivity 中 setContentView 方法之后调用`activity.getWindow().getDecorView()`,然后获取 content 根布局`FrameLayout content = viewGroup.findViewById(android.R.id.content); content.addView(slideBackView);`将我们带有返回 View 的布局添加到 FameLayout 的顶层,这样就达到的最小化的修改代码实现效果.

# 对于返回图形的绘制,主要运用了触摸事件、贝塞尔曲线等知识点
* 触摸事件,在 View 内重写 onTouchEvent 方法,判断左右滑动并且根据滑动距离绘制返回图形宽度
* 对于返回图形可以余弦公式,也使用贝塞尔曲线画成,但贝塞尔曲线看起来更加顺滑自然,但是 Android 原生最高支持三阶,对于我们所要绘制的图形三阶会不够自然,使用六阶会比较好,但是又不支持,所以只能由三个二阶组合成返回图形
![三阶贝塞尔曲线](https://def-201655.cos.ap-shanghai.myqcloud.com/bezier_3.png)
![六阶贝塞尔曲线](https://def-201655.cos.ap-shanghai.myqcloud.com/bezier_6.png)

## View触摸事件
``` java
//左边视差阈值,为 View 的 1/3
private float thresholdLeft;
//右边边视差阈值,为 View 的 2/3
private float thresholdRight;
//返回图形的最大宽度
private float backMaxWidth;
//绘制返回图形的最小边距
private float backEdgeWidth;
//视差的变量
private float bufferX;

    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        //当前动作下 View 的X坐标点,不能用 getRawX,否则获取的是屏幕上的坐标
        float currentX = ev.getX();
        currentY = ev.getY();
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                //按下时 View 的X坐标点
                downX = ev.getX();
                forwardX = downX;
                if (downX <= backEdgeWidth) {
                //当按下的X坐标<最小触发边距时,认为有效触发绘制逻辑并绘制左边返回图形
                    isEdge = true;
                    left = true;
                    right = false;
                } else if (downX >= getWidth() - backEdgeWidth) {
                //当按下的X坐标>(View 宽度-最小触发边距)时,认为有效触发绘制逻辑并绘制右边返回图形
                    isEdge = true;
                    right = true;
                    left = false;
                }
                break;
            case MotionEvent.ACTION_MOVE:
                //当前滑动到的点与按下时的点X坐标的距离
                deltaX = currentX - downX;
                //滑动距离 判断连续滑动时的方向
                float diff = forwardX - currentX;
                if (diff > 0) {
                    //向左滑动
                    if (currentX < thresholdLeft && left) {
                        //当绘制左边图形、向左滑动时、当前小于屏幕1/3处时,绘制时差效果
                        deltaX = backMaxWidth;
                        deltaX -= (thresholdLeft - currentX) / 2;
                        bufferX = deltaX;
                        if (deltaX < 0) {
                            deltaX = 0;
                            bufferX = 0;
                        }
                    } else if ((currentX > thresholdRight) && right) {
                        //当绘制右边图形、向左滑动时、当前大于屏幕2/3处时,绘制时差效果
                         bufferX -= diff;
                         deltaX = bufferX;
                    }
                } else {
                    //向右滑动
                    if ((currentX < thresholdLeft) && left) {
                        //当绘制左边图形、向右滑动时、当前小于屏幕1/3处时,绘制时差效果
                        bufferX += Math.abs(diff);
                        deltaX = bufferX;
                    } else if (currentX > thresholdRight && right) {
                         //当绘制右边边图形、向右滑动时、当前大于屏幕2/3处时,绘制时差效果
                        deltaX = -backMaxWidth;
                        deltaX += (currentX - thresholdRight) / 2;
                        bufferX = deltaX;
                        if (deltaX < -backMaxWidth) {
                            deltaX = -backMaxWidth;
                            bufferX = -backMaxWidth;
                        }
                    }
                }
                //记录当前滑动的X坐标,与下次滑动点做比较
                forwardX = currentX;
                if (isEdge) {
                    invalidate();
                }
                break;
            case MotionEvent.ACTION_UP:
                if (isEdge) {
                    if (deltaX >= backMaxWidth && left) {
                        //当绘图形宽度为最大时并抬起,表示返回
                        back();
                    } else if (Math.abs(deltaX) >= backMaxWidth && right) {
                        if (!isOnlyLeftBack) {
                            //如果绘制右边时,出发返回逻辑
                            back();
                        }
                    }
                    deltaX = 0;
                    invalidate();
                }
                left = false;
                right = false;
                isEdge = false;
                bufferX = 0;
                break;
            default:
                break;
        }
        return isEdge;
    }
```
通过以上代码,分析了手势与绘制图形的关系,具体逻辑可以根据效果分析,视差部分的效果可以根据自身进行调整.
## 贝塞尔曲线绘制
使用贝塞尔曲线绘制返回图形和里面的返回图标.
``` java
    //返回图形画笔
    private Paint backPaint;
    //返回图标画笔
    private Paint arrowPaint;
    //返回图形路径
    private Path backPath;
    //返回图标路径
    private Path arrowPath;

    private void init() {
        backPath = new Path();
        arrowPath = new Path();

        backPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        backPaint.setColor(0xAA000000);
        backPaint.setStyle(Paint.Style.FILL_AND_STROKE);

        arrowPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        arrowPaint.setColor(Color.WHITE);
        arrowPaint.setStyle(Paint.Style.STROKE);
        arrowPaint.setStrokeWidth(6);
    }

@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //当deltaX超过最大距离时,设置为最大距离
        if (deltaX > backMaxWidth && left) {
            deltaX = backMaxWidth;
        } else if (deltaX < -backMaxWidth && right) {
            deltaX = -backMaxWidth;
        }
        //初始化绘制坐标点Y
        float deltaY = currentY - backViewHeight / 2;
        backPath.reset();
        arrowPath.reset();

        if (deltaX > 0 && left) {
            backPath.moveTo(0, deltaY);
            backPath.quadTo(0, backViewHeight / 4 + deltaY, deltaX / 3, backViewHeight * 3 / 8 + deltaY);
            backPath.quadTo(deltaX * 5 / 8, backViewHeight / 2 + deltaY, deltaX / 3, backViewHeight * 5 / 8 + deltaY);
            backPath.quadTo(0, backViewHeight * 6 / 8 + deltaY, 0, backViewHeight + deltaY);
            canvas.drawPath(backPath, backPaint);

            arrowPath.moveTo(deltaX / 6 + (15 * (deltaX / (width / 6))), backViewHeight * 15 / 32 + deltaY);
            arrowPath.lineTo(deltaX / 6, backViewHeight * 16.1f / 32 + deltaY);
            arrowPath.moveTo(deltaX / 6, backViewHeight * 15.9f / 32 + deltaY);
            arrowPath.lineTo(deltaX / 6 + (15 * (deltaX / (width / 6))), backViewHeight * 17 / 32 + deltaY);
            canvas.drawPath(arrowPath, arrowPaint);
        } else if (deltaX < 0 && right) {
            if (!isOnlyLeftBack) {
                deltaX = Math.abs(deltaX);
                backPath.moveTo(width, deltaY);
                backPath.quadTo(width, backViewHeight / 4 + deltaY, width - deltaX / 3, backViewHeight * 3 / 8 + deltaY);
                backPath.quadTo(width - deltaX * 5 / 8, backViewHeight / 2 + deltaY, width - deltaX / 3, backViewHeight * 5 / 8 + deltaY);
                backPath.quadTo(width, backViewHeight * 6 / 8 + deltaY, width, backViewHeight + deltaY);
                canvas.drawPath(backPath, backPaint);

                arrowPath.moveTo(width - deltaX / 6 - (15 * (deltaX / (width / 6))), backViewHeight * 15 / 32 + deltaY);
                arrowPath.lineTo(width - deltaX / 6, backViewHeight * 16.1f / 32 + deltaY);
                arrowPath.moveTo(width - deltaX / 6, backViewHeight * 15.9f / 32 + deltaY);
                arrowPath.lineTo(width - deltaX / 6 - (15 * (deltaX / (width / 6))), backViewHeight * 17 / 32 + deltaY);
                canvas.drawPath(arrowPath, arrowPaint);
            }
        }
        setAlpha(deltaX / backMaxWidth);
    }
```
返回图形及返回坐标的形状都可以通过这是不同的结束点和控制点去调节,形成自己想要的效果,不过最好先通过绘图软件,分析好想要的图形及其绘制比例.

# 总结
SlideBack用法
``` java
//setContent(layoutResID) 替换为
SlideBack.Builder().init(layoutResID).build(this)
```
使用自定义 View 可以作出各种设计师想实现的动画效果,一般使用 onTouchEvent、onMeasure、onLayout、onDraw 等方法,根据实际情况重写这些方法即可实现.通过实现这个效果可以更好的理解 Android 中各种设计,提升自己开发水平.
# 项目地址
[GitHub项目地址](https://github.com/Thewhitelight/Tinder) 里面有一些 library 集合
`cn.libery:slideback:latest` 
[ ![Maven](https://api.bintray.com/packages/light/Avatar/SlideBack/images/download.svg) ](https://bintray.com/light/Avatar/SlideBack/_latestVersion) 