# Activitythread main方法在哪儿调用

## 回答

* ActivityThread是真正的核心类，它的**main方法**，是**整个应用进程的入口**。

```
在一个Android 程序开始运行的时候，会单独启动一个Process。
默认的情况下，所有这个程序中的Activity或者Service（Service和 Activity只是Android提供的Components中的两种， 除此之外还有Content Provider和Broadcast Receiver）都会跑在这个Process。一个Android 程序默认情况下也只有一个Process，但一个Process下却可以有许多个Thread。

在这么多Thread当中，有一个Thread，我们称之为UI Thread。
UI Thread 在 Android 程序运行的时候就被创建，是一个 Process 当中的主线程 MainThread， 主要是负责控制 UI界面的显示、更新和控件交互。

在 Android 程序创建之初，一个 Process 呈现的是单线程模型，所有的任务都在一个线程中运行。因此，我们认为，UI Thread 所执行的每一个函数，所花费的时间都应该是越短越好。而其他比较费时的工作（访问网络，下载数据，查询数据库等），都应该交由子线程去执行，以免阻塞主线程。

那么，UI Thread 如何和其他 Thread 一起工作呢？常用方法是：
1.诞生一个主线程的 Handler 物件，当做 Listener 去让子线程能将讯息 Push 到主线程的 MessageQuene 里，以便触发主线程的 handlerMessage（）函数，让主线程知道子线程的状态，并在主线程更新UI。

2.在子线程的状态发生变化时，我们需要更新UI。
如果在子线程中直接更新UI,通常会抛出下面的异常：
ERROR/JavaBinder(1029):android.view.ViewRoot$CalledFromWrongThreadException:Only the original thread that created a view hierarchy can touch its views.
意思是，无法在子线程中更新UI。

3.我们需要通过 Handler 物件，通知主线程 Ui Thread 来更新界面。
```

* 每个应用程序对应着一个 ActivityThread 实例，应用程序由 ActivityThread.main 打开消息循环。每个应用程序同时也对应着一个ApplicationThread对象。该对象是 activityThread 和 ActivityManagerService 之间的桥梁。


* 在attach中还做了一件事情，就是通过代理调用 attachApplication，并利用 binder 的 transact 机制，在ActivityManagerService 中建立了 ProcessRecord 信息。


* ActivityManagerService 和 ActivityStack 位于同一个进程中，而 ApplicationThread 和 ActivityThread 位于另一个进程中

1. 在客户端和服务端分别有一个管理 activity 的地方，服务端是在 mHistory 中，处于 mHistory栈顶 的就是当前处于 running 状态的 activity，客户端是在 mActivities 中。
2. 在 startActivity 时，首先会在 ActivityManagerService 中建立 HistoryRecord ，并加入到 mHistory 中，然后通过 scheduleLaunchActivity 在客户端创建 ActivityRecord 记录并加入到 mActivities 中。最终在ActivityThread 发起请求，进入消息循环，完成 activity 的启动和窗口的管理等；

* ActivityThread 功能：它管理应用进程的主线程的执行(相当于普通Java程序的main入口函数)，并根据 AMS 的要求（通过IApplicationThread接口，AMS为Client、ActivityThread.ApplicationThread 为 Server）负责调度和执行activities、broadcasts 和其它操作。

