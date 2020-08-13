<font size=3>

#### View 的Draw过程
    1、从ViewRootImpl开始，触发View 的绘制流程，从performDraw方法开始，调用View 的draw方法，并传入canvas，
    2、在draw方法内调用onDraw方法实现绘制，然后调用dispatchDraw方法，
    3、如果是ViewGroup会重写该方法，分发绘制操作，调用子View 的draw方法。
    
    View的draw方法的步骤
        1、绘制View的背景
        2、如果需要的话，保持canvas的图层，为fading做准备
        3、绘制View的内容
        4、分发View的绘制操作
        5、如果需要的话，绘制View的fading边缘并恢复图层
        6、绘制View的装饰(例如滚动条等等)

```
private void performDraw() {
    ...
    draw(fullRefrawNeeded);
    ...
}

private void draw(boolean fullRedrawNeeded) {
    ...
    if (!drawSoftware(surface, mAttachInfo, xOffest, yOffset, 
    scalingRequired, dirty)) {
        return;
    }
    ...
}

private boolean drawSoftware(Surface surface, AttachInfo attachInfo, 
int xoff, int yoff, boolean scallingRequired, Rect dirty) {
    ...
    mView.draw(canvas);
    ...
}

// 绘制基本上可以分为六个步骤
public void draw(Canvas canvas) {
    ...
    // 步骤一：绘制View的背景
    drawBackground(canvas);

    ...
    // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
    saveCount = canvas.getSaveCount();
    ...
    canvas.saveLayer(left, top, right, top + length, null, flags);

    ...
    // 步骤三：绘制View的内容
    onDraw(canvas);

    ...
    // 步骤四：绘制View的子View
    dispatchDraw(canvas);

    ...
    // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
    canvas.drawRect(left, top, right, top + length, p);
    ...
    canvas.restoreToCount(saveCount);

    ...
    // 步骤六：绘制View的装饰(例如滚动条等等)
    onDrawForeground(canvas)
}
```
##### setWillNotDraw的作用


```
// 如果一个View不需要绘制任何内容，那么设置这个标记位为true以后，
// 系统会进行相应的优化。
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```
- 默认情况下，View没有启用这个优化标记位，但是ViewGroup会默认启用这个优化标记位。
- 当我们的自定义控件继承于ViewGroup并且本身不具备绘制功能时，就可以开启这个标记位从而便于系统进行后续的优化。
- 当明确知道一个ViewGroup需要通过onDraw来绘制内容时，我们需要显示地关闭WILL_NOT_DRAW这个标记位。
