<h1>Chapter 9: 四大组件的工作过程</h1>
Android的四大组件中除了BroadcastReceiver以外，其他三种组件都必须在AndroidManifest中注册。

使用隐式Intent，不仅可以启动自己程序内的活动，还可以启动其他程序的活动。这使得 Android 多个应用程序之间的功能共享成为了可能。
例如，调用系统的浏览器来打开网页。startActivityForResult()可以返回数据给上一个Activity。

<h2>9.2 Activity 的工作过程</h2>
