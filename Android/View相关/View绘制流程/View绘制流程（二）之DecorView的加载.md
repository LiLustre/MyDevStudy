<font size=3>
从Activity的startActivity开始，最终调用到ActivityThread的handleLaunchActivity方法来创建Activity，相关核心代码如下：

- **handleLaunchActivity**
> 
    handleLaunchActivity的主要操作
    1、初始化WindowManagerService
    2、实例化Activity、初始化Activity的Window等参数、回调Activity的onCreate（）
    3、执行Acttivity的onResume（）
```
 private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        ...略
        // 初始化WindowManagerService
        WindowManagerGlobal.initialize();
        //实例化Activity，初始化Activity的Window等参数并回调Activity的onCreate、onStart
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            //执行Activity的onResume
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        ...略            
        } else {
            ...略
        }
    }
```
- **performLaunchActivity**
>
    performLaunchActivity的操作步骤：
    1、获取Activity信息
    2、实例化Activity 通过反射获取Activity的类，并创建Activity
    4、获取Application
    5、调用Activity的attach函数 
        初始化Activity的成员变量包括Window、Application、和mainThread等。初始化PhoneWindow的构造函数的内部初始化了PhoneWindow的DecorView。
    6、执行Activity的生命周期函数
        1、回调Activity的onCreate方法
            通常Activity的onCreate的内我们会调用setContentView
            而setContentView 内部调用PhoneWindow的setContentView，
            PhoneWindow的setContentView通过LayoutInflater.inflate()将我们的布局加载到DecorView中
        2、如果Activity没有Finish则调用Activity的onStart()
        3、设置Activity的savedInstanceState
        4、设置Window的title
```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
       
        //1、获取Activity信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

       ...略
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            //2、通过反射获取到activity类,并创建Activity实例
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            //3、如果没在清单里注册就会出现这个错误
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            //4、获取Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                ...略
                appContext.setOuterContext(activity);
                //5、初始化Activity的成员变量包括Window、Application、和mainThread等
                //         初始化PhoneWindow的构造函数的内部初始化了PhoneWindow的DecorView。

                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //6、回调Activity的onCreate方法
                // 通常Activity的onCreate的内我们会调用setContentView
                // 而setContentView 内部调用PhoneWindow的setContentView，
                // PhoneWindow的setContentView通过LayoutInflater.inflate()将我们的布局加载到DecorView中
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                //7、如果Activity没有Finish则调用Activity的onStart()
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                // 8、设置Activity的savedInstanceState
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                //9、设置Window的title
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```
- **handleResumeActivity**
>
    handleResumeActivity操作步骤：
    1、执行Activity的onResume
    2、调用 WindowManager的addView(decor, l);
```
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    // 调用Activity的onResume方法
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null &&& !a.mFinished && willBeVisible) {
            //获取Activity的Window
            r.window = r.activity.getWindow();
            //获取Window的DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //获取Activity的WindowManagerImpl
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            //将Window的DecorView赋值个Activity的mDecor
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                // WindowManager的实现类是WindowManagerImpl，
                // 所以实际调用的是WindowManagerImpl的addView方法
                wm.addView(decor, l);
            }
        }
    }
}
```
- **WindowManager的addView(View view, ViewGroup.LayoutParams params)**

     实际上WindowManager的实现类是WindowManagerImpl，而WindowManagerImpl的addView()方法内部，调用了WindowManagerGlobal的addView。所以接下来分析WindowManagerGlobal的addView。源码如下：

    1、实例化ViewRootImpl
    2、把DecorView加载到Window中，触发View的测量、布局、绘制
```
// WindowManagerGlobal的addView方法
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View pannelParentView = null;
    synchronized (mLock) {
        ...
        // 创建ViewRootImpl实例
        root = new ViewRootImpl(view..getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    try {
        // 把DecorView加载到Window中，触发View的测量、布局、绘制
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        synchronized (mLock) {
            final int index = findViewLocked(view, false);
            if (index >= 0) {
                removeViewLocked(index, true);
            }
        }
        throw e;
    }
}
// ViewRootImpl的方法
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
        ...
            requestLayout();        
        ...
            
        }
    }

public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
           ...
            scheduleTraversals();
        }
    }
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            ...
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
           ...
        }
    }
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
 void doTraversal() {
        if (mTraversalScheduled) {
            ...
            performTraversals();
            ...
        }
    }
```
最终ViewRootImpl的setView方法利用执行Runable的方式执行了performTraversals();

- **DecorView加载程序流执行程图**

![image](https://note.youdao.com/yws/api/personal/file/9B05E31829BC4DF191679A99710FEAE8?method=download&shareKey=7b3e9738865908dc0c3e2f9d433eca2a)



#### 总结：
    1、Activity从startActivity开始，最终调用带ActivityThread的handleLaunchActivity方法内，
        这个方法的内部调用performLaunchActivity方法，再这个方法内实例化Activity。
        实例化之后调用Activity的attach方法初始化Activity的成员变量包括Window、Application、和mainThread等，
        其中Window实例化的是PhoneWindow，PhoneWindow在实例化的时候初始化DecorView，
        然后执行Activity的OnRusume之前的生命周期函数，
        其中而setContentView内部调用PhoneWindow的setContentView，PhoneWindow的setContentView通过LayoutInflater.inflate()将我们的布局加载到DecorView中，
    2、然后执行ActivityThread的handleResumeActivity方法，其内部执行Activity的onRusume
        然后从Activity获取WindowManagerImpl，执行addView()方法把DecorView加载到Window中，在该方法内会初始化一个ViewRootImpl对象，然后通过ViewRootImpl执行Runable的形式触发View的测量、布局、绘制
</font>