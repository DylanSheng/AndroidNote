<h1>Chapter 11: 线程和线程池</h1>
对于AsyncTask来说，它的底层用到了线程池，对于IntentService和HandlerThread来说，它们的底层则直接使用了线程。
AsyncTask封装了线程池和Handler，它主要为了方便在子线程中更新UI。
HandlerThread是一种具有消息循环的线程，在它的内部可以使用Handler。
IntentService是一个服务，系统对其进行了封装使其可以更方便地执行后台任务，内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。

当系统中存在大量的线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的并行，除非线程数量小于等于CPU的核心数。因此，在一个进程中频繁地创建和销毁线程显然不是高效的做法。正确的做法是采用线程池，一个线程池中会缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。
<h2>主线程和子线程</h2>
从Android 3.0开始系统要求网络访问必须在子线程中进行。

<h2>Android中的线程形态</h2>
<h3>AsyncTask</h3>
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
