<font size=3>

##### 涉及事件传递的方法
    1·  View内的dispatchTouchEvent方法
    2.  OnTouchListener的onTouch 方法
    3.  onTouchEvent方法
    
    三个方法的执行顺序 dispatchTouchEvent-> OnTouchListener的onTouch -> onTouchEvent
    

```
dispatchTouchEvent源码
public boolean dispatchTouchEvent(MotionEvent event) {
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
        return true;
    }
    return onTouchEvent(event);
}
```

##### dispatchTouchEvent方法
    事件的传递都最先经过这个方法。如果事件由这个View处理，则为true，否则为false
    
    该方法内逻辑：
    1、 先判断OnTouchListener 是否为空、控件是否可用、mOnTouchListener.onTouch 是否返回True，当三个条件都返回True时，该方法也返回True，否则 执行onTouchEvent方法。

##### OnTouchListener的onTouch 方法
    该方法内返回值是我们自己返回的，所以当我们设置触摸监听，实现该方法时，如果返回True 则 onTouchEvent()方法不会执行。

##### onTouchEvent
    在该方法内，有一种情况时控件可点击的时候，该方法对 事件的ACTION_UP、ACTION_DOWN、ACTION_CANCEL、ACTION_MOVE、分别处理之后，返回True。如果控件不可点击则返回false ，此时调用完该方法的dispatchTouchEvent也返回false。
    
#### 总结：
    事件的分发由 dispatchTouchEvent完成，而它的返回值决定是否继续分发下一个事件，返回True 则下一个事件才会继续处理，返回false，则下一个事件将不会被处理。
    而他的返回值 有 onTouch 和 onTouchEvent方法分别决定，
    当 onTouch返回True 并且该控件可用  此时dispatchTouchEvent返回True，当 onTouch返回False，则会执行onTouchEvent方法，
    如果onTouchEvent方法返回True，则dispatchTouchEvent返回True，onTouchEvent方法返回False 则dispatchTouchEvent返回False。


