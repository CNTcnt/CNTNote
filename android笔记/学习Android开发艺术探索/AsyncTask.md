[TOC]



# AsyncTask

## 简介

* 异步任务类，主要是处理不推荐在主线程中执行的任务，处理完后便于访问主线程；
* 后台任务是在线程池中执行，然后把执行的结果（进度也可以）传递给主线程便于在主线程中更新 ui ;
* 其实 AsyncTask 就是一个异步任务类。封装了：通过线程池来执行耗时的任务，得到结果后，通过 Handler 将结果传递到 UI 线程 来执行；

## 使用方法

* AsyncTask 是一个抽象的泛型类，它提供了 Params（参数类型），Progress（需要传递出来给外界的进度的参数），Result（最终需要传递出来的结果的参数） 这3个泛型参数，实际使用时可以根据需要，不需要的用 void
* 使用时可以覆写4个核心方法,
  1.  onPreExecute()：最先在主线程中被调用，可以用于做一些准备工作
  2. doInBackground(Params...params)：在线程池中执行后台异步任务，执行途中如果想要给外界传递比如此时的任务进度，可以调用 publishProgress 方法来更新（调用了它之后就它会去调用 onProgressUpdate 方法）
  3. onProgressUpdate(Progress...values)：这个方法当调用了 publishProgress 方法会被调用，需要注意的是这个方法是在主线程中执行的，所以可以更新 ui；至于为什么在主线程中了等等看原理分析
  4. onPostExecute(Result result)：在主线程中执行，当异步任务执行完成后会被调用，参数当然是doInBackground的返回值
* 其实还有一个比较重要的方法可以覆写：onCancelled()，它在主线程中运行；搭配 isCancelled() 方法，这个方法返回异步任务是否被外界人为取消掉了，如果是，则返回 true，此时 onCancelled() 就会被调用，可以进行 ui 操作了
* 科普：Java中 ... 表示参数的数量补丁，它是一种数组型参数；

## 使用遵守规则

* 使用的时候有一些规则是需要遵守的；这里不说为什么这么做，在分析原理的时候解释；
  1. AsyncTask 的类必须在主线程加载，这里不用担心，因为在安卓4.1后就由系统自动完成了，可以查看 ActivityThread 的 main 方法，它会调用 AsyncTask 的 init 方法；
  2. AsyncTask 的对象必须在主线程中创建；
  3. execute 方法必须在 主线程中调用；
  4. 不可以在程序中直接去调用上面我们覆写的方法，这些方法是 AsyncTask 框架自己去调用执行的，不要认为调用
  5. 一个 AsyncTask 对象只能调用一次 execute 方法；
  6. 在 Android 3.0 后采用一个线程来串行执行任务，如果需要并行执行任务，则是调用 executeOnExecutor 方法；（串行队列：串行队列的特点是队列内的线程是一个一个执行，直到结束。并行队列：并行队列的特点是队列中所有线程的执行结束时必须是一块的，队列中其他线程执行完毕后，会阻塞当前线程等待队列中其他线程执行然后一块执行完毕。）

## 源码解释

* 我比较喜欢先去构造方法去看

  ~~~java
  public AsyncTask(@Nullable Looper callbackLooper) {
    		//拿到 UI 线程 handler，为什么要拿呢？当然是我们这个 AsyncTask 用来执行异步任务的，执行完你就不想去 UI 展示一下你的执行成果呀
          mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
              ? getMainHandler()
              : new Handler(callbackLooper);
  		//先把异步任务执行过程封装，使用时调用 call 方法即可
          mWorker = new WorkerRunnable<Params, Result>() {
              public Result call() throws Exception {
                  mTaskInvoked.set(true);
                  Result result = null;
                  try {
                      Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                      //noinspection unchecked我们要做的任务我们覆写在 doInBackground 的任务
                      result = doInBackground(mParams);
                      Binder.flushPendingCommands();
                  } catch (Throwable tr) {
                      mCancelled.set(true);
                      throw tr;
                  } finally {
                      postResult(result);
                  }
                  return result;
              }
          };
  		//再封装成 FutureTask 
          mFuture = new FutureTask<Result>(mWorker) {
              @Override
              protected void done() {
                  try {
                      postResultIfNotInvoked(get());
                  } catch (InterruptedException e) {
                      android.util.Log.w(LOG_TAG, e);
                  } catch (ExecutionException e) {
                      throw new RuntimeException("An error occurred while executing doInBackground()",
                              e.getCause());
                  } catch (CancellationException e) {
                      postResultIfNotInvoked(null);
                  }
              }
          };
  }

  private void postResultIfNotInvoked(Result result) {
          final boolean wasTaskInvoked = mTaskInvoked.get();
          if (!wasTaskInvoked) {
              postResult(result);
          }
  }

  private Result postResult(Result result) {
          @SuppressWarnings("unchecked")
          Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                  new AsyncTaskResult<Result>(this, result));
          message.sendToTarget();
          return result;
  }
  ~~~


* 再看一些静态的变量意义

  ~~~java
  //线程池，目前不知道它干啥用
  public static final Executor THREAD_POOL_EXECUTOR;
  static {
          ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                  CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                  sPoolWorkQueue, sThreadFactory);
          threadPoolExecutor.allowCoreThreadTimeOut(true);
          THREAD_POOL_EXECUTOR = threadPoolExecutor;
  }
  //看样子也是个线程池，为啥子有2个线程池嘞？
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
  private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
  //Java语言提供了一种稍弱的同步机制，即volatile变量，用来确保将变量的更新操作通知到其他线程。当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。
  ~~~

  ​

* 带着上面这些疑问我们现在再去它的 execute 方法去看

  ~~~java
  @MainThread
  public final AsyncTask<Params, Progress, Result> execute(Params... params) {
      return executeOnExecutor(sDefaultExecutor, params);
  }
  ~~~

  ~~~java
  public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
              Params... params) {
          if (mStatus != Status.PENDING) {//这里控制了一个 AsyncTask 对象只能执行一次 execute
              switch (mStatus) {
                  case RUNNING:
                      throw new IllegalStateException("Cannot execute task:"
                              + " the task is already running.");
                  case FINISHED:
                      throw new IllegalStateException("Cannot execute task:"
                              + " the task has already been executed "
                              + "(a task can be executed only once)");
              }
          }
  
          mStatus = Status.RUNNING;
  //onPreExecute()此时是运行在主线程中的，我们覆写它可以写一些准备去执行异步任务前的操作
          onPreExecute();
  //把我们传进来的任务需要的参数给封装好的 mWorker
          mWorker.mParams = params;
  //再把 mFuture 任务给 sDefaultExecutor（SerialExecutor()对象）去执行
          exec.execute(mFuture);
          return this;
  }
  ~~~

* 此时我们要到 SerialExecutor 的 execute 去一探究竟

  ~~~java
  // Executor 其内部使用了线程池机制，它在java.util.cocurrent 包下，通过该框架来控制线程的启动、执行和关闭，可以简化并发编程的操作。因此，在Java 5之后，通过Executor来启动线程比使用Thread的start方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免this逃逸问题——如果我们在构造器中启动一个线程，因为另一个任务可能会在构造器结束之前开始执行，此时可能会访问到初始化了一半的对象用Executor在构造器中。
  private static class SerialExecutor implements Executor {
    //首先创建了一个 Runnable 的双端队列，任务嘛，我认为如果是串行执行的话，当然是先到先得咯，用队列没错，而为什么要用容器来装载任务呢？很简单嘛，可能有多个任务执行呀；而我们不可能每次有一个任务就一个线程吧，多消耗资源哟，所以使用线程池，任务多的话就先用容器装起来咯
          final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
          Runnable mActive;
    
          public synchronized void execute(final Runnable r) {
            //先把任务先放进 mTasks 中
              mTasks.offer(new Runnable() {
                  public void run() {
                      try {
                          r.run();
                      } finally {
                          scheduleNext();
                      }
                  }
              });
            //再去 THREAD_POOL_EXECUTOR 这个线程池中执行，此时就回答了上面的为什么有2个线程池的问题，一个用于任务排队，一个用于真正的执行任务，因为我们可能有多个 AsyncTask 对象的不同任务要执行
              if (mActive == null) {
                  scheduleNext();
              }
          }
  
          protected synchronized void scheduleNext() {
              if ((mActive = mTasks.poll()) != null) {
                //此时就在线程池中执行我们的任务，此时我们就去我们的run方法看一下
                  THREAD_POOL_EXECUTOR.execute(mActive);
              }
          }
  }
  
  ~~~

  此时我们应该到的是 FutureTask 的 run 方法看，因为我们在构造的时候把我们的任务封装成 FutureTask 对象了

  ~~~java
  public void run() {
          if (state != NEW ||
              !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
              return;
          try {
              Callable<V> c = callable;
              if (c != null && state == NEW) {
                  V result;
                  boolean ran;
                  try {
                    //就是这里，这里调用了 call 方法，就是 我们在构造函数中的匿名内部类对象WorkerRunnable 对象
                      result = c.call();
                      ran = true;
                  } catch (Throwable ex) {
                      result = null;
                      ran = false;
                      setException(ex);
                  }
                  if (ran)
                      set(result);
              }
          } finally {
              // runner must be non-null until state is settled to
              // prevent concurrent calls to run()
              runner = null;
              // state must be re-read after nulling runner to prevent
              // leaked interrupts
              int s = state;
              if (s >= INTERRUPTING)
                  handlePossibleCancellationInterrupt(s);
          }
  }
  ~~~

  此时就跳到 我们在构造函数中创建的 WorkerRunnable 匿名内部类对象的 call 方法中去执行；

  ~~~java
  mWorker = new WorkerRunnable<Params, Result>() {
              public Result call() throws Exception {
                  mTaskInvoked.set(true);
                  Result result = null;
                  try {
                      Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                      //noinspection unchecked我们要做的任务我们覆写在 doInBackground 的任务
                      result = doInBackground(mParams);
                      Binder.flushPendingCommands();
                  } catch (Throwable tr) {
                      mCancelled.set(true);
                      throw tr;
                  } finally {
                    //当我们的任务执行完后就到这里
                      postResult(result);
                  }
                  return result;
              }
  };
  private Result postResult(Result result) {
          @SuppressWarnings("unchecked")
    //此时就到了我们在构造函数中创建的 handler 出场的时候了，切回主线程中
          Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                  new AsyncTaskResult<Result>(this, result));
    //private static class AsyncTaskResult<Data> {
    //      final AsyncTask mTask;
    //      final Data[] mData;
  
    //      AsyncTaskResult(AsyncTask task, Data... data) {
    //          mTask = task;
    //          mData = data;
    //      }
    //  }
          message.sendToTarget();
          return result;
  }
  private Handler getHandler() {
          return mHandler;
  }
  private static Handler getMainHandler() {
          synchronized (AsyncTask.class) {
              if (sHandler == null) {
                //我们的 handler 是 InternalHandler 对象
                  sHandler = new InternalHandler(Looper.getMainLooper());
              }
              return sHandler;
          }
  }
  private static class InternalHandler extends Handler {
          public InternalHandler(Looper looper) {
              super(looper);
          }
  
          @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
          @Override
          public void handleMessage(Message msg) {
              AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
              switch (msg.what) {
                  case MESSAGE_POST_RESULT:
                      // There is only one result执行最后的动作，当然是我们覆写的 onPostExecute 方法
                      result.mTask.finish(result.mData[0]);
                      break;
                  case MESSAGE_POST_PROGRESS:
                      result.mTask.onProgressUpdate(result.mData);
                      break;
              }
          }
  }
  
  private void finish(Result result) {
          if (isCancelled()) {
              onCancelled(result);
          } else {
            //就是这里，最后一步，此时当然已经在主线程中执行了
              onPostExecute(result);
          }
          mStatus = Status.FINISHED;
  }
  ~~~

* 开头我们说 doInBackground 的时候说了它可以中途返回任务进度一类的数据（当然，其他数据随便你），只要在doInBackground 中调用 publishProgress 方法，我们到方法内部看一下

  ~~~java
  @WorkerThread
      protected final void publishProgress(Progress... values) {
          if (!isCancelled()) {
              getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                      new AsyncTaskResult<Progress>(this, values)).sendToTarget();
          }
      }
  
  //此时看上面的 handler 就知道了
  result.mTask.onProgressUpdate(result.mData);
  ~~~

* 到这里我们的原理分析就结束了；吃饭去咯；

## 以模板模式来看待 AsyncTask

* 其实 AsyncTask 就是一个异步任务类。封装了：通过线程池来执行耗时的任务，得到结果后，通过 Handler 将结果传递到 UI 线程 来执行；

* 如果只是看了上面的源码分析，让你自己写一个 AsyncTask 是一个非常有难度的事情，但是如果你知道模板模式这个设计模式的话，设计一个 AsyncTask 也不是一个很难得事情；

* 使用 AsyncTask 时，就是继承它然后覆写几个方法，就是填代码，我们把耗时的方法放在 doInBackground 中，如果执行耗时任务前还想做一些初始化工作就实现在 onPreEecute ，当 耗时任务即 doInBackground 执行完后，就会执行 onPostExecute ，而我们覆写完方法后，只需创建 AsyncTask 对象调用 execute 方法即可，这就是典型的模板模式；我们可以看到，它整个执行流程其实是一个固定的框架，具体的实现都需要子类来完成，而且它执行的算法框架是固定的;

* 而 AsyncTask 无非就是流程封装，这是典型的模板模式，就是把某个固定的流程封装到一个 final 函数中，并且让子类能够定制这个流程中的指定步骤；

* 我们现在不去理代码细节，分析一下代码大致框架看看；

* 首先，我们调用 execute 方法

  ~~~java
  @MainThread
  public final AsyncTask<Params, Progress, Result> execute(Params... params) {
      return executeOnExecutor(sDefaultExecutor, params);
  }
  ~~~

  ~~~java
  //看到没有，一个 final 方法，就是不让你碰了，算法框架逻辑就在这里面
  @MainThread
      public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
              Params... params) {
          //在任务在running或者任务已经执行过了的时候，你再去同一个 asynctask 对象中调用 execute 方法是会抛异常的；
          if (mStatus != Status.PENDING) {
              switch (mStatus) {
                  case RUNNING:
                      throw new IllegalStateException("Cannot execute task:"
                              + " the task is already running.");
                  case FINISHED:
                      throw new IllegalStateException("Cannot execute task:"
                              + " the task has already been executed "
                              + "(a task can be executed only once)");
              }
          }
  
          mStatus = Status.RUNNING;
          //首先先执行我们覆写的 onPreExecute 
          onPreExecute();
  
          mWorker.mParams = params;
          //然后就肯定是在线程池 异步 地去执行我们的任务了呀，我们去构造方法中看一下这个mFuture
          exec.execute(mFuture);
  
          return this;
      }
  ~~~

* 构造方法，按顺序看下面注释

  ~~~java
  public AsyncTask(@Nullable Looper callbackLooper) {
          mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
              ? getMainHandler()
              : new Handler(callbackLooper);
      //3. mWorker 它在这里，call ，调用了我们需要覆写的 doInBackground 任务方法
          mWorker = new WorkerRunnable<Params, Result>() {
              public Result call() throws Exception {
                  mTaskInvoked.set(true);
                  Result result = null;
                  try {
                      Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                      //noinspection unchecked
                      result = doInBackground(mParams);
                      Binder.flushPendingCommands();
                  } catch (Throwable tr) {
                      mCancelled.set(true);
                      throw tr;
                  } finally {
                      //任务执行完后，就是执行 返回任务结果的方法了，我们去看看
                      postResult(result);
                  }
                  return result;
              }
          };
      //1. 看到没有，就是这个 mFuture，其实就是把 mWorker 对象再包了一层；我们先去 mFuture 去看一下它的 run 方法，再去看一下这个 mWorker，看名字就知道它是任务执行者了
          mFuture = new FutureTask<Result>(mWorker) {
              @Override
              protected void done() {
                  try {
                      postResultIfNotInvoked(get());
                  } catch (InterruptedException e) {
                      android.util.Log.w(LOG_TAG, e);
                  } catch (ExecutionException e) {
                      throw new RuntimeException("An error occurred while executing doInBackground()",
                              e.getCause());
                  } catch (CancellationException e) {
                      postResultIfNotInvoked(null);
                  }
              }
          };
      }
  
  //2. 这个是 FutureTask 对象的 run 方法；
  public void run() {
          if (state != NEW ||
              !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
              return;
          try {
              Callable<V> c = callable;
              if (c != null && state == NEW) {
                  V result;
                  boolean ran;
                  try {
                      //看到没有，执行 c.call()，这个 c 是啥呢？在构造方法中 this.callable = callable;这个 callable 就是 AsyncTask 构造方法中的 mWorker 对象，现在我们就去 它看看
                      result = c.call();
                      ran = true;
                  } catch (Throwable ex) {
                      result = null;
                      ran = false;
                      setException(ex);
                  }
                  if (ran)
                      set(result);
              }
          } finally {
              // runner must be non-null until state is settled to
              // prevent concurrent calls to run()
              runner = null;
              // state must be re-read after nulling runner to prevent
              // leaked interrupts
              int s = state;
              if (s >= INTERRUPTING)
                  handlePossibleCancellationInterrupt(s);
          }
      }
  
  //4. 任务结果有可能是需要在 UI 线程中显示的，所以我们需要借助 handler 去执行
  private Result postResult(Result result) {
          @SuppressWarnings("unchecked")
          Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                  new AsyncTaskResult<Result>(this, result));
          message.sendToTarget();
          return result;
  }
  //5. 这个 getHandler 是获取 InternalHandler 对象，
  private static class InternalHandler extends Handler {
          public InternalHandler(Looper looper) {
              super(looper);
          }
  
          @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
          @Override
          public void handleMessage(Message msg) {
              AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
              switch (msg.what) {
                  case MESSAGE_POST_RESULT:
                      // There is only one result
                      //执行到这里，在 UI 线程中 执行 finish 方法，这里 result.mTask 是 AsyncTask 对象，所以我们去 AsyncTask 对象的 finish 方法看看即可；
                      result.mTask.finish(result.mData[0]);
                      break;
                  case MESSAGE_POST_PROGRESS:
                      result.mTask.onProgressUpdate(result.mData);
                      break;
              }
          }
      }
  //6. 
  private void finish(Result result) {
          if (isCancelled()) {
              onCancelled(result);
          } else {
              //终于看到你了，在 UI 线程中执行我们覆写的 onPostExecute 方法，
              onPostExecute(result);
          }
          mStatus = Status.FINISHED;
  }
  ~~~

* 经过上面的分析，整个异步任务类已经很清楚地呈现，就是把异步任务类的执行算法框架用 final 方法封装起来：通过线程池来执行耗时的任务，得到结果后，通过 Handler 将结果传递到 UI 线程 来执行；然后开放异步任务类几个用户关心的重要步骤的方法给我们用户覆盖实现，用户根本不需要去知道这个任务怎么去异步执行就可以使用异步任务，简单好用；

* 模板模式真是形象好用；看完上面这个 模板模式 的实例后，重新写一个 AsyncTask 类也不是什么难事吧

