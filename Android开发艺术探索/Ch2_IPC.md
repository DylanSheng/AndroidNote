<h1>Chapter 2: IPC</h1>

IPC is Inter-Process Communication  

Binder可以实现进程间通信  

<h2>2.2 多进程</h>

在Android中使用多进程只有一种办法，就是给四大组件(Activity, Service, Receiver, ContentProvider) 在AndroidManifest 中指定android:process属性。 或非常规办法是在native层用JNI实现。

~~~
<activity
  android:name=".SecondActivity"
  android:process=":remote">
</activity>
~~~
