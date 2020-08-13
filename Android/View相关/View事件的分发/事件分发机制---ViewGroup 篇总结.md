<font size=3>

##### 事件传递的方法
    1、ViewGroup 中特有的方法 onInterceptTouchEvent
    2、和与View相同的Touch()、onTouchEvent()、和dispatchTouchEvent()
    
##### onInterceptTouchEvent
    这个方法代表是否拦截事件，返回true 代表拦截事件，不允许事件继续向子View传递、返回false代表不拦截事件，
    
    onInterceptTouchEvent的调用时机：
    1、当前事件是按下事件 或者 2、触摸的子View目标不为空
    并且
    3、disallowIntercept是false，拦截功能没有被禁止
    所以：
    1、当ViewGroup 分发事件发生区域的子View目标为空并且，不是Down事件时，onInterceptTouchEvent不会被执行
    2、当拦截功能被禁止，onInterceptTouchEvent不会被执行
```
  public boolean dispatchTouchEvent(MotionEvent ev) {
        ...
      if (actionMasked == MotionEvent.ACTION_DOWN
                        || mFirstTouchTarget != null) {
                    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                    if (!disallowIntercept) {
                        intercepted = onInterceptTouchEvent(ev);
                        ev.setAction(action); // restore action in case it was changed
                    } else {
                        intercepted = false;
                    }
                } else {
                    // There are no touch targets and this action is not an initial down
                    // so this view group continues to intercept touches.
                    intercepted = true;
                }
        ...
    }
    
```

#### dispatchTouchEvent
    这个方法与View的内部实现有所不同，主要的不同之处在于内部多了一些是否拦截事件的判断和分发事件给子View
    
    它的内部判断通过disallowIntercept 和onInterceptTouchEvent方法来判断，当disallowIntercept为True 或者onInterceptTouchEvent返回false 是事件向子view传递
    disallowIntercept：是否禁用掉事件拦截的功能，默认是false，没有禁用拦截功能，True则代表禁用了拦截功能。也可以通过调用requestDisallowInterceptTouchEvent方法对这个值进行修改。
    
    当disallowIntercept为True 或者onInterceptTouchEvent返回false 时 ，ViewGroup 会寻找事件发生区域的子View，然后调用子View 的dispatchTouchEvent方法，如子View 的dispatchTouchEvent返回true,则不会响应viewgroup 的touch事件，否则将会按照View的方式响应事件，从dispatchTouchEvent()开始 到 touche 再到 onTouchEvent
    
    
    
1.  Android事件分发是先传递到ViewGroup，再由ViewGroup传递到View的。

2. 在ViewGroup中可以通过onInterceptTouchEvent方法对事件传递进行拦截，onInterceptTouchEvent方法返回true代表不允许事件继续向子View传递，返回false代表不对事件进行拦截，默认返回false。

3. 子View中接收到传递的事件，ViewGroup中onTouchEvent将无法接收到任何事件。如果子View中的dispatchTouchEvent返回false，则Activity的OnTouchEvent将会接收到事件

#### 事件在哪些对象之间进行传递？

    事件的分发在Activity、ViewGroup、View及其派生类之间。 Android 的UI界面 由Activity、ViewGroup、View 及其派生类组成。 
    Activity：控制生命周期、处理事件、通过回调方法 与Window、View进行交互 View：
    所有UI组件的基类 ViewGroup：View的集合，是Android中父布局的父类，
    事件分发顺序

事件的分发顺序：Activity->ViewGroup->View 即：1个点击事件发生后，事件先传到Activity、再传到ViewGroup、最终再传到 View
    