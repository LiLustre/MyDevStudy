#### 概念
    Handler机制主要用于线程间发送、处理消息runable。可用于处理异步操作
#### 组成部分
    由 Handle、MessageQuene、Message、Looper 四部分组成
    
    Handle：发送、处理消息runable
    Message：消息的载体
    MessageQuene：用于存储消息，属于消息队列，际上是通过一个单链表的数据结构来维护消息列表，MessageQuene 并没有使用集合的方式存储消息，而是内部只存储一个Msssage，而这个Message内部 存储下一个Message 方式维护消息队列
    Looper：循环消息队列，按分发机制将消息分发给目标处理者。
#### 消息机制运行流程

    1、Handler使用send或者post方法发送消息，它的内部通过MessageQuene.enqueueMessage，向消息队列中添加消息，无论Handler发送msg 还是runable 其实最终都是message，发送runable只是将message 的callback设置为runable。
        Handler.post(Runnable)，处理消息只是通过调用runable的run方法，并没有新建线程
    2、当使用Looper.loop()开始循环后，会不断的从消息队列取出消息。
    3、然后调用Message的目标Handler的dispatchMessage方法传递消息
    4、Handler收到消息，并处理消息
        处理消息的方法：优先级从高到低
                        1、消息的Runable不为空时，使用Runable处理消息
                        2、Handler的CallBack不为空时，使用CallBack处理消息
                        3、Handler的handleMessage()处理消息
##### 问题总结
- Handler并不是在创建实例的线程中处理消息，而是在绑定的Looper线程中处理消息的
-   **Handler为什么能造成内存泄漏**

    
```
    Handler本身并不会造成内存泄漏，而造成内存泄漏的主要原因是匿名内部类的引用，匿名内部类的隐形的持有外部类的引用    (如果不持有引用怎么可以使用外部类的变量方法呢？)
        所以当匿名内部类的生命周期较长，例如正在跑一个耗时的线程，而碰巧Activity不外类此时退出，需要回收，而内部类让然引用外部类，导致内存不能回收。
        
    Handler发生内存泄漏的原因分析
    Handler持有Activity，Message持有Handler、Message在MessageQueue中、
    MessageQueue在Looper中、而ThreadLoca静态持有Looper。因为 sThreadLocal 是方法区常量，所以不会被回收，即 sThreadLocal 间接的持有了 Activity 的引用，
    当 Handler 发送的消息还没有被处理完毕时，比如延时消息，
    而 Activity 又被用户返回了，即 onDestroy() 后，系统想要对 Activity 对象进行回收，但是发现还有引用链存在，回收不了，就造成了内存泄漏
```
- **怎么防止 Handler 内存泄漏**
    
    1、把ThreadLocal与Activity的引用链断开
    
        最简单的方法就是在 onPause()中使用 Handler 的 removeCallbacksAndMessages(null)方法清除所有消息及回调。就可以把引用链断开了。Android 源码中这种方式也很常见，不在 onDestroy()里面调用主要是 onDestroy() 方法不能保证每次都能执行到
    
    2、第二种方法就是使用静态类加弱引用的方式
        
        因为静态类不会持有外部类的引用，所以需要传一个 Activity 过来,并且使用一个弱引用来引用 Activity 的实例，
        弱引用在 gc的时候会被回收，所以也就相当于把强引用链给断了，自然也就没有内存泄漏了。
