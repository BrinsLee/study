# View工作原理

问题：View什么时候添加到屏幕上的？

### handleLanunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);
    
    //分析1 : 这里是创建Activity,并调用了Activity的onCreate()和onStart()
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        //分析2 : 这里调用Activity的onResume()
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);
    }
    ....
}
```

handleLaunchActivity()主要是为了调用performLaunchActivity()和handleResumeActivity()

### performLaunchActivity

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ......
    //分析1 : 这里底层是通过反射来创建的Activity实例
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);

    //底层也是通过反射构建Application,如果已经构建则不会重复构建,毕竟一个进程只能有一个Application
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);

    if (activity != null) {
        Context appContext = createBaseContextForActivity(r, activity);
        CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
        Configuration config = new Configuration(mCompatConfiguration);
        if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                + r.activityInfo.name + " with config " + config);
        //分析2 : 在这里实例化了PhoneWindow,并将该Activity设置为PhoneWindow的Callback回调,还初始化了WindowManager
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config);

        //分析3 : 间接调用了Activity的performCreate方法,间接调用了Activity的onCreate方法.
        mInstrumentation.callActivityOnCreate(activity, r.state);
        
        //分析4: 这里和上面onCreate过程差不多,调用Activity的onStart方法
        if (!r.activity.mFinished) {
            activity.performStart();
            r.stopped = false;
        }
        ....
    }
}
```

主要过程

1. 通过反射创建Activity
2. 实例化PhoneWindow，并将Activity设置为PhoneWindow的Callback回调
3. 调用Activity#onCreate
4. 调用Activity#onStart

setContentView仅仅是创建DecorView，并将xml布局添加到DecorView上

```java
 handleResumeActivity
   ---> performResumeActivity
  	    ---> r.activity.performResume
  		  	 ---> mInstrumentation.callActivityOnResume
   ---> wm.addView(decor, l) (WindowManagerImpl.java)
   	    ---> WindowManagerGlobal.addView(decor, l)
   	    	 ---> root = new ViewRootImpl(view.getContext(), display)
   	    	 ---> mViews.add(view) // view: DecorView
   	    	 	  mRoots.add(root) // root: ViewRootImpl
   	    	 	  mParams.add(wparams) // wparams: WindowManager.LayoutParams
   	    	 ---> root.setView(view, wparams, panelParentView, userId)
   	    	 
WindowManagerImpl: 确定View属于哪个屏幕、哪个父窗口
WindowManagerGlobal: 管理整个进程所有窗口信息
ViewRootImpl: WindowManagerGlobal 实际操作者，操作自己的窗口

ViewRootImpl.setView
	---> requestLayout()
		---> scheduleTraversals()
			---> mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null)
				---> doTraversal()
					---> performTraversals()// 绘制View
	---> res = mWindowSession.addToDisplayAsUser// 将窗口绘制到WMS上
	---> view.assignParent(this) //控件树的根部 ViewRootImpl

ViewRootImpl 构造方法
mThread = Thread.currentThread()// 拿到当前线程
mDirty = new Rect()// 脏区域
mAttachInfo = new View.attachInfo()// 保存一些窗口信息

    
performTraversals@ViewRootImpl.java
    ---> windowSizeMayChange |= measureHierarchy()//预测量
    ---> relayoutResult = relayoutWindow(params, viewVisibility, insetsPending)// 布局窗口
    ---> performMeasure(childWidthMeasureSpec, childHeightMeasureSpec)// 控件树测量
    ---> performLayout(lp, mWidth, mHeight)
    ---> performDraw()
    
measureHierarchy
1、设置一个值，进行第一次测量
2、获取一个状态值
3、改变大小 baseSize = (baseSize + desiredWindowWidth) / 2
4、进行第二次测量
5、如果还不满意，直接给自己的最大值，然后第三次测量 -- 不确定大小

如果windowSizeMayChange = true, 表示仍需要测量
performMeasure
    ---> mView.measure
    	---> onMeasure ---> 一定调用setMeasuredDimension()
    
performLayout
    ---> 
```

**handleResumeActivity**

```java
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    .....
    //分析1 : 在其内部调用Activity的onResume方法
    r = performResumeActivity(token, clearHide, reason);

    .....
    r.window = r.activity.getWindow();
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    //获取WindowManager
    ViewManager wm = a.getWindowManager();
    WindowManager.LayoutParams l = r.window.getAttributes();
    a.mDecor = decor;

    if (a.mVisibleFromClient) {
        .....
        //分析2 : WindowManager添加DecorView
        wm.addView(decor, l);
        ...
    }
    .....

}
```

分析2

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

mGlobal是WindowManagerGlobal,WindowManagerGlobal是全局单例.WindowManagerImpl的方法都是由WindowManagerGlobal完成的

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ....

    ViewRootImpl root;

    synchronized (mLock) {
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        .....

        root.setView(view, wparams, panelParentView);
    }
}
```

在这里我们看到了,实例化ViewRootImpl,然后建立ViewRootImpl与View的联系.跟着进入setView方法

**屏幕绘制**

ViewRootImpl requestLayout流程

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView, int userId) {
	synchronized (this) {
        ...
        requestLayout();
        ...
        res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);
    }
}
```

requestLayout 这个方法的主要目的就是请求布局操作，其中包括 View 的测量、布局、绘制等

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread(); // 检查是否合法线程，一般情况下检查是否为主线程
            mLayoutRequested = true; // 将请求布局标识符设置为 true，这个参数决定了后续是否需要执行 measure 和 layout 操作。
            scheduleTraversals();
        }
    }
```

scheduleTraversals

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 向主线程消息队列中插入SyncBarrier Message，该方法发送一个没有target的message到Queue中，在next方法中获取消息时，如果发现没有target的message，则在一定时间内跳过同步消息，优先执行异步消息，保证UI绘制操作优先执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

```java
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            // 移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

```

可以看出，在 run 方法中调用了 doTraversal 方法，并最终调用了 performTraversals() 方法，这个方法就是真正的开始 View 绘制流程：measure –> layout –> draw 。

```java
private void performTraversals() {
    ...
    // 调用performMeasure进行预测量
    measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);
    // 执行onLayout操作
    performLayout(lp, mWidth, mHeight);
    // 执行onDraw操作
    performDraw();
    ...
}
```

ViewRootImpl 的 measureHierarchy

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp, final Resources res, final int desiredWindowWidth, final int desiredWindowHeight{
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        boolean windowSizeMayChange = false;
    	...
         if (!goodMeasure) {
            // 通过 getRootMeasureSpec 方法获取了根 View的MeasureSpec，实际上 MeasureSpec 中的宽高此处获取的值是 Window 的宽高
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
    
    return windowSizeMayChange;
}
```

ViewRootImpl 的 performMeasure

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

只是执行了 mView 的 measure 方法，这个 mView 就是 DecorVIew。其 DecorView 的 measure 方法中，会调用 onMeasure 方法，而 DecorView 是继承自 FrameLayout 的，因此最终会执行 FrameLayout 中的 onMeasure 方法，并递归调用子 View 的 onMeasure 方法。

performTraversals()会依次调用performMeasure、performLayout、performDraw方法,而这三个方法是View的绘制流程的核心所在.

- performMeasure : 在performMeasure里面会调用measure方法,然后measure会调用onMeasure方法,而在onMeasure方法中则会对所有的子元素进行measure过程.这相当于完成了一次从父元素到子元素的measure传递过程,如果子元素是一个ViewGroup,那么继续向下传递,直到所有的View都已测量完成.测量完成之后,我们可以根据getMeasureHeight和getMeasureWidth方法获取该View的高度和宽度.
- performLayout : performLayout的原理其实是和performMeasure差不多,在performLayout里面调用了layout方法,然后在layout方法会调用onLayout方法,onLayout又会对所有子元素进行layout过程.由父元素向子元素传递,最终完成所有View的layout过程.确定View的4个点: left+top+right+bottom,layout完成之后可以通过getWidth和getHeight获取View的最终宽高.
- performDraw : 也是和performMeasure差不多,从父元素从子元素传递.在performDraw里面会调用draw方法,draw方法再调用drawSoftware方法,drawSoftware方法里面回调用View的draw方法,然后再通过dispatchDraw方法分发,遍历所有子元素的draw方法,draw事件就这样一层层地传递下去.

### 1. MeasureSpec

MeasureSpec意为测量规格，是一个32位int值，高2位代表测量模式SpecMode，低30位代表该测量模式下的规格大小SpecSize。通过MeasureSpec.makeMeasureSpec()可以得到一个measureSpec；代码如下：

```java
 public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,@MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
```

##### 1.1 SpecMode

SpecMode有三类，分别是

- **UNSPECIFIED : 父容器不对view有任何限制，想要多大就给多大，一般只有系统内部使用**
- **EXACTLY :  父容器已经检测出view的大小，即SpecSize的值，一般对应于LayoutParams中的match_parent和具体数值 如 100dp**
- **AT_MOST ：父容器给定一个具体的值SpecSize，view的具体大小由自身实现，但是不能超过SpecSize，对应于;LayoutParams中的wrap_content**

### 2. LayoutParams

LayoutParams主要保存了一个View的布局参数，包括width，height，margin等

### 3. MeasureSpec和LayoutParams关系

顶级DecorView：由于没有父容器，因此MeasureSpec由屏幕尺寸和自身LayoutParams决定

普通view：其MeasureSpec由父容器的MeasureSpec和自身LayoutParams决定

##### 3.1 DecorView：

```java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

其中desiredWindowWidth、desiredWindowHeight为屏幕尺寸，我们来看一下getRootMeasureSpec()

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            //MATCH_PARENT，精确模式时，大小为窗口大小
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            //WRAP_CONTENT，最大模式时，大小不定，但是不能超过最大值，也就是窗口的大小
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            //固定大小 如100dp，为LayoutParams指定的大小
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

#####  3.2 普通view

普通view的MeasureSpec由父容器根据自身MeasureSpec和子view的LayoutParams计算而来，measureChildWithMargin()

```java
 protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

首先会获取child的LayoutParams然后调用getChildMeasureSpec()方法分别计算出child的宽高规格，计算childWidthMeasureSpec时传入的参数为parentWidthMeasureSpec，左右padding和margin，以及child自身的width。

我们来看看getChildMeasureSpec做了什么

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;
    
    switch (specMode) {
    // Parent has imposed an exact size on us
    //父容器有一个确定的大小
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            //子控件也是确定的大小,那么最终的大小就是子控件设置的大小,SpecMode为EXACTLY
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            // 子控件想要占满剩余的空间,那么就给它吧.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            //子控件想要自己定义大小,但是不能超过剩余空间 size
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

先取出父容器的测量规格和减去间距后剩余大小

```java
switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
```

接着，根据父容器的SpecMode和自身的宽高值开始计算测量规格，

1. 当父容器为EXACTLY时：
   - childDimension > 0，即确定值如100dp，那么子view的测量模式为EXACTLY，大小就是childDimension给定的大小。
   - childDimension == MATCH_PARENT，那么子View测量模式为EXACTLY，大小为父容器剩余大小specSize - padding。
   - childDimension == WRAP_CONTENT，那么子View测量模式为AT_MOST，大小不超过父容器剩余空间
2. 当父容器为AT_MOST时：
   - childDimension > 0，即确定值如100dp，那么子view的测量模式为EXACTLY，大小就是childDimension给定的大小。
   - childDimension == MATCH_PARENT，那么子View测量模式为AT_MOST，大小为父容器剩余大小specSize - padding。
   - childDimension == WRAP_CONTENT，那么子View测量模式为AT_MOST，大小不超过父容器剩余空间

如图：

![962203ddb6f6c03594a57dea94fb5fd](C:\Users\ADMINI~1\AppData\Local\Temp\WeChat Files\962203ddb6f6c03594a57dea94fb5fd.png)



### 4.Measure过程

单一view只需完成自身测量，而ViewGroup除了完成自身测量，还要去遍历调用子元素的measure

##### 4.1 view的measure过程

测量流程由measure方法开始，然后在其中调用onMeasure()

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(),widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

我们先看看setMeasuredDimension，只是用于设置view的宽高测量值

```java
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
```

接着看getDefaultSize()

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

当view测量模式为EXACTLY或者AT_MOST时，返回的就是measureSpec中的大小。上面代码中的size由getSuggestedMinimumWidth()计算而来

```java
protected int getSuggestedMinimumWidth() {
   return (mBackground == null) ? mMinWidth:max(mMinWidth,mBackground.getMinimumWidth());
}
```

当view设置了背景时，返回值为mMinWidth与mBackground大小两者的最大值，mMinWidth通过android:minWidth设置，默认为0。



##### 4.2 viewGroup的measure过程

viewGroup除了完成自身测量过程，还会遍历调用子view的measure。ViewGroup中并没有重写onMeasure，而是提供了一个measureChildren方法

```java
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

它会去遍历子view，计算出子view的measureSpec，然后调用其measure方法，我们看下measureChild方法

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

就是根据父容器的measureSpec和子View的layoutParams计算出子View的measureSpec。getChildMeasureSpec方法我们上面已经分析过了。

值得注意的是，ViewGroup并没有对测量过程进行统一实现，因为不同的布局各有特点，所以onMeasure由ViewGroup子类根据自身特点去实现，比如LinearLayout，RelativeLayout



### 5.Layout过程

Layout是用来确定元素位置的，layout方法确定View本身的位置，然后在其中调用onLayout方法确定子元素位置。

##### 5.1 view的Layout过程

```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) 		 {
            onLayout(changed, l, t, r, b);
			...
        }
}
```

在layout方法中首先会调用setFrame初始View四个顶点的位置，这样自身的位置也就确定了，接着调用onLayout确定子元素位置。onLayout在View中是空实现，交由具体的布局去实现

#####  5.2 ViewGroup的Layout过程

以LinearLayout为例

```java
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```

根据orientation不同调用layoutVertical或者layoutHorizontal方法，我们只看layoutVertical()

```java
void layoutVertical(int left, int top, int right, int bottom) {
    ...
	for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();

                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
	...
                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }

                childTop += lp.topMargin;
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }
```

遍历子view，获取累计每个子view的高度和间距，然后调用setChildFrame，这个方法会调用子View的layout方法，从而确定子View的位置，因为没确定一个子View，childTop会逐渐增大，也就使得子View按照从上到下的方式线性排列。



### 6.Draw过程

步骤

1. 绘制背景 drawBackground(canvas)
2. 绘制自身内容 onDraw()
3. 绘制子View dispatchDraw()
4. 绘制装饰 onDrawScrollBars()

```java
public void draw(Canvas canvas) {
       ...

        
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);// Step 1, draw the background, if needed
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            return;
        }
```

