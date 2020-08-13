<font size=3>

##### 目录
![image](https://note.youdao.com/yws/api/personal/file/3D72883D4EF54B05B6EBEE2A57849595?method=download&shareKey=70bba33625c513ed586217f5b219727a)

##### Handler概念
    Handler主要用于发送和处理消息和Runable对象。处理异步操作
##### 源码解析
 - **Handler的创建**
    
Handler共有7个构造函数，如下：

1、无参构造函数，内部调用【6】构造函数  
```
  public Handler() {
        this(null, false);
    }
```
2、参数:只有Callback的构造函数，内部调用【6】构造函数 
```
 public Handler(Callback callback) {
        this(callback, false);
    }
```
3、参数：只有async的构造函数，被系统隐藏，我们不能调用。内部调用【6】构造函数 

```
  public Handler(boolean async) {
        this(null, async);
    }
```

4、参数:只有一个Looper的构造函数，内部调用

```
 public Handler(Looper looper) {
        this(looper, null, false);
    }
```
5、参数：有Looper和Callback的构造函数，内部调用

```
public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }
```
6、参数：Callback和async的构造函数，被系统隐藏，我们不能调用
> Callback:用于处理消息的接口。
  async：是否是异步处理消息，默认为false，我创建Handler时，系统为我们指定为false 
  
    执行步骤：
    1、调用 Looper.myLooper()，获取Looper。myLooper()方法实际使用ThreadLocal.get()方法，获取当前线程指定的Looper。

    2、从获取的Looper中获取消息队列，MessageQueue

    3、初始化赋值 mCallback和 mAsynchronous

```
  public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
> Callback接口源码
```
public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
7、参数：Looper、Callback、async 构造函数。
主要就是初始化赋值

```
public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
- **ThreadLocal**
1. 简介
>  ThreadLocal只有用于线程内的数据存储，通过他可以在指定的线程内存储数据，数据存储以后，其他线程无法获取到该数据，只有该线程可以获取到存储的数据

> 在系统内，虽然使用的是同一个ThreadLocal对象，但是通过TheadLocal获取的数据不一样，例如Looper

> 一般线程，某些数据是以线程为作用于，并且不同线程用于不同的数据副本的时候，就可以考虑使用ThreadLocal

2.ThreadLocal的get()方法
>执行流程
    
    1、获取当前线程
    2、获取当前线程的TheadLocalMap
    3、如果TheadLocalMap不为空，则以ThreaLocal当前实例作为Key去取值，然后返回
    4、如果ThreaLocalMap为空，或者取值为空，创建ThreadLocal或者设置默认值
```
public T get() {
    //1. 获取当前的线程
    Thread t = Thread.currentThread();
    //2. 以当前线程为参数，获取一个 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //3. map 不为空，则以当前 ThreadLocal 对象实例作为key值，去map中取值，有找到直接返回
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    //4. map 为空或者在map中取不到值，那么走这里，返回默认初始值
    return setInitialValue();
}
```

- **Looper初始化/Looper.perpare()**
> 

    1、实例化MessageQueue
    2、获取当前线程
```
 private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

>  
    1、Handler初始化获取，当在主线程定义Handler时，通过Looper.myLooper()获取主线程的Looper。
    2、在子线程内调用Looper.prepare()方法初始化，
    Looper.prepare()实际是new一个Looper之后通过ThreadLoca的set方法，以ThreadLoca当前实例为key放入到线程内的ThreadLocalMap中
    如果Looper已经被初始化，则抛出异常
    RuntimeException("Only one Looper may be created per thread")

总结：

1、所以在主线程创建Handler时，我们不需要实例Looper，因为在ActivityThread主线程中已经初始化了Looper，而之后我们在主线程使用的Looper都是同一个Looper

2、在子线程实例Handler时，需要我们在子线程内调用Looper的prepare()方法实例化Looper


- **Handler发送消息**
>
    实际发送一个Message主要是调用MessageQueue的enqueueMessage方法将消息放入队列
    
```
 //调用sendMessageDelayed(Message msg, long delayMillis)
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
 // 调用sendMessageAtTime(Message msg, long uptimeMillis)  
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    //调用enqueueMessage(queue, msg, uptimeMillis)
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    //调用MessageQueue的enqueueMessage(msg, uptimeMillis);将消息放入队列
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        //设置target为当前Handler实例
        msg.target = this;
        //如果是异步处理,则设置Message为同步，这个 mAsynchronous 标识是我们构造 Handler 的时候传递的参数，默认为  false
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        最后就是真正的进队方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }    
```
MessageQueue将消息放入队列
>消息队列是使用链表作为数据的存储结构。是可以插队的，即不存在发送了延时消息不会阻塞消息队列。 
MessageQueue中存储一个头结点的Message ，而每个Message中都存储一个Message，这样根据消息的执行顺序设置Message的next形成了消息队列
```
  boolean enqueueMessage(Message msg, long when) {
        //如果target为空则抛出异常。target实际是发送消息的Handler实例，之后被用来处理消息
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        //同步锁
        synchronized (this) {
            //如果MessageQueue退出，则释放消息
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
            
            msg.markInUse();
            ///赋值调用时间
            msg.when = when;
            //获取头结点
            Message p = mMessages;
            boolean needWake;
            //  //队列中没有消息 或者 时间为0 或者 比头结点的时间早
            //插入到头结点中
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                /类似插入排序，找到合适的位置,找到一个Messgae的next为空或者next的执行实际大于当前message的Message
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                // 将上一个Message的next设置个当前Message。将上一个Message的next设置为当前Message
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

         
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

- **Looper循环消息**
> 
    1、loop首先判断Looper是否为空，如果为空则抛出异常   RuntimeException("No Looper; Looper.prepare() wasn't called on this thread."); 
    这就是我们创建非主线程的 Handler为什么要调用Looper.prepare()的原因。
    而主线程中会在ActivityThread.main() 方法里面调用了 prepare 方法，
    所以我们创建默认（主线程）的 Handler 不需要额外创建 Looper 
    
    2、loop内是一个死循环，取出消息如果为空则退出循环，如果不为空则执行msg的响应的Handler的dispatchMessage(msg)处理消息。for循环内通过queue.next()取出消息。
```
// Looper.java ，省略部分代码
loop(){
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block  , 从队列取出一个msg
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        msg.target.dispatchMessage(msg); //Handler处理消息
        msg.recycleUnchecked();  //回收msg
    }
}
```
    MessageQueue的next()如下


```
//MessageQueue.java ,删减部分代码
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        //如果队列已经停止了（quit or dispose）
        return null;
    }
    for (;;) {
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();   //获取当前时间
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                 //msg == target 的情况只能是屏障消息，即调用postSyncBarrier()方法
                //如果存在屏障，停止同步消息，异步消息还可以执行
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());  //找出异步消息，如果有的话
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当前消息还没准备好(时间没到)
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 消息已准备，可以取出
                    if (prevMsg != null) {
                        //有屏障，prevMsg 为异步消息 msg 的前一节点，相当于拿出 msg ,链接前后节点
                        prevMsg.next = msg.next;
                    } else {
                        //没有屏障,msg 即头节点，将 mMessages 设为新的头结点
                        mMessages = msg.next;
                    }
                    msg.next = null;  //断开即将执行的 msg
                    msg.markInUse(); //标记为使用状态
                    return msg;  //返回取出的消息，交给Looper处理
                }
            } 
            // Process the quit message now that all pending messages have been ha
            if (mQuitting) {
                //队列已经退出
                dispose();
                return null;  //返回null后Looper.loop()方法也会结束循环
            }
    }
}
```
异步消息并不会立刻执行，而是根据时间，完全跟同步消息一样的顺序插入队列中。异步消息与同步消息唯一的区别就是当有消息屏障时，异步消息还可以执行，而同步消息则不行

总结：发送消息的所在的线程，不处理消息，只有Looper将消息取出后处理，所以在无论Handler发送消息所在在哪个线程，最终处理消息是在Looper.loop()所在的线程
    
- **Handler处理消息**
>     
    Looper.loop()取出消息后，调用msg.target.dispatchMessage(msg)方法处理消息，msg.target就是发送消息的Handler
    
    消息处理：
    1、消息的Runable不为空时，使用Runable处理消息
    2、Handler的CallBack不为空时，使用CallBack处理消息
    3、Handler的handleMessage()处理消息

```
public void dispatchMessage(Message msg) {
        //如果消息的CallBack不为空，则调用 handleCallback(msg);.Callback就是Runable
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            // 如果Handler的CallBack不为空，则调用Callback处理下消息，如果mCallback.handleMessage(msg)返回true，则处理结束，否则，将继续执行Handler的handleMessage(msg);
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //最后上面情况都不满足时，调用Handler的handleMessage(msg)处理消息
            handleMessage(msg);
        }
    }
    
    
private static void handleCallback(Message message) {
        message.callback.run();
}
```
>总结：
    
    如果 msg 没有 callback 的话，那么将会判断 mCallback 是否为空，
    这个 mCallback 就是构造方法种传递的那个 Callback ,如果 
    mCallback为空,那么就调用 Handler 的 handleMessage(msg) 
    方法，否则就调用 mCallback.handleMessage(msg) 方法，然后根据 
    mCallback.handleMessage(msg)的返回值判断是否拦截消息，如果拦截(返回 
    true)，则结束，否则还会调用 Handler#handleMessage(msg)方法。
    也就是说：Callback.handleMessage() 
    的优先级比Handler.handleMessage()要高 
    。如果存在Callback,并且Callback#handleMessage() 返回了 true 
    ,那么Handler#handleMessage()将不会调用。
- **Handler.post(Runnable)**
> 无论消息发送Runable还是msg，最终消息的载体都是Message，Runable只是用来处理消息，而最终使用Runable的处理消息使用handleCallback(Message message)，而该方法内部调用Runable的run方法，并没有新建线程
```
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    //创建消息，将Runable放入现在Message内，用来处理消息
 private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```
    
- **消息创建**
>
    消息的创建分为：
    New消息
    Message.obtain()
    Handler.obtainMessage() 
    而Handler.obtainMessage() 最终也是调用的 Message.obtain(Handler)，所传参数会将Handler本身赋值给Message的Target用来处理消息
    接下来分析Message.obtain()
    

```
//Message.java

private static Message sPool;

public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```
>
    obtain() 方法会在以 sPool作为头结点的消息池（链表）中遍历,如果找到，那么取出来，并置为非使用状态，然后返回，如果消息池为空，则新建一个消息
- **消息池的获取**
>这是一个回收的方法，方法体内将 Message 对象的各种参数清空，如果消息池的数量小于最大数量(50)的话，就当前消息插入缓存池的头结点中
```
//Message.java

private static final int MAX_POOL_SIZE = 50;

void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
>上方法的其中一个调用在 Looper.looperzhong 使用，调用时机是在 Handler 处理事件之后，既然是 Handler 处理后就会回收：源码如下
    
```
// Looper.java ，省略部分代码
loop(){
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block  , 从队列取出一个msg
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        msg.target.dispatchMessage(msg); //Handler处理消息
        ...
        msg.recycleUnchecked();  //回收msg
    }
}
```
>
    总结：
        所以在消息处理方中：
        Handler的Callback接口中
        Runable的run方法中
        Handler的handlerMessage
        消息处理完成后系统就会回收消息，所以不要在消息处理方法中异步的处理消息，以为在loop方法中执行了处理方法后，就执行回收。如果真要在异步中使用，那么可以创建一个新的 Message 对象，并将值赋值过去
**总而言之，因为 Handler 机制在整个 Android 系统中使用太频繁，所以 Android 就采用了一个缓存策略。就是 Message 里面会缓存一个静态的消息池，当消息被处理或者移除的时候就会被回收到消息池，所以推荐使用 Message.obtain()来获取消息对象。**
        
        
- **总结流程**
  ![image](https://note.youdao.com/yws/api/personal/file/1F00B87DAB5142789A9EB1B5D5E7BFF8?method=download&shareKey=40034fa393933967298c810f74c2eab1)

>把整个Handler机制比作一个流水线的话，那么 Handler 就是工人，可以在不同线程传递 Message到传送带(MessageQueue)，而传送带是被马达（Looper）运输的，马达又是一开始就运行了(Looper.loop())，并且只会在一开始的线程，所以无论哪个工人(Handler)在哪里(任意线程)传递产品(Message)，都只会在一条传送带(MessageQueue)上被唯一的马达(Looper)运送到终点处理，即 Message 只会在调用 Looper.loop() 的线程被处理。

---

##### 问题总结
-   **Handler为什么能造成内存泄漏**
> 
    Handler本身并不会造成内存泄漏，而造成内存泄漏的主要原因是匿名内部类的引用，匿名内部类的隐形的持有外部类的引用    (如果不持有引用怎么可以使用外部类的变量方法呢？)
    所以当匿名内部类的生命周期较长，例如正在跑一个耗时的线程，而碰巧Activity不外类此时退出，需要回收，而内部类让然引用外部类，导致内存不能回收。
    
    Handler发生内存泄漏的原因分析
    Handler持有Activity，Message持有Handler、Message在MessageQueue中、
    MessageQueue在Looper中、而ThreadLoca静态持有Looper。因为 sThreadLocal 是方法区常量，所以不会被回收，即 sThreadLocal 间接的持有了 Activity 的引用，当 Handler 发送的消息还没有被处理完毕时，比如延时消息，而 Activity 又被用户返回了，即 onDestroy() 后，系统想要对 Activity 对象进行回收，但是发现还有引用链存在，回收不了，就造成了内存泄漏

![image](https://note.youdao.com/yws/api/personal/file/9F67EDBCCDFB4AC383997D5FE8D3AF73?method=download&shareKey=cce73701fc297f6f525eebd009c5baed)

##### 怎么防止 Handler 内存泄漏
    
    1、把ThreadLocal与Activity的引用链断开
    
        最简单的方法就是在 onPause()中使用 Handler 的 removeCallbacksAndMessages(null)方法清除所有消息及回调。就可以把引用链断开了。
        Android 源码中这种方式也很常见，不在 onDestroy()里面调用主要是 onDestroy() 方法不能保证每次都能执行到
    
    2、第二种方法就是使用静态类加弱引用的方式
        因为静态类不会持有外部类的引用，所以需要传一个 Activity 过来,并且使用一个弱引用来引用 Activity 的实例，
        弱引用在 gc的时候会被回收，所以也就相当于把强引用链给断了，自然也就没有内存泄漏了。
##### Loop.loop() 为什么不会造成应用卡死？
        loop() 方法是一个死循环，那么肯定会占用大量的 cpu 而导致应用卡顿，甚至说 ANR 。
        但是 Android 中即使使用大量的 Looper，也不会造成这种问题，问什么呢？
        由于这个问题涉及到的知识比较深，主要是通过 Linux 的 epoll 机制实现的，这里需要 Linux 、 jni 等知识，我等菜鸟就不分析了，推荐一些相关文章：

>Android中为什么主线程不会因为Looper.loop()里的死循环卡死？
    [https://www.zhihu.com/question/34652589](https://www.zhihu.com/question/34652589)



##### 总结
    1. Handler 的回调方法是在 Looper.loop()所调用的线程进行的；
    2. Handler 的创建需要先调用 Looper.prepare() ，然后再手动调用 loop()方法开启循环；
    3. App 启动时会在ActivityThread.main()方法中创建主线程的 Looper ,并开启循环，所以主线程使用 Handler 不用调用第2点的逻辑；
    4. 延时消息并不会阻塞消息队列；
    5. 异步消息不会马上执行，插入队列的方式跟同步消息一样，唯一的区别是当有消息屏障时，异步消息可以继续执行，同步消息则不行；
    6. Callback.handleMessage() 的优先级比 Handler.handleMessage()要高*
    7. Handler.post(Runnable)传递的 Runnale 对象并不会在新的线程执行；
    8. Message 的创建推荐使用 Message.obtain() 来获取，内部采用缓存消息池实现；
    9. 不要在 handleMessage()中对消息进行异步处理；
    10. 可以通过removeCallbacksAndMessages(null)或者静态类加弱引用的方式防止内存泄漏；
    11. Looper.loop()不会造成应用卡死，里面使用了 Linux 的 epoll 机制。

### 参考资料
[Handler全家桶之 —— Handler 源码解析](https://www.jianshu.com/p/13c8a66d3b5c)

[深入理解 ThreadLocal (这些细节不应忽略)](https://www.jianshu.com/p/56f64e3c1b6c)