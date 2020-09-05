####  一、Activity启动过程的三大角色

1、AMS（ActivityManagerService）：是Android中最核心的服务之一，主要负责管理四大组件、处理应用进程、记录应用运行时的状态

2、ActivityRecord：AMS的眼里，并不直接与Activity打交道，而是通过ActivityRecord作为媒介记录了有关于Activity运行数据与状态信息。

3、ProcessRecord：则记录了进程的所有信息

#### 二、启动场景

1、在手机桌面、手指点击App图片、启动Activity，

2、桌面为Launcher，并且Launcher也是一个Activity，内聚了其它APP的信息，以方便用户操作。

3、如果手机不使用Launcher的话，那么当你启动手机后，屏幕所呈现的模样，与终端无异

####  三、 Launcher启动App的Activity流程

1、桌面LauncherActivity 在创建之后 会从PMS中获取各个App的启动信息，收集的关键信息如在各Manifest定义的App包名、定义为action为MAIN，category为LAUNCHER的Activity的信息、应用图标等。关联这些信息并生成桌面图标。

2、当点击图标是会开始App的根启动

3、App图标点击之后 ，接收到点击事件，之前关联的关键启动信息存于View tag中，这里取出并用以启动。链路为Launcher.startActivitySafely() -> Launcher.startActivity() -> Activity.startActivity(）

4、在 上面的过程中，当前AMS中如没有关于此APP的相关TASK，理所应当为此次启动新建TASK进行处理，会为启动所用的Intent设置FLAG_ACTIVITY_NEW_TASK的标识位。Launcher本身为Activity，最终以Activity.startActivity(）进行启动

5、而以Activity.startActivity(）启动，不管以何种形式启动，最终到达Activity.startActivityForResult()

```java
Launcher

    public void onClick(View v) {
        ......
        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            ......
            boolean success = startActivitySafely(v, intent, tag);
            ......
        } 
        ......
    }
```

大致流程如下：

![](https://lize1.gitee.io/mystudyphoto/Android/FrameWork/Activity%E8%B7%9F%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png)



#### 四、Activity.startActivity()调用流程

**1、startActivity**

源码如下：

```java
@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        //第二个参数为-1表示Launcher不需要知道根Activity的启动结果
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }


    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        //mParent表示当前Activity的父类，当根活动还没创建则mParent==null    
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
           ... 
        } 
        ...
    }
```

1、从上面看出，startActivity的调用会调用到startActivityForResult，startActivityForResult第二个参数requestCode为-1表示Launcher不需要知道根Activity的启动结果

2、`startActivityForResult`会判断mParent是否为Null来以合适的方式启动Activtiy。mParent表示当前Activity的父类，当根活动还没创建则mParent==null 

3、然后回调用到 调用Instrumentation的`execStartActivity`

4、`Instrumentation`:这个类很重要，重要体现在对Activity生命周期的调用根本离不开她,每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象



**2、Instrumentation.execStartActivity**

大致源码如下：

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
        try {
		   ...
		    //获取AMS的代理对象
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

1、调用ActivityManager的getService方法来得到AMS的代理对象，

2、然后调用这个代理对象的startActivity方法

3、AMS引用作为服务端处于SystemServer进程中，当前Launcher进程作为客户端，与服务端不在同一个进程，所以ActivityManager返回的是IActivityManager.Stub的代理对象，此时如果要实现客户端与服务端进程间的通信，只需要在AMS继承了IActivityManager.Stub类并实现了相应的方法，这样Launcher进程作为客户端就拥有了服务端AMS的代理对象，然后就可以调用AMS的方法来实现具体功能了，就这样Launcher的工作就交给AMS实现了

**3、AMS启动Activity**