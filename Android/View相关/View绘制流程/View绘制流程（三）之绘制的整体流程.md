<font size=3>

#### View绘制整体流程
    View的绘制最终是从ViewRootImpl的performTraversals()方法开始，从上到下遍历整个视图树，
    每个View控件负责绘制自己，而ViewGroup还需要负责通知自己的子View进行绘制操作。ViewRootImpl的performTraversals()方法的主要源码如下：


```
private void performTraversals() {
    ...
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    ...
    //执行测量流程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    //执行布局流程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    ...
    //执行绘制流程
    performDraw();
}
```
流程图如下：
![image](https://note.youdao.com/yws/api/personal/file/2E6E09635C304E179BE6DC98B9C47962?method=download&shareKey=2f59d09f696d1bb3043e6050c454dc36)

- 注：preformLayout和performDraw的传递流程和performMeasure是类似的，唯一不同的是，performDraw的传递过程是在draw方法中通过dispatchDraw来实现的，不过这并没有本质区别。
</font>