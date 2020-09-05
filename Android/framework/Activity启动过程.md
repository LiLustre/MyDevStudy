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

![]([[https://lize1.gitee.io/mystudyphoto/Android/FrameWork/Activity%E8%B7%9F%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png](https://lize1.gitee.io/mystudyphoto/Android/FrameWork/Activity跟启动流程.png)











