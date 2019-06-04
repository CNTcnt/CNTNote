[TOC]

# 从Android设计模式源码分析再来一次Hanlder

## 概述分析

- 其实 Android 是事件驱动的，每个事件都会转化为一个系统消息，即 Message 。消息中包含了事件的信息以及整个消息的处理人——Handler；每个进程都有一个默认的消息队列，也就是我们的 MessageQueue，这个消息队列维护了一个待处理的消息列表，有一个消息循环不断地从这个队列中取出消息，处理消息，这样就使得应用动态地运作起来；

- 我们从使用方法入手

  ```java
  Handler handler = new Handler(Loop.getMainLopper());//创建一个 Looper 是 UI 线程的 Handler
      private void doSomeThing(){
          new Thread(){
              @Override
              public void run() {
                  //在子线程可以进行耗时操作
                  handler.post(new Runnable() {//需要进行 UI 操作时就要切回 UI 线程
                      @Override
                      public void run() { 
                      //在这里直接进行 UI 操作
                      }
                  });
              }
          };
      }
  ```

- 在上面我们通过 Handler post 了一个 Runnable 给 UI 线程；我们进去看一下；

  ```java
  public final boolean post(Runnable r){
         return  sendMessageDelayed(getPostMessage(r), 0);//其实也是通过 sendMessage方式；只是在send 时需要把 Runnable 转化为需要的 Message,使用 getPostMessage(r) 来转化，不管 post 的是一个 Runnable 还是一个 Message ，都会调用 sendMessageDelayed 方法；
  }
  private static Message getPostMessage(Runnable r) {//将 Runnable 对象包装成 Message 对象
          Message m = Message.obtain();
          m.callback = r;//把 Runnable 对象放到 Message 对象的 callback 参数里面
          return m;
  }
  ```

  ```java
  public final boolean sendMessageDelayed(Message msg, long delayMillis){
          if (delayMillis < 0) {
              delayMillis = 0;
          }
          return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
  }
  ```

  ```java
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
          MessageQueue queue = mQueue;//获取当前 Handler 所在的消息队列，那么这个 mQueue 怎么来的呢，
      //mQueue = mLooper.mQueue;//就是这么来的
      //其实它就是 当前 Handler 对应的 Looper 创建时就会跟着创建的 queue；
      
      //  private Looper(boolean quitAllowed) {
      //      mQueue = new MessageQueue(quitAllowed);
      //      mThread = Thread.currentThread();
      //  }
      
          if (queue == null) {
              RuntimeException e = new RuntimeException(
                      this + " sendMessageAtTime() called with no mQueue");
              Log.w("Looper", e.getMessage(), e);
              return false;
          }
          return enqueueMessage(queue, msg, uptimeMillis);
  }
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
          msg.target = this;//这里注意，target 其实就是发送消息的 Handler；在 Message 对象里有一个变量 /*package*/ Handler target;实际上就是转了一圈，通过 Handler 将消息传递给 消息队列，而消息队列又会将消息分发给发送这条消息的 Handler 处理（看下面的代码）；这里是一个典型的命令模式
          if (mAsynchronous) {
              msg.setAsynchronous(true);
          }
          return queue.enqueueMessage(msg, uptimeMillis);
      }
  ```

* 这里是很重要的操作，在 sendMessageAtTime 方法中，先判断当前 Handler 的消息队列是否为空，如果不为空那么就会将消息追加到消息队列中，即 enqueueMessage(queue, msg, uptimeMillis); 此时注意，参数里的 queue 刚才已经分析过，这里提个神；我们的 Handler 在创建时就关联了 UI 线程的 Looper ，Handler 在发送消息时，是从这个 Looper 中获取 queue 哦，那么，发送的 Message 或 Runnable 就会被放到 UI 线程的 queue ，此时，我们的 Message 或 Runnable 在后续的某个时刻就会在 UI 线程里面执行了，此时应该就解开了疑惑：为什么在别的线程发送的消息可以在 UI 线程执行；（如果不手动传递 Looper ，那么 Handler 所持有的 Looper 就是当前线程的 Looper ，也就是说在哪个线程创建的 Handler ，就是哪个线程的 Looper）；

* ~~~java
  public void dispatchMessage(Message msg) {
          if (msg.callback != null) {
              handleCallback(msg);//如果 msg 的 Runnable 类型的 callback 参数为空，那么 Handler 就不是使用post(Runnable run) 方法来发送消息的，如果不为空，那么就会执行 handlerCallback，该方法会调用 calback 的 run 方法；
          } else {
              if (mCallback != null) {//这里提及另一种使用 Handler 的方式，就是：Handler handler = new Handler(Callback callback);把实现了 Callback 接口对象当作参数创建 Handler；如果是使用这种方式，就会触发下面的 mCallback.handleMessage(msg) 
                //public interface Callback{
    			 //		public boolean handleMessage(Message msg);
  			 //}
                  if (mCallback.handleMessage(msg)) {
                      return;
                  }
              }
              handleMessage(msg);//如果不是上面的2种，那么就是平常最熟悉的派生一个Handler的子类的方式，就会调用这个；
          }
  }
  ~~~

* ​

  ​

## 小结

* 消息通过 Handler 投递到消息队列，这个消息队列在 Handler 关联的 Looper 中，消息循环启动之后会不断地从队列中获取消息，获取消息之后会调用消息的 callback 或者分发给对应的 Handler 的 handleMessage 方法进行处理，这样就将消息，消息的分发，处理隔离开来，降低各个角色之间的耦合；