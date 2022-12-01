## 一、概述

在使用getContxext方法的时候，在不同场景下，取得的Context到底有什么不同，View, Fragment, Activity, Application的getContext究竟是什么样的

## 二、getContext

### DecorView

DecorView的Context是Application Context

ActivityThread.addView ---> PhoneWindow.generateDecor

```java
protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```

### View

一般的View是从LayoutInflater类中inflate生成的，查看inflate方法，会调用rInflate

```java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
		//...
         else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        //...
    }
```

context是入参，其实就是LayoutInflater.from传进来的Context

对于xml布局文件中的View，Context是什么样的呢？

通过查看Activity#setContentView() ---> PhoneWindow#setContentView() ---> LayoutInflater#inflate。而这个LayoutInflater是在PhoneWindow的构造方法内创建的，回到Activity.attach方法，看到构造方法的参数是Activity的Context

于是增加一个结论，在xml中解析的View的Context属于Activity。

那Fragment也是吗？看一下Fragment.onCreateView中参数有LayoutInflater，跟踪一下

Fragment.onCreateView <- Fragment.performCreateView -> FragmentManagerImpl.moveToState <- Fragment.performGetLayoutInflater <- Fragment.getLayoutInflater <- FragmentHostCallback.onGetLayoutInflater

结果是用了FramgentHostCallback构造方法的参数Context。结论也是Activity的Context。

#### FragmentActivity

一般使用FragmentActivity.this和FragmentActivity.getContext方法取到Context，最终取到的都是Activity的Context，不再赘述。

### Fragment

通过Fragment.getContext取到Context，结果是取到FragmentHostCallback.getContext也是Activity的Context。

### Application

取到的是Application Context。

## Applicaiton与Activity的区别

所以最终的目的是区分Application与Activity的Context有什么不同。所以看它们各自的实现，Application与Activity都继承于ContextWrapper，ContextWrapper就是包装了一下抽象类Context，在构造方法里传入Context对象，根本在于构建Application和Activity的地方。

追踪一下发现Context构建都在ContextImpl类内，Application对应createAppContext，而Activity则对应createActivityContext，都在ActivityThread中调用各自创建Context方法进行初始化。

```java
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
    context.setResources(packageInfo.getResources());
    return context;
}

static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId, Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");

        String[] splitDirs = packageInfo.getSplitResDirs();
        ClassLoader classLoader = packageInfo.getClassLoader();

        if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "SplitDependencies");
            try {
                classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
                splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
            } catch (NameNotFoundException e) {
                // Nothing above us can handle a NameNotFoundException, better crash.
                throw new RuntimeException(e);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
            }
        }

        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName, activityToken, null, 0, classLoader);

       //...
    }
```

对比ContextImpl的构建，前三个参数都一致，Activity的Context多了activityToken和classLoder，其中activityToken对应ActivityRecord类，接着调用Context.setResource方法设置Activity的配置和逻辑显示相关的信息Display。

![](E:\workspace\Android-Notes\Images\need_new_task.png)

看一下这个熟悉的错误，这个是使用Application的Context启动Activity时报的错误，这个mSourceRecord就是AcitivityRecord对象，对于Application而言为null，所以需要指定New_Task这个标志。

两者之间更重要的一个区别是：生命周期。

对于Application的Context而言，在整个应用的生命周期内都不会改变；而对于不同的Activity，其Context有可能不同，例如添加一个Dialog必须附着在Activity上，所以使用Application就会报错。

![](E:\workspace\Android-Notes\Images\Context使用.png)

## 总结

除了Application，DecorView和getApplicationContext方法会取到Application Context外，其他方法getContext都会取到Activity Context或者传入的Context。

一般来说，Application Context存在于整个应用的生命周期中，不会随场景变化而改变，所以对于打开不同的Activity，Activity Context可能存在不同，而且生命周期跟Activity的生命周期一致。