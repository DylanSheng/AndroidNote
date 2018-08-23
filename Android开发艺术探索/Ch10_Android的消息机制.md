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
