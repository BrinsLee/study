## 使用

```groovy
dependencies {
    // most often used way, enable notification to notify block event
    implementation 'com.github.markzhai:blockcanary-android:1.5.0'

    // this way you only enable BlockCanary in debug package
    // debugImplementation 'com.github.markzhai:blockcanary-android:1.5.0'
    // releaseImplementation 'com.github.markzhai:blockcanary-no-op:1.5.0'
}
```

```java
// Application中注册
public class DemoApplication extends Application {
    @Override
    public void onCreate() {
        // ...
        // Do it on main process
        BlockCanary.install(this, new BlockCanaryContext()).start();
    }
}
```

检测结果

![img](https://pic3.zhimg.com/80/v2-22ad81c923f6ef7b5449823088c9a9ee_720w.webp)

## 原理

Android，应用的卡顿，主要是在主线程阻塞导致的。Looper是主线程消息的调度者，以他为突破口

### Looper#loop()

在Looper#loop方法中有个Printer，在每个Message处理的前后被调用，而如果主线程卡住了，就是dispatchMessage里面卡住了

```java
public static void loop() {
    // ....

    for (;;) {
        // ...

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
        }

        // ...
        msg.target.dispatchMessage(msg);
        // ...

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
       // ...
    }
}
```

```java
//把mBlockCanaryCore中的monitor设置MainLooper中进行监听
Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
```

### 创建自定义Printer

```java
@Override
    public void println(String x) {
        //如果再debug模式，不执行监听
        if (mStopWhenDebugging && Debug.isDebuggerConnected()) {
            return;
        }
        if (!mPrintingStarted) {//dispatchMesage前执行的println
            //记录开始时间
            mStartTimestamp = System.currentTimeMillis();
            mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
            mPrintingStarted = true;
            //开始采集栈及cpu信息
            startDump();
        } else {//dispatchMesage后执行的println
            //获取结束时间
            final long endTime = System.currentTimeMillis();
            mPrintingStarted = false;
            //判断耗时是否超过阈值
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
            stopDump();
        }
    }
 //判断是否超过阈值
 private boolean isBlock(long endTime) {
        return endTime - mStartTimestamp > mBlockThresholdMillis;
    }
//回调监听
 private void notifyBlockEvent(final long endTime) {
        final long startTime = mStartTimestamp;
        final long startThreadTime = mStartThreadTimestamp;
        final long endThreadTime = SystemClock.currentThreadTimeMillis();
        HandlerThreadFactory.getWriteLogThreadHandler().post(new Runnable() {
            @Override
            public void run() {
                mBlockListener.onBlockEvent(startTime, endTime, startThreadTime, endThreadTime);
            }
        });
    }
```

当发现时间差超过阈值后，会回调onBlockEvent。具体的实现在BlockCanaryInternals的构造方法中，如下：

```java
setMonitor(new LooperMonitor(new LooperMonitor.BlockListener() {

            @Override
            public void onBlockEvent(long realTimeStart, long realTimeEnd,
                                     long threadTimeStart, long threadTimeEnd) {
                //根据开始及结束时间，从栈的map当中获取记录信息
                ArrayList<String> threadStackEntries = stackSampler
                        .getThreadStackEntries(realTimeStart, realTimeEnd);
                if (!threadStackEntries.isEmpty()) {
                    //构建 BlockInfo对象，设置相关的信息
                    BlockInfo blockInfo = BlockInfo.newInstance()
                            .setMainThreadTimeCost(realTimeStart, realTimeEnd, threadTimeStart, threadTimeEnd)
                            .setCpuBusyFlag(cpuSampler.isCpuBusy(realTimeStart, realTimeEnd))
                            .setRecentCpuRate(cpuSampler.getCpuRateInfo())
                            .setThreadStackEntries(threadStackEntries)
                            .flushString();
                    //记录信息
                    LogWriter.save(blockInfo.toString());
                    //遍历拦截器，通知
                    if (mInterceptorChain.size() != 0) {
                        for (BlockInterceptor interceptor : mInterceptorChain) {
                            interceptor.onBlock(getContext().provideContext(), blockInfo);
                        }
                    }
                }
            }
        }, getContext().provideBlockThreshold(), getContext().stopWhenDebugging()));

```

