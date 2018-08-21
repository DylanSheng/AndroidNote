<h1>Chapter 2: IPC</h1>

IPC is Inter-Process Communication  

Binder可以实现进程间通信  

<h2>2.2 多进程</h2>

在Android中使用多进程只有一种办法，就是给四大组件(Activity, Service, Receiver, ContentProvider) 在AndroidManifest 中指定android:process属性。 或非常规办法是在native层用JNI实现。

~~~
<activity
  android:name=".SecondActivity"
  android:process=":remote">
</activity>

<activity
  android:name=".ThirdActivity"
  android:process="com.dylansheng.remote">
</activity>
~~~

进程名以“:”开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以”：”开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

所有运行在不同进程中的四大组件，只要他们之间需要通过内存来共享数据，都会共享失败。
使用多进程会造成如下几方面的问题：
1. 静态成员和单例模式完全失效。
2. 线程同步机制完全失效。
3. SharedPreference的可靠性下降。因为SharedPreference不支持两个进程同时执行写操作，否则会导致一定几率的数据丢失。因为SharedPreference是通过通过读写XML来实现的。
4. Application会多次创建。

_每个进程对应一个虚拟机_

<h2>2.3.1 Serializable</h2>
http://www.blogjava.net/jiangshachina/archive/2012/02/13/369898.html
Java平台允许我们在内存中创建可复用的Java对象，但一般情况下，只有当JVM处于运行时，这些对象才可能存在，即，这些对象的生命周期不会比JVM的生命周期更长。但在现实应用中，就可能要求在JVM停止运行之后能够保存(持久化)指定的对象，并在将来重新读取被保存的对象。Java对象序列化就能够帮助我们实现该功能。

一般来说，我们应该手动指定serialVersionUID的值。可以避免反序列化失败。比如当版本升级后，我们可能删除或增加了一些新的成员变量，这个时候我们的反向序列化过程依然能够成功。

静态成员变量属于类不属于对象，所以不会参与序列化过程；其次用transient关键字标记的成员变量不参与序列化过程。

<h2>2.3.2 Parcable</h2>
Serializable是Java中的序列化接口，使用简单但是开销很大，序列化和反序列化过程需要大量I/O操作。
Parcelable是Android中的序列化方式，使用略麻烦但是效率很高，首选Parcelable。
Parcelable主要用在内存序列化上。
将对象序列化到存储设备中，或者将对象序列化后通过网络传输，建议使用Serializable。

<h2>2.3.3 Binder---_not finished_</h2>
从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager等）和相应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通讯的媒介。

<h1>2.4 Android中的IPC方式</h1>
<h2>2.4.1 Bundle</h2>
四大组件中的（Activity, Service, Receiver） 都是支持在Intent中传递Bundle数据的。因为Bundle实现了Parcelable。但并不支持所有类型数据。
<h2>2.4.2 文件共享</h2>
_Windows有排斥锁，但是Linux对读写并发没有限制_
适合对数据同步要求不高的进程之间通信，并且要妥善处理读写并发的问题。
SharedPreference是特例，不建议多进程使用。
<h2>2.4.3 Messenger</h2>
<h2>2.4.4 AIDL</h2>
<h2>2.4.5 ContentProvider</h2>
<h2>2.4.6 Socket</h2>
