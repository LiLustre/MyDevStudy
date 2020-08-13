#### 介绍
    1、定义：一个Android已经封装好的轻量级异步类
    2、作用：
            1、实现多线程
            2、异步通信和消息传递，在工作线程和主线程直接通信，在工作线程执行耗时操作，在主线程执行UI操作
    3、优点：方便实现异步通信
    4、通过集成Thread，快速创建带有Looper的信工作线程，通过内部封装Handler类，快速创建Handler与其他线程通信
    

#### 使用
    1、创建HandlerThread实例对象
    2、启动线程
    3、创建工作线程Handler使用HandlerThread的Looper为参数 & 复写handleMessage（）
    4、使用工作线程Handler向工作线程的消息队列发送消息
    5、结束线程，即停止线程的消息循环
    
    
```
// 步骤1：创建HandlerThread实例对象
// 传入参数 = 线程名字，作用 = 标记该线程
   HandlerThread mHandlerThread = new HandlerThread("handlerThread");

// 步骤2：启动线程
   mHandlerThread.start();

// 步骤3：创建工作线程Handler & 复写handleMessage（）
// 作用：关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
// 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
  Handler workHandler = new Handler( handlerThread.getLooper() ) {
            @Override
            public boolean handleMessage(Message msg) {
                ...//消息处理
                return true;
            }
        });

// 步骤4：使用工作线程Handler向工作线程的消息队列发送消息
// 在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
  // a. 定义要发送的消息
  Message msg = Message.obtain();
  msg.what = 2; //消息的标识
  msg.obj = "B"; // 消息的存放
  // b. 通过Handler发送消息到其绑定的消息队列
  workHandler.sendMessage(msg);

// 步骤5：结束线程，即停止线程的消息循环，HandlerThread的quit方法就是调用Looper的quit()方法;
  mHandlerThread.quit();
```

#### 注意点
    内存泄漏 & 连续发送消息
    1、不要使用匿名内部类Handler，使用静态内部类和若应用的方式持有Activity
    2、连续发送消息，并不会并行，按照Handler原理消息会存储到消息队列中，然后Looper循环，一条条执行