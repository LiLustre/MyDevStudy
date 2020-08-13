<font size=3>

#### Model的理解
    模型，包含业务模型以及负责处理业务层逻辑，业务数据以及针对业务数据的操作共同构成了 Model 层，
    但数据的最终获取操作不应该是Model来完成，而应该有一个Repository来完成。Repository最终负责数据在本地或者网络上操作，Model负责调用Repository，并处理业务逻辑，例如数据获取之后的整合，数据存储之前的数据转换。
#### Model 层与 Presenter/ViewModel 层的依赖关系
    Presenter/ViewModel 通过接口的形式依赖 Model 层，Model 层完全不依赖 Presenter/ViewModel。Model 层必然不会出现任何 presenter 这样的单词，上层通过观察者模式来监听 Model 层的数据变化（ LoadCallback 接口也算是一种），Model 层也不用关心上层是 Presenter 还是 ViewModel
#### MVC
    定义：MVC 即分为Model、View 、Controller，简单来说就是通过controller的控制去操作model层的数据，并且返回给view层展示
    
    
   
![image](https://note.youdao.com/yws/api/personal/file/D790A149B72D41B5A495111869025F2C?method=getImage&version=8630&cstk=TmLiF-Q_)
- 角色分工:
>     Model 模型，负责数据的加载和存储，业务数据以及对业务数据的操作
>     View 视图，负责界面的展示
>     Controller 控制器，负责逻辑控制

- Android中的MVC的角色：

>     model：（数据提供者）就是读取数据库，网络请求这些我们一般有专门的类，
>     controller：就是我们的activity
>     View：一般用自定义控件

- 工作原理:

>     当用户出发事件的时候，view层会发送指令到controller层，
>     接着controller去通知model层更新数据，
>     model层更新完数据以后直接显示在view层上

- 优点
>     1.结构清晰，职责划分清晰
>     2.降低耦合
>     3.有利于组件重用
- 缺点
>     
>     1.一般来说，Activity / Fragment 会承担 View 和 Controller 两个角色，就会导致 Activity / Fragment 中代码较多
>     
>     2.Model 直接操作 View，View 的修改会导致 Controller 和 Model 都进行改动
>     
>     3.增加了代码结构的复杂性

#### MVP 架构
    Model-View-Presenter，MVC的一个演变模式，将Controller换成了Presenter，
    主要为了解决MVC中View对Model的依赖，会导致View也包含了业务逻辑，将View和Model解耦，不过MVC中Controller会变得很厚很复杂这个问题依然没有解决
![image](https://note.youdao.com/yws/api/personal/file/3A5BBD4D4C4A4269A18D43E1A08D75CA?method=getImage&version=8679&cstk=TmLiF-Q_)

- 角色分工
    
>     Model 模型，负责数据的加载和存储，业务数据以及对业务数据的操作
>     View 视图，负责界面的展示
>     Presenter 控制器，负责表现层逻辑控制

- 工作原理
    
>     MVP 中的各个角色划分，和 MVC 基本上相似，那么区别在哪里呢？区别就在角色的通信上
>     MVP 和 MVC 最大的不同，就是 View 和 Model 不相互持有，都通过 Presenter 做中转。
>     1、View 产生事件，通知给 Presenter，
>     2、Presenter 中进行逻辑处理后，通知 Model 更新数据，
>     3、Model 更新数据后，通知数据结构给 Presenter，
>     4、Presenter 再通知 View 更新界面    
    
- 实现

>     定义了 View，Model，Presenter 三个接口，其中 View 会持有 Presenter，Presenter 中持有View和Model，Model层数据操作结果通过接口的形式回传给presenter。

- 优点：
    
>     1、结构清晰，职责划分清晰
>     2、模块间充分解耦， （1.当View发生变化，或者改版，只需要修改View层，Presenter和model无需改动  2.当业务逻辑或者数据获取方式发生变化只需要修改Model ）
>     3、有利于组件的重用

- 缺点：
>     会引入大量的接口，导致项目文件数量激增
>     增大代码结构复杂性，三层划分，函数的调用栈变深，不熟悉的开发人员遇到问题时，不能快速定位问题所在，或者各层职责认识不清，导致不同层之间代码乱入。

#### MVVM
    MVVM将MVP中的presenter层换成了viewmodel层，还有一点就是view层和viewmodel层是相互绑定的关系，这意味着当你更新viewmodel层的数据的时候，view层会相应的变动ui，MVVM 要解决的问题和 MVC，MVP 大同小异：控制逻辑，数据处理逻辑和界面交互耦合，并且同时能将 MVC 中的 View 和 Model 解耦，还可以把 MVP 中 Presenter 和 View 也解耦
![image](https://note.youdao.com/yws/api/personal/file/6770603D87954A618A7D88CDFD30E72E?method=getImage&version=8756&cstk=TmLiF-Q_)

- 角色分工
    
>     Model 模型，负责数据的加载和存储，业务数据以及对业务数据的操作
>     View 视图，负责界面的展示
>     ViewModel 控制器，负责表现层逻辑控制

- 工作原理
    
>    View和ViewModel的双向绑定通过 data binding这个框架，可以轻松的实现MVVM。 View 产生事件，自动通知给 ViewMode，ViewModel 中进行逻辑处理后，通知 Model 更新数据，Model 更新数据后，通知数据结构给 ViewModel，ViewModel 自动通知 View 更新界面。

- 优点
> 
>     1.在 MVP 的基础上，MVVM 把 View 和 ViewModel 也进行了解耦
>     2.模块间充分解耦
>     3.结构清晰，职责划分清晰

- 缺点
    
>     1.Debug 困难，由于 View 和 ViewModel 解耦，导致 Debug 时难以一眼看出 View 的事件传递
>     2.代码复杂性增大


#### 总结
![image](https://note.youdao.com/yws/api/personal/file/E9FA1E41921E4678B73B4A36B21B9FDC?method=getImage&version=8787&cstk=TmLiF-Q_)