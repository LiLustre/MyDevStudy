###  1、基础认知
#### 事件分发的本质
将点击事件（MotionEvent）传递到某个具体的View & 处理的整个过程

#### 事件在哪些对象之间进行传递？
事件的分发在Activity、ViewGroup、View及其派生类之间。
Android 的UI界面 又Activity、ViewGroup、View 及其派生类组成。
Activity：控制生命周期、处理事件、通过回调方法 与Window、View进行交互
View：所有UI组件的基类
ViewGroup：View的集合，是Android中父布局的父类，

#### 事件分发顺序
事件的分发顺序：Activity->ViewGroup->View
即：1个点击事件发生后，事件先传到Activity、再传到ViewGroup、最终再传到 View

###### 事件分发过程由哪些方法协作完成？
dispatchTouchEvent() 、onInterceptTouchEvent()和onTouchEvent()

dispatchTouchEvent() :分发事件，当事件传递到当前View时，就会被调用

onTouchEvent()：处理点击事件，在dispatchTouchEvent（）中调用

onInterceptTouchEvent()：拦截事件，只存在于ViewGroup中，在DispatchTouchEvent()中调用

### 2.Activity的事件分发机制
1、首先Activity 调用DispatchTouchEvent()方法，方法内如果是Down事件 则调用onUserInteraction()方法。onUserInteraction()方法在Activity 在栈顶时，用户点击home、back、menu、时调用。

2、然后调用getWindow.superDispatchTouchEvent(),如果过返回true，则Activity的dispatchTouchEvent()返回true，否则继续向下调用Activity的OnTouchEvent()方法。

2.1 getWindow.获取window对象，而Window是抽象类，唯一实现是PhoneWindow，所以获取的PhoneWindow对象，调用PhoneWindow中的superDispatchTouchEvent()方法。

2.2 PhoneWindow的superDispatchTouchEvent() 方法调用内部类继承 FrameLayout 的dispatchTouchEvent（），这样事件分发到ViewGroup。

3、当事件为被Activity的任何子View 接收处理，则调用Activity的onTouchEvent（）处理，
public boolean onTouchEvent(MotionEvent event) {
当触摸事件发生在window的边界外时，返回true。否则返回false。
   
```
 if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    return false;
}
```

### 3.ViewGroup的事件分发机制
1.首先判断 是否拦截事件，通过onIntercepterTouchEvent()或disllowIntercept 判断是否需要拦截。
-     disllowIntercept 是否禁用事件拦截的功能默认是false  可以通过requestDisllowInterceptTouchEvent（）方法修改。
-     通过onIntercepterTouchEvent()返回值取反判断是否继续分发，如onIntercepterTouchEvent() 返回true，则代表拦截事件，如果返回false则代表不拦截事件
2.如果拦截事件 则遍历子View，并判断是否是当前事件发生的位置，如果是，则调用子View的dispatchTouchEvent()方法，这样实现了事件的传递到子View，如果当前位置没有子View，则跳出判断，并调用父类即View的dipatchTouchEvent(),并处理自身事件。
3.如果不拦截，调用父类即View的dipatchTouchEvent(),并处理自身事件

onTouchEvent():调用父类即View的onTouchEvent方法，返回true 代表处理事件，当setOnClickLinstener()方法时会返回true，返回false则代表不处理事件。

### 4.View事件的分发机制
1.首先在dispatchTouchEvent中 有三个判断条件 
- 条件一：是否设置了触摸监听
- 条件二：控件是否可用
- 条件三：触摸监听的onTouch()返回值。

一般情况下，如果设置了触摸监听则条件一成立，而且 默认控件是enable的，所以条件二成立，而条件三代表是onTouch()返回值，所以需要我们手动修改，如果三个条件都成立，则直接返回true，如果不成立，则执行onTouchEvent()方法。
2.如果上述条件不成立，则执行onTouchEvent()方法，如果控件可点击，则除非onClick方法，并返回true，若该控件不可点击，就一定返回false

##### ***如果onTouchEvent() 手动设置返回false 无论是否拦截事件，只要返回false，当前View 都不会接受事件列的其他事件
##### ***如果ViewGroup的子View 的onTouchEvent()返回true，则在此事件列的事件将不会回传给ViewGroup的onTouchEvent

##### *** 如果ViewGroup的onInterceptTouchEvent()返回true，拦截事件，并且onTouchEvent()返回true 消费此事件，
- onInterceptTouchEvent()返回true，拦截事件，事件将不再向下分发

- 此事件列的其他事件将不会再进入到onInterceptTouchE()方法中。因为onInterceptTouchEvent()返回一次true，则此事件列的其他事件将不再进行判
- onTouchEvent()事件也将不会回传到父View或者Activity的onTouchEvent事件中。

- 如果 onTouchEvent()返回非false，则此事件列的除了down事件外将不再接受其他事件

##### *** 当ViewGroup在MOVE时 返回true，拦截事件，onTouchEvent返回true，子View的onTouchEvent()返回。
- 因为Move在Down事件之前，down事件还没有被拦截，所以down事件被onTouchEvent()方法处理，不会回传到父布局ViewGroup的onTouchEvent();
- 当Move事件发生，被onInterceptTouchEvent()返回true，事件拦截，move事件将被转化成cancel 分发给了子View，子View会执行onTouchEvent()方法，但之后此事件列的其他Move事件将不会再分发给子View，而是在ViewGroup的onTouchEvent中处理，onInterceptTouchEvent()也不会执行，即使此事件列的up事件发生。up事件也将不会分发给子View。





        



