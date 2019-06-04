[TOC]



# Android的消息机制

## 简介

* 先了解几点东西；
  1. Handler 是 Android 消息机制的上层接口，这使得开发中只需要和 Handler 交互即可；
  2. 通过 Handler 可以轻松地将一个任务切换到Handler所在的线程中去执行；
  3. Android 的消息机制主要是指 Handler 的运行机制，Handler 的运行需要底层的 MessageQueue 和 Looper 的支持；
  4. MessageQueue：消息队列（内部其实是常用单链表的数据结构），它只是一个消息的储存单元，不能处理数据
  5. Looper：消息循环，它处理 MessageQueue 的数据，以无限循环的形式去查找有没有新消息，没有就等待；线程默认是没有 Looper 的（只有 UI 线程（即ActivityThread）默认可以使用，因为 ActivityThread 被创建时就会初始化 Looper ），所以需要使用 Handler 就必须为线程创建 Looper（除了UI线程）；
  6. Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，通过 ThreadLocal 来获取每个线程的 Looper
  7. ThreadLocal：它可以在不同线程之间互不干扰地存储并提供数据，所以通过 ThreadLocal 可以很轻松来获取每个线程的 Looper；
  8. Handler的主要作用是将一个任务切换到某个指定的线程去执行；由于 Android 的 UI 控件不是线程安全的，如果在 多线程中并发访问可能会导致UI控件处于不可预期的状态；那看到这里，可能就会有疑惑，既然 UI控件 不是线程安全的，那加上锁机制不就可以了，为什么还要大费周章来搞出这个 Handler ？这是因为加上锁机制会让UI访问的逻辑变得复杂，还会降低 UI 访问的效率（大问题），因为锁机制会阻塞某些线程的执行；所以最简单最高效的方法就是采用单线程模型来处理 UI 操作，对于开发者只需要通过 Handler 切换到 UI 的执行线程即可；
* Android的消息机制概述
  * Android 的消息机制主要是 Handler 的运行机制以及 Handler 所附带的 MessageQueue 和 Looper 的工作过程，这三者是一个整体；
  * Handler 创建时就会采用当前线程的 Looper 来构建内部的消息循环系统，如果当前线程没有 Looper，就会报错；（在 Handler 的工作流程的分析总结中解释）解决方法就是为当前线程创建 Looper 即可，或者在一个有Looper 的线程中去创建 Handler 也行；
  * 其实 Android 是事件驱动的，每个事件都会转化为一个系统消息，即 Message 。消息中包含了事件的信息以及整个消息的处理人——Handler；每个进程都有一个默认的消息队列，也就是我们的 MessageQueue，这个消息队列维护了一个待处理的消息列表，有一个消息循环不断地从这个队列中取出消息，处理消息，这样就使得应用动态地运作起来；
  * Handler 创建完毕后，这个时候其内部的 Looper 以及 MessageQueue 就可以和 Handler 一起协同工作，需要切回 handler 所在的线程时，通过 Handler 的 post 方法将一个 Runnable 投递到 Handler 内部的 Looper 中去处理，也可以通过 send 方法发送一个消息，这个消息同样会在 Looper 中去处理。（post 方法最终也是通过 send方法来完成的）；send 方法的工作流程是：当 send 方法被调用时，它会调用 MessageQueue 的enqueueMessage 方法将这个消息放入消息队列中，然后 Looper 发现有新消息到来时，就会处理这个消息，最终消息中的 Runnable 或者 Handler 的 handleMessage 方法就会被调用。（由于 Looper 是运行在创建 Handler 所在的线程中，这样一来 Handler 中的业务逻辑就被切换到创建 Handler 所在的线程中去执行）

## ThreadLocal的工作原理

### 简介

* ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，存储数据后，只有在指定线程中可以获取到存储的数据，对于其他线程来说，则无法获取到它的数据；
* 当某些数据是以线程为作用域，并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal
* 对于 Handler 来说，他需要获取当前线程的 Looper，很显然 Looper 的作用域就是线程并且不同线程具有不同的Looper，这个时候通过 ThreadLocal 就可以轻松实现 Looper 在线程中的存取；

### 分析总结

* ThreadLocal是一个泛型类：public class ThreadLocal<T>

* 在不同线程中访问同一个 ThreadLocal 对象时，它们通过 ThreadLocal 获取到的值却是不一样的！！！这就是ThreadLocal 的奇妙之处；

* ThreadLocal 之所以有这么奇妙的效果，是因为不同线程访问同一个 ThreadLocal 的 get() 方法，ThreadLocal 内部会从各自的线程中取出一个数组，然后再从数组中根据当前 ThreadLocal 的索引去查找出对应的 value 值；

* 那么，这个各个线程中的数组具体是什么数组呢？其实，在 Thread 类的内部有一个成员专门用于存储线程的ThreadLocal 的数据：ThreadLocal.Values  localValues；在 localValues 中有一个数组：private  Object[ ]  table，ThreadLocal 的值就存在这个 table 数组中；

  ~~~java
  //首先先看一下ThrealLocal的set方法；
  public void set(T value){
    Thread currentThread = Thread.currentThread();//获取当前线程的引用
    Values values = Values(currentThread);//获取当前线程中专门存储数据的一个成员
    if(values == null){
    	values = initializeValues(currentThread);
    }
    values.put(this,value);//将value的值put在values中；这里就不去分析put方法了，只需要知道是把值put在						  values中的table数组中
  }
  ~~~

* 分析了 ThreadLocal 的 set 方法得出，数据存储进这个 table 数组的一个存储规则是：那就是 ThreadLocal 的值在table 数组中的存储位置总是 ThreadLocal 的 reference（参照，索引）字段所标识的对象的下一个位置；比如ThreadLocal 的 reference 对象在 table 数组中的索引为 index，即 table [ index + 1 ] = value;

* 分析了 ThreadLocal 的 get 方法得出，ThreadLocal 的 get 方法可以取出 ThreadLocal 的值，规则是先找到ThreadLocal 的 reference 对象在 table 数组中的位置，然后 table 数组中的下一个位置所存储的数据就是ThreadLocal 的值；

* 从 ThreadLocal 的 set 和 get 方法可以看出它们所操作的对象都是当前线程的 localValues 对象的 table 数组，因此在不同线程中访问同一个 ThreadLocal 的 set 和 get 方法，它们对 ThreadLocal 所做的读/写操作仅限于各自线程的内部，这就是为什么 ThreadLocal 可以在多个线程i中互不影响地存储和修改数据；

## MessageQueue 的工作原理

### 简介

* MessageQueue 也叫做消息队列，虽然叫队列，但是它的内部实现并不是用队列，实际上是通过一个单链表来维护消息列表，这是因为 MessageQueue 主要包括两个操作：插入和读取（读取本身会伴随着删除操作），而单链表在插入和删除上比较有优势；
* 在 Looper 的构造方法中被创建；
* 插入和读取对应的方法分别为 enqueueMessage 和 next
  * enqueueMessage：往 MessageQueue 中插入一条消息；
  * next：从 MessageQueue 取出一条消息并将其移除；

### 分析总结

* 分析 MessageQueue 主要从 enqueueMessage 和 next 这2个方法下手即可；
* enqueueMessage主要操作就是单链表的插入操作，没什么好分析的；
* next 方法是一个无限循环的方法，如果队列中没有消息，那么 next 方法会一致阻塞在这里，当有新消息到来时，next 方法会返回这条消息并将它从单链表移除；


## Looper的工作原理

### 简介

* Looper：消息循环
* Looper 它会不停的从 MessageQueue 中查看是否有新消息，如果有新消息就会立刻处理，否则就一致阻塞在那里；
* 每个线程只能拥有一个 Looper，它的 looper()方法负责循环读取 MessageQueue 中的消息并将读取到的消息交给发送该消息的 handler 进行处理。
* Android 中的 Looper 类，是用来封装消息和循环消息队列的一个类，用于在 android 线程中进行消息处理。handler事实上能够看做是一个工具类。用来向消息队列中插入消息的。
* 一个线程仅仅能有一个 Looper。相应一个 MessageQueue。

### 分析总结

* 首先看一下 Looper 的构造器；

  ~~~java
  private Looper（boolean quitAllowed）{//私有构造，也就是说不能直接new
      mQueue = new MessageQueue(quieAllowed);//在创建Looper时会创建一个与之关联的MessageQueue对象。构造器是private修饰的,所以用户是无法创建Looper对象的，就是说创建Looper同时就创建了MessageQueue对象
      mThread = Thread.currentThread();
  }
  ~~~

  前面说 Handler 作用有两个---发送消息和处理消息,Handler 发送的消息必须被送到指定的 MessageQueue ,也就是说,要想 Handler 正常工作必须在当前线程中有一个 MessageQueue ,否则消息没法保存。而 MessageQueue 是由Looper 负责管理的,因此要想 Handler 正常工作,必须在当前线程中有一个 Looper 对象,这里分为两种情况:

  1. 主线程(UI线程),系统已经初始化了一个 Looper 对象,因此程序直接创建Handler即可；
  2. 自己创建的子线程,这时,必须创建一个 Looper 对象,并启动它。

* 既然 Looper 的构造器是私有的，那么它怎么创建这个对象呢？

  通过 Looper.prepare( )：即可为 当前线程 创建一个 Looper；

  接着**通过 Looper.loop() 来开启消息循环**，只有调用了 loop 之后，消息循环系统才会真正起作用；

  ~~~java
  new Thread("Thread#1"){
      @Ovrride
      public void run(){
          Looper.prepare();//为当前线程 创建一个Looper；
          Handler handler = new Handler();
          Looper.loop();//这里是Looper类的一个静态方法，它是处理消息循环的；那么消息是怎么返回到创建Handler的那个线程中去执行呢？这里留个问题，下面解答
      }
  }.start(); 
  ~~~

* Looper 除了 prepare 方法外，还提供了 prepareMainLopper 方法，主要是给主线程（即ActivityThread）创建 Looper使用的，其本质也是通过 prepare 方法来实现的；由于主线程的 Looper 比较特殊，所以 Looper 提供了一个getMainLooper 方法，通过它可以在**任何地方**获取到主线程的 Looper 。

* Looper 的退出

  Looper提供了quit 和quitSafely 来退出一个Looper，区别是：quit 会直接退出Looper；而quitSafely 只是设定要给退出标记，然后把消息队列中的已有消息处理完毕后才安全地退出。（在子线程中在不需要的时候建议终止Looper）；

* Looper 退出后，通过Handler发送的消息会失败，这个时候 Handler 的 send 方法会返回false ；

* 在子线程中，如果手动为其创建了Looper ，那么在所有的事情完成以后应该调用 quit 方法来终止消息循环，否则这个子线程就会一直处于等待的状态，而如果退出 Looper 以后，这个线程就会立刻终止；

* Looper **最重要的就是这个 loop 方法**，只有调用了loop之后，消息循环系统才会真正起作用；

  ~~~java
  public static void loop(){
      final Looper me = myLooper();
      if(me == null){
          throw new RuntimeException("抛异常的信息就不写了");
      }
      final MessageQueuequeue = me.mQueue;
      Binder.clearCallingIdentity();
      final long ident = Binder.clearCallintIdentity();
      for(;;){
          Message msg = queue.next();//loop方法调用MessageQueue的next方法来获取新消息，而next方法是一个阻塞操作，当没有消息时，next方法会一致阻塞在那里，这才导致了loop方法一致阻塞在那里；
        //如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息；  
          if(mes == null){//当Looper的quit被调用时，Looper就会调用MessageQUeue的quit或者quitSafely方法来通知消息队列退出，当MessageQueue被标记为退出状态时，它的next方法就会返回null，此时Looper才会瑞出死循环；也就是说，Looper必须退出，否则loop方法就会无限循环下去；
              return;
          }
          Printer logging = me.Logging;
          if(logging != null){
              logging.pringln(">>>>> Dispatching to "+msg.target+" "+msg.callback+":"+msg.what);
          }
          msg.target.disparchMessage(msg);//msg.target就是发送这条消息的Handler对象；这样Handler发送的消息最终又交给它的dispatchMessage方法来处理，但这里不用的是，Handler的dispatchMessage方法是在创建Handler时所使用的Looper中执行的，这样就成功的将代码逻辑切换到指定的线程中去执行了；这里就回答了开头的问题；
          if(logging != null){
              logging.println("<<<<< Finished to "+msg.target +);
          }
          
          final long newIdent = Binder.clearCallingIdentity();
          if(ident != newIdent){
              Log.wtf(TAG+"log内容就不写了");
          }
          msg.recycleUnchecked();
      }
  }
  ~~~

## Handler的工作流程

### 简介

* Handler的工作主要包含消息的发送和接受过程；消息的发送可以通过post 的一系列方法以及send的一系列方法来实现，post的一系列方法最终是通过send的一系列方法来实现的；

### 分析总结

~~~java
//现在分析send方法
public final boolean sendMessage(Message msg){//追寻源码一步一步往下看
    return secdMessageDdlayed(msg,0);
}
public final boolean sendMessageDelayed(Message msg,long delayMillis){
    if(delayMillis < 0){
        delayMillis = 0;
    }
    return sendMessageAtTime(msg,SystemClock.uptimeMillis()+delayMillis);
}
public boolean secdMessageAtTime(Message msg,long uptimeMillis){
    MessageQueue queue = mQueue;
    if(queue == null){
        return false;
    }
    return enqueueMessage(queue,msg,uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue,Message msg,long upTimaMillis){
    msg.target = this;
    if(mAsynchronous){
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg,uptimeMillis);//可以看到，Handler 发送消息的过程仅仅是向 MessageQueue 中插入了一条消息；接着 MessageQueue 的 next 方法就会返回这条消息给 Looper ，Looper 收到消息之后就开始处理，处理后最终消息就会由 Looper 交由 Handler 处理（即调用 msg.target 的 dispatchMessage 方法,这里的mas.target 就是Handler，Looper分析中有写，msg.target就是发送这条消息的Handler对象，那么 向发送这条消息的Handler对象 的 MessageQueue 插入一条数据，那么向发送这条消息的线程中的 Looper 就会接受到这条数据，此时业务自然而然地 回到 Looper 所在的线程中）；调用后，此时Handler 就进入了处理消息的阶段；
}
~~~

~~~java
//dispatchMessage的实现如下：
public void dispatchMessage(Message msg){
    if(msg.callback != null){//msg.callback是一个Runnable对象，实际上就是Handler 的 post 方法所传递的Runnable参数；如果它为 null ,那么此方法直接结束
        handleCallback(msg);//如果Handler传递过来的是一个Runnable对象，直接去处理它即可，handleCallback它的逻辑一目了然，那你就错了，其实此时 msg 处理时并不是把它当成一个 线程对象，而是把它当成一个普通类的对象，然后我们就像使用一个普通方法一样使用 run() 方法，而不是开启线程，这点注意！即 msg.callback.run();
    }else {//如果Handler传递过来的不是一个Runnable对象，则
        if(mCallback != null){//先判断传递过来的是不是一个Callback对象，对于这个Callback对象看最下面
            if(mCallback.handleMessage(msg)){//如果传递过来的确实是一个Callback对象，那么就去处理消息
                return ;
            }
        }
        handleMessage(msg);////如果传递过来的不是一个 Callback 对象或者 mCallback.handleMessage(msg)返回 false ，最后去调用 handleMessage 方法最终去处理消息即可；
    }
}
//这个Callback是一个接口，用来创建一个Handler的实例但不需要派生Handler的子类：Handler handler = new Handler(callback);即安卓给我们提供了另外一种使用Handler的方式；
public interface Callback{
    public boolean handleMessage(Message msg);
}
//ps：在日常开发中，创建Handler最常见的方式就是派生一个Handler的子类并重写其handleMessage方法来处理具体的消息；
~~~

* 下面是解决疑问的时候，在没有Looper的子线程中创建Handler 会引发程序异常；

  ~~~java
  //Handler的一个默认构造方法 public Handler(),这个构造方法会调用下面的构造方法；
  public Handler(Callback callback , boolean async){
      ...
      mLooper = Looper.myLooper();
      if(mLooper == null){
          throw new RuntimeException("Can't create handler inside thread that has not called 	  Looper.prepare()");
      }
      mQueue = mLooper.mQueue;
      mCallback = callback;
      mAsynchronous = async;
  }
  ~~~

## 主线程的消息循环

## 分析

* 安卓的主线程定如就是ActivityThread，我们在java中的入口函数main就在这个里面，在 main 方法中系统会通过 Looper.prepareMainLooper()来创建主线程的 Looper 以及 MessageQueue，并通过Looper.loop()来开启主线程的消息循环；

  ~~~java
  public static void main(String args){
      '''
      Process.setAtgV0("<pre-initialized>");
      
      Looper.preparedMainLooper();
      
     // 事实上，会在进入loop()死循环之前便创建了新binder线程：ActivityThread实际上并非线程，不像HandlerThread类，ActivityThread并没有真正继承Thread类，只是往往运行在主线程，该人以线程的感觉，其实承载ActivityThread的主线程就是由Zygote fork而创建的进程。
      ActivityThread thread = new ActivityThread();
      //建立 Binder 通道（具体是指ApplicationThread，Binder的服务端，用于接收系统服务AMS发送来的事件），该 Binder 线程通过 Handler 将 Message 发送给主线程) 简单说Binder用于进程间通信；
      thread.attach(false);
      
      if(sMainThreadHandler == null){
          sMainThreadHandler = thread.getHandler();
      }
      
      AsyncTask.init();
      
      if(false){
          Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG,"ActivityThread"));
      }
      Looper.loop();
      throw new RuntimeException("Main thread loop unexpectedly exited");
  }
  ~~~

* 主线程的消息循环开始了以后，ActivityThread 肯定还需要一个Handler 来和消息队列进行交互，这个Handler 就是 ActivityThread.H，它的内部定义了一组消息类型，主要包含了四大组件的启动和停止等过程；

  ~~~java
  private class H exTends Handler{
      public static final int LAUNCH_ACTIVITY = 100;
      public static final int PAUSE_ACTIVITY  = 101;
      '''//还有很多
  }
  ~~~


## 特殊要点

* ActivityThread 通过 ApplicationTread.H 和 AMS 进行通信，AMS 以进程间通信的方式完成 ActivityThread 的请求后会回调 ApplicationThread 中的 Binder 的方法，然后 ApplicationThread 会向 H 发送消息，H收到消息后会将ApplicationThread 中的逻辑切换到ActivityThread 中去执行，即切换到主线程中去执行，这个过程就是主线程的消息循环模型；

* **Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施：**
  在H.handleMessage(msg)方法中，根据接收到不同的msg，执行相应的生命周期。

  ​    比如收到msg=H.LAUNCH_ACTIVITY，则调用ActivityThread.handleLaunchActivity()方法，最终会通过反射机制，创建Activity实例，然后再执行Activity.onCreate()等方法；

  ​    再比如收到msg=H.PAUSE_ACTIVITY，则调用ActivityThread.handlePauseActivity()方法，最终会执行Activity.onPause()等方法。 上述过程，我只挑核心逻辑讲，真正该过程远比这复杂。

  **主线程的消息又是哪来的呢？**当然是App进程中的 其他线程 通过 Handler 发送给主线程，

* Binder用于不同进程之间通信，由一个进程的Binder客户端向另一个进程的服务端发送事务；而handler用于同一个进程中不同线程的通信；

* 结合图说说Activity生命周期，比如暂停Activity，流程如下：

  ![](https://pic1.zhimg.com/80/7fb8728164975ac86a2b0b886de2b872_hd.jpg)


1. 线程1（Binder服务端）的AMS中调用线程2（Binder客户端）的ATP；（由于同一个进程的线程间资源共享，可以相互直接调用，但需要注意多线程并发问题）
2. 线程2通过binder传输到App进程的线程4；
3. 线程4通过handler消息机制，将暂停Activity的消息发送给主线程；
4. 主线程在looper.loop()中循环遍历消息，当收到暂停Activity的消息时，便将消息分发给ActivityThread.H.handleMessage()方法，再经过方法的调用，最后便会调用到Activity.onPause()，当onPause()处理完后，继续循环loop下去。


## 为什么主线程不会因为Looper.loop()里的死循环卡死

* 简单一句话是：Android应用程序的主线程在进入消息循环过程前，会在内部创建一个Linux管道（Pipe），这个管道的作用是使得Android应用程序主线程在消息队列为空时可以进入空闲等待状态，并且使得当应用程序的消息队列有消息需要处理时唤醒应用程序的主线程。

  这一题是需要从消息循环、消息发送和消息处理三个部分理解Android应用程序的消息处理机制了，这里我对一些要点作一个总结：

  A. Android应用程序的消息处理机制由消息循环、消息发送和消息处理三个部分组成的。

  

  B. Android应用程序的主线程在进入消息循环过程前，会在内部创建一个Linux管道（Pipe），这个管道的作用是使得Android应用程序主线程在消息队列为空时可以进入空闲等待状态，并且使得当应用程序的消息队列有消息需要处理时唤醒应用程序的主线程。阻塞在loop的queue.next()中，而message.next()会阻塞在nativePollOnce（）方法中，当有消息到达消息队列时，比如调用了enqueueMessage方法，这个方法里面会调用到nativeWake(mPtr);方法，这个方法会往管道中的写端写入内容，此时阻塞住的nativePollOnce（）方法监听到管道发生了事件，然后就唤醒唤醒正在等待消息到来的应用程序主线程。（这里的消息可能是SystemServer进程或者别的进程发的或者Binder线程发的）

  
  
  C. Android应用程序的主线程进入空闲等待状态的方式实际上就是在管道的读端等待管道中有新的内容可读，具体来说就是是通过Linux系统的Epoll机制中的epoll_wait函数进行的。
  
  
  
   D. 当往Android应用程序的消息队列中加入新的消息后，会同时调用方法往管道中的写端写入内容，通过这种方式就可以唤醒正在等待消息到来的应用程序主线程。






