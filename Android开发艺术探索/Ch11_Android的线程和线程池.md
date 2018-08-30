<h1>Chapter 11: 线程和线程池</h1>
对于AsyncTask来说，它的底层用到了线程池，对于IntentService和HandlerThread来说，它们的底层则直接使用了线程。
AsyncTask封装了线程池和Handler，它主要为了方便在子线程中更新UI。
HandlerThread是一种具有消息循环的线程，在它的内部可以使用Handler。
IntentService是一个服务，系统对其进行了封装使其可以更方便地执行后台任务，内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。

当系统中存在大量的线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的并行，除非线程数量小于等于CPU的核心数。因此，在一个进程中频繁地创建和销毁线程显然不是高效的做法。正确的做法是采用线程池，一个线程池中会缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。
<h2>主线程和子线程</h2>
从Android 3.0开始系统要求网络访问必须在子线程中进行。

<h2>11.2 Android中的线程形态</h2>
<h3>11.2.1 AsyncTask</h3>
AsyncTask并不适合进行特别耗时的后台任务，建议使用线程池。
`public abstract class AsyncTask<Params, Progress, Result>`
其中Params是参数类型，Progress表示后台任务的执行进度的类型，Result表示后台任务的返回结果的类型。

在Java中，”...”表示参数的数量不定，它是一种数组型参数。

```kotlin
class MainActivity : AppCompatActivity() {

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    Log.d("MainActivity", "onCreate")
    DownloadTask(txtview).execute()
  }

  private class DownloadTask(textView: TextView) : AsyncTask<Void, Int, Void>() {
    val mTextView = textView
    override fun doInBackground(vararg params: Void): Void? {
      //Log.d("MainActivity", "doInBackground")
      for (i in 1.. 100) {
        try {
          Thread.sleep(500)
        } catch (e: InterruptedException) {
          Thread.interrupted()
        }

        Log.d("MainActivity", "doInBackground $i")
        publishProgress(i)
      }
      return null
    }

    override fun onProgressUpdate(vararg values: Int?) {
      Log.d("MainActivity", "onProgressUpdate " + values[0])
      mTextView?.text = values[0].toString()
    }

    override fun onPostExecute(result: Void?) {
      super.onPostExecute(result)
      Log.d("MainActivity", "onPostExecute")
    }
  }
}
```

一个AsyncTask对象只能执行一次，即只能调用一次execute()，否则会报运行时异常。AsyncTasks采用一个线程来串行执行任务。我们仍然可以通过AsyncTask()的executeOnExecutor方法来并行地执行任务。

”至少在Android-23 SDK里面，多个AsyncTask对象是串行执行的。“ https://blog.csdn.net/zj510/article/details/51622597
SDK 28也是串行


>AsyncTask是Android提供的轻量级的异步类, 可以直接继承AsyncTask,在类中实现异步操作，并提供接口反馈当前异步执行进度(可以通过接口实现UI进度更新)，最后反馈执行的结果给UI主线程。AsyncTask 有两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handler(InternalHandler), 其中SerialExecutor是用来任务排队的 ,而线程池THREAD_POOL_EXECUTOR是用来真正执行任务的;  AsyncTask异步任务底层是封装了线程池和Handler。<cite>-------https://www.jianshu.com/p/aaca5d83eb65 </cite>

From AsyncTask.java
```java
/**
 * An {@link Executor} that executes tasks one at a time in serial
 * order.  This serialization is global to a particular process.
 */
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
//sDefaultExecutor 实现了 the SerialExecutor.

@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}

@MainThread
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

//很明显是用一个ArrayDeque来schedule。
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
```

From Executor.java
```Java
public interface Executor {
  void execute(Runnable command);
}
```

<h3>11.2.3 HandlerThread</h3>
HandlerThread继承了Thread，它是一个可以使用Handler的Thread，它就是在run方法中通过Looper.prepare()来创建消息队列，并通过Looper.loop()来开启消息循环，这样在实际的使用中就允许在HandlerThread中创建Handler了。
```java
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
```

普通Thread主要用于在run方法中执行一个耗时任务，而HandlerThread在内部创建了消息队列，外界需要通过Handler的消息方式来通知HandlerThread执行一个具体的任务。

<h3>IntentService</h3>
IntentService可用于执行后台耗时的任务，当任务执行后它会自动停止，同时由于IntentService是服务的原因，这导致了它的优先级比单纯的线程要高很多，所以IntentService比较适合执行一些高优先级的后台任务，因为它优先级高不容易被系统杀死。


<h2>11.3 线程池</h2>
优点：
1. 重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销。
2. 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致的阻塞现象。
3. 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能。

<h3>11.3.1 ThreadPoolExecutor</h3>
```Java
/**
* @param corePoolSize the number of threads to keep in the pool, even
*        if they are idle, unless {@code allowCoreThreadTimeOut} is set
*        默认情况下，核心线程会在线程池中一直存活，即使它们处于闲置状态。
* @param maximumPoolSize the maximum number of threads to allow in the
*        pool
* @param keepAliveTime when the number of threads is greater than
*        the core, this is the maximum time that excess idle threads
*        will wait for new tasks before terminating.
* @param unit the time unit for the {@code keepAliveTime} argument
* @param workQueue the queue to use for holding tasks before they are
*        executed.  This queue will hold only the {@code Runnable}
*        tasks submitted by the {@code execute} method.
* @param threadFactory the factory to use when the executor
*        creates a new thread
*/
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
```

ThreadPoolExecutor执行任务时大致遵循如下规则：
1. 如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个核心线程来执行任务。
2. 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3. 如果在步骤2中无法将任务插入到任务队列中，这往往是由于任务队列已满，这个时候如果线程数量未达到线程池规定的最大值，那么会立刻启动一个非核心线程来执行任务。
4. 如果步骤3中线程数量已经达到了线程池规定的最大值，那么就拒绝执行此任务，ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法来通知调用者。

AsyncTask中线程池的配置 (SDK-28)
```java
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

<h3>11.3.2 线程池的分类</h3>
1. FixedThreadPool
2. CachedThreadPool
3. ScheduledThreadPool
4. SingleThreadExecutor
