<font size=3>

##### 目录

#### 二、简介

- **概念**
> AsyncTask 是Android已经封装好的轻量级的异步操作的抽象类，即使用时需要自己实现
- **作用**

>1.实现多线程，在子线程执行耗时操作

>2.异步通信、消息传递,即实现工作线程与主线程（UI）线程之间通信【使用上来说实现将工作线程执行结果返会给UI线程，实现UI线程的UI更新】，
- **优点**
> 1.**方便的实现异步通信**【不需要我们自己手动去使用Thread+Handler 的方式实现异步】

>2.**节省资源**，内部使用线程线程池来复用线程和缓存线程，减小了频繁创建和销毁线程的开销

---
#### 三、类定义

- **类中参数为3种泛型类型**

>     a. Params：开始异步任务执行时传入的参数类型，对应excute（）中传递的参数
>     
>      b. Progress：异步任务执行过程中，返回下载进度值的类型
>     
>      c. Result：异步任务执行完成后，返回的结果类型，与doInBackground()的返回值类型保持一致

- **核心方法**
   
- 方法介绍
   

核心方法 |作用|调用时机|说明
---|---|---|---|
execute(Params... params) | 触发执行异步线程任务|手动调用|1、必须在UI线程中调用 （内部调用executeOnExecutor方法 被@MainThread注解）2、运行在主线程
onPreExecute()| 执行线程前的操作（根据需求重写）|（由系统调用）执行线程前自动调用，在我们调用execute后就会调用|用于界面的初始化操作等
doInBackground| 接受输入参数，执行耗时操作（必须重新的方法），返回执行结果|（由系统调用）执行线程任务时自动调用|不能更新UI，执行过程中可以使用publishProgress()方法更新进度信息
onProgressUpdate(Progress value)| 在主线程显示执行任务的进度|（由系统调用）我们调用publishProgress()是会自动调用
onPostExecute| 接收线程任务的执行结果,将执行结果反映在UI上|（由系统调用）线程任务结束后自动调用
onCancelled| 将异步任务设置为取消状态（并不是真正的取消任务）|异步任务被取消时自动调用|该方法被调用时，onPostExecute将不会被调用
   
   - 执行顺序
    

类型 | 顺序
---|---
基本使用| execute()->onPreExecute()->doInBackground()->onPostExecute()
显示进度|execute()->onPreExecute()->doInBackground()-publishProgress()->onProgressUpdate()->onPostExecute()
执行线程任务时被取消|execute()->onPreExecute()->doInBackground()->onCancelled
#### 四、基本使用
    1、创建 AsyncTask 子类 & 根据需求实现核心方法
    2、创建 AsyncTask子类的实例对象（即 任务实例）
    3、手动调用execute(（）从而执行异步线程任务
    
```
/**
  * 步骤1：创建AsyncTask子类
  * 注： 
  *   a. 继承AsyncTask类
  *   b. 为3个泛型参数指定类型；若不使用，可用java.lang.Void类型代替
  *   c. 根据需求，在AsyncTask子类内实现核心方法
  */

  private class MyTask extends AsyncTask<Params, Progress, Result> {

        ....

      // 方法1：onPreExecute（）
      // 作用：执行 线程任务前的操作
      // 注：根据需求复写
      @Override
      protected void onPreExecute() {
           ...
        }

      // 方法2：doInBackground（）
      // 作用：接收输入参数、执行任务中的耗时操作、返回 线程任务执行的结果
      // 注：必须复写，从而自定义线程任务
      @Override
      protected String doInBackground(String... params) {

            ...// 自定义的线程任务

            // 可调用publishProgress（）显示进度, 之后将执行onProgressUpdate（）
             publishProgress(count);
              
         }

      // 方法3：onProgressUpdate（）
      // 作用：在主线程 显示线程任务执行的进度
      // 注：根据需求复写
      @Override
      protected void onProgressUpdate(Integer... progresses) {
            ...

        }

      // 方法4：onPostExecute（）
      // 作用：接收线程任务执行结果、将执行结果显示到UI组件
      @Override
      protected void onPostExecute(String result) {

         ...// UI操作

        }

      // 方法5：onCancelled()
      // 作用：将异步任务设置为：取消状态
      @Override
        protected void onCancelled() {
        ...
        }
  }

/**
  * 步骤2：创建AsyncTask子类的实例对象（即 任务实例）
  * 注：AsyncTask子类的实例必须在UI线程中创建
  */
  MyTask mTask = new MyTask();

/**
  * 步骤3：手动调用execute(Params... params) 从而执行异步线程任务
  * 注：
  *    a. 必须在UI线程中调用
  *    b. 同一个AsyncTask实例对象只能执行1次，若执行第2次将会抛出异常
  *    c. 执行任务中，系统会自动调用AsyncTask的一系列方法：onPreExecute() 、doInBackground()、onProgressUpdate() 、onPostExecute() 
  *    d. 不能手动调用上述方法
  */
  mTask.execute()；

```

#### 五、基本原理

- **线程介绍**
 
![image](https://note.youdao.com/yws/api/personal/file/032CCF3F9F344617821F77488EC93F5C?method=download&shareKey=c96c10efecf30b914a2fea58439108fc)

- **线程与进程的区别**

类型  | 定义| 作用|资源拥有性|地址空间|系统开销|通信
---|---|---|---|---|---|---|---
进程  | 资源拥有的基本单位|使多个程序并发运行，提高系统的资源的利用率和吞吐量|拥有资源|进程的地址空间相互独立|大（创建、回收系统资源等）|复杂（需要进程同步和互斥手段辅助）
线程 | 操作系统中，调度的最小单元|减小程序并发执行是的时间和空间开销,提高程序的并发性能|拥有很少的基本资源（但可以访问隶属进程的系统资源）|同一进程的各个线程间共享进程的资源|小（保存和恢复少量的寄存器）|简单（可直接读写进程数据进行通信）

- **原理介绍**
> AsyncTask主要由2个线程池和Handler组合实现，Handler主要完成消息的传递和异步通信，2个线程池分别完成任务的调度和线程执行

#### 六、源码分析
- **创建AsyncTask**
>
    主要操作：
    1、初始化异步通信的Handler
    
    2、初始化WorkerRunable类的实例，并实现call()方法
    
    3、初始化FutureTask类 的实例对象 并实现了 done()
    
    WorkerRunnable：真正执行耗时的操作，call()函数内执行doInBackground()
    
    FutureTask:检查WorkerRunnable的执行结果，如果WorkerRunaable为执行，将未执行的结果的发送的主线程
```
  public AsyncTask(@Nullable Looper callbackLooper) {
        //初始化Handler,
        //1、如果Looper是Null或者 是MainLooper，则Handler是主线的Handler
        //2、如果传入Looper是其他子线程的，则Handler是其他子线程的
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
        //初始化mWorker，WorkerRunnable实现Callable的抽象类
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                //设置线程的调度标识
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    //设置线程的优先级
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //执行doInBackground（）
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    //抛出异常时，设置取消表示
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //最后将返回结果发送到主线程
                    postResult(result);
                }
                return result;
            }
        };
        //初始化mFuture，参数是上面初始化的mWorker
        mFuture = new FutureTask<Result>(mWorker) {
            // done（）简介：FutureTask内的Callable执行完后的调用方法
            // 在这里 是mWorker执行完后调用done()
            // 作用：复查任务的调用、将未被调用的任务的结果通过InternalHandler传递到UI线程
            @Override
            protected void done() {
                try {
                    //检查将没被调用的Result也一并发出
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    //若 发生异常,则将发出null
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
- **WorkerRunnable类的构造函数**
>  Callable与Runable的区别：Callable存在返回值，其泛型是返回值类型
```
  private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        // 此处的Callable也是任务；
        // 与Runnable的区别：Callable<T>存在返回值 = 其泛型
        Params[] mParams;
    }
```
- **FutureTask类的构造函数**
> FutureTask是包装任务的包装类

>注：内部包含Callable<T> 、增加了一些状态标识 & 操作Callable<T>的接口
```
  public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;      
    }
    // 回到调用原处

```
- **postResultIfNotInvoked**
```
 private void postResultIfNotInvoked()(Result result) {
        // 取得任务标记
        final boolean wasTaskInvoked = mTaskInvoked.get();
        // 若任务无被执行,将未被调用的任务的结果通过InternalHandler传递到UI线程
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```
- **手动调用execute(Params... params)**
>
    @MainThread：这个方法必须在主线线程（UI线程）调用
    
```
 @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        //如果任务没有处于未被执行的状态，根据状态抛出异常，
        // 这就是一个AsyncTask的实例只能调用一次execute
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
        //设置正在运行的状态
        mStatus = Status.RUNNING;
        //调用onPreExecute
        onPreExecute();
        //将执行参数设置给mWorker
        mWorker.mParams = params;
        //线程池执行mFuture（FutureTask）
        exec.execute(mFuture);

        return this;
    }
```
- **exec.execute(mFuture)**
> 

    SerialExecutor属于任务队列 线程池类，是AsyncTask的静态内部类
    
    ArrayDeque： ArrayDeque 被成为双端队列（即从两端添加或移除元素）， ArrayDeque是Deque接口的一种具体实现，是依赖于可变数组来实现的。ArrayDeque 没有容量限制，可根据需求自动进行扩容。ArrayDeque不支持值为 null 的元素

    FutureTask是Runable的子类
    
    操作：
    -执行任务前，通过SerialExecutor将任务按顺序放入到队列中（通过同步锁 修饰execute（）从而保证AsyncTask中的任务是串行执行的）
    -之后的线程任务执行是 通过任务线程池类（THREAD_POOL_EXECUTOR） 进行的。
    
    
```
 private static class SerialExecutor implements Executor {
        // // SerialExecutor 内部维持了1个双向队列；
        //容量根据元素数量调节
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;
        // execute（）被同步锁synchronized修饰
        // 即说明：通过锁使得该队列保证AsyncTask中的任务是串行执行的
        //即 多个任务需1个个加到该队列中；然后 执行完队列头部的再执行下一个，
        public synchronized void execute(final Runnable r) {
            // // 将实例化后的FutureTask类 的实例对象传入
            // 即相当于：向队列中加入一个新的任务
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        //运行FutureTask
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            // // 若当前无任务执行，则去队列中取出1个执行
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            // 1. 取出队列头部任务
            if ((mActive = mTasks.poll()) != null) {
                //执行取出的队列头部任务
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```


```
    用于真正执行任务的线程池
   /**
  * 源码分析：THREAD_POOL_EXECUTOR.execute（）
  * 说明：
  *     a. THREAD_POOL_EXECUTOR实际上是1个已配置好的可执行并行任务的线程池
  *     b. 调用THREAD_POOL_EXECUTOR.execute（）实际上是调用线程池的execute()去执行具体耗时任务
  *     c. 而该耗时任务则是步骤2中初始化WorkerRunnable实例对象时复写的call（）
  * 注：下面先看任务执行线程池的线程配置过程，看完后请回到步骤2中的源码分析call（）
  */

    // 步骤1：参数设置
        //获得当前CPU的核心数
        private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
        //设置线程池的核心线程数2-4之间,但是取决于CPU核数
        private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
        //设置线程池的最大线程数为 CPU核数*2+1
        private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
        //设置线程池空闲线程存活时间30s
        private static final int KEEP_ALIVE_SECONDS = 30;

        //初始化线程工厂
        private static final ThreadFactory sThreadFactory = new ThreadFactory() {
            private final AtomicInteger mCount = new AtomicInteger(1);

            public Thread newThread(Runnable r) {
                return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
            }
        };

        //初始化存储任务的队列为LinkedBlockingQueue 最大容量为128
        private static final BlockingQueue<Runnable> sPoolWorkQueue =
                new LinkedBlockingQueue<Runnable>(128);

    // 步骤2： 根据参数配置执行任务线程池，即 THREAD_POOL_EXECUTOR
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        // 设置核心线程池的 超时时间也为30s
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

- **InternalHandler类**
    
```
 private static class InternalHandler extends Handler {

        // 构造函数
        public InternalHandler() {
            super(Looper.getMainLooper());
            // 获取的是主线程的Looper()
            // 故 AsyncTask的实例创建 & execute（）必须在主线程使用
        }

        @Override
        public void handleMessage(Message msg) {

            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;

            switch (msg.what) {
                // 若收到的消息 = MESSAGE_POST_RESULT
                // 则通过finish() 将结果通过Handler传递到主线程
                case MESSAGE_POST_RESULT:
                    result.mTask.finish(result.mData[0]); ->>分析3
                    break;

                // 若收到的消息 = MESSAGE_POST_PROGRESS
                // 则回调onProgressUpdate()通知主线程更新进度的操作
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
 ```
- **result.mTask.finish(result.mData[0])**

```
private void finish(Result result) {
        // 先判断是否调用了Cancelled()
            // 1. 若调用了则执行我们复写的onCancelled（）
            // 即 取消任务时的操作
            if (isCancelled()) {
                onCancelled(result);
            } else {

            // 2. 若无调用Cancelled()，则执行我们复写的onPostExecute(result)
            // 即更新UI操作
                onPostExecute(result);
            }
            // 注：不管AsyncTask是否被取消，都会将AsyncTask的状态变更为：FINISHED
            mStatus = Status.FINISHED;
        }
```

##### 总结
> 
- 任务线程池类（THREAD_POOL_EXECUTOR）实际上是1个已配置好的可执行并行任务的线程池    


- 调用THREAD_POOL_EXECUTOR.execute（）实际上是调用线程池的execute()去执行具体耗时任务
- 而该耗时任务则是 WorkerRunnable实例对象时复写的call（）内容
- 在call（）方法里，先调用 我们复写的doInBackground(mParams)执行耗时操作
- 再调用postResult(result)， 通过 InternalHandler 类 将任务消息传递到主线程；根据消息标识（MESSAGE_POST_RESULT）判断，最终通过finish(）调用我们复写的onPostExecute(result)，从而实现UI更新操作