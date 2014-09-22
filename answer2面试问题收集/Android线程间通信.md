##Android线程间通信
Android线程间通讯机制的内部实现原理，即`Handler、Message、MessageQueue、Looper、HandlerThread、AsyncTask`类的实现以及之间的关系。
    
### 一、Handler+Message+Runnable内部解析
问题：我们在使用`Handler`类的时候，都知道有`sendMessage（Message）`等发送消息的功能和`post（Runnable）`发送任务的功能，然后还有能够处理接受到的`Message`的功能。这时候我就会提出这样的问题：
1. 有发送、接受`Message`的功能，是不是`sendMessage`方法是直接调用`handleMessage`的重写方法里呢？  
2. 不是有按时间计划发送`Message`和`Runnable`吗？如果问题1成立的话，`handleMessage`可能会同时接受多个`Message`，但是此方法不是线程安全的（没有synchronized修饰），这样会出现问题了。

解决问题：如果对API有任何疑惑，最根本的方法就是查看源代码。
在看源代码之前，需要了解几个类：

* `Handler`：负责发送`Message`和`Runnable`到`MessageQueue`中，然后依次处理`MessageQueue`里面的队列。  

* `MessageQueue`：消息队列。负责存放一个线程的`Message`和`Runnable`的集合。  

* `Message`：消息实体类。  

* `Looper`：消息循环器。负责把`MessageQueue`中的`Message`或者`Runnable`循环取出来，然后分发到`Handler`中。  

四者的关系：一个线程可以有多个`Handler`实例，一个线程对应一个`Looper`，一个`Looper`也只对应一个`MessageQueue`，一个`MessageQueue`对应多个`Message`和`Runnable`。所以就形成了一对多的对应关系，一方：线程、`Looper`、`MessageQueue`；多方：`Handler`、`Message`。同时可以看出另一个一对一关系：一个`Message`实例对应一个`Handler`实例。    
一个`Handler`实例都会与一个线程和消息队列捆绑在一起，当实例化`Handler`的时候，就已经完成这样的工作。源码如下：
####`Handler`类
<pre><code>
/**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }
    
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
</code></pre>
可以从mLooper = Looper.myLooper()
mQueue = mLooper.mQueue;看出，实例化`Handler`就会绑定一个`Looper`实例，并且一个`Looper`实例包涵一个`MessageQueue`实例。    
问题来了，为什么说一个线程对应一个`Looper`实例？我们通过`Looper.myLooper()`找原因：
####`Looper`类
<pre><code>
// sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
     
    public static Looper myLooper() {
        return sThreadLocal.get();
</code></pre>
ThreadLocal类    
Implements a thread-local storage, that is, a variable for which each thread has its own value. All threads sharethe sameThreadLocal object, but each sees a different value when accessing it, and changes made by onethread do not affect the other threads. The implementation supportsnull values.    
所以说，每个线程都会存放一个独立的`Looper`实例，通过`ThreadLocal.get()`方法，就会获得当前线程的`Looper`的实例。    
    
好了，接下来就要研究一下`Handler`发送`Runnable`，究竟怎么发送？    
<pre><code>
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }     
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
</code></pre>   

可以看出，其实传入的`Runnable`对象都是封装到`Message`类中，看下`Message`是存放什么信息：    
    
####`Message`类
<pre><code>
public final class Message implements Parcelable {  
    public int what;  
    public int arg1;  
    public int arg2;  
    public Object obj;  
    public Messenger replyTo;  
    long when;  
    Bundle data;  
    Handler target;       
    Runnable callback;   
    Message next;  
    private static Object mPoolSync = new Object();  
    private static Message mPool;  
    private static int mPoolSize = 0;  
    private static final int MAX_POOL_SIZE = 10; 
</code></pre>

* When: 向`Handler`发送`Message`生成的时间     

* Data: 在Bundler 对象上绑定要线程中传递的数据     

* Next: 当前`Message` 对一下个`Message` 的引用     
  
* Handler: 处理当前`Message` 的`Handler`对象.     

* mPool: 通过字面理解可能叫他`Message`池,但是通过分析应该叫有下一个`Message`引用的`Message`链更加适合.         

其中`Message.obtain()`,通过源码分析就是获取断掉`Message`链关系的第一个`Message`.    
 **对于源码的解读，可以明确两点：    
        1）`Message.obtain()`是通过从全局`Message pool`中读取一个`Message`，回收的时候也是将该`Message` 放入到`pool`中。       
        2）`Message`中实现了`Parcelable`接口**
所以接下来看下`Handler`如何发送`Message`：    

####`Handler`类
<pre><code>
 /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
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
     private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
</code></pre>
其实无论是按时间计划发送`Message`或者Runnable，最终是调用了`sendMessageAtTime`方法，里面核心执行的是enqueueMessage方法，就是调用了`MessageQueu`e中的`enqueueMessage`方法，就是把消息`Message`加入到消息队列中。
    
这时候问题又来了，如果发送消息只是把消息加入到消息队列中，那谁来把消息分发到`Handler`中呢？    
不妨我们看看`Looper`类：    

####`Looper`类
<pre><code>
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.<span style="padding:0px; margin:0px; color:rgb(255,0,0)"><strong>dispatchMessage</strong></span>(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycle();
        }
    }
     public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
</code></pre>    
`dispatchMessage`最终是回调了`handleMessage`。换句话说，`Loop`的`loop()`方法就是取得当前线程中的`MessageQueue`实例，然后不断循环消息分发到对应的`Handler`实例上。就是只要调用Looper.loop()方法，就可以执行消息分发。    
**整个机制实现原理流程：当应用程序运行的时候，会创建一个主线程（UI线程）`ActivityThread`,这个类里面有个`main`方法，就是java程序运行的最开始的入口**    

<pre><code>
public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();
        if (sMainThreadHandler == null) {
            sMainThreadHandler = new Handler();
        }

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        <span style="padding:0px; margin:0px; color:rgb(255,0,0)">Looper.loop();</span>

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
</code></pre> 
UI线程就开始就已经调用了loop消息分发，所以当在UI线程实例的`Handler`对象发送消息或者任务时，会把`Message`加入到`MessageQueue`消息队列中，然后分发到`Handle`r的`handleMessage`方法里。    
###二、`HandlerThread`
其实上述就是线程间通讯机制的实现，而`HandlerThread`和`AsyncTask`只是对通讯机制进行进一步的封装，要理解也很简单：    
####HandlerThread类
<pre><code>
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }

    public void run() {
        mTid = Process.myTid();
        <span style="padding:0px; margin:0px; color:rgb(255,0,0)">Looper.prepare();</span>
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        <span style="padding:0px; margin:0px; color:rgb(255,0,0)">Looper.loop();</span>
        mTid = -1;
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
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
     * Ask the currently running looper to quit.  If the thread has not
     * been started or has finished (that is if {@link #getLooper} returns
     * null), then false is returned.  Otherwise the looper is asked to
     * quit and true is returned.
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    
    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    public int getThreadId() {
        return mTid;
    }
}
</code></pre> 
 可以看得出,HandlerThread继承了Thread，从run()方法可以看出，HandlerThread要调用start()方法，才能实例化HandlerThread的Looper对象，和消息分发功能。    
**所以使用HandlerThread，必须先运行HandlerThread,才能取出对应的Looper对象，然后使用Handler（Looper）构造方法实例Handler，这样Handler的handleMessage方法就是子线程执行了。**    
###三、AsyncTask    
`AsyncTas`k现在是`android`应用开发最常用的工具类，这个类面向调用者是轻量型的，但是对于系统性能来说是重量型的。这个类很强大，使用者很方便就能使用，只需要在对应的方法实现特定的功能即可。就是因为`AsyncTask`的强大封装，所以说不是轻量型的，先看下源代码吧：
<pre><code>
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";

    private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static final InternalHandler sHandler = new InternalHandler();

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;

    private volatile Status mStatus = Status.PENDING;
    
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    /**
     * Indicates the current status of the task. Each status will be set only once
     * during the lifetime of a task.
     */
    public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         */
        PENDING,
        /**
         * Indicates that the task is running.
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         */
        FINISHED,
    }

    /** @hide Used to force static handler to be created. */
    public static void init() {
        sHandler.getLooper();
    }

    /** @hide */
    public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }

    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

    
    public final Status getStatus() {
        return mStatus;
    }

    
    protected abstract Result doInBackground(Params... params);

   
    protected void onPreExecute() {
    }

    
    @SuppressWarnings({"UnusedDeclaration"})
    protected void onPostExecute(Result result) {
    }

    
    @SuppressWarnings({"UnusedDeclaration"})
    protected void onProgressUpdate(Progress... values) {
    }

   
    @SuppressWarnings({"UnusedParameters"})
    protected void onCancelled(Result result) {
        onCancelled();
    }    
    
    
    protected void onCancelled() {
    }

    
    public final boolean isCancelled() {
        return mCancelled.get();
    }

    
    public final boolean cancel(boolean mayInterruptIfRunning) {
        mCancelled.set(true);
        return mFuture.cancel(mayInterruptIfRunning);
    }

    
    public final Result get() throws InterruptedException, ExecutionException {
        return mFuture.get();
    }

    
    public final Result get(long timeout, TimeUnit unit) throws InterruptedException,
            ExecutionException, TimeoutException {
        return mFuture.get(timeout, unit);
    }

    
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

   
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
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

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

    
    public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }

    
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            sHandler.obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
}
</code></pre>
要理解这个工具类，主要是理解这几个成员对象：    

* private static final InternalHandler sHandler = new InternalHandler(); 

* private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR; 

* private final WorkerRunnable<Params, Result> mWorker; 

* private final FutureTask<Result> mFuture; 

分析：sHandler    
消息的发送者和处理者     
 sDefualtExecutor    
线程执行者。实际上就是一个线程池。    
 mWorker    
WorkerRunnable实现了Callable接口,就是有返回值的线程任务.    
 mFuture    
FutureTask是对Callable执行的一个管理类，能够获得线程执行返回的结果，和取消执行等操作。我们再深入一下FutureTask，其中的done()方法是回调方法：
<pre><code>
  /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

       <span style="padding:0px; margin:0px; color:rgb(204,0,0)"> done();</span>

        callable = null;        // to reduce footprint
    }
</code></pre>
只要线程移除或者挂起（取消）的时候，就会调用done()方法，然后在AsyncTask类中的mTask实现了done()方法，最后回调onCancelled()方法。    
    
具体的流程原理是这样的: 

1. 当第一次`AsyncTask`在UI线程实例化，其实是实例化`Handler`，同时UI线程的`Looper`和`MessageQueue`绑定在sHandler对象中，之后再去实例话`AsyncTask`不会在初始化`Handler`，因为`sHandler`是类变量. 

2. 当执行`execute`方法的时候，实际上是调用线程池的`execute`方法运行线程 . 

3. callable线程执行体就是调用了`doInBackground(mParams)`方法，然后以返回结果`result`当参数，又调用`postResult(Result result)`，实际上就是利用`sHandler`来发送`result`到UI线程的MessageQueue中，最后sHandler接受到result后，回调onPostExecute方法。 

4. 如果主动调用`publishProgress(Progress... values)`方法，就会利用`sHandler`把value发送到UI线程的`MessageQueue`中，然后`sHandler`接收到value后，回调`onProgressUpdate(Progress... values)`方法。 

注意：`sHandler`和`mDefaultExecutor`是类变量
  `mWorker`和`mFuture`是实例变量
所以，无论进程中生成多少个`AysncTask`对象，`sHandler`和`mDefaultExecutor`都是同一个，只是任务不同而已。
    
转自：[线程间通讯机制——深入浅出实现原理](http://blog.csdn.net/jackchen95/article/details/13631761)










