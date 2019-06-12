[TOC]

# Android问题

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

### Service与Activity之间通信的几种方式

- 通过Binder对象
- 通过broadcast(广播)的形式
- 使用事件总线（观察者模式）

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

## 什么情况下容易导致内存泄露

* *内存泄*漏（Memory Leak）是指程序中己动态分配的堆*内存*由于某种原因程序未释放或无法释放，造成系统*内存*的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。
* 内存溢出（OutOfMemory）是指：当JVM因为没有足够的内存来为对象分配空间并且垃圾回收器也已经没有空间可回收时，就会抛出这个error

* 参考至`[http://www.jackywang.tech/AndroidInterview-Q-A/interview/%E4%BB%80%E4%B9%88%E6%83%85%E5%86%B5%E5%AF%BC%E8%87%B4%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F-%E7%BE%8E%E5%9B%A2.html](http://www.jackywang.tech/AndroidInterview-Q-A/interview/什么情况导致内存泄漏-美团.html)`

1. 资源对象没关闭造成的内存泄漏

   描述： 资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于 java虚拟机内，还存在于java虚拟机外。如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄漏。因为有些资源性对象，比如 SQLiteCursor(在析构函数finalize(),如果我们没有关闭它，它自己会调close()关闭)，如果我们没有关闭它，系统在回收它时也会关闭它，但是这样的效率太低了。因此对于资源性对象在不使用的时候，应该调用它的close()函数，将其关闭掉，然后才置为null.在我们的程序退出时一定要确保我们的资源性对象已经关闭。 程序中经常会进行查询数据库的操作，但是经常会有使用完毕Cursor后没有关闭的情况。如果我们的查询结果集比较小，对内存的消耗不容易被发现，只有在常时间大量操作的情况下才会复现内存问题，这样就会给以后的测试和问题排查带来困难和风险。

2. 构造Adapter时，没有使用缓存的convertView

   描述： 以构造ListView的BaseAdapter为例，在BaseAdapter中提供了方法： public View getView(int position, ViewconvertView, ViewGroup parent) 来向ListView提供每一个item所需要的view对象。初始时ListView会从BaseAdapter中根据当前的屏幕布局实例化一定数量的 view对象，同时ListView会将这些view对象缓存起来。当向上滚动ListView时，原先位于最上面的list item的view对象会被回收，然后被用来构造新出现的最下面的list item。这个构造过程就是由getView()方法完成的，getView()的第二个形参View convertView就是被缓存起来的list item的view对象(初始化时缓存中没有view对象则convertView是null)。由此可以看出，如果我们不去使用 convertView，而是每次都在getView()中重新实例化一个View对象的话，即浪费资源也浪费时间，也会使得内存占用越来越大。 ListView回收list item的view对象的过程可以查看: android.widget.AbsListView.java --> voidaddScrapView(View scrap) 方法。 示例代码：

```java
public View getView(int position, ViewconvertView, ViewGroup parent) {
	View view = new Xxx(...); 
	... ... 
	return view; 
} 
```

​	修正示例代码：

```java
public View getView(int position, ViewconvertView, ViewGroup parent) {
	View view = null; 
	if (convertView != null) { 
		view = convertView; 
		populate(view, getItem(position)); 
		... 
	} else { 
		view = new Xxx(...); 
		... 
	} 
	return view; 
} 
```

3. Bitmap对象不在使用时调用recycle()释放内存

   描述： 有时我们会手工的操作Bitmap对象，如果一个Bitmap对象比较占内存，当它不在被使用的时候，可以调用Bitmap.recycle()方法回收此对象的像素所占用的内存，但这不是必须的，视情况而定。可以看一下代码中的注释：

4. 试着使用关于application的context来替代和activity相关的context

   这是一个很隐晦的内存泄漏的情况。有一种简单的方法来避免context相关的内存泄漏。最显著地一个是避免context逃出他自己的范围之外。使用Application context。这个context的生存周期和你的应用的生存周期一样长，而不是取决于activity的生存周期。如果你想保持一个长期生存的对象，并且这个对象需要一个context,记得使用application对象。你可以通过调用 Context.getApplicationContext() or Activity.getApplication()来获得。因为有时候当我们的Activity需要回收的时候，如果还被别的地方引用了这个 Activity 的 context ，那么回收失败

5. 注册没取消造成的内存泄漏

   一些Android程序可能引用我们的Anroid程序的对象(比如注册机制，观察者模式)。即使我们的Android程序已经结束了，但是别的引用程序仍然还有对我们的Android程序的某个对象的引用，泄漏的内存依然不能被垃圾回收。调用registerReceiver后未调用unregisterReceiver。 比如:假设我们希望在锁屏界面(LockScreen)中，监听系统中的电话服务以获取一些信息(如信号强度等)，则可以在LockScreen中定义一个 PhoneStateListener的对象，同时将它注册到TelephonyManager服务中。对于LockScreen对象，当需要显示锁屏界面的时候就会创建一个LockScreen对象，而当锁屏界面消失的时候LockScreen对象就会被释放掉。 但是如果在释放 LockScreen对象的时候忘记取消我们之前注册的PhoneStateListener对象，则会导致LockScreen无法被垃圾回收。如果不断的使锁屏界面显示和消失，则最终会由于大量的LockScreen对象没有办法被回收而引起OutOfMemory,使得system_process 进程挂掉。 虽然有些系统程序，它本身好像是可以自动取消注册的(当然不及时)，但是我们还是应该在我们的程序中明确的取消注册，程序结束时应该把所有的注册都取消掉。

6. 集合中对象没清理造成的内存泄漏。

   我们通常把一些对象的引用加入到了集合中，当我们不需要该对象时，并没有把它的引用从集合中清理掉，这样这个集合就会越来越大。如果这个集合是static的话，那情况就更严重了。



