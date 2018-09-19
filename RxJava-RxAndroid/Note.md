<h1>RxJava/RxAndroid</h1>

```java
public class MainActivity extends AppCompatActivity {

  private TextView textView = null;
  private Subscriber<Integer> subscriber = null;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Button button = findViewById(R.id.button);
    textView = findViewById(R.id.text);

    subscriber = new Subscriber<Integer>() {
      @Override
      public void onSubscribe(Subscription s) {
        s.request(Long.MAX_VALUE);
      }

      @Override
      public void onNext(Integer integer) {
        Log.e("Test", "i = " + integer);
        textView.setText(integer.toString());
      }

      @Override
      public void onError(Throwable t) {

      }

      @Override
      public void onComplete() {

      }
    };

    button.setOnClickListener(new View.OnClickListener() {
      @Override
      public void onClick(View view) {
        onButtonClick();
      }
    });
  }

  private void onButtonClick() {
    Flowable.create(new FlowableOnSubscribe<Integer>() {
      @Override
      public void subscribe(@NonNull FlowableEmitter<Integer> e) throws Exception {
        int i = 0;
        while (i < Long.MAX_VALUE) {
          try {
            Thread.sleep(100);
          } catch (InterruptedException ex) {
            ex.printStackTrace();
          }
          e.onNext(i);
          i++;
        }
      }
    }, BackpressureStrategy.DROP)
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())   
        .subscribe(subscriber);
  }
}
```

https://www.heqiangfly.com/2017/10/14/open-source-rxjava-guide-flowable/

<h2>Backpressure（背压）</h2>
背压是指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略。
这里上游数据发射和下游的数据处理在各自的独立线程中执行，如果在同一个线程中不存在背压的情形。下游对数据的处理会堵塞上游数据的发送，上游发送一条数据后会等下游处理完之后再发送下一条。
在例子中，上游发射数据时，并不知道下游数据有没有处理完，就会源源不断的发射数据，而下游数据会间隔两秒钟才处理一次，这样就会产生很多下游没来得及处理的数据，这些数据既不会丢失，也不会被垃圾回收机制回收，而是存放在一个异步缓存池中，如果缓存池中的数据一直得不到处理，越积越多，最后就会造成内存溢出，这便是 Rxjava 中的背压问题。

Flowable 的异步缓存池不同于 Observable，Observable的异步缓存池没有大小限制，可以无限制向里添加数据，直至OOM,而 Flowable 的异步缓存池有个固定容量，其大小为128。

`MISSING`：这种策略模式下相当于没有指定任何的背压策略，不会对数据做缓存或丢弃处理，需要下游通过背压操作符（onBackpressureBuffer()/onBackpressureDrop()/onBackpressureLatest()）指定背压策略。
`ERROR`：这种策略模式下如果缓存池中的数据超限了，则会抛出 MissingBackpressureException 异常
`BUFFER`：这种策略模式下没有为异步缓存池限制大小，可以无限制向里添加数据，不会抛出 MissingBackpressureException 异常，但会导致OOM。
`DROP`：这种策略模式下如果异步缓存池满了，会丢掉将要放入缓存池中的数据。
`LATEST`：这种策略模式下与 Drop 策略一样，如果缓存池满了，会丢掉将要放入缓存池中的数据，不同的是，不管缓存池的状态如何，LATEST都会将最后一条数据强行放入缓存池中。

`request(long n)` 用于发起<b>接收数据</b>的请求，如果不调用这个方法，虽然被观察者会正常发送数据，但是观察者是不会去接收数据的。参数 n 代表请求的数据量。
`cancel()` 方法用于取消订阅关系。

Observable/Observer
Flowable/Subscriber
Single/SingleObserver
Completable/CompletableObserver
Maybe/MaybeObserver

<h3>Single</h3>

结合 Single 本身的名字，我们可以联想到，这个组合是只能发射单个数据或者一条异常通知，不能发射完成通知，其中数据与通知只能发射一个。
那么 Single/SingleObserver 有什么使用场景呢？
我们在实际应用中，有时候需要发射的数据并不是数据流的形式，而只是一条单一的数据，比如发起一次网络请求。在这种情况下，如果我们使用 Observable，onComplete 会紧跟着 onNext 被调用，为什么不能将这连个方法合二为一呢。如果再这种情况下我们再使用 Observable 就显得有点大材小用，因为我们不需要处理 onNext() 的数据。于是，为了满足这种单一数据的使用场景，便出现了 Single。

<h3>Completable</h3>

和前面 Single/SingleObserver 的用法比较类似，只是这里不对数据进行处理，只有个通知的结果。比如：我们向服务器发起一个更新数据的请求，服务器更新数据以后是返回的是更新的结果。这个时候我们或许只是关心的是服务器更新数据是否成功，而不需要对数据进行处理，那么这个时候用 Completable/CompletableObserver 就可以了。
Completable 也提供了对 Observable、Flowable、Single 和 Maybe 的转换。

<h3>Maybe</h3>

Maybe可发射一条单一的数据，以及发射一条完成通知，或者一条异常通知，其中完成通知和异常通知只能发射一个，发射数据只能在发射完成通知或者异常通知之前，否则发射数据无效。

Disposable : It is used to dispose the subscription when an observer no longer wants to listen to Observable. For avoiding memory leaks.
CompositeDisposable : This can maintain list of subscriptions in a poll and dispose them at once. We use the .add() to add subscriptions to it.
We can unsubscribe or dispose the subscriptions on destroy.
