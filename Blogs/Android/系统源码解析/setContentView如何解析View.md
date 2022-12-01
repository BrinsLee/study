### 入口

继承Activity

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_custom_parse)
 } 
```

```java
public void setContentView(@LayoutRes int layoutResID) {
    // getWindow() 返回phoneWindow    
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
 }
```

`phoneWindow`创建时机`ActivityThread#performLaunchActivity`

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent{
    ...
    Activity activity = null;
    try{
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //通过反射创建Activity
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
    }catch (Exception e) {
        ...
    }
    
    ...
    if(activity != null) {
        ...
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
    }
}
```

`Activity#attach`

```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        
        attachBaseContext(context);
        mFragments.attachHost(null /*parent*/);
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        ...
    }
```

`回到PhoneWindow.setContentView()`

```java
@Override
public void setContentView(int layoutResID) {
	if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
}
```

```java
 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
     ...
 }
```

总结

```java
ActivityThread.performLaunchActivity
---> activity.attach
	---> new PhoneWindow()//创建PhoneWindow
---> mInstrumentation.callActivityOnCreate   

PhoneWindow.setContentView//主要目的 创建DecorView 拿到Content
---> installDecor() //创建DecorView 拿到Content
    ---> mDecor = generateDecor(-1)
    	---> new DecorView();
    ---> mContentParent = generateLayout(mDecor)
        ---> R.layout.screen_simple ---> @android:id/content ---> mContentParent
        ---> mDecor.onResourcesLoaded(mLayoutInflater, layoutResource); // 将R。layout.screen_simple 添加到DecorView
        ---> ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);  
---> mLayoutInflater.inflate(layoutResID, mContentParent);//将Rlayout.activity_main 渲染到ContentParent中

```



继承AppCompatActivity

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_custom_parse)
 } 
```

```java
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        // 创建 AppCompatDelegate
        getDelegate().setContentView(layoutResID);
    }
```

```java
    @Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mOriginalWindowCallback.onContentChanged();
    }
```

```java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
          mSubDecor = createSubDecor();
    }
}
```

总结

```java
AppCompatDelegate.setContentView
---> ensureDecor()
    ---> mSubDecor = createSubDecor()
    	---> ensureWindow() //从Activity 拿PhoneWindow
    	---> mWindow.getDecorView()
    		---> installDecor()// mContentParent
    	---> final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content)//拿到R.layout.screen_simple 里面的content, 并把content里面的View复制到R.id.action_bar_activity_content
    	---> windowContentView.setId(View.NO_ID)// 把原始content id置为空
    	---> contentView.setId(android.R.id.id_content)// subDecorView id R.id.action_bar_activity_content --> 置为id_content
---> ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content)
---> LayoutInflater.from(mContext).inflate(resId, contentParent)
```

![image-20221011145859358](E:\学习资料\Android\view\setContentView\image-20221011145859358.png)

### 渲染

```java
LayoutInflater.from(mContext).inflate(resId, contentParent);
	// 创建View --- 布局的rootView
	---> final View temp = createViewFromTag(root, name, inflaterContext, attrs);
			// 是否是sdk里面的布局 
		---> if (-1 == name.indexOf('.')) { //name = androidx.constraintlayout.widget.ConstraintLayout
                    view = onCreateView(parent, name, attrs);
            			---> PhoneLayoutInflater.onCreate(name, attrs);
            				---> View view = createView(name, prefix, attrs);
              } else {
                    view = createView(name, null, attrs);
            				// 通过反射调用创建View对象
            			---> clazz = mContext.getClassLoader().loadClass(prefix != null ? (prefix + name) : name).asSubclass(View.class);
            			---> constructor = clazz.getConstructor(mConstructorSignature);
            			---> final View view = constructor.newInstance(args);

              }
	---> rInflateChildren(parser, temp, attrs, true);// 创建子View
		---> rInflate
            ---> 


// LayoutInflate参数作用
// 方式1 布局添加成功
// View view inflater.inflate(R.layout.inflate_layout, root, true)

// 方式2 报错，一个View只能有一个父布局
// View view inflater.inflate(R.layout.inflate_layout, root, true)
// root.addView(view)

// 方式3 布局添加成功，第三个参数为false
// 目的 想要inflate_layout的根节点属性（宽高）有效，又不想让其处于某个容器中
// View view inflater.inflate(R.layout.inflate_layout, root, false)
// root.addView(view)

// 方式4 root = null
// inflate_layout的根节点的属性（宽高）设置无效，只是包裹子view
// 但是子View有效，因为子View是包含于容器下的
// View view inflater.inflate(R.layout.inflate_layout, null, false)
            

private static final String[] sClassPrefixList = {
    "android.widget.",
    "android.webkit.",
    "android.app.",
    "android.view"
};
```

#### merge、include、ViewStub 标签 

- merge: 

   	1. 用于优化布局，减少层级
   	2. 只能做rootView，不能作为子布局

- include:

  1. 注意id设置，优先使用include标签设置的id 
  2. 不能作为rootView

- ViewStub:

  1. 隐藏作用，懒加载，跟include差不多 

  

  

  
