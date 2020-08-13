<font size=3>

#### Binder通信模型
    Binder框架定义了四个角色：
    Server，Client，ServiceManager以及Binder驱动，其中Server，Client，ServiceManager运行于用户空间，驱动运行于内核空间。
    Client：跨进程通讯的客户端（运行在某个进程）
    Server：跨进程通讯的服务端（运行在某个进程）
    Binder驱动：跨进程通讯的介质
    ServiceManager：跨进程通讯中提供服务的注册和查询（运行在System进程）
- **ServiceManager**
    
    ServiceManager作为Android进程间通信binder机制中的重要角色，运行在native层，由c++语言实现，任何Service被使用之前，例如播放音乐的MediaService，例如管理activity的ActivityManagerService，均要向SM注册，同时客户端使用某个service时，也需要向ServiceManager查询该Service是否注册过了
    
    ==作用==：
    
    1、负责与Binder driver通信，维护一个死循环，不断地读取内核binder driver。即不断读取看是否有对service的操作请求。
   
    2、维护一个svclist列表来存储service信息。
    
    3、向客户端提供Service的代理，也就是BinderProxy。
    延伸：客户端向ServiceManager查找Service并获取BinderProxy，通过BinderProxy实现与Service端的通信。
    
    4、负责提供Service注册服务
其实，ServiceManager就像是一个路由，首先，Service把自己注册在ServiceManager中，调用方（客户端）通过ServiceManager查询服务
    
    ==ServeiceManager如何提供自己的binder==
    
    当ServerManager作为Serve端的时候，它提供的Binder比较特殊，它没有名字也不需要注册，当一个进程使用BINDER_SET_CONTEXT_MGR命令将自己注册成SMgr时Binder驱动会自动为它创建Binder实体，这个Binder的引用在所有Client中都固定为0而无须通过其它手段获得。也就是说，一个Server若要向ServerManager注册自己Binder就必需通过0这个引用号和ServerManager的Binder通信
#### Binder的通信原理
![image](https://note.youdao.com/yws/api/personal/file/D0BC7F4F61374342AC5B8C6E4FEE8D3C?method=getImage&version=9052&cstk=8JUM-qAa)    
- **通讯原理**
    
    1、Service端通过Binder驱动在ServiceManager的查找表并注册Object对象的add方法
    
    2、Client端通过Binder驱动在ServiceManager的查找表中找到Object对象的add方法，并返回proxy对象的add方法，add方法是个空实现，proxy对象也不是真正的Object对象，是通过Binder驱动封装好的代理类的add方法
    
    3、当Client端调用add方法时，Client端会调用proxy对象的add方法，通过Binder驱动去请求ServiceManager来找到Service端真正对象，然后调用Service端的add方法

#### Java层Binder相关类：
    Binder类：是Binder本地对象
    
    BinderProxy类：是Binder类的内部类，它代表远程进程的Binder对象的本地代理
    
    Parcel类：是一个容器，它主要用于存储序列化数据，然后可以通过Binder在进程间传递这些数据
    
    IBinder接口：代表一种跨进程传输的能力，实现这个接口，就能将这个对象进行跨进程传递
    
    IInterface接口：client端与server端的调用契约，实现这个接口，就代表远程server对象具有什么能力，因为IInterface接口的asBinder方法的实现可以将Binder本地对象或代理对象进行返回


#### 抛弃aidl手动实现AIDL
1.定义一个接口服务，也就是上述的服务端要具备的能力来提供给客户端,定义一个接口继承IInterface，代表了服务端的能力

```
public interface PersonManger extends IInterface {
    void addPerson(Person mPerson);
    List<Person> getPersonList();
}
```
2.定义一个Server中的Binder实体对象了，首先肯定要继承Binder，其次需要实现上面定义好的服务接口
    
```
public abstract class BinderObj extends Binder implements PersonManger {
    public static final String DESCRIPTOR = "com.example.taolin.hellobinder";
    public static final int TRANSAVTION_getPerson = IBinder.FIRST_CALL_TRANSACTION;
    public static final int TRANSAVTION_addPerson = IBinder.FIRST_CALL_TRANSACTION + 1;
    public static PersonManger asInterface(IBinder mIBinder){
        IInterface iInterface = mIBinder.queryLocalInterface(DESCRIPTOR);
        if (null!=iInterface&&iInterface instanceof PersonManger){
            return (PersonManger)iInterface;
        }
        return new Proxy(mIBinder);
    }
    @Override
    protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
        switch (code){
            case INTERFACE_TRANSACTION:
                reply.writeString(DESCRIPTOR);
                return true;

            case TRANSAVTION_getPerson:
                data.enforceInterface(DESCRIPTOR);
                List<Person> result = this.getPersonList();
                reply.writeNoException();
                reply.writeTypedList(result);
                return true;

            case TRANSAVTION_addPerson:
                data.enforceInterface(DESCRIPTOR);
                Person arg0 = null;
                if (data.readInt() != 0) {
                    arg0 = Person.CREATOR.createFromParcel(data);
                }
                this.addPerson(arg0);
                reply.writeNoException();
                return true;
        }
        return super.onTransact(code, data, reply, flags);

    }

    @Override
    public IBinder asBinder() {
        return this;
    }
    
}

```
> asInterface方法，Binder驱动传来的IBinder对象，通过queryLocalInterface方法，查找本地Binder对象，如果返回的就是PersonManger，说明client和server处于同一个进程，直接返回，如果不是，返回给一个代理对象

3.定义代理对象，也是需要实现服务接口

```
public class Proxy implements PersonManger {
    private IBinder mIBinder;
    public Proxy(IBinder mIBinder) {
        this.mIBinder =mIBinder;
    }

    @Override
    public void addPerson(Person mPerson) {
        Parcel data = Parcel.obtain();
        Parcel replay = Parcel.obtain();

        try {
            data.writeInterfaceToken(DESCRIPTOR);
            if (mPerson != null) {
                data.writeInt(1);
                mPerson.writeToParcel(data, 0);
            } else {
                data.writeInt(0);
            }
            mIBinder.transact(BinderObj.TRANSAVTION_addPerson, data, replay, 0);
            replay.readException();
        } catch (RemoteException e){
            e.printStackTrace();
        } finally {
            replay.recycle();
            data.recycle();
        }
    }

    @Override
    public List<Person> getPersonList() {
        Parcel data = Parcel.obtain();
        Parcel replay = Parcel.obtain();
        List<Person> result = null;
        try {
            data.writeInterfaceToken(DESCRIPTOR);
            mIBinder.transact(BinderObj.TRANSAVTION_getPerson, data, replay, 0);
            replay.readException();
            result = replay.createTypedArrayList(Person.CREATOR);
        }catch (RemoteException e){
            e.printStackTrace();
        } finally{
            replay.recycle();
            data.recycle();
        }
        return result;
    }

    @Override
    public IBinder asBinder() {
        return null;
    }
}

```
代理对象实质就是client最终拿到的代理服务，通过这个就可以和Server进行通信了，首先通过Parcel将数据序列化，然后调用 remote.transact()将方法code，和data传输过去，对应的会回调在在Server中的onTransact()中
  
  
4.Server进程,onBind方法返回mStub对象，也就是Server中的Binder实体对象  

```
public class ServerSevice extends Service {
    private static final String TAG = "ServerSevice";
    private List<Person> mPeople = new ArrayList<>();


    @Override
    public void onCreate() {
        mPeople.add(new Person());
        super.onCreate();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mStub;
    }
    private BinderObj mStub = new BinderObj() {
        @Override
        public void addPerson(Person mPerson) {
            if (mPerson==null){
                mPerson = new Person();
                Log.e(TAG,"null obj");
            }
            mPeople.add(mPerson);
            Log.e(TAG,mPeople.size()+"");
        }

        @Override
        public List<Person> getPersonList() {
            return mPeople;
        }
    };
}

```
5.我们在客户端进程，bindService传入一个ServiceConnection对象，在与服务端建立连接时，通过我们定义好的BinderObj的asInterface方法返回一个代理对象，再调用方法进行交互


```
public class MainActivity extends AppCompatActivity {
    private boolean isConnect = false;
    private static final String TAG = "MainActivity";
    private PersonManger personManger;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        start();
        findViewById(R.id.textView).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (personManger==null){
                    Log.e(TAG,"connect error");
                    return;
                }
                personManger.addPerson(new Person());
                Log.e(TAG,personManger.getPersonList().size()+"");
            }
        });
    }

    private void start() {
        Intent intent = new Intent(this, ServerSevice.class);
        bindService(intent,mServiceConnection,Context.BIND_AUTO_CREATE);
    }
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e(TAG,"connect success");
            isConnect = true;
            personManger = BinderObj.asInterface(service);
            List<Person> personList = personManger.getPersonList();
            Log.e(TAG,personList.size()+"");
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.e(TAG,"connect failed");
            isConnect = false;
        }
    };
}

```
**相关方法解读**

1、**asInterface**
    
    当客户端bindService的onServiceConnecttion的回调里面，通过asInterface方法获取远程的service的。
    其函数的参数IBinder类型的obj，这个对象是驱动给我们的，如果是Binder本地对象，那么它就是Binder类型，如果是Binder代理对象，那就是BinderProxy类型。
    asInterface方法中会调用queryLocalInterface，查找Binder本地对象，如果找到，说明Client和Server都在同一个进程，这个参数直接就是本地对象，直接强制类型转换然后返回，
    果找不到，说明是远程对象那么就需要创建Binder代理对象，让这个Binder代理对象实现对于远程对象的访问
2、addPerson
    
    当客户端调用add方法时，首先用Parcel把数据序列化，然后调用mRemote.transact方法，mRemote就是new Stub.Proxy(obj)传进来的
3、transact
    
    transact方法是native方法，它的实现在native层，它会去借助Binder驱动完成数据的传输，
    当完成数据传输后，会唤醒Server端，调用了Server端本地对象的onTransact函数
4、onTransact
    
    在Server进程里面，onTransact根据调用code会调用相关函数，接着将结果写入reply并通过super.onTransact返回给驱动，驱动唤醒挂起的Client进程里面的线程并将结果返回