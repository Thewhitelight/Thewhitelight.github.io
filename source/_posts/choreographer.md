---
title: Android 9.0 Choreographer 原理解析
date: 2019-03-15 09:39:26
tags:
    - Android
    - 源码解析
categories: 
    - 技术
---
## 前言
为了监测 APP 运行时是否流畅,项目使用 Choreographer 配合 ActivityLifecycleCallbacks 监测页面掉帧情况,然后在记录到数据库,方便回顾那些页面出现频繁卡顿情况,目前只是在测试环境使用,还未实现上传至服务器做更直观的分析,都需要dump 出 db 文件再分析.项目最低版本也是 4.1 以上,所以使用了 Choreographer 这种方式.
<!-- more -->
## Choreographer
### getInstance
``` java
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };
```
获取当前线程的 Looper,而 Choreographer 我们是初始化在主线程,所以这里相当于是主线程,所以这样就是我们在使用 doFrame 时需要起一个子线程去打印 Log 和保存到数据库的原因.
### 私有构造函数
``` java
    private Choreographer(Looper looper, int vsyncSource) {
        //方法内代码有省略
        //FrameHandler 处理消息
        mHandler = new FrameHandler(looper);
        //接收 Native 方法的 VSYNC 信号
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;
        //刷新间隔时间 也就是16.67ms  getRefreshRate()在 VirtualDisplayDevice 中定义为 60,也就是我们所说的默认刷新率
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
        //创建回调对象队列 共有四种回调 所以初始化 size 为 4
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
    }
```
### FrameHandler
``` java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //渲染帧
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                //请求 VSYNC
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                //请求回调
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```
### FrameDisplayEventReceiver
``` java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        //looper 此时也是主线程
        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        //系统 native 方法会调用此方法
        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            //判断是否是主显示屏幕
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                Log.d(TAG, "Received vsync from secondary display, but we don't support "
                        + "this case yet.  Choreographer needs a way to explicitly request "
                        + "vsync for a specific display to ensure it doesn't lose track "
                        + "of its scheduled vsync.");
                scheduleVsync();
                return;
            }
            //修正时间确保时间顺序正确
            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            //向主线程发送异步消息处理下一帧 即调用 run()
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```
onVsync 向主线程发送**异步消息**时由于 FrameDisplayEventReceiver 实现了 Runnable 所以会直接调用其 run 方法
### doFrame
``` java
    //方法内有省略
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; //如果为渲染完成,则直接返回,这样就渲染不了当前帧
            }
            //原来帧的绘制时间点
            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                //SKIPPED_FRAME_WARNING_LIMIT=30 如果页面复杂时,我们经常会看到 Log 里输出下面的信息,也就是说官方认为掉帧数=30时就认为太卡,建议优化了
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                frameTimeNanos = startNanos - lastFrameOffset;
            }

            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG_JANK) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                //请求 VSYNC
                scheduleVsyncLocked();
                return;
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
            //优先处理输入事件
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        if (DEBUG_FRAMES) {
            final long endNanos = System.nanoTime();
            Log.d(TAG, "Frame " + frame + ": Finished, took "
                    + (endNanos - startNanos) * 0.000001f + " ms, latency "
                    + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
        }
    }
```
### scheduleVsync
``` java scheduleVsyncLocked.java
    private void scheduleVsyncLocked() {
        //调用了 FrameDisplayEventReceiver 父类 DisplayEventReceiver
        mDisplayEventReceiver.scheduleVsync();
    }
```
``` java DisplayEventReceiver.java
      public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            //调用了 Native 方法
            nativeScheduleVsync(mReceiverPtr);
        }
    }
```
由于请求 VSYNC 最终执行了 Native 方法,由于自己不熟悉所以先不深究做了什么,但是可以肯定,最终会执行 onVsync 方法.
### scheduleFrameLocked
在 FrameHandler 收到 MSG_DO_SCHEDULE_CALLBACK 时调用 doScheduleCallback 方法,最终调用了 scheduleFrameLocked;
``` java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }
                if (isRunningOnLooperThreadLocked()) {
                    //如果在当前Looper线程 则立即请求VSYNC
                    scheduleVsyncLocked();
                } else {
                    //否则向主线程发送消息
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
               //只分析用户使用 VSYNC 的情况
            }
        }
    }
```
### doScheduleCallback
在 FrameHandler 收到 MSG_DO_SCHEDULE_CALLBACK 时调用 doScheduleCallback(callbackType) 方法,根据回调队列获取当前时间的回调类型,最终调用了 scheduleFrameLocked 方法;
### doCallbacks
doFrame 方法里在最后调用了四种类型的 callback
``` java
    //输入类型
    public static final int CALLBACK_INPUT = 0;
    //动画类型
    @TestApi
    public static final int CALLBACK_ANIMATION = 1;
    //遍历类型 其他异步消息处理完之后,处理执行 layout 和 draw
    public static final int CALLBACK_TRAVERSAL = 2;
    //提交类型 在 traversal完成后 处理 post-draw 操作
    public static final int CALLBACK_COMMIT = 3;

    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            final long now = System.nanoTime();
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                                + " ms which is more than twice the frame interval of "
                                + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                                + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                                + " ms in the past.");
                        mDebugPrintNextFrameTimeDelta = true;
                    }
                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                //如果token=FRAME_CALLBACK_TOKEN,则调用下一个回调的 doFrame 否则继续调用 run
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
### CallbackRecord
``` java
    private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        //Runnable 类型 或者 FrameCallback类型
        public Object action;
        public Object token;

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                //执行输入、动画或者遍历的 Runnalbe 2.8节有阐述
                ((Runnable)action).run();
            }
        }
    }
```
### CallbackQueue
``` java CallbackQueue.addCallbackLocked
 public void addCallbackLocked(long dueTime, Object action, Object token) {
            CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
            CallbackRecord entry = mHead;
            if (entry == null) {
                mHead = callback;
                return;
            }
            //按照时间降序入队
            if (dueTime < entry.dueTime) {
                callback.next = entry;
                mHead = callback;
                return;
            }
            while (entry.next != null) {
                if (dueTime < entry.next.dueTime) {
                    callback.next = entry.next;
                    break;
                }
                entry = entry.next;
            }
            entry.next = callback;
        }
```
CallbackQueue 是一个由 CallbackRecord 构成的单向链表,把 CallbackRecord 按照事件发生顺序插入到队列中,在 doFrame 方法中可以看出最先放入 input 回调,这样也保证输入事件最先执行,这样也正好对应到`ViewRootImpl`中`ConsumeBatchedInputRunnable InvalidateOnAnimationRunnable TraversalRunnable`他们三个分别对应了三种回调类型的 Runnalbe.
``` java
    @TestApi
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
```
ViewRootImpl 中初始化 Choreographer 后则通过调用上面这个方法,执行不同的事件类型.
## 总结
通过查看源码知道整个屏幕刷新机制,平时看到 Log 里打印`The application may be doing too much work on its main thread`也知道了出处.也知道了经常使用的 requestLayout 其实最终其实执行的是 ViewRootImpl 的 TraversalRunnable,对于卡顿的形成也稍微有点了解.
## 参考
[Android8.1 Choreographer机制与源码分析](https://www.jianshu.com/p/c2d93861095a)
[Choreographer原理](http://gityuan.com/2017/02/25/choreographer/)