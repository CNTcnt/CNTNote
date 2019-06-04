[TOC]



# Activity的启动过程

## 简述

* Activity 是一种展示型组件，用于向用户直接展示一个界面，并且可以接收用户的输入信息进行交互；
* Activity 组件的主要作用是展示一个界面并和用户交互，扮演一种前台界面的角色；
* 先了解以下3个要点
* ActivityThread 是真正的核心类，它的 **main 方法**，是**整个应用进程的入口**。
* ActivityThread 功能：它管理应用进程的主线程的执行(相当于普通Java程序的 main 入口函数)，并根据 AMS 的要求（通过 IApplicationThread 接口，AMS 为 Client、ActivityThread.ApplicationThread 为 Server）负责调度和执行activities、broadcasts 和其它操作。
* ActivityManagerService 和 ActivityStack 位于同一个进程中，而 ApplicationThread 和 ActivityThread 位于另一个进程中;

## Activity的工作过程

~~~java
startActivity(new Intent(this,TestActivity.class));
~~~

~~~java
//到 Activity 中去执行，Activity 复写了 Context 的 startActivity 方法；
public void startActivity(Intent intent) {
        this.startActivity(intent, null);
}
~~~

~~~java
public void startActivity(Intent intent, @Nullable Bundle options) {//startActivity 有好几种重载方式，但最终都会调用到 startActivityForResult 方法；
     if (options != null) {
         startActivityForResult(intent, -1, options);
     } else {
         // Note we want to go through this call for compatibility with
         // applications that may have overridden the method.
         startActivityForResult(intent, -1);
     }
}
~~~

* 下面是 startActivityForResult 方法

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
      //启动过程跳到 Instrumentation 的 execStartActivity() 中去执行；Instrumentation 把
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
      //这里 mMainThread.getApplicationThread() 获取的是 ApplicationThread，它是 ActivityThread 的一个内部类，在后面会用到，后面再说，现在只需要知道它在 Activity 的启动过程发挥着很重要的作用;那么这个 mMainThread 怎么来的呢，我们现在要启动的这个 Activity 是在 已存在的 Activity 的基础上启动的，这个 mMainThread 是 ActivityThread 类型，是已存在的 Activity 它在被创建时通过 attach 方法传进来的；
     .......  
}
```

```java
//你可以将Instrumentation理解为一种没有图形界面的，具有启动能力的，用于监控其他类(用Target Package声明)的工具类。任何想成为Instrumentation的类必须继承android.app.Instrumentation。简单的说，它是ActivityThread的一个管家吧
// Instrumentation  的 execStartActivity()
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                ActivityResult result = null;
                if (am.ignoreMatchingSpecificIntents()) {
                    result = am.onStartActivity(intent);
                }
                if (result != null) {
                    am.mHits++;
                    return result;
                } else if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
      //这里才是我们要关注的重点，拿到 AMS 再转到 AMS 来启动 Activity，你启动个 Activity 不得让 AMS 过过目啊，无论是通过Launcher来启动Activity，还是通过Activity内部调用startActivity接口来启动新的Activity，都通过Binder进程间通信进入到ActivityManagerService进程中，并且调用ActivityManagerService.startActivity接口； 开始跨进程访问系统进程的 AMS
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);//这个方法用来检查启动 Activity 的结果
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}

//下面一目了然，IActivityManager是一个Binder接口，所以 AMS 其实也是一个 Binder；
public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
}

//这是一个单例的封装类
 private static final Singleton<IActivityManager> IActivityManagerSingleton =
       new Singleton<IActivityManager>() {
           @Override
          protected IActivityManager create() {
               final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
               final IActivityManager am = IActivityManager.Stub.asInterface(b);
               return am;
          }
 };

public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
  ...... 
}
```

* 既然 Activity 的启动在这里就开始交给了 AMS 了，这是一个重要的结点，我们就需要到 AMS 中去看看；下面代码就在 AMS 中去了，此时注意，AMS 在系统进程中，那么与我们的应用进程的通信当然是使用 Binder 机制，那么通过哪个 Binder 对象呢？这里先说，通过 ApplicationThread 通过 Hander 机制（H 类） 发送消息 给 ActivityThread  ，ActivityThread 后面会 执行 performLaunchActivity 方法，这个方法就是最终完成了 Activity 的启动；

~~~java
public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
}

public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
  //Activity 的启动又转移到了 mActivityStarter 中，它是 ActivityStarter 类型；
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null,
                "startActivityAsUser");
}
~~~

* 现在到 ActivityStarter （Activity启动器）中去看

~~~java
// ActivityManagerService 调用 ActivityStack.startActivityMayWait 来做准备要启动的 Activity 的相关信息；
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask, String reason) {
            ......
            int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                    aInfo, rInfo, voiceSession, voiceInteractor,
                    resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                    options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                    inTask, reason);            
}

int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask, String reason) {

        if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        mLastStartReason = reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord[0] = null;
//跳到 startActivity 中去
        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                container, inTask);

        if (outActivity != null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] = mLastStartActivityRecord[0];
        }
        return mLastStartActivityResult;
}

private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
        int result = START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        } 
  ...
}

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
				....
                  //跳到 ActivityStackSupervisor (Activity栈管理者) 中去；
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
       			...
}
                                                                
~~~

* 现在到 ActivityStackSupervisor (Activity栈管理者) 中去

~~~java
boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
          //跳到 ActivityStack (Activity栈) 去
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.state == RESUMED) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }
        return false;
}

~~~

* 跳到 ActivityStack (Activity栈) 去

~~~java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        // When resuming the top activity, it may be necessary to pause the top activity (for
        // example, returning to the lock screen. We suppress the normal pause logic in
        // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the end.
        // We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here to ensure
        // any necessary pause logic occurs.
        mStackSupervisor.checkReadyForSleepLocked();

        return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
       ...
       //现在跳回 ActivityStackSupervisor 的 startSpecificActivityLocked 方法
       mStackSupervisor.startSpecificActivityLocked(next, true, false);
       ...  
}
~~~

* 现在跳回 ActivityStackSupervisor 的 startSpecificActivityLocked 方法

~~~java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.getStack().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
              	//再到 realStartActivityLocked 中
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
}

final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
			......
             //此时就到了这里，app.thread 是 IApplicationThread 类型，那么这个对象具体是什么呢？IApplicationThread 是一个继承了 IInterface 的接口，所以它是一个 Binder ，这个接口里面包含了大量启动，停止 Activity 的接口，即 IApplicationThread 这个接口的实现者完成了大量和 Activity 以及 Service 启动/停止相关的功能；那么是哪个类实现了它呢？而 app.thread 是什么时候有的呢？那这个 binder 用来干什么呢？又开始跨进程，系统进程的 AMS 跨进程调用了用户进程的 ApplicationThread
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global and
                    // override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
			......
           
}

//其实就是 ActivityThread 中的 内部类 ApplicationThread 类，这个类实现了大量和 Activity 以及 Service 启动/停止相关的功能，那 app.thread 是什么时候有的呢？记不记得在最上面 startActivityForResult 方法中我们有个参数没有说明，就是它： mMainThread.getApplicationThread();
//这个 binder 的用处：该对象是 activityThread 和 ActivityManagerService 之间的桥梁。
private class ApplicationThread extends IApplicationThread.Stub {......}

final ApplicationThread mAppThread = new ApplicationThread();
public ApplicationThread getApplicationThread(){
    return mAppThread;
}

~~~

* 此时程序跳到 ActivityThread 中的 内部类 ApplicationThread 的 scheduleLaunchActivity 方法；

~~~java
@Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);
//发送一个启动 Activity 的消息交由 Handler 处理；这个Handler 的名字叫做 H,也是 ActivityThread 的一个内部类
            sendMessage(H.LAUNCH_ACTIVITY, r);
}

private class H extends Handler {......}

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

public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    //H 接受到启动 Activity 的消息就开始调用 handleLaunchActivity，这个方法是 ActivityThread 的方法，也就是说，启动 Activity 跳回 ActivityThread 来完成的；想想就知道 Activity 的启动过程就应该是 ActivityThread 去管理的
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
             } break;
             ......
}
~~~

* 此时到 ActivityThread 中看看怎么启动；

~~~java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }

        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
	   //看这个方法名，启动创建Activity ，就是它了；下面我们继续分析这个方法
        Activity a = performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
          //在创建完成 Activity 后，会调用 handleResumeActivity 方法来调用被启动 Activity 的 onResume 这一生命周期；
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
		   ......
        }
}

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
       
//1. 从 ActivityClientRecord 中获取启动的 Activity 的组件信息；
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
//2.通过 Instrumentation 的 newActivity 方法使用类加载器创建 Activity 对象；
        ContextImpl appContext = createBaseContextForActivity(r);//这里先创建了 ContextImpl 对象
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
  //     public Activity newActivity(ClassLoader cl, String className,  Intent intent)
  //     throws InstantiationException, IllegalAccessException,ClassNotFoundException {
  //      	return (Activity)cl.loadClass(className).newInstance();
  //     }
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
//3.通过 LoadedApk 的 makeApplication 方法来尝试创建 Application 对象；
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());
//4.通过 Activity 的 attach 方法来完成一些重要数据的初始化 ；
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
              // ContextImpl 是通过 Activity 的 attach 方法来和 Activity 建立联系，在 activity 方法中还会完成 Window 的创建并建立自己和 Window 的关联，这样当 Window 接收到外部输入事件就可以将事件传递给 Activity
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }
//5.调用 Activity 的 onCreat 方法，这里也是 mInstrumentation 来做 
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                  //就是这里
                    mInstrumentation.callActivityOnCreate(activity, r.state);
     //  public void callActivityOnCreate(Activity activity, Bundle icicle) {
  	 //      prePerformCreate(activity);
  	 //      activity.performCreate(icicle);//这一行就是调用 Activity 的 onCreate 方法；
  	 //      postPerformCreate(activity);
  	 //  }
     //  final void performCreate(Bundle icicle) {
     //     restoreHasCurrentPermissionRequest(icicle);
     //     onCreate(icicle);//这里看到了没有，onCreate，整个流程结束
     //     mActivityTransitionState.readState(icicle);
     //     performCreateCommon();
     //  }
    }
        ......
        return activity;
 }
~~~

* 这里说一下上面的步骤3

~~~java
//这里说一下步骤3 ，通过 从 参数 ActivityClientRecord 对象 r 中的 LoadedApk 对象 activityInfo 的 makeApplication 方法来尝试创建 Application 对象
 public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {//看见这里没有，如果 Application 已经创建过了就不再重新创建，也就是说，一个应用只有一个 Application 对象；这里很重要：每个应用程序同时也对应着一个ApplicationThread对象。该对象是 activityThread 和 ActivityManagerService 之间的桥梁。
            return mApplication;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
          //Application 对象的创建也是通过 Instrumentation 来完成的；也是通过类加载器来实现的；
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            try {
              //这里，Application 创建完成后，系统会通过 instrumentation.callApplicationOnCreate(app)来调用 Application 的 onCreate() 方法；
                instrumentation.callApplicationOnCreate(app);
 //   public void callApplicationOnCreate(Application app) {
 //       app.onCreate();
 //   }
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }

        // Rewrite the R 'constants' for all library apks.
        SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
        final int N = packageIdentifiers.size();
        for (int i = 0; i < N; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        return app;
}
~~~

## 小结

* ActivityManagerService 和 ActivityStack 位于同一个进程中，而 ApplicationThread 和 ActivityThread 位于另一个进程中。其中，ActivityManagerService 是负责管理 Activity 的生命周期的，ActivityManagerService 还借助 ActivityStack是来把所有的 Activity 按照后进先出的顺序放在一个堆栈中；

* 对于每一个应用程序来说，都有一个**ActivityThread来表示应用程序的主进程**，而每一个ActivityThread都包含有一个ApplicationThread 实例，它是一个 Binder 对象，负责和其它进程进行通信。

* 下面简要介绍一下启动的过程：

  ​        Step 1. 无论是通过Launcher来启动Activity，还是通过Activity内部调用startActivity接口来启动新的Activity，都通过Binder进程间通信进入到ActivityManagerService进程中，并且调用ActivityManagerService.startActivity接口； 

  ​        Step 2. ActivityManagerService调用ActivityStack.startActivityMayWait来做准备要启动的Activity的相关信息；

  ​        Step 3. ActivityStack 通知 ApplicationThread 要进行 Activity 启动调度了，这里的 ApplicationThread 代表的是调用 ActivityManagerService.startActivity 接口的进程，对于通过点击应用程序图标的情景来说，这个进程就是Launcher 了，而对于通过在 Activity 内部调用 startActivity 的情景来说，这个进程就是这个 Activity 所在的进程了；

  ​        Step 4. ApplicationThread 不执行真正的启动操作，它通过调用 ActivityManagerService.activityPaused 接口进入到 ActivityManagerService 进程中，看看是否需要创建新的进程来启动 Activity；（这一步上面代码没有）

  ​        Step 5. 对于通过点击应用程序图标来启动Activity的情景来说，ActivityManagerService在这一步中，会调用startProcessLocked来创建一个新的进程，而对于通过在Activity内部调用startActivity来启动新的Activity来说，这一步是不需要执行的，因为新的Activity就在原来的Activity所在的进程中进行启动；

  ​        Step 6. ActivityManagerServic 调用 ApplicationThread.scheduleLaunchActivity 接口，通知相应的进程执行启动Activity 的操作；

  ​        Step 7. ApplicationThread 把这个启动 Activity 的操作通过 Handler 机制转发给 ActivityThread，ActivityThread通过 ClassLoader 导入相应的 Activity 类，然后把它启动起来。


* ActivityManagerService 和 ActivityStack 位于同一个进程中，而 ApplicationThread 和 ActivityThread 位于另一个进程中，而2方之间的通信当然是通过 binder 机制，那么是通过哪个 binder 呢，答案在上面就可以看出来，就是 ApplicationThread;

* **ActivityManagerService**作为Client端调用 ApplicationThread 的接口，目的是用来调度管理 Activity；

* 我们知道，系统一开始就会跑 ActivityThread 的 main 方法，里面有一句代码

  ~~~java
  thread.attach(false);

  //此时主要完成两件事

  1.调用 RuntimeInit.setApplicationObject() 方法，把对象mAppThread（Binder）放到了RuntimeInit类中的静态变量mApplicationObject中。

      public static final void More ...setApplicationObject(IBinder app) {
          mApplicationObject = app;
      }
  mAppThread的类型是ApplicationThread，它是ActivityThread的成员变量，定义和初始化如下：
  		final ApplicationThread mAppThread = new ApplicationThread();
  2.就是调用 ActivityManagerService 的 attachApplication() 方法，将mAppThread 作为参数传入ActivityManagerService，这样 ActivityManagerService 就可以调用 ApplicaitonThread 的接口了。这与我们刚才说的，ActivityManagerService作为Client端调用ApplicaitonThread的接口管理Activity，就不谋而合了。

  作者：MeloDev
  链接：https://www.jianshu.com/p/0efc71f349c8
  來源：简书
  简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。
  ~~~

  ​