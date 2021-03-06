<h1>Chapter 10: Android的消息机制</h1>
Android的消息机制主要是指Handler的运行机制以及Handler所附带的MessageQueue和Looper的工作过程。
Handler的主要作用是将一个任务切换到某个制定的线程中去执行。用来解决子线程中无法访问UI的矛盾。
ViewRootImpl对UI操作做了验证，由其中的checkTread方法来完成。
Android的UI控件不是线程安全的，多线程并发访问会导致UI控件不可控。
为何不加锁？第一，上锁机制会让UI访问的逻辑变得复杂，其次，会降低访问效率，因为锁机制会阻塞某些线程的执行。
Handler创建时会采用当前线程的Looper来创建内部消息循环系统，如果当前线程没有Looper，那么就会报错。

MessageQueue实际上是Singly Linked List实现的。MessageQueue只是一个消息的存储单元，他不能去处理消息，而Looper会以无限循环的形式去查找是否有新消息。如果有新消息，处理，否则就一直等待着。Looper中有ThreadLocal，他并不是线程，作用是可以在每个线程中互不干扰地存储数据。通过ThreadLocal可以轻松获取每个线程的Looper。线程是默认没有Looper的，如果需要使用Handler就必须为线程创建Looper。
UI线程，就是ActivityThread，被创建时就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。

![Alt text](Images/Handler.png?raw=true "Handler")

<h2>10.2.1 ThreadLocal</h2>
对于Handler来说，它需要获取当前线程的Looper，很显然Looper的作用域就是线程并且不同线程具有不同的Looper，这个时候通过ThreadLocal就可以轻松实现Looper在线程中的存储。如果不使用ThreadLocal，那么系统就必须提供一个全局的哈希表供Handler查找指定线程的Looper。

详见书中例子

核心：ThreadLocal可以在不同的线程中维护一套数据的副本并且彼此互不干扰。本质上每个Thread上都要建独立的Map

```java
//ThreadLocal的set函数

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

<h2>10.2.2 MessageQueue</h2>
MessageQueue主要包含两个操作：插入和读取。本质是Singly Linked List，这样的话插入和删除是O(1)。

``
_“… the volatile modifier guarantees that any thread that reads a field will see the most recently written value.” - Josh Bloch_
``

next方法是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next方法会返回这条消息并将其从单链表中移除。

<h2>10.2.3 Looper</h2>
Looper会不停地从MessageQueue中查看是否有新消息。如果有新消息就会立刻处理，否则就一直阻塞在那里。
通过Looper.prepare()即可为当前线程创建一个Looper，接着通过Looper.loop()来开启消息循环。
```java
  new Thread("Thread#2") {
    @Override
    public void run() {
      Looper.prepare();
      Handler handler = new Handler();
      Looper.loop();
    };
  }.start();
```
Looper除了prepare方法外，还提供了prepareMainLooper方法，这个方法主要是给主线程也就是ActivityThread创建Looper用的。通过它可以在任何地方获取到主线程的Looper。
quit会直接退出Looper，而quitSafely只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才安全地退出。建议不需要的时候终止Looper。

<h3>10.2.4 Handler</h3>
Handler发送消息的过程仅仅是向消息队列中插入了一条消息，MessageQueue的next方法就会返回这条消息给Looper，Looper收到消息后就开始处理了，最终消息由Looper交给Handler处理，即Handler的dispatchMessage方法会被调用，这时Handler就进入了处理消息的阶段。
Handler的处理方式。首先，检查Message的callback是否为null，不为null就通过handleCallback来处理消息。Message的callback是一个Runnable对象。其次，检查mCallback是否为null，不为null就调用mCallback的handleMessage方法来处理消息。

![Alt text](Images/Handler消息处理.png?raw=true "Handler消息处理")

<h2>10.3 主线程的消息循环</h2>
Android的主线程即使ActivityThread，主线程的入口方法为main，在main方法中系统会通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue，并通过Looper.loop()来开启主线程的消息循环。
主线程的消息循环开始了以后，ActivityThread还需要一个Handler来和消息队列进行交互，这个Handler就是ActivityThread.H，它内部定义了一组消息类型，主要包含了四大组件的启动和停止等过程。
ActivityThread通过ApplicationThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中去执行，即切换到主线程中去执行，这个过成就是主线程的消息循环模型。
