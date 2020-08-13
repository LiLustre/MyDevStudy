<font size=3>

#### View的Measure测量过程
    1、View的测量从ViewRootImpl触发，从DocerView开始遍历View树，计算出子View的宽和高的MeasureSpace，
    2、调用子View的measure方法将MeasureSpace传递给子View，子View在measure方法内调用OnMeasure方法，计算并设置View的宽和高，
    3、ViewGroup在onMeasure内调用measureChildren方法触发子View的measure操作
    以上就是整体的测量流程
    在View的子View内会根据控件的特性重写onMeasure，完成自定义的测量。在原View内，会调用getDefaultSize方法获取View的宽高。
    
相关的核心代码流程如下
```
private void perormMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    ...
    // 具体的测量操作分发给ViewGroup
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}

// 在ViewGroup中的measureChildren()方法中遍历测量ViewGroup中所有的View
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        // 当View的可见性处于GONE状态时，不对其进行测量
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

// 测量某个指定的View
protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    // 根据父容器的MeasureSpec和子View的LayoutParams等信息计算
    // 子View的MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom, lp.height);
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

// View的measure方法
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    // ViewGroup没有定义测量的具体过程，因为ViewGroup是一个
    // 抽象类，其测量过程的onMeasure方法需要各个子类去实现
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
}

// 不同的ViewGroup子类有不同的布局特性，这导致它们的测量细节各不相同，如果需要自定义测量过程，则子类可以重写这个方法
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // setMeasureDimension方法用于设置View的测量宽高
    setMeasureDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec), 
    getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

// 如果View没有重写onMeasure方法，则会默认调用getDefaultSize来获得View的宽高
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = sepcSize;
            break;
    }
    return result;
}
```
##### 对getSuggestMinimumWidth的分析
    如果View没有设置背景，那么返回android:minWidth这个属性所指定的值，这个值可以为0；
    如果View设置了背景，则返回android:minWidth和背景的最小宽度这两者中的最大值。    
```
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinmumWidth());
}

protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
}

public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

##### 在Activity中获取某个View的宽高
    由于View的measure过程和Activity的生命周期方法不是同步执行的，如果View还没有测量完毕，那么获得的宽/高就是0。
    所以在onCreate、onStart、onResume中均无法正确得到某个View的宽高信息
1.Activity或者View重写onWindowFocusChanged，在你该方法内获取View 的宽高
    
```
// 此时View已经初始化完毕
// 当Activity的窗口得到焦点和失去焦点时均会被调用一次
// 如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁地调用
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if (hasFocus) {
        int width = view.getMeasureWidth();
        int height = view.getMeasuredHeight();
    }
}
```
2.view.post(runnable) 

```
/ 通过post可以将一个runnable投递到消息队列的尾部，// 然后等待Looper调用次runnable的时候，View也已经初
// 始化好了
protected void onStart() {
    super.onStart();
    view.post(new Runnable() {

        @Override
        public void run() {
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```
3.使用ViewTreeObserver的addOnGlobalLayoutListener监听

```
// 当View树的状态发生改变或者View树内部的View的可见性发生改变时，onGlobalLayout方法将被回调
protected void onStart() {
    super.onStart();

    ViewTreeObserver observer = view.getViewTreeObserver();
    observer.addOnGlobalLayoutListener(new OnGlobalLayoutListener() {

        @SuppressWarnings("deprecation")
        @Override
        public void onGlobalLayout() {
            view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```

</font>