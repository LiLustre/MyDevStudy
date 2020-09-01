<font size=3>

[TOC]



#### 一、Android 屏幕刷新显示机制概述

- **显示系统中的三要素**
  
    在一个典型的显示系统中，一般包括CPU、GPU、display三个部分
    
     CPU：负责计算数据，把计算好数据交给GPU,
    
    GPU：会对图形数据进行渲染，渲染好后放到buffer里存起来
    
     display（屏幕）：负责把buffer里的数据呈现到屏幕上  
    
- **屏幕刷新过程**
  
    显示过程，简单的说就是CPU/GPU准备好数据，存入buffer，display从buffer中取出数据，然后一行一行显示出来。
     所以屏幕刷新的三个步骤：
     1、CPU 计算屏幕数据
     2、GPU 进一步处理并缓存
     3、最后Display从缓存中取出屏幕数据并显示
    
    对应Android 而言，CPU的计算屏幕数据指的就是Android的View树绘制流程，就是从DecorView开始遍历View，执行测量、布局、绘制三大流程。   
    

![屏幕刷新机制](https://note.youdao.com/yws/api/personal/file/3532FF97254541279E01D6B4A09F18D7?method=getImage&version=9173&cstk=qPiLh-_D)
    
- 流程解释 
  
     **底层**：
        
    Display 可以理解为屏幕，底层已固定频率发出一个Vsync信号， 这个固定频率就是每16ms发出一个Vsync信号。
    
    继续看图，Display 黄色的这一行里有一些数字：0, 1, 2, 3, 4。，可以看到每次屏幕刷新信号到了的时候，数字就会变化，所以这些数字其实可以理解成每一帧屏幕显示的画面。也就是说，屏幕每一帧的画面可以持续 16.6ms，当过了 16.6ms，底层就会发出一个屏幕刷新信号，而屏幕就会去显示下一帧的画面。
    
    **APP层**
    
    继续看图，CPU 蓝色的这行，CPU 这块的耗时其实就是我们 app 绘制当前 View 树的时间，而这段时间就跟我们自己写的代码有关系了，如果你的布局很复杂，层次嵌套很多，每一帧内需要刷新的 View 又很多时，那么每一帧的绘制耗时自然就会多一点。
    
    继续看图，CPU 蓝色这行里也有一些数字，其实这些数字跟 Display 黄色的那一行里的数字是对应的，在 Display 里我们解释过这些数字表示的是每一帧的画面，那么在 CPU 这一行里，其实就是在计算对应帧的画面数据，也叫屏幕数据。也就是说，在当前帧内，CPU 是在计算下一帧的屏幕画面数据，当屏幕刷新信号到的时候，屏幕就去将 CPU 计算的屏幕画面数据显示出来；同时 CPU 也接收到屏幕刷新信号，所以也开始去计算下一帧的屏幕画面数据。

    CPU 跟 Display 是不同的硬件，它们是可以并行工作的。要理解的一点是，我们写的代码，只是控制让 CPU 在接收到屏幕刷新信号的时候开始去计算下一帧的画面工作。而底层在每一次屏幕刷新信号来的时候都会去切换这一帧的画面，这点我们是控制不了的，是底层的工作机制。之所以要讲这点，是因为，当我们的 app 界面没有必要再刷新时（比如用户不操作了，当前界面也没动画），这个时候，我们 app 是接收不到屏幕刷新信号的，所以也就不会让 CPU 去计算下一帧画面数据，但是底层仍然会以固定的频率来切换每一帧的画面，只是它后面切换的每一帧画面都一样，所以给我们的感觉就是屏幕没刷新。
- **Android 每隔16ms 刷新一次屏幕**

    指底层每隔16毫秒从buffer中取出需要显示的数据并显示在屏幕上


#### 二、APP层源码分析屏幕的刷新
    从View的的invalidate()、requestLayout()、Activity的加载中的流程可以看出都会调用ViewRootImpl的scheduleTraversals()方法，
    scheduleTraversals()方法会使用Choreographer来执行一个Runable（mTraversalRunnable），
    这个Runable（mTraversalRunnable）内的run方法内包括View树的测量、布局、绘制三大流程。

-  **View#invalidate()**
   
    invalidate()方法会调用view父布局的invalidateChild，invalidateChild内也可以看到内部其实是有一个 do{}while() 循环来不断寻找 mParent，这个mParent是ViewGroup也可以是ViewRootImpl，然后调用mParent的invalidateChildInParent()所以最终才会走到 ViewRootImpl 里去。 ViewRootImpl的invalidateChildInParent()方法通过invalidate()最终调用到scheduleTraversals().
- **View#requestLayout()**
  
    从View的requestLayout()方法开始，会逐级向上调用requestLayout()方法，也就是调用view调用自己mParent的requestLayout()，mParent的requestLayout()内再调用自己mParent的requestLayout()，最终调用到ViewRootImpl的requestLayout()方法，View的requestLayout内部调用scheduleTraversals()。

- **Activity的加载**
  
    从ActivityThread的handleLaunchActivity开始，其内部会依次调用Activity的生命周期函数:onCreate()、onStart()、onResume(),然后WindowManager会实例化一个ViewRootImpl，并调用ViewRootImpl的setView()将DecorView和ViewRootImpl绑定，绑定过程中调用ViewRootImpl的requestLayout(),最终调用到scheduleTraversals()

- **ViewRootImpl#scheduleTraversals()**
  
    1、判断mTraversalScheduled为false时执行，并将mTraversalScheduled设置为True
    
    2、设置同步屏障

    3、将绘制操作的mTraversalRunnable放入Choreographer的Callback

```
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```
**设置同步屏障**

这个同步屏障的作用可以理解成拦截同步消息的执行，主线程的 Looper 会一直循环调用 MessageQueue 的 <code>next()</code> 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。当 <code>next()</code> 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 <code>next()</code> 方法陷入阻塞状态。如果 <code>next()</code> 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除出队列，否则主线程就一直不会去处理同步屏幕后面的同步消息。</p>
而所有消息默认都是同步消息，只有手动设置了异步标志，这个消息才会是异步消息。另外，同步屏障消息只能由内部来发送，这个接口并没有公开给我们使用。



- **Choreographer#postCallback()**
  
    1、因为 postCallback() 调用 postCallbackDelayed() 时传了 delay = 0 进去，所以在 postCallbackDelayedInternal() 里面会先根据当前时间戳将这个 Runnable 保存到一个 mCallbackQueue 队列里，这个队列跟 MessageQueue 很相似，里面待执行的任务都是根据一个时间戳来排序。   Choreographer 里有多个队列，而第一个参数 Choreographer.CALLBACK_TRAVERSAL 这个参数是用来区分队列的，可以理解成各个队列的 key 值。然后走了 scheduleFrameLocked() 方法这边，看看做了些什么：

```
 public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
  public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }
    
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                如果是延迟执行则使用Handler发送异步消息的时候执行
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```
- **scheduleFrameLocked**
  
    判断是否是在主线程，如果在主线程则直接执行scheduleVsyncLocked();，否则使用Handler切换线程并将该消息放入队列头，设置最高优先级保证第一时间Handler处理该消息
```
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            //USE_VSYNC 默认是Ture
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                //判断执行是否现在主线程
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    //如果不在主线程则使用Handler切换线程执行
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    //将消息发送到对头，可以理解为当前msg的优先级最高
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
    
    FrameHandler
    
     private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                ......
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                .......
            }
        }
    }
    
    void doScheduleVsync() {
        synchronized (mLock) {
            if (mFrameScheduled) {
                scheduleVsyncLocked();
            }
        }
    }
    
```
- **Choreographer#scheduleVsyncLocked()**
  
    最终执行DisplayEventReceiver的scheduleVsync，接收Vsync信号，并开始处理UI，


​    
>FrameDisplayEventReceiver继承自DisplayEventReceiver接收底层的VSync信号开始处理UI过程。
> VSync信号由SurfaceFlinger实现并定时发送。FrameDisplayEventReceiver收到信号后，调用onVsync方法组织消息发送到主线程处理。
> 这个消息主要内容就是run方法里面的doFrame了，这里mTimestampNanos是信号到来的时间参数。


```
 private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
    
    
 DisplayEventReceiver 
 public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
```
<p>也就是说，<code>onVsync()</code> 是底层会回调的，可以理解成每隔 16.6ms 一个帧信号来的时候，底层就会回调这个方法，当然前提是我们得先注册，这样底层才能找到我们 app 并回调。当这个方法被回调时，内部发起了一个 Message，注意看代码对这个 Message 设置了 callback 为 this，Handler 在处理消息时会先查看 Message 是否有 callback，有则优先交由 Message 的 callback 处理消息，没有的话再去看看Handler 有没有 callback，如果也没有才会交由 <code>handleMessage()</code> 这个方法执行。</p>
<p>这里这么做的原因，猜测可能 <code>onVsync()</code> 是由底层回调的，那么它就不是运行在我们 app 的主线程上，毕竟上层 app 对底层是隐藏的。但这个 <code>doFrame()</code> 是个 ui 操作，它需要在主线程中执行，所以才通过 Handler 切到主线程中。</p>


这就有点类似于观察者模式，或者说发布-订阅模式。既然上层 app 需要知道底层每隔 16.6ms 的帧信号事件，那么它就需要先注册监听才对，这样底层在发信号的时候，直接去找这些观察者通知它们就行了。

还有一点，scheduleVsync() 注册的监听应该只是监听下一个屏幕刷新信号的事件而已，而不是监听所有的屏幕刷新信号。比如说当前监听了第一帧的刷新信号事件，那么当第一帧的刷新信号来的时候，上层 app 就能接收到事件并作出反应。但如果还想监听第二帧的刷新信号，那么只能等上层 app 接收到第一帧的刷新信号之后再去监听下一帧。


```
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
     
            ......
            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```
- **Choreographer#doFrame*()*
    最终从Callback队列中取出我们之前放入绘制操作的Runable，并执行
```
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
         ...

        try {
            ....
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        if (DEBUG_FRAMES) {
            final long endNanos = System.nanoTime();
            Log.d(TAG, "Frame " + frame + ": Finished, took "
                    + (endNanos - startNanos) * 0.000001f + " ms, latency "
                    + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
        }
    }
    
      void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            ...
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
         ..
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
    
    
```
- **调用流程梳理**
  
     1、我们知道一个 View 发起刷新的操作时，最终是走到了 ViewRootImpl 的 scheduleTraversals() 里去，然后这个方法会将遍历绘制 View 树的操作 performTraversals() 封装到 Runnable 里，传给 Chorerographer，以当前的时间戳放进一个 mCallbackQueue 队列里，然后调用了 native 层的方法向底层注册监听下一个屏幕刷新信号事件。
    2、当下一个屏幕刷新信号发出的时候，如果我们 app 有对这个事件进行监听，那么底层它就会回调我们 app 层的 onVsync() 方法来通知。当 onVsync() 被回调时，会发一个 Message 到主线程，将后续的工作切到主线程来执行。
    3、切到主线程的工作就是去 mCallbackQueue 队列里根据时间戳将之前放进去的 Runnable 取出来执行，而这些 Runnable 有一个就是遍历绘制 View 树的操作 performTraversals()。在这次的遍历操作中，就会去绘制那些需要刷新的 View。
    
    5、所以说，当我们调用了 invalidate()，requestLayout()，等之类刷新界面的操作时，并不是马上就会执行这些刷新的操作，而是通过 ViewRootImpl 的 scheduleTraversals() 先向底层注册监听下一个屏幕刷新信号事件，然后等下一个屏幕刷新信号来的时候，才会去通过 performTraversals() 遍历绘制 View 树来执行这些刷新操作。

##### 三、疑问解答

- ==为什么会过滤一帧内重复的刷新请求==
  
    当我们调用了一次 scheduleTraversals()之后，直到下一个屏幕刷新信号来的时候，doTraversal() 被取出来执行。在这期间重复调用 scheduleTraversals() 都会被过滤掉的。那么为什么需要这样呢？
    
    其实，想想就能明白了。View 最终是怎么刷新的呢，就是在执行 performTraversals() 遍历绘制 View 树过程中层层遍历到需要刷新的 View，然后去绘制它的吧。既然是遍历，那么不管上一帧内有多少个 View 发起了刷新的请求，在这一次的遍历过程中全部都会去处理的吧。这也是我们从代码上看到的，每一个屏幕刷新信号来的时候，只会去执行一次 performTraversals()，因为只需遍历一遍，就能够刷新所有的 View 了。
    
    而 performTraversals() 会被执行的前提是调用了 scheduleTraversals() 来向底层注册监听了下一个屏幕刷新信号事件，所以在同一个 16.6ms 的一帧内，只需要第一个发起刷新请求的 View 来走一遍 scheduleTraversals() 干的事就可以了，其他不管还有多少 View 发起了刷新请求，没必要再去重复向底层注册监听下一个屏幕刷新信号事件了，反正只要有一次遍历绘制 View 树的操作就可以对它们进行刷新了。




```
ViewRootImpl.java

void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
           ....
        }
    }
void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
           ...
        }
    }
```
- ==为什么使用postSyncBarrier()---同步屏障消息==
  
    主线程同一时间只能处理一个 Message，这些 Message 就肯定有先后的问题，也就是说，当我们的 app 接收到屏幕刷新信号时，来不及第一时间就去执行刷新屏幕的操作，这样一来，即使我们将布局优化得很彻底，保证绘制当前 View 树不会超过 16ms，但如果不能第一时间优先处理绘制 View 的工作，那等 16.6 ms 过了，底层需要去切换下一帧的画面了，我们 app 却还没处理完，这样也照样会出现丢帧了吧。而且这种场景是非常有可能出现的吧，毕竟主线程需要处理的事肯定不仅仅是刷新屏幕的事而已，那么这个问题是怎么处理的呢？
    在逻辑走进 Choreographer 前会先往队列里发送一个同步屏障，而当 doTraversal() 被调用时才将同步屏障移除。这个同步屏障又涉及到消息机制了，不深入了，这里就只给出结论。

    <p>在逻辑走进 Choreographer 前会先往队列里发送一个同步屏障，而当 <code>doTraversal()</code> 被调用时才将同步屏障移除。这个同步屏障又涉及到消息机制了，不深入了，这里就只给出结论。</p>
    <p>这个同步屏障的作用可以理解成拦截同步消息的执行，主线程的 Looper 会一直循环调用 MessageQueue 的 <code>next()</code> 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。当 <code>next()</code> 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 <code>next()</code> 方法陷入阻塞状态。如果 <code>next()</code> 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除出队列，否则主线程就一直不会去处理同步屏幕后面的同步消息。</p>
    
    <p>而所有消息默认都是同步消息，只有手动设置了异步标志，这个消息才会是异步消息。另外，同步屏障消息只能由内部来发送，这个接口并没有公开给我们使用。</p>
    <p>最后，仔细看上面 Choreographer 里所有跟 message 有关的代码，你会发现，都手动设置了异步消息的标志，所以这些操作是不受到同步屏障影响的。这样做的原因可能就是为了尽可能保证上层 app 在接收到屏幕刷新信号时，可以在第一时间执行遍历绘制 View 树的工作。</p>
    
    <p>因为主线程中如果有太多消息要执行，而这些消息又是根据时间戳进行排序，如果不加一个同步屏障的话，那么遍历绘制 View 树的工作就可能被迫延迟执行，因为它也需要排队，那么就有可能出现当一帧都快结束的时候才开始计算屏幕数据，那即使这次的计算少于 16.6ms，也同样会造成丢帧现象。</p>
    <p>只能说，同步屏障是尽可能去做到，但并不能保证一定可以第一时间处理。因为，同步屏障是在 <code>scheduleTraversals()</code> 被调用时才发送到消息队列里的，也就是说，只有当某个 View 发起了刷新请求时，在这个时刻后面的同步消息才会被拦截掉。如果在 <code>scheduleTraversals()</code> 之前就发送到消息队列里的工作仍然会按顺序依次被取出来执行。</p>
```
ViewRootImpl.java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            ....
        }
    }
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

         ...
        }
    }
```

- Q1：==Android 每隔 16.6 ms 刷新一次屏幕到底指的是什么意思？是指每隔 16.6ms 调用 onDraw() 绘制一次么？==
- Q2：==如果界面一直保持没变的话，那么还会每隔 16.6ms 刷新一次屏幕么？==

答：我们常说的 Android 每隔 16.6 ms 刷新一次屏幕其实是指底层会以这个固定频率来切换每一帧的画面，而这个每一帧的画面数据就是我们 app 在接收到屏幕刷新信号之后去执行遍历绘制 View 树工作所计算出来的屏幕数据。而 app 并不是每隔 16.6ms 的屏幕刷新信号都可以接收到，只有当 app 向底层注册监听下一个屏幕刷新信号之后，才能接收到下一个屏幕刷新信号到来的通知。而只有当某个 View 发起了刷新请求时，app 才会去向底层注册监听下一个屏幕刷新信号。

也就是说，只有当界面有刷新的需要时，我们 app 才会在下一个屏幕刷新信号来时，遍历绘制 View 树来重新计算屏幕数据。如果界面没有刷新的需要，一直保持不变时，我们 app 就不会去接收每隔 16.6ms 的屏幕刷新信号事件了，但底层仍然会以这个固定频率来切换每一帧的画面，只是后面这些帧的画面都是相同的而已。


- Q3：==界面的显示其实就是一个 Activity 的 View 树里所有的 View 都进行测量、布局、绘制操作之后的结果呈现，那么如果这部分工作都完成后，屏幕会马上就刷新么？==

答：我们 app 只负责计算屏幕数据而已，接收到屏幕刷新信号就去计算，计算完毕就计算完毕了。至于屏幕的刷新，这些是由底层以固定的频率来切换屏幕每一帧的画面。所以即使屏幕数据都计算完毕，屏幕会不会马上刷新就取决于底层是否到了要切换下一帧画面的时机了。

- Q4：==网上都说避免丢帧的方法之一是保证每次绘制界面的操作要在 16.6ms 内完成，但如果这个 16.6ms 是一个固定的频率的话，请求绘制的操作在代码里被调用的时机是不确定的啊，那么如果某次用户点击屏幕导致的界面刷新操作是在某一个 16.6ms 帧快结束的时候，那么即使这次绘制操作小于 16.6 ms，按道理不也会造成丢帧么？这又该如何理解？==

答：之所以提了这个问题，是因为之前是以为如果某个 View 发起了刷新请求，比如调用了 invalidte()，那么它的重绘工作就马上开始执行了。梳理完屏幕刷新机制后就清楚了，代码里调用了某个 View 发起的刷新请求，这个重绘工作并不会马上就开始，而是需要等到下一个屏幕刷新信号来的时候才开始，所以现在回过头来看这些图就清楚多了。

- Q5：==主线程耗时的操作会导致丢帧，但是耗时的操作为什么会导致丢帧？它是如何导致丢帧发生的？==

答：造成丢帧大体上有两类原因，一是遍历绘制 View 树计算屏幕数据的时间超过了 16.6ms；二是，主线程一直在处理其他耗时的消息，导致遍历绘制 View 树的工作迟迟不能开始，从而超过了 16.6 ms 底层切换下一帧画面的时机。

第一个原因就是我们写的布局有问题了，需要进行优化了。而第二个原因则是我们常说的避免在主线程中做耗时的任务。

针对第二个原因，系统已经引入了同步屏障消息的机制，尽可能的保证遍历绘制 View 树的工作能够及时进行，但仍没办法完全避免，所以我们还是得尽可能避免主线程耗时工作。

其实第二个原因，可以拿出来细讲的，比如有这种情况， message 不怎么耗时，但数量太多，这同样可能会造成丢帧。如果有使用一些图片框架的，它内部下载图片都是开线程去下载，但当下载完成后需要把图片加载到绑定的 view 上，这个工作就是发了一个 message 切到主线程来做，如果一个界面这种 view 特别多的话，队列里就会有非常多的 message，虽然每个都 message 并不怎么耗时，但经不起量多啊。

#### 四、总结

1.界面上任何一个 View 的刷新请求最终都会走到 ViewRootImpl 中的 scheduleTraversals() 里来安排一次遍历绘制 View 树的任务；

2.scheduleTraversals() 会先过滤掉同一帧内的重复调用，在同一帧内只需要安排一次遍历绘制 View 树的任务即可，这个任务会在下一个屏幕刷新信号到来时调用 performTraversals() 遍历 View 树，遍历过程中会将所有需要刷新的 View 进行重绘；

3.接着 scheduleTraversals() 会往主线程的消息队列中发送一个同步屏障，拦截这个时刻之后所有的同步消息的执行，但不会拦截异步消息，以此来尽可能的保证当接收到屏幕刷新信号时可以尽可能第一时间处理遍历绘制 View 树的工作；

4.发完同步屏障后 scheduleTraversals() 才会开始安排一个遍历绘制 View 树的操作，作法是把 performTraversals() 封装到 Runnable 里面，然后调用 Choreographer 的 postCallback() 方法；

5.postCallback() 方法会先将这个 Runnable 任务以当前时间戳放进一个待执行的队列里，然后如果当前是在主线程就会直接调用一个native 层方法，如果不是在主线程，会发一个最高优先级的 message 到主线程，让主线程第一时间调用这个 native 层的方法；

6.native 层的这个方法是用来向底层注册监听下一个屏幕刷新信号，当下一个屏幕刷新信号发出时，底层就会回调 Choreographer 的onVsync() 方法来通知上层 app；

7.onVsync() 方法被回调时，会往主线程的消息队列中发送一个执行 doFrame() 方法的消息，这个消息是异步消息，所以不会被同步屏障拦截住；

8.doFrame() 方法会去取出之前放进待执行队列里的任务来执行，取出来的这个任务实际上是 ViewRootImpl 的 doTraversal() 操作；

9.上述第4步到第8步涉及到的消息都手动设置成了异步消息，所以不会受到同步屏障的拦截；

10.doTraversal() 方法会先移除主线程的同步屏障，然后调用 performTraversals() 开始根据当前状态判断是否需要执行performMeasure() 测量、perfromLayout() 布局、performDraw() 绘制流程，在这几个流程中都会去遍历 View 树来刷新需要更新的View；



![image](https://note.youdao.com/yws/api/personal/file/EF8B8106A1524F09977E052C5919CB02?method=getImage&version=9412&cstk=qPiLh-_D)