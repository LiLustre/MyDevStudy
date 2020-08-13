<font size=3>

#### View的Layout过程
    布局过程从ViewRootImpl开始，调用layout方法并使用Measure获取的宽和高，确定自身的View位置，然后调用onLayout，该方法在View内是空方法，在ViewGroup的子类会重写该方法，遍历子View然后调用子View 的layout方法。
    
相关核心代码如下

```
/ ViewRootImpl.java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    ...
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    ...
}

// View.java
public void layout(int l, int t, int r, int b) {
    ...
    // 通过setFrame方法来设定View的四个顶点的位置，即View在父容器中的位置
    boolean changed = isLayoutModeOptical(mParent) ? 
    set OpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    ...
    onLayout(changed, l, t, r, b);
    ...
}

// 空方法，子类如果是ViewGroup类型，则重写这个方法，实现ViewGroup
// 中所有View控件布局流程
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

}
```
LinearLayout的onLayout方法实现解析

```
protected void onlayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l,)
    }
}

// layoutVertical核心源码
void layoutVertical(int left, int top, int right, int bottom) {
    ...
    final int count = getVirtualChildCount();
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasureWidth();
            final int childHeight = child.getMeasuredHeight();

            final LinearLayout.LayoutParams lp = 
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            ...
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            // 为子元素确定对应的位置
            setChildFrame(child, childLeft, childTop + getLocationOffset(child), childWidth, childHeight);
            // childTop会逐渐增大，意味着后面的子元素会被
            // 放置在靠下的位置
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child,i)
        }
    }
}

private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```
##### getWidth()（getHeight()）与getMeasuredWidth()(getMeasuredHeight())获取的宽(高)的区别
    1、getWidth()/getHeight()：获得View最终的宽/高
    2、getMeasuredWidth()/getMeasuredHeight()：获得View测量的宽/高
先看下各自得源码：

```
// 获得View测量的宽 / 高
  public final int getMeasuredWidth() {  
      return mMeasuredWidth & MEASURED_SIZE_MASK;  
      // measure过程中返回的mMeasuredWidth
  }  

  public final int getMeasuredHeight() {  
      return mMeasuredHeight & MEASURED_SIZE_MASK;  
      // measure过程中返回的mMeasuredHeight
  }  


// 获得View最终的宽 / 高
  public final int getWidth() {  
      return mRight - mLeft;  
      // View最终的宽 = 子View的右边界 - 子view的左边界。
  }  

  public final int getHeight() {  
      return mBottom - mTop;  
     // View最终的高 = 子View的下边界 - 子view的上边界。
  }  
```

类型 | 作用|赋值时机|赋值方法|值大小|使用场景
---|---|---|---|---|---|
getMeasuredWidth()/getMeasuredHeight() | 获取View的测量宽高|measure过程|setMeasuredDimension()|一般情况下两者的宽高相同|measure过程完成之后获取测量的宽高
getWidth()/getHeight() | 获取View的最终宽高|layout过程|layout传递的四个参数之间运算|一般情况下两者的宽高相同|layout过程完成之后，获取View 的宽高

一般情况下，二者获取的宽高是相等的。那么，"非一般"情况是什么？
答：人为设置：通过重写View的layout()强行设置


```
@Override
public void layout( int l , int t, int r , int b){
  
   // 改变传入的顶点位置参数
   super.layout(l，t，r+100，b+100)；

   // 如此一来，在任何情况下，getWidth() / getHeight()获得的宽/高 总比 getMeasuredWidth() / getMeasuredHeight()获取的宽/高大100px
   // 即：View的最终宽/高 总比 测量宽/高 大100px

}

```
