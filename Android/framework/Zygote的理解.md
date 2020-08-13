##### Zygote的理解

    1.1 zygote的作用
        1、启动SystemServer、2、孵化应用进程
        
    2.2 zygote的启动流程
        Android 进程启动的三段式
            1、启动进程
            2、准备工作
            3、Loop循环 消息（消息来源MessageQuene、底层驱动、socket 来）
        zygote的启动流程 
            1、进程如何启动
                init 进程是linus启动后第一个进程，找init.rc配置文件，看看哪些系统服务（包括zygote服务 ）需要启动（启动方式fork+execve以及fork+handle）zygote进程的启动方式是fork+execve,zygote进程作为子进程（fork后给子进程的pid=0）继承init进程(子进程的pid)系统资源,然后zygote进程执行execve,会将继承父进程的系统资源清理
            2、启动之后做了什么
                1、在native层 
                    1、创建虚拟机java虚拟机（我们的进程jni调用的时候不需要创建java虚拟机，因为我们的进程有zygote fork出来的，fork的时候我们的虚拟机从zygote 继承而来，fork的时候充值虚拟机的状态）
                    2、注册Android系统的JNI函数
                    3、通过JNI调用进入Java层（如何进入java层调用：1、创建虚拟机 2、反射机制调用java方法）
                2、java层
                    1、预加载资源，这些资源是将来fork子进程时继承给他们（包含类、主题相关的资源、共享库）
                    2、启动SystemServer
                    3、Loop循环 等待Socket消息 通过Socket其他进程与zegote通信   
    3.3 zygote 的工作原理
        通过通过fork+execve 的方式孵化进程，通过Sccket的方式孵化进程

