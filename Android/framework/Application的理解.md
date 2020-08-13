##### Application
    1、Application的作用
         1、保存应用进程内的全局变量（当时对应这种全局变量建议在单例中做，按照全局变量的不同功能来实现不同的单例）
         2、 应用进程启动之后的初始化操作。（Application的创建是在四大组件之前）
         3、提供上下文。 （Application 可以提供横跨生命周期的context，所以不用担心内存泄漏）
    Application 是跟着进程走，开几个进程，就有几个Application 
    2、 Application的类继承关系以及生命周期
        1、类继承关系
        1、Application 继承ContextWaraper，ContextWarpper 是对Context的包装，所以Application是继承Context的，Application可以当做Context来用 。
        2、ContextWarpper内有一个Context的mBase的变量，这个变量是在构造函数被赋值，也可以在attachBaseContext()方法被赋值。
        3、这是一个典型的代理，其中ContextWarppe或者Application的内部实现都是由mBase来处理。
        2、生命周期
            1、构造函数
            2、attachBaseContext
            3、onCreate
            4、onTerminate （在模拟器中使用，在手机上不生效的）
    3、Application的初始化原理
        application的初始化过程是在activityThread的main()函数中的attach()(binder跨进程通信)函数开始的，
        然后执行ams中的attachApplicationLocked()函数，
        该函数中，会调用新进程的binder对象的bindApplication()(这个函数是binder跨进程通信回到了新进程中)
        然后拿到应用安装包信息，反射的机制new Application
        初始化流程
        1、通过反射 new Application 的实例
        2、通过调用Application的attachBaseContext()设置Application的Context的mBase变量，实际上传入的是ContextImpl
        3、调用Application的onCreate
        注意
        1、
        所以不要在Application中构造函数初始化使用Context，应为Context 还没准备好
        2、不再Application的生命周期函数执行耗时操作，会阻碍UI线程，会耽误应用进程启动组件。
        3、尽量不要在Application中缓存全局静态变量，如果ActivityA设置Application内部的name变量值，在跳转到ActivityB中，B使用这个name，而这时应用切到后台，由于内存不足等原因，应用被清理，此时我们回到应用，应用重写初始化Application，恢复ActivityB，这是B在使用name，则name变为null了。这种场景的解决办法：ActivityB中onSaveInstanceState 方法, 在这个方法中我们可以保存一下 name，在onCreate中在获取