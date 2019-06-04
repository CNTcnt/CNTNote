[TOC]

# Android面试题

## 四大组件问题

### 一个 app 被杀掉进程后，是否还能收到广播

* 能够想到的方案是在AndroidMainifest.xml中静态注册一个广播，监听系统的某些广播达到触发应用完成操作的目的，但现象是：程序安装后，在未启动的情况下无法接收到系统的广播；但在启动一次后，就能够正常收到系统广播。

* 通过查阅资料发现，这个问题只有在Android 3.1及以上的版本才会出现，我用的是4.2.2的版本测试，自然会有这个问题，具体原因是：从Android3.1开始，新安装的程序会被置于"stopped"状态，并且只有在至少手动启动这个程序一次后该程序才会改变状态，能够正常接收到指定的广播消息。Android这样做的目的是防止广播无意或者不必要地开启未启动的APP后台服务。

* 也就是说在Android3.1及以上的版本，在未启动的情况下通过应用自身完成一些操作是不可能的，但Android提供了一种借助其它应用发送指定Flag广播的方式，达到应用在未启动的情况下仍然能够收到消息的效果。

* 从Android 3.1开始，系统给Intent定义了两个新的Flag，分别为FLAG_INCLUDE_STOPPED_PACKAGES（表示包含未启动的App）和FLAG_EXCLUDE_STOPPED_PACKAGES（表示不包含未启动的App），用来控制Intent是否要对处于停止状态的App起作用，具体的操作方式如下：

  1.在需要接收广播的应用中静态注册广播，并定义好action，并且需要指定android:exported="true"；

### Android如何在service中弹出对话框

* 系统可以在低电量的时候弹出电量不足的提示，那么我们也可以按同样的方法做到

* 下面介绍在service中弹出对话框的两种方法：

  1。**将dialog的Type设置为TYPE_SYSTEM_ALERT**，还要在AndroidMainfest.xml中添加权限

  ~~~java
  <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
  dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
  ~~~

  这种方式不管当前处于应用内还是应用外，甚至在其他应用内，都可以弹出了dialog；

  还有一个适配问题：在android O以及以上版本要是用悬浮窗功能，你需要把悬浮窗的type设置成`WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY`

  2.**将Activity设置成dialog主题弹出**，注意此时弹出的是activity而不是上面的dialog

  ~~~java
  //在AndroidManifest.xml中加上android:theme=”@android:style/Theme.Dialog”
  //在service中启动
  Intent intent = new Intent(getApplicationContext(), DialogActivity.class);
                  startActivity(intent);
  ~~~

  这种方式**在弹出对话框的时候，会先跳转到我们的应用，然后再弹出对话框**

  

  

## 为什么不建议使用Intent传递大的数据

* Intent 在 Activity 间传递基础类型数据或者可序列化的对象数据。但是 Intent 对数据大小是有限制的，当超过这个限制后，就会触发 TransactionTooLargeException 传输过大异常。

* 为什么会出现异常

  * 简单来说，Intent 传输数据的机制中，用到了 Binder。Intent 中的数据，会作为 Parcel 被存储在 Binder 的事务缓冲区(Binder transaction buffer)中的对象进行传输。

    而这个 **Binder 事务缓冲区具有一个有限的固定大小，当前为 1MB**。你可别以为传递 1MB 以下的数据就安全了，这里的 1MB 空间并不是当前操作独享的，而是由当前进程所共享。也就是说 Intent 在 Activity 间传输数据，本身也不适合传递太大的数据。

* Bundle 存储数据，到底是值传递(深拷贝)还是引用传递？

  * 这个不是绝对的，要根据使用场景来用
  * Intent 使用 Bundle 存储数据，Activity 之间传递数据，首先要考虑跨进程的问题，而 Android 中又是通过 Binder 机制来解决跨进程通信的问题。涉及到跨进程，对于复杂数据就要涉及到序列化和反序列化的过程，这就注定是一次值传递（深拷贝）的过程。
  * 在 Android 中，使用 Bundle 传输数据，并非 Intent 独有的。例如使用弹窗时，DialogFragment 中也可以通过 `setArguments(Bundle)` 传递一个 Bundle 对象给对话框。Fragment 本身是不涉及跨进程的，这里虽然使用了 Bundle 传输数据，但是并没有通过 Binder，也就是不存在序列化和反序列化。和 Fragment 数据传递相关的 Bundle，其实传递的是原对象的引用。

* 如何解决这个异常？

  * 可以尽量不要去传输大数据比如图片,长字符串、Bitmap 

  * 还有就是不要走 Binder 通道就可以杜绝这个异常

  * 解决大数据传递问题，可以从数据源出发，根据数据的标识，还原数据，或者先持久化再还原。也可以使用 EventBus 的粘性事件来解决。

  * 此前阿里发布的《**Android 开发者手册**》中，就提到了这个问题的解决建议。

    ~~~java
    //通过 EventBus 来传递数据。
    ~~~

  * 这里利用的 EventBus 的粘性事件(Sticky Event)来实现，EventBus 内部维护了一个 Map 对象 `stickyEvents`，用于缓存粘性事件。

  * 粘性事件使用 `postSticky()` 方法发送，它会将事件缓存到 `stickyEvents` 这个 Map 对象中，以待下次注册时，将这个事件取出，抛给注册的组件。以此来达到一个粘性的滞后事件发送和接收。



