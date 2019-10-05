---
title: HandlerThread与IntentService
date: 2018-01-05 21:17:23
tags: Android
---
HandlerThread和IntentService都是Android提供方便开发者使用的类，通过这两个类我们可以方便快捷的执行异步任务。而将这两个类放在一起分析，是因为IntentService中使用了HandlerThread。

### HandlerThread
在项目中我们经常会为了耗时操作来开启线程执行耗时的代码，但是在系统中创建线程和销毁线程都会损耗系统的性能。而当我们线程过多时，系统在调度切换线程时也同样会性能的开销。因此，我们可以选择线程池来防止创建过多的线程，而如果在任务不需要并发，允许任务以队列的顺序执行的场景下，我们可以使用HandlerThread，相比起线程池，HandlerThread减少线程创建与销毁，切换线程的开销。

#### HandlerThread 的使用
先来看看HandlerThread在代码中的使用

``` Java
public class HandlerThreadActivity extends Activity implements View.OnClickListener {

    private Handler handler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler_thread);

        HandlerThread handlerThread = new HandlerThread("Test Handler Thread");
        handlerThread.start();
        handler = new Handler(handlerThread.getLooper()){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                Log.i("HandlerThread", Thread.currentThread().getName() + " get Msg : " + msg.obj.toString());
            }
        };

        Button btn = findViewById(R.id.handler_thread_btn);
        btn.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        if (handler != null) {
            Message message = Message.obtain();
            message.obj = "This is Msg From " + Thread.currentThread().getName() + " in " + System.currentTimeMillis();
            handler.sendMessage(message);
        }
    }
}
```

可以看到代码中通过HandlerThread创建了一个Handler，在点击按钮的时候通过Handler发送信息，并附上当前线程名与时间戳。在Handler接收信息的时候打印Handler所在的线程名与接收到的信息。连续点击按钮5次，看看Log的打印日志。从日志可以看到，任务是以队列的顺序来执行的。

```
Test Handler Thread get Msg : This is Msg From main in 1515155429407
Test Handler Thread get Msg : This is Msg From main in 1515155429819
Test Handler Thread get Msg : This is Msg From main in 1515155430128
Test Handler Thread get Msg : This is Msg From main in 1515155430381
Test Handler Thread get Msg : This is Msg From main in 1515155430551
```

#### HandlerThread 的实现
HandlerThread内部主要实现是与Handler一样的机制，Looper，这也是为什么这个类的名字有Handler存在的原因。HandlerThread的源码并不多，加上注释只有一百多行，我们看下代码。

``` Java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    /**
     * @hide
     */
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}
```
从源码我们可以看到它有两个构造方法，在构造方法我们可以设置线程名与线程的优先级。HandlerThread是继承与Thread，因此使用的时候记得要调用start方法，在run方法中做了同步方法的Looper的初始化，在Looper初始化之后调用notifyAll来唤醒其他等待线程。这么做的原因是当我们创建HandlerThread独有的Handler时需要用到其Looper，HandlerThread提供了getLooper方法来提供Looper，使用时有可能获取Looper时还未初始化完毕，在getLooper方法中当Looper还未初始化时会释放锁进入阻塞状态，此时notifyAll会唤醒这些阻塞的线程。当没有使用start方法就调用getLooper时，会通过isAlive()判断线程是否启动，如果没有启动会返回null值，这点需要注意一下。

因为有Looper的原因，所以HandlerThread一直都在运行没有停止，当我们想关闭HandlerThread时，可以使用quit()或者quitSafely() 来退出HandlerThread，那么这两个函数有什么区别呢？从代码可以看出，两者的区别就是分别调用了looper.quit()和looper.quitSafely()。查看Looper代码可知Looper.quit()最终会调用MessageQueue.removeAllMessagesLocked()，而Looper.quitsafely()会调用MessageQueue.removeAllFutureMessagesLocked()。quit()实际上是把消息队列全部清空，然后让MessageQueue.next()返回null令Looper.loop()循环结束从而终止Handler机制，但是可能有些消息在消息队列没来得及处理。而quitsafely()只清除消息队列中延迟信息，等待消息队列剩余信息处理完之后再终止Looper循环。

### IntentService
相比起Service，IntentService提供了异步操作，而且在使用完之后会自己销毁，不需要像Service那样需要调用stop方法来停止。先来看看使用方式。

#### IntentService 的使用
``` Java
public class TestIntentService extends IntentService {

    public TestIntentService() {
        super("TestIntentService");
    }
    
     @Override
    public void onCreate() {
        super.onCreate();
        Log.i("TestIntentService", "onCreate");
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        super.onStart(intent, startId);
        Log.i("TestIntentService", "onStart");
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        Log.i("TestIntentService", "onStartCommand");
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        Log.i("TestIntentService", "onHandleIntent : " + Thread.currentThread().getName());
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i("TestIntentService", "onDestroy");
    }
    
}

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btn = findViewById(R.id.intent_service_btn);
        btn.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        Intent intent = new Intent(this, TestIntentService.class);
        startService(intent);
    }

}

// 记得在AndroidManifest中注册
<service android:name=".touch.TestIntentService" />

```
使用IntentService时需要创建类继承IntentService，因为IntentService是个抽象类，同时需要调用IntentService的构造方法，传入一个mName参数，同时需要实现构造方法onHandleIntent。在Activity实现时跟平时使用Service一样调用start方法就可以了。在调用start方法后，打印出各个方法的log，可以看出在onHandleIntent的方法中是在异步的线程，线程名是IntentService[mName]，mName是我们之前传入的参数。
``` Java
TestIntentService: onCreate
TestIntentService: onStartCommand
TestIntentService: onStart
TestIntentService: onHandleIntent : IntentService[TestIntentService]
TestIntentService: onDestroy
```

### IntentService 的实现
IntenteService的代码也是不多，先看下代码。
``` Java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```
从代码我们了解到，IntentService是继承了Service的一个抽象类，提供了onHandleIntent的抽象方法。在onCreate中创建了HandlerThread，并以构造函数接收到的name作为线程名，同时获取了HandlerThread的Looper和创建所属的Handler。可以看到，在IntentService的onStartCommand调用了onStart方法，多次调用startService时只会调用onStartCommand方法，onStart是为了向下兼容，在onStart方法中通过发送消息到Handler处理，在Handler的HandleMessage中调用了抽象方法onHandleIntent，因此onHandleIntent是运行在HandlerThread的异步线程中，也是在onHandleIntent能做耗时操作的原因。而在onHandleIntent后立即调用了stopSelf方法，这就是我们不用手动停止IntentService的原因，因为它会自己停止。

**需要注意的点**

1. 我们知道，Service还有一种启动方式，bindService，通常我们会使用onBind返回的IBinder与Service交互，而在IntentService中，onBind返回的是null，即当我们使用bindService时我们不能享受IntentService提供的异步功能，只能得到一个无法交互的Service。
2. 当我们多次调用startService时，从Log可以看到多次调用了onCreate和onDestroy，这是因为我们在onHandleIntent只是做了打印工作，所以执行完很快就调用stopSelf。尝试修改下onHandleIntent，同时打印出onStartCommand的startId，然后再多次执行startService，查看Log。
``` Java
 @Override
protected void onHandleIntent(@Nullable Intent intent) {
    Log.i("TestIntentService", "onHandleIntent : " + Thread.currentThread().getName());
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}

@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    Log.i("TestIntentService", "onStartCommand : " + startId);
    return super.onStartCommand(intent, flags, startId);
}

// Log
TestIntentService: onCreate
TestIntentService: onStartCommand : 1
TestIntentService: onStart
TestIntentService: onHandleIntent : IntentService[TestIntentService]
TestIntentService: onStartCommand : 2
TestIntentService: onStart
TestIntentService: onStartCommand : 3
TestIntentService: onStart
TestIntentService: onHandleIntent : IntentService[TestIntentService]
TestIntentService: onStartCommand : 4
TestIntentService: onStart
TestIntentService: onHandleIntent : IntentService[TestIntentService]
TestIntentService: onHandleIntent : IntentService[TestIntentService]
TestIntentService: onDestroy
```
当执行耗时操作，我们连续四次startService时，从Log我们可以看出，onCreate和onDestroy只会调用一次，onStartCommand的startId会递增。原因是在IntentService调用stopSelf(msg.arg1)时传入了Message的参数，在onStart方法中我们可以看到msg.arg1就是startId。Service提供了stopSelf和stopSelf(startId)方法，stopSelf其实也是调用了stopSelf(-1)，它的意义是，在销毁时会判断当前的startId是否与参数的一致，如果一致则销毁，如果不一致则不销毁，而多次startService会改变startId的值，所以onCreate和onDestroy只会调用一次。这样做的好处是，减少了HandlerThread的创建，即线程的创建，提高了IntentService的利用率。

