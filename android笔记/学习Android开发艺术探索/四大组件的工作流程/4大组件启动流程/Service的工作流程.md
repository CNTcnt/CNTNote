[TOC]



# Service的启动流程

## 简介

* Service 是一种计算型组件，用于在后台执行一系列计算任务，顾名思义，就是提供服务，但是它工作在后台，因此用户无法直接感知到服务；
* Service 组件有2个状态：启动状态 和 绑定状态；
* Service 是运行在主线程中的，道理你懂吧，就是不能在 Service 中进行耗时的任务；可以在服务中开一一个线程，在线程中做耗时动作。
* Service 处于启动状态的时候，这个时候 Service 内部可以做一些后台计算，不同理会其他东西，不需要和外界有直接的交互；（基于它运行在主线程，不能做耗时操作）；停止一个启动状态的服务只需要调用 stopService 方法；
* Service 处于和 Activity 的绑定状态时（我只学过服务和活动绑定。。），这个时候 Service 内部不仅可以做一些后台计算，而且外界可以很方便地和 Service 组件进行通信；停止一个绑定状态的服务需要调用 stopService 和 unBindService 方法；

## 启动流程

### 启动一个服务但不与 activity 绑定

~~~java
startService(Intent service);
~~~

* 此时 Service 就开始启动，到了 ContextWrapper （ Activity 是间接继承它的 ）的 startService ；

  ~~~java
  @Override
  public ComponentName startService(Intent service) {
      //？？？这个 mBase 是 Context 类型，那么它什么时候创建的呢？？？
      return mBase.startService(service);
      //可以从这里看出一点端倪，这里的操作是另外通过 mBase 来操作的实现的，是一种典型的桥接模式；
  }
  //这个要我找我也找不到啊。。。书里说是在 Activity 被创建时会通过 attach 方法将一个 ContextImpl 关联起来，我们现在先去找它；
  //代码在 ActivityThread 启动 Activity 的 performLaunchActivity 方法中
  ContextImpl appContext = createBaseContextForActivity(r);
  activity.attach(appContext, this, getInstrumentation(), r.token,
                          r.ident, app, r.intent, r.activityInfo, title, r.parent,
                          r.embeddedID, r.lastNonConfigurationInstances, config,
                          r.referrer, r.voiceInteractor, window, r.configCallback);
  //到 Activity 中
  final void attach(Context context, ActivityThread aThread,
              Instrumentation instr, IBinder token, int ident,
              Application application, Intent intent, ActivityInfo info,
              CharSequence title, Activity parent, String id,
              NonConfigurationInstances lastNonConfigurationInstances,
              Configuration config, String referrer, IVoiceInteractor voiceInteractor,
              Window window, ActivityConfigCallback activityConfigCallback) {
          
      attachBaseContext(context);//就是这里了，进去看看
  	......
  }
  @Override
  protected void attachBaseContext(Context newBase) {
      super.attachBaseContext(newBase);//再进去看看，不知道是不是这里
      newBase.setAutofillClient(this);
  }
  //到 ContextThemeWrapper extends ContextWrapper
  @Override
  protected void attachBaseContext(Context newBase) {
      super.attachBaseContext(newBase);
  }
  //到 ContextWrapper extends Context 
  protected void attachBaseContext(Context base) {
      if (mBase != null) {
           throw new IllegalStateException("Base context already set");
      }
      //就是这里了，一目了然，确实在 Activity 的启动过程中，Activity 的 attach 方法把  ActivityThread 创建的 ContextImpl 对象给了 ContextWrapper 的 mBase 引用；这里就可以回答开头的问题，ContextWrapper 里的 mBase 对象是 ContextImpl 类型；
      mBase = base;
  }
  ~~~

* 此时就该到 mBase 引用所指向的 ContextImpl 对象去看

  ~~~java
  @Override
  public ComponentName startService(Intent service) {
       warnIfCallingFromSystemProcess();
       return startServiceCommon(service, false, mUser);
  }
  private ComponentName startServiceCommon(Intent service, boolean requireForeground,
              UserHandle user) {
          try {
              validateServiceIntent(service);
              service.prepareToLeaveProcess(this);
              //又到了我们最熟悉的地方；通过 AMS 来开启服务，这里和启动 Acticity 一样都需要传进一个 ApplicationThread Binder参数，AMS 作为Client端调用 ApplicationThread 的接口，目的是用来调度管理 Service；
              ComponentName cn = ActivityManager.getService().startService(
                  mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                              getContentResolver()), requireForeground,
                              getOpPackageName(), user.getIdentifier());
              if (cn != null) {
                  if (cn.getPackageName().equals("!")) {
                      throw new SecurityException(
                              "Not allowed to start service " + service
                              + " without permission " + cn.getClassName());
                  } else if (cn.getPackageName().equals("!!")) {
                      throw new SecurityException(
                              "Unable to start service " + service
                              + ": " + cn.getClassName());
                  } else if (cn.getPackageName().equals("?")) {
                      throw new IllegalStateException(
                              "Not allowed to start service " + service + ": " + cn.getClassName());
                  }
              }
              return cn;
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
  ~~~

* 此时又到了 AMS 的 startService 方法

  ~~~java
  public ComponentName startService(IApplicationThread caller, Intent service,
              String resolvedType, boolean requireForeground, String callingPackage, int userId)
              throws TransactionTooLargeException {
          enforceNotIsolatedCaller("startService");
          // Refuse possible leaked file descriptors
          if (service != null && service.hasFileDescriptors() == true) {
              throw new IllegalArgumentException("File descriptors passed in Intent");
          }

          if (callingPackage == null) {
              throw new IllegalArgumentException("callingPackage cannot be null");
          }

          if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                  "*** startService: " + service + " type=" + resolvedType + " fg=" + requireForeground);
          synchronized(this) {
              final int callingPid = Binder.getCallingPid();
              final int callingUid = Binder.getCallingUid();
              final long origId = Binder.clearCallingIdentity();
              ComponentName res;
              try {
                  //此时看到这里，由  mServices 对象的 startServiceLocked 方法来实现
                  // final ActiveServices mServices;ActiveServices 是一个辅助 AMS 进行 Service 管理的类，包括 Service 的启动，绑定和停止等。
                  res = mServices.startServiceLocked(caller, service,
                          resolvedType, callingPid, callingUid,
                          requireForeground, callingPackage, userId);
              } finally {
                  Binder.restoreCallingIdentity(origId);
              }
              return res;
          }
  }
  ~~~

* mServices 对象的 startServiceLocked 方法来实现

  ~~~java
  ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
              int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
              throws TransactionTooLargeException {
          .............
          ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
          return cmp;
  }
  //  ServiceRecord 意思是 Service记录，一直贯穿着整个 Service 的启动过程
  ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
              boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
          ServiceState stracker = r.getTracker();
          if (stracker != null) {
              stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
          }
          r.callStart = false;
          synchronized (r.stats.getBatteryStats()) {
              r.stats.startRunningLocked();
          }
          String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
          .............
          return r.name;
  }
  private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
              boolean whileRestarting, boolean permissionsReviewRequired)
              throws TransactionTooLargeException {
          ............
              realStartServiceLocked(r, app, execInFg);      
          ............
          return null;
  }
  private final void realStartServiceLocked(ServiceRecord r,
              ProcessRecord app, boolean execInFg) throws RemoteException {
          ..............
              //终于看到这个对象了，这个对象就是 ActivityThread 传给 AMS 的 ApplicationThread 对象，此时通过 scheduleCreateService 来创建 Service 对象并调用其 onCreate；
              app.thread.scheduleCreateService(r, r.serviceInfo,
                      mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                      app.repProcState);
          ................
              //接着后面调用了 sendServiceArgsLocked 方法来调用 Service 的其他方法，如 onStartCommand； 
          sendServiceArgsLocked(r, execInFg, true);
         ..................
  }
  ~~~

* 接着就到 ActivityThread 的内部类的 ApplicationThread 的 scheduleCreateService 

  ~~~java
  public final void scheduleCreateService(IBinder token,
                  ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
         updateProcessState(processState, false);
         CreateServiceData s = new CreateServiceData();
         s.token = token;
         s.info = info;
         s.compatInfo = compatInfo;
  //接下来的流程就和启动一个 Activity 的过程一样了，还是发送消息给 ActivityThread 的内部类 H 的对象 mH
         sendMessage(H.CREATE_SERVICE, s);
  }
  private void sendMessage(int what, Object obj) {
          sendMessage(what, obj, 0, 0, false);
  }
  private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
          if (DEBUG_MESSAGES) Slog.v(
              TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
              + ": " + arg1 + " / " + obj);
          Message msg = Message.obtain();
          msg.what = what;
          msg.obj = obj;
          msg.arg1 = arg1;
          msg.arg2 = arg2;
          if (async) {
              msg.setAsynchronous(true);
          }
          mH.sendMessage(msg);
  }
  ~~~

* 此时就是 mH 这个 Handler 对象来 接收消息并处理

  ~~~java
  public void handleMessage(Message msg) {
              if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
              switch (msg.what) {
  	case CREATE_SERVICE:
              Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
            //调用 ActivityThread 的 handleCreateService 方法
              handleCreateService((CreateServiceData)msg.obj);
              Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              break;
       ..............
       }
      .............
  }    
  ~~~

* 调用 ActivityThread 的 handleCreateService 方法

  ~~~java
  private void handleCreateService(CreateServiceData data) {
          // If we are getting ready to gc after going to the background, well
          // we are back active so skip it.
          unscheduleGcIdler();

          LoadedApk packageInfo = getPackageInfoNoCheck(
                  data.info.applicationInfo, data.compatInfo);
          Service service = null;
          try {
              //通过类加载器来创建一个 Service 
              java.lang.ClassLoader cl = packageInfo.getClassLoader();
              service = (Service) cl.loadClass(data.info.name).newInstance();
          } catch (Exception e) {
              if (!mInstrumentation.onException(service, e)) {
                  throw new RuntimeException(
                      "Unable to instantiate service " + data.info.name
                      + ": " + e.toString(), e);
              }
          }

          try {
              if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

              ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
              context.setOuterContext(service);
  		 //接着创建 Application 对象并调用其 onCreate ，当然 Application 的创建过程只会有一次
              Application app = packageInfo.makeApplication(false, mInstrumentation);
              //将创建好的 ContextImpl 对象通过 Service 的 attach 方法建立二者之间的关系，这个过程和Activity 实际上是类似的，毕竟 Service 和 Activity 都是一个 Context
              service.attach(context, this, data.info.name, data.token, app,
                      ActivityManager.getService());
              //紧接着就调用了 Service 的 onCreate 方法
              service.onCreate();
              //然后将 service 对象储存到 ActivityThread 中的一个列表中；
              //这个 mServices 列表的定义是 
              //  final ArrayMap<IBinder, Service> mServices = new ArrayMap<>();
              mServices.put(data.token, service);
              try {
                  ActivityManager.getService().serviceDoneExecuting(
                          data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
              } catch (RemoteException e) {
                  throw e.rethrowFromSystemServer();
              }
          } catch (Exception e) {
              if (!mInstrumentation.onException(service, e)) {
                  throw new RuntimeException(
                      "Unable to create service " + data.info.name
                      + ": " + e.toString(), e);
              }
          }
  }
  ~~~

* 前面说了 AMS 通过 ApplicationThread 执行了 scheduleCreateService 方法去间接执行 ActivityThread 的 handleCreateService 方法，之后还会执行 sendServiceArgsLocked(r, execInFg, true); 方法，我们接着看

  ~~~java
  private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
              boolean oomAdjusted) throws TransactionTooLargeException {
          ..............
              r.app.thread.scheduleServiceArgs(r, slice);
         ............ 
  }
  ~~~

* 又到 ActivityThread 的内部类的 ApplicationThread 的 sendServiceArgsLocked 

  ~~~java
  public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
              List<ServiceStartArgs> list = args.getList();
              for (int i = 0; i < list.size(); i++) {
                  ServiceStartArgs ssa = list.get(i);
                  ServiceArgsData s = new ServiceArgsData();
                  s.token = token;
                  s.taskRemoved = ssa.taskRemoved;
                  s.startId = ssa.startId;
                  s.flags = ssa.flags;
                  s.args = ssa.args;

                  sendMessage(H.SERVICE_ARGS, s);
              }
  }
  ~~~

* 此时也是 mH 这个 Handler 对象来 接收消息并处理

  ~~~java
  public void handleMessage(Message msg) {
              if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
              switch (msg.what) {
  	case SERVICE_ARGS:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
                      handleServiceArgs((ServiceArgsData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
       ..............
       }
      .............
  }   
  private void handleServiceArgs(ServiceArgsData data) {
          Service s = mServices.get(data.token);
          if (s != null) {
              try {
                  if (data.args != null) {
                      data.args.setExtrasClassLoader(s.getClassLoader());
                      data.args.prepareToEnterProcess();
                  }
                  int res;
                  if (!data.taskRemoved) {
                      //这里执行了 Service 的 生命周期 onStartCommand
                      res = s.onStartCommand(data.args, data.flags, data.startId);
                  } else {
                      s.onTaskRemoved(data.args);
                      res = Service.START_TASK_REMOVED_COMPLETE;
                  }

                  QueuedWork.waitToFinish();

                  try {
                      ActivityManager.getService().serviceDoneExecuting(
                              data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                  } catch (RemoteException e) {
                      throw e.rethrowFromSystemServer();
                  }
                  ensureJitEnabled();
              } catch (Exception e) {
                  if (!mInstrumentation.onException(s, e)) {
                      throw new RuntimeException(
                              "Unable to start service " + s
                              + " with " + data.args + ": " + e.toString(), e);
                  }
              }
          }
  }
  ~~~

* 到这里 Service 的启动就结束了；

####小结

* 过程和 启动一个 Activity 的过程差不多，区别是用 启动这个 Service 的 Activity 的 ContextImpl 对象 去让 AMS 来启动这个 Service，AMS 通过 ApplicationThread 这个 binder 来发出一条消息让 ActivityThread 启动 Service ；
* 这里为什么不让 Activity 直接通知 AMS 来启动 Service ？？？比如一个方法里面直接调用  ActivityManager.getService().startService(）方法，仔细想想我还是太蠢了，启动 Service 的工作还是要由 ActivityThread 来完成的，那我们就需要给 AMS 传递 ApplicationThread 这个 binder ， ActivityManager.getService().startService(）这个方法时就需要到这个参数；所以我们还是需要到这个 context 参数，这个 context 参数的实际对象就是 ActivityThread 创建 Activity（这个 Activity 是启动这个 Service 的 Activity） 时通过 attach 方法传进来的 ContextImpl 对象，就携带着 ApplicationThread 这个 binder；

### 启动一个服务并且与 activity 绑定

* 即调用 bindService 方法

  ~~~java
  bindService(Intent service,ServiceConnection conn,int flags);
  //bindService有三个参数，第一个是传入intent，第二个传入ServiceConnection，第三个传入flag~上文的BIND_AUTO_CREATE的意思是，如果服务不存在则创建一个的意思；ServiceConnection，其中有两个回调方法，其中一个是与service建立起连接的时候回调的，另一个是与Service断开连接的时候回调的~其中onServiceConnected传进两个参数，其中一个是IBinder，该参数正是在 Service 的 onBind 返回的值

  public boolean bindService(Intent service, ServiceConnection conn,
                int flags) {
            return mBase.bindService(service, conn, flags);
  } 
  ~~~

* 接下来当然是到 ContextImpl 的 bindService

  ~~~java
  @Override
  public boolean bindService(Intent service, ServiceConnection conn,
              int flags) {
     warnIfCallingFromSystemProcess();
     return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
             Process.myUserHandle());
  }

  private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
              handler, UserHandle user) {
          // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
          IServiceConnection sd;
          if (conn == null) {
              throw new IllegalArgumentException("connection is null");
          }
          if (mPackageInfo != null) {
          //这里的 sd 是 IServiceConnection 类型，看名字就有点像 Binder 类，其实它就是，我们在 Activity 中启动并与 Service 绑定，我们可以使用 ServiceConnection 来与 Activity 交互，但是，服务的绑定有可能是跨进程的，比如去绑定另一个进程的服务，此时 ServiceConnection 是不能被跨进程调用的；解决方法是使用 IPC 机制来让远程服务端回调自己的方法，所以我们需要把 ServiceConnection 转化为 binder 对象，所以 getServiceDispatcher 方法就是把客户端的 ServiceConnection 转化为 ServiceDispatcher.InnerConnection 这个 Binder 对象，InnerConnection 是一个内部类，ServiceDispatcher 起着连接 ServiceConnection 和 InnerConnection 的作用；具体怎么连接看下面
              sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
          } else {
              throw new RuntimeException("Not supported in system context");
          }
          validateServiceIntent(service);
          try {
              IBinder token = getActivityToken();
              if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                      && mPackageInfo.getApplicationInfo().targetSdkVersion
                      < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                  flags |= BIND_WAIVE_PRIORITY;
              }
              service.prepareToLeaveProcess(this);
              //启动 Service 当然要先过问 AMS ，要把 IServiceConnection 对象同时给它
              int res = ActivityManager.getService().bindService(
                  mMainThread.getApplicationThread(), getActivityToken(), service,
                  service.resolveTypeIfNeeded(getContentResolver()),
                  sd, flags, getOpPackageName(), user.getIdentifier());
              if (res < 0) {
                  throw new SecurityException(
                          "Not allowed to bind to service " + service);
              }
              return res != 0;
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
  }
  ~~~

* 到 Loadeapk 中看看 getServiceDispatcher；

  ~~~java
  public final IServiceConnection getServiceDispatcher(ServiceConnection c,
              Context context, Handler handler, int flags) {
      //mServices 是一个 ArrayMap，它储存了一个应用当前 Activity 的 ServiceConnection 和 ServiceDispatcher 的映射关系（还是一个 ArrayMap）
      //private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices = new ArrayMap<>();
          synchronized (mServices) {
              LoadedApk.ServiceDispatcher sd = null;
              ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
              //先查找是否存在相同的 ServiceConnection ，如果存在就不再重新创建；
              if (map != null) {
                  if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
                  sd = map.get(c);
              }
              //如果不存在就重新创建一个 ServiceDispatcher 对象并将其储存在 mServices 中，其中 储存的映射关系 ArrayMap 的 key 是 ServiceConnection ，value 是 ServiceDispatcher；然后，在 ServiceDispatcher 中又储存了 ServiceConnection 和 InnerConnection 对象；此时的 ServiceDispatcher 起着连接 ServiceConnection 和 InnerConnection 的作用
              if (sd == null) {
                  sd = new ServiceDispatcher(c, context, handler, flags);
                  if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
                  if (map == null) {
                      map = new ArrayMap<>();
                      //把调用方的 context 和 一个映射关系 放进 mServices 中
                      mServices.put(context, map);
                  }
                  //给 映射关系 赋值，key 是 ServiceConnection ，value 是 刚创建好的 ServiceDispatcher；
                  map.put(c, sd);
              } else {
                  sd.validate(context, handler);
              }
              return sd.getIServiceConnection();
          }
  }
  ~~~

* 看看 ServiceDispatcher 的创建

  ~~~java
  ServiceDispatcher(ServiceConnection conn,
                  Context context, Handler activityThread, int flags) {
      //内部有着一个 InnerConnection 对象，外部需要用时可以从 ServiceDispatcher 的getIServiceConnection() 得到 ；
              mIServiceConnection = new InnerConnection(this);
              mConnection = conn;
              mContext = context;
              mActivityThread = activityThread;
              mLocation = new ServiceConnectionLeaked(null);
              mLocation.fillInStackTrace();
              mFlags = flags;
   }
  ~~~

* 到 InnerConnection 看看

  ~~~java
  private static class InnerConnection extends IServiceConnection.Stub {
      //弱引用
              final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

              InnerConnection(LoadedApk.ServiceDispatcher sd) {
                  mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
              }
  //当 Service 和客户端建立连接后，系统会通过 InnerConnection 来调用 ServiceConnection 中的 onServiceConected 方法
              public void connected(ComponentName name, IBinder service, boolean dead)
                      throws RemoteException {
                  LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                  if (sd != null) {
                      sd.connected(name, service, dead);
                  }
              }
   }
  public void connected(ComponentName name, IBinder service, boolean dead) {
              if (mActivityThread != null) {
                  mActivityThread.post(new RunConnection(name, service, 0, dead));
              } else {
                  doConnected(name, service, dead);
              }
  }
  public void doConnected(ComponentName name, IBinder service, boolean dead) {
              ServiceDispatcher.ConnectionInfo old;
              ServiceDispatcher.ConnectionInfo info;
  		   ...........................................
              // If there was an old service, it is now disconnected.
              if (old != null) {
                  mConnection.onServiceDisconnected(name);
              }
              if (dead) {
                  mConnection.onBindingDied(name);
              }
              // If there is a new service, it is now connected.
              if (service != null) {
                  mConnection.onServiceConnected(name, service);
              }
  }
  ~~~


* 现在我们重新回到之前  ContextImpl 的 bindServiceCommon 方法 里调用到 AMS 的 bindService 方法中

  ~~~java
  public int bindService(IApplicationThread caller, IBinder token, Intent service,
              String resolvedType, IServiceConnection connection, int flags, String callingPackage,
              int userId) throws TransactionTooLargeException {
          enforceNotIsolatedCaller("bindService");

          // Refuse possible leaked file descriptors
          if (service != null && service.hasFileDescriptors() == true) {
              throw new IllegalArgumentException("File descriptors passed in Intent");
          }

          if (callingPackage == null) {
              throw new IllegalArgumentException("callingPackage cannot be null");
          }

          synchronized(this) {
              //此时看到这里，由  mServices 对象的 startServiceLocked 方法来实现
              // final ActiveServices mServices;ActiveServices 是一个辅助 AMS 进行 Service 管理的类，包括 Service 的启动，绑定和停止等。
              return mServices.bindServiceLocked(caller, token, service,
                      resolvedType, connection, flags, callingPackage, userId);
          }
  }
  ~~~

* mServices 对象的 bindServiceLocked 方法来实现

  ~~~java
  int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
              String resolvedType, final IServiceConnection connection, int flags,
              String callingPackage, final int userId) throws TransactionTooLargeException { 
      ...............
  	if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                  s.lastActivity = SystemClock.uptimeMillis();
          //这个 bringUpServiceLocked，进而调用  realStartServiceLocked(r, app, execInFg);这个realStartServiceLocked 中调用了 app.thread.scheduleCreateService 这个方法完成了 Service 实例的创建并执行 onCreate 方法；看下面具体代码
                  if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                          permissionsReviewRequired) != null) {
                      return 0;
                  }
        }
      ............
      if (s.app != null && b.intent.received) {
                  // Service is already running, so we can immediately
                  // publish the connection.
                  try {
                      c.conn.connected(s.name, b.intent.binder, false);
                  } catch (Exception e) {
                      Slog.w(TAG, "Failure sending service " + s.shortName
                              + " to connection " + c.conn.asBinder()
                              + " (in " + c.binding.client.processName + ")", e);
                  }

                  // If this is the first app connected back to this binding,
                  // and the service had previously asked to be told when
                  // rebound, then do so.
                  if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                      //看上面的英语注释，
                      //启动了 Service 之后，还有绑定的操作啊，具体看下面
                      requestServiceBindingLocked(s, b.intent, callerFg, true);
                  }
              } else if (!b.intent.requested) {
                  requestServiceBindingLocked(s, b.intent, callerFg, false);
       }
  	............
       return 1;
  }

  ~~~

  bringUpServiceLocked 方法实现了什么？

  ~~~java
  // bringUpServiceLocked，进而调用  realStartServiceLocked(r, app, execInFg);在  realStartServiceLocked 中调用了 app.thread.scheduleCreateService 这个方法完成了 Service 实例的创建并执行 onCreate 方法；这个 app.thread 我们看过太多次了，其实就是 ApplicationThread ，ApplicationTHread 的一系列以 schedule 开头的方法，其内部都是通过 Handler H 来中转的；
  public final void scheduleCreateService(IBinder token,
                  ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
              updateProcessState(processState, false);
              CreateServiceData s = new CreateServiceData();
              s.token = token;
              s.info = info;
              s.compatInfo = compatInfo;

              sendMessage(H.CREATE_SERVICE, s);
   }
   case CREATE_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                      handleCreateService((CreateServiceData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
  //ActivityThread 创建 Service 实例
  private void handleCreateService(CreateServiceData data) {
          // If we are getting ready to gc after going to the background, well
          // we are back active so skip it.
          unscheduleGcIdler();

          LoadedApk packageInfo = getPackageInfoNoCheck(
                  data.info.applicationInfo, data.compatInfo);
          Service service = null;
          try {
              //也是通过类加载器来创建 Service
              java.lang.ClassLoader cl = packageInfo.getClassLoader();
              service = (Service) cl.loadClass(data.info.name).newInstance();
          } catch (Exception e) {
              if (!mInstrumentation.onException(service, e)) {
                  throw new RuntimeException(
                      "Unable to instantiate service " + data.info.name
                      + ": " + e.toString(), e);
              }
          }

          try {
              if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

              ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
              context.setOuterContext(service);

              Application app = packageInfo.makeApplication(false, mInstrumentation);
              service.attach(context, this, data.info.name, data.token, app,
                      ActivityManager.getService());
              //这里执行了 Service 的生命周期 onCreate()
              service.onCreate();
              //把 service 放进 mServices 列表中
              mServices.put(data.token, service);
              try {
                  ActivityManager.getService().serviceDoneExecuting(
                          data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
              } catch (RemoteException e) {
                  throw e.rethrowFromSystemServer();
              }
          } catch (Exception e) {
              if (!mInstrumentation.onException(service, e)) {
                  throw new RuntimeException(
                      "Unable to create service " + data.info.name
                      + ": " + e.toString(), e);
              }
          }
  }
  ~~~

* 上面说了，在 AMS 执行完 bringUpServiceLocked 方法之后要执行 requestServiceBindingLocked 方法，即 创建完 Service 就要开始 绑定操作 用来绑定服务；

  ~~~java
  private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
          ...............
          if ((!i.requested || rebind) && i.apps.size() > 0) {
              try {
                  bumpServiceExecutingLocked(r, execInFg, "bind");
                  r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                  //这里就可以让 ApplicationThread 来执行绑定服务的操作
                  r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                          r.app.repProcState);
                  if (!rebind) {
                      i.requested = true;
                  }
                  i.hasBound = true;
                  i.doRebind = false;
              } 
          ................
          return true;
  }
                                                    
  public final void scheduleBindService(IBinder token, Intent intent,
                  boolean rebind, int processState) {
              updateProcessState(processState, false);
              BindServiceData s = new BindServiceData();
              s.token = token;
              s.intent = intent;
              s.rebind = rebind;

              if (DEBUG_SERVICE)
                  Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                          + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
              sendMessage(H.BIND_SERVICE, s);
  }
                                                    
  case BIND_SERVICE:
              Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
              handleBindService((BindServiceData)msg.obj);
              Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              break;
                                                    
  private void handleBindService(BindServiceData data) {
      //首先根据 Service 的 token 取出 Service 对象，然后调用 Service 的 onBind 方法，Service 的 onBiner 对象给客户端使用，
      //还记得我们在把创建 Service 把创建好的 service 放进 mServices 列表中，现在检查有没有
          Service s = mServices.get(data.token);
          if (DEBUG_SERVICE)
              Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
          if (s != null) {
              try {
                  data.intent.setExtrasClassLoader(s.getClassLoader());
                  data.intent.prepareToEnterProcess();
                  try {
                      if (!data.rebind) {
                         //原则上，Service 到下面这里 onBinder 方法被调用 ，Service 已经处于绑定状态了；
                          IBinder binder = s.onBind(data.intent);
                          //但是，你看清楚，这是 Service 调用它自己的 onBinder 进行绑定操作，是不是 客户端都不知道它与 Service 进行了绑定了没有？那就对了，我们还需要去通知 客户端，调用客户端绑定完 Service 之后需要调用 Service 的方法 ；而我们一开始执行 bindService 传进去的第二个参数 ServiceConnection 就起到回调这个的作用；而我们现在就需要去回调 ServiceConnection 中的 onServiceConnected 让客户端知道我们已经绑定成功了，可以做一些与服务交互的操作，而这个过程还是要借助 AMS ，因为我们现在是在 ActivityThread 中，我们可以去借助 AMS 来操作
                          ActivityManager.getService().publishService(
                                  data.token, data.intent, binder);
                      } else {
                          s.onRebind(data.intent);
                          ActivityManager.getService().serviceDoneExecuting(
                                  data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                      }
                      ensureJitEnabled();
                  } catch (RemoteException ex) {
                      throw ex.rethrowFromSystemServer();
                  }
              } catch (Exception e) {
                  if (!mInstrumentation.onException(s, e)) {
                      throw new RuntimeException(
                              "Unable to bind to service " + s
                              + " with " + data.intent + ": " + e.toString(), e);
                  }
              }
          }
  }
  ~~~

* 到 AMS 中 的 publishService

  ~~~java
  public void publishService(IBinder token, Intent intent, IBinder service) {
          // Refuse possible leaked file descriptors
          if (intent != null && intent.hasFileDescriptors() == true) {
              throw new IllegalArgumentException("File descriptors passed in Intent");
          }

          synchronized(this) {
              if (!(token instanceof ServiceRecord)) {
                  throw new IllegalArgumentException("Invalid service token");
              }
              mServices.publishServiceLocked((ServiceRecord)token, intent, service);
          }
  }
  ~~~

* mServices.publishServiceLocked

  ~~~java
  void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
          final long origId = Binder.clearCallingIdentity();
          try {
              if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                      + " " + intent + ": " + service);
              if (r != null) {
                  Intent.FilterComparison filter
                          = new Intent.FilterComparison(intent);
                  IntentBindRecord b = r.bindings.get(filter);
                  if (b != null && !b.received) {
                      b.binder = service;
                      b.requested = true;
                      b.received = true;
                      for (int conni=r.connections.size()-1; conni>=0; conni--) {
                          ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                          for (int i=0; i<clist.size(); i++) {
                              ConnectionRecord c = clist.get(i);
                              if (!filter.equals(c.binding.intent.intent)) {
                                  if (DEBUG_SERVICE) Slog.v(
                                          TAG_SERVICE, "Not publishing to: " + c);
                                  if (DEBUG_SERVICE) Slog.v(
                                          TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                                  if (DEBUG_SERVICE) Slog.v(
                                          TAG_SERVICE, "Published intent: " + intent);
                                  continue;
                              }
                              if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                              try {
                                  //就是这里了，调用了 客户端传进来的 ServiceConnection 的 方法
                                  c.conn.connected(r.name, service, false);
                              } catch (Exception e) {
                                  Slog.w(TAG, "Failure sending service " + r.name +
                                        " to connection " + c.conn.asBinder() +
                                        " (in " + c.binding.client.processName + ")", e);
                              }
                          }
                      }
                  }

                  serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
              }
          } finally {
              Binder.restoreCallingIdentity(origId);
          }
  }
  ~~~

* c.conn.connected();//c.conn 是一个 Binder 对象，具体实现是 InnerConnection，里面的 mDispatcher 就有 ServiceConnection；

  ~~~java
  private static class InnerConnection extends IServiceConnection.Stub {
              final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

              InnerConnection(LoadedApk.ServiceDispatcher sd) {
                  mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
              }

              public void connected(ComponentName name, IBinder service, boolean dead)
                      throws RemoteException {
                  LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                  if (sd != null) {
                      sd.connected(name, service, dead);
                  }
              }
  }
  public void connected(ComponentName name, IBinder service, boolean dead) {
              if (mActivityThread != null) {
                  mActivityThread.post(new RunConnection(name, service, 0, dead));
              } else {
                  doConnected(name, service, dead);
              }
  }
  public void doConnected(ComponentName name, IBinder service, boolean dead) {
              ServiceDispatcher.ConnectionInfo old;
              ServiceDispatcher.ConnectionInfo info;
  		   ...........................................
              // If there was an old service, it is now disconnected.
              if (old != null) {
                  mConnection.onServiceDisconnected(name);
              }
              if (dead) {
                  mConnection.onBindingDied(name);
              }
              // If there is a new service, it is now connected.就是这里了，跨进程调用 ServiceConnection 的 onServiceConnected
              if (service != null) {
                  //这里就执行客户端的 onServiceConnected 方法，把 Service 给客户端去交互 
                  mConnection.onServiceConnected(name, service);
              }
  }
  ~~~

  ​



