[TOC]



# BroadcastReceiver 的工作流程

# 简介

* ​

## 注册源码分析

* 由动态注册 BroadcastReceiver 的 方法进入,这个方法是在 Activity 的父类 ContextWrapper 中实现；

  ~~~java
  @Override
  public Intent registerReceiver(
          BroadcastReceiver receiver, IntentFilter filter) {
    //和 Service 一样，ContextWrapper 并没有做实际的事，而是交给 ContextImpl 对象去做，而 Contextimpl 对象的来源这和 Service 一样 
          return mBase.registerReceiver(receiver, filter);
  }
  ~~~

* 现在就到 ContextImpl 中

  ~~~java
  @Override
  public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
          return registerReceiver(receiver, filter, null, null);
  }
  @Override
  public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
              String broadcastPermission, Handler scheduler) {
          return registerReceiverInternal(receiver, getUserId(),
                  filter, broadcastPermission, scheduler, getOuterContext(), 0);
  }
  private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
              IntentFilter filter, String broadcastPermission,
              Handler scheduler, Context context, int flags) {
          IIntentReceiver rd = null;
          if (receiver != null) {
              if (mPackageInfo != null && context != null) {
                  if (scheduler == null) {
                      scheduler = mMainThread.getHandler();
                  }
                //一般是在这里获取 IIntentReceiver 对象 
                  rd = mPackageInfo.getReceiverDispatcher(
                      receiver, context, scheduler,
                      mMainThread.getInstrumentation(), true);
              } else {
                  if (scheduler == null) {
                      scheduler = mMainThread.getHandler();
                  }
                  rd = new LoadedApk.ReceiverDispatcher(
                          receiver, context, scheduler, null, true).getIIntentReceiver();
              }
          }
          try {
            //这里看到了熟悉的 AMS ，我们看到了是用 AMS 来 registerReceiver ，把我们的 广播接收器 BroadcastReceiver 进行注册。wtf,这里怎么没看到传进来的 receiver ，那还注册什么，不过这里的 rd 对象是什么，传这个干什么？原来我又忘了一点，与 AMS 的交互，是不可以随便传的，要用 Binder 来中转一下，把 receiver 封装成一个 Binder 使用；
              final Intent intent = ActivityManager.getService().registerReceiver(
                      mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                      broadcastPermission, userId, flags);
              if (intent != null) {
                  intent.setExtrasClassLoader(getClassLoader());
                  intent.prepareToEnterProcess();
              }
              return intent;
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
  }
  ~~~

* 现在到 LoadeApk 中去看看 的过程  mPackageInfo.getReceiverDispatcher();

  ~~~java
  public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
              Context context, Handler handler,
              Instrumentation instrumentation, boolean registered) {
          synchronized (mReceivers) {
              LoadedApk.ReceiverDispatcher rd = null;
              ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            //先看看 是不是已经注册过了
              if (registered) {
                  map = mReceivers.get(context);
                  if (map != null) {
                      rd = map.get(r);
                  }
              }
              if (rd == null) {
                //这个 rd 对象是 ReceiverDispatcher ，它的内部保存了 BroadcastReceiver 和 InnerReceiver,而这个 InnerReceiver 就是我们所需要的 Binder 对象 IIntentReceiver 的具体实现；
                  rd = new ReceiverDispatcher(r, context, handler,
                          instrumentation, registered);
                  if (registered) {
                      if (map == null) {
                          map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                          mReceivers.put(context, map);
                      }
                    //把 BroadcastReceiver 和 ReceiverDispatcher 储存在 ContextImpl 的 mReceivers 中
                      map.put(r, rd);
                  }
              } else {
                  rd.validate(context, handler);
              }
              rd.mForgotten = false;
            //返回 IIntentReceiver 对象；
              return rd.getIIntentReceiver();
          }
  }
  // ReceiverDispatcher 的构造
  ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                  Handler activityThread, Instrumentation instrumentation,
                  boolean registered) {
              if (activityThread == null) {
                  throw new NullPointerException("Handler must not be null");
              }
              mIIntentReceiver = new InnerReceiver(this, !registered);
              mReceiver = receiver;
              mContext = context;
              mActivityThread = activityThread;
              mInstrumentation = instrumentation;
              mRegistered = registered;
              mLocation = new IntentReceiverLeaked(null);
              mLocation.fillInStackTrace();
  }
  static final class ReceiverDispatcher {
    //这里就可以看到 InnerReceiver 确实是继承了 IIntentReceiver 这个 Binder
          final static class InnerReceiver extends IIntentReceiver.Stub {
              final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
              final LoadedApk.ReceiverDispatcher mStrongRef;

              InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                  mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                  mStrongRef = strong ? rd : null;
              }
  ~~~

* 现在到 AMS 中的 registerReceiver 中

  ~~~java
  //只贴重要代码
  //把远程的 InnerReceiver 对象以及 IntentFilter 对象储存起来；
  mRegisteredReceivers.put(receiver.asBinder(), rl);
  .................
  BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                      permission, callingUid, userId, instantApp, visibleToInstantApps);
  rl.add(bf);
  if (!bf.debugCheck()) {
        Slog.w(TAG, "==> For Dynamic broadcast");
  }
  mReceiverResolver.addFilter(bf);
  .................
  ~~~

  ​

  ​

