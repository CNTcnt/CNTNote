[TOC]

# CNTNote
* 听闻室友硬盘坏掉了然后换硬盘所有东西都没有了，所以把电脑里的重要文档备份在 github 上

## 学习历程

### 看过一些书，虽然大部分都忘记了，很惭愧
![](https://wx3.sinaimg.cn/mw690/007duwLrgy1g3owq8bdbrj31400u0wlu.jpg)
* 第一行代码就是入门书了（我也忘记了什么时候买的了，好像是大二的开头或者大一的暑假），其他一些都是后面感兴趣买的，里面有两本在某宝上买二手书然后给我送过来了盗版书，很生气，纸质太差，里面一本书的侧面封面还是我自己填上去的（书里还没有页码，都是我自己填上去的，说多了都是泪），不过也凑合着看。
* 大二是在迷茫的探索阶段，就不多说了（虽然现在也很迷茫）
* 大三上学期看的东西很杂，像设计模式就看了蛮久时间，虽然领会的不是很多（可能因为只看了一遍）所以笔记懒得做直接写在书上了（打字比较慢所以没做）。
* 这里说一下，Android 是一个 OS ，如果不止做C端的应用开发，比如做 framework 层开发，Android 系统开发，学习难度其实蛮大的，因为等于去学一个操作系统去怎么运作的，比如一个应用程序是怎么启动的，开机过程中做了什么，你的每一下点击屏幕都会触发点击事件，那么这个点击事件是怎么产生的，怎么分发的，怎么传递到我们的应用程序的窗口的，而窗口是怎么知道有 Touch 事件发生了，怎么接收，怎么分发，有时候还会接触到驱动，比如 Binder 机制。

### 到大三下学期就比较努力一些
* 开始去了解一下一些开源框架的源码(Android源码太复杂了，去一些开源框架放松一下)，比如 EventBus，okHttp3 什么的
* 也了解了一些 framework 层的东西，比如 我们应用层所使用的 handler 机制其实也有 jni 层的参与，要理解整个 handler 机制就要了解 framework；比如开机过程中做了什么准备工作，启动了哪些进程，ANdroid 第一个进程如何创建。再比如一个应用程序的创建过程，从利用 AMS 通过 Zygote 在 Java 框架层中创建的 Socket 通知 Zygote 来创建新的应用程序，从创建进程开始一步一步都要了解；再比如一个 Touch 事件如何发生（关于发生部分应该在驱动，我没有去探索），如何分发到应用程序的窗口都是在 framework 完成的，应用程序的窗口是如何接收到的也是一个可以探究的问题。总的来说学无止境，因为要学的真的太多了。
* 对于 Java 这一部分也感觉到了自身的不足，所以去了解了 Java 虚拟机，内存模型，动态代理，反射，线程池，GC机制，底层类的实现原理，类加载机制等等。
* 对于 线程安全也了解了一些。还有一些杂七杂八的东西只是想了解一下而已。
* 暂时就这么多，溜了

### 小结
* 学的深入一些些，就感觉其实基础是最重要的（以前一直听别人讲基础很重要，不以为意，现在发现有些感受是要有自己的体验才深有所感的），感同身受我认为只有自身真的经历过同样的事情，才能对这件事件有所感触，才能感同身受。
* 现在觉得：操作系统，计算机网络，数据结构真的很重要，基本功。
