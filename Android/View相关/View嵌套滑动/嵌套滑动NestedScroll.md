#### 一、机制介绍
    1、子View在滑动之前，将滑动的信息传递给父View
    2、父View在接收到上面的值之后，决定是否部分消耗或者全部消耗，利用这个值父View可以在子View滑动之前做自己想做的事。
    3、子View会接收父View销耗值的剩余值，然后根据这个值滑动
    4、滑动完成之后如果子View没有完全消耗掉这个剩余的值就再告知一下父View，我滑完了，但是还有剩余的值你还要不要

Lollipop及以上版本的所有View都已经支持了NestedScrolling，Lollipop之前版本可以通过Support包进行向前兼容

#### 二、嵌套滑动4个重要角色
    1、NestedScrollingParent
    2、NestedScrollingParentHelper
    3、NestedScrollingChild
    4、NestedScrollingChildHelper
 
NestedScrollingParent和NestedScrollingChild都是接口。分别对应ViewGroup和View。NestedScrollingChildHelper 和 NestedScrollingParentHelper 是两个帮助类，当我们在实现 NestedScrollingChild 和 NestedScrollingParent 接口时，使用这两个帮助类可以简化我们的工作。

- **NestedScrollingChild**

    **相关方法**
    > 
        1、 boolean startNestedScroll(@ScrollAxis int axes);开始滑动
        2、 void stopNestedScroll();停止滑动
        3、触摸滑动相关
        boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
                @Nullable int[] offsetInWindow);
        boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
                int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow);
        4、惯性滑动
        boolean dispatchNestedPreFling(float velocityX, float velocityY);
        boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);

    **1、public boolean startNestedScroll(int axes);**
        
         开启嵌套滚动流程(实际上是进行了一些嵌套滚动前准备工作)。当找到了能够配合当前 子view 进行嵌套滚动的 父view 时，返回值为 true。
        
    **2、public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);**
        
        在子view 自己进行滚动之前调用此方法，询问 父view 是否要在 子view 之前进行滚动。
        此方法的前两个参数用于告诉 父View 此次要滚动的距离；
        而 第三 第四 个参数用于 子view 获取 父view 消费掉的距离和 父view 位置的偏移量。
        第一 第二 个参数为输入参数，即常规的函数参数，调用函数的时候我们需要为其传递确切的值。
        而 第三 第四个参数为输出参数，调用函数时我们只需要传递容器（在这里就是两个数组），在调用结束后，我们就可以从容器中获取函数输出的值。
        如果 parent 消费了一部分或全部距离，则此方法返回 true。
    
    **3、public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);**
        
        在 子view 自己进行滚动之后调用此方法，询问 父view 是否还要进行余下(unconsumed)的滚动。
        前四个参数为输入参数，用于告诉 父view 已经消费 和 尚未消费 的距离，最后一个参数为输出参数，用于 子view 获取 父view 位置的偏移量。
    
    **4、stopNestedScroll()**
        
        与 startNestedScroll(int axes) 对应，用于结束嵌套滚动流程；而惯性滚动相关的两个方法与触摸滚动相关的两个方法类似，

- **NestedScrollingParent接口**
    
    **相关介绍**
        
        //当开启、停止嵌套滚动时被调用
        public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
        public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
        public void onStopNestedScroll(View target);
        //当触摸嵌套滚动时被调用
        public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
        public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed);
        
        //当惯性嵌套滚动时被调用
        public boolean onNestedPreFling(View target, float velocityX, float velocityY);
        public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);

    **NestedScrollingChild与NestedScrollingParent与之相对应的方法就会被回调。方法之间的具体对应关系如下**
        
        
        子(发起者) | 父(被回调)
        ---|---
        startNestedScroll | onStartNestedScroll、onNestedScrollAccepted
        dispatchNestedPreScroll | onNestedPreScroll
        dispatchNestedScroll |onNestedScroll
        stopNestedScroll|onStopNestedScroll
        …|…
    
    1.public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
        target:发起嵌套滚动的子View，此子view必须实现NestedScrollingChild接口。上面提到过，此子view并不需要是当前view的直接子view
        child:当前view的包含target的直接子view
        nestedScrollAxes:嵌套滚动的方向，可能是SCROLL_AXIS_HORIZONTAL 或 SCROLL_AXIS_VERTICAL 或 二者都有
        
    2、onNestedPreScroll()、onNestedScroll()、onNestedPreFling()、onNestedFling()
        这几个方法分别对应NestedScrollingChild中的dispatchNestedPreScroll()、dispatchNestedScroll()、dispatchNestedPreFling()和dispatchNestedFling()。
        它们的参数也是基本对应的，以onNestedPreScroll()为例，参数dx、dy、consumed实际就是dispatchNestedPreScroll()中的dx、dy、consumed
- **NestedScrollingChildHelper\NestedScrollingParentHelper**
    > 
    NestedScrollingChildHelper是实现NestedScrollingChild接口的View的辅助类。
    在源码中我们看到当实现NestedScrollingChild接口的View调用startNestedScroll()方法时，会调用NestedScrollingChildHelper的startNestedScroll方法。
        
    ```
    // View.javapublic 
    boolean startNestedScroll(int axes) { 
        // ... 
        if (isNestedScrollingEnabled()) { 
            ViewParent p = getParent(); 
            View child = this; 
            while (p != null) { 
                try { 
                    // 关键代码 
                    if (p.onStartNestedScroll(child, this, axes)) { 
                        mNestedScrollingParent = p; 
                        p.onNestedScrollAccepted(child, this, axes); 
                        return true; 
                    }
                } catch (AbstractMethodError e) { 
                    // ... 
                } 
                if (p instanceof View) { 
                    child = (View) p; 
                } 
                p = p.getParent(); 
            } 
        } 
        return false;
    }
    ```
    > 从源码中我们看到会获取当前View的父View，然后调用父控件的onStartNestedScroll()，如果onStartNestedScroll()返回true，则继续调用父View的onNestedScrollAccepted(child, this, axes)，最终返回true。
    
    >onStartNestedScroll的方法说明, 返回true表示接收内控件的滑动信息.
    
    >找到了外控件后ACTION_DOWN事件就没嵌套滑动的事了, 要滑动肯定会在onTouchEvent中处理ACTION_MOVE事件, 接着我们看ACTION_MOVE事件是怎样处理的.
        
    ```
    // NestedScrollView#onTouchEvent
    case MotionEvent.ACTION_MOVE: 
        // ... 
        final int y = (int) MotionEventCompat.getY(ev, activePointerIndex); 
        int deltaY = mLastMotionY - y; 
        // 让外控件先处理滑动距离 
        if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) { 
            deltaY -= mScrollConsumed[1];// 消耗滑动距离 
            // ... 
        } 
        // ... 
        if (mIsBeingDragged) { 
            // ... 
            // 内控件处理滑动距离 
            if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0, 
                          0, true) && !hasNestedScrollingParent()) { 
                // ... 
            } 
    
            final int scrolledDeltaY = getScrollY() - oldY; 
            final int unconsumedY = deltaY - scrolledDeltaY; 
            if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) { 
                // ... 
            } 
            // ... 
        } 
        break;
    ```
    > 在Action的Move的事件中，先调用dispatchNestedPreScroll(),dispatchNestedPreScroll()内调用NestedScrollChildHelper的dispatchNestedPreScroll();dispatchNestedPreScroll()内部获取父View，调用父View 的onNestedPreScroll()。如果父View消耗了滑动距离，则返回True。然后子View调用dispatchNestedScroll(),内部根据NestedScrollChildHelper调用他的dispatchNestedScroll(),在其内部如果调用了父View的onNestedScroll则返回True.
    
    
    
#### 流程图
![image](https://note.youdao.com/yws/api/personal/file/DACB380928B24C7E868EFC3054C41EC7?method=download&shareKey=3419a01cb9202040a774549f017e8d20)


