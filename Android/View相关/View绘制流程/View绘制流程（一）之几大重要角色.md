#### 一、目录
 <font size=4>![image](https://note.youdao.com/yws/api/personal/file/4E336A0A6E1B4A98B333D4B2E65A8318?method=download&shareKey=0f5661d776984452ab11bcf65338bac7)
#### 二、简介
    在理解View绘制流程，之前我们先解析下View绘制中的几大重要成员。
- **Window**
    - 介绍：Window表示一个窗口，用来承载View的抽象类。不管是Activity、Dialog还是Toast,它们的视图实际上都是附加在Window上的，因此Window实际是View的直接管理者。他的唯一实现是PhoneWindow
    
    ![image](https://note.youdao.com/yws/api/personal/file/D20C68C51A6941E5A3E40564047169A6?method=download&shareKey=5b1016e0404da88d173edbddd941bed2)

- **WindowManager**
    - 简介：WindowManager继承自ViewManager的接口。WindowManager主要用来管理窗口的一些状态、属性、view增加、删除、更新、窗口顺序、消息收集和处理等。
    
    ![image](https://note.youdao.com/yws/api/personal/file/CBD71AC5754145D3B426ED1EE8186FC5?method=download&shareKey=344a95c47ac6a5539c000e347eff73a3)
- **ViewRootImpl**
    - 简介：ViewRootImpl是View中的最高层级，属于所有View的根（但ViewRootImpl不是View，只是实现了ViewParent接口），是实现 View 的绘制的类，它实现了View和WindowManager之间的通信协议。大多数情况下，他也是WindowManagerGloable的内部功能具体实现 

- **DecorView**
    - DecorView是Activity内所有View的父布局，DecorView 是 FrameLayout 的子类，FrameLayout 是 GroupView 的子类，所以DecorView 是一个 GroupView。内部包含上面是标题，下面是内容栏（id是content） 

#### 三、Activity、PhoneWindow、DecorView三者之间的关系
    每一个Acyiviy 包含一个Window对象，这个window对象是phoneWindow
    PhoneWindow内包含一个DecorView。同时，PhoneWindow也是Activity和View系统交互的接口。DecorView是整个窗口内的所有View的跟布局
    DecorView将屏幕划分两个区域：1.titleView 2.ContentView
    DecorView本质上是一个FrameLayout，是Activity中所有View的祖先
    而我们平时所写的就是展示在ContentView中的，下图表示Activity的构成。
    

![image](https://note.youdao.com/yws/api/personal/file/9BE92E1EB97642BEADBA017DD2438642?method=download&shareKey=4b462df191c7776a33ed9eaab0139793)
