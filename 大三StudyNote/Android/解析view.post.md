[TOC]

# 关于view.post()和view.postDelay()的解析

2016年11月08日 21:01:18 阅读数：472



https://blog.csdn.net/qq_29425853/article/details/53087742

## 1.view.post()

![](https://img-blog.csdn.net/20161108210514882?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



运行程序，得到的结果如下：

![](https://img-blog.csdn.net/20161108210647344?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 2.view.postDelay()

大家都有过这样的经历，重复点击同一个view，这个view的点击事件总是重复执行，可以通过view.postDelay()这个方法来达到这个效果：

![](https://img-blog.csdn.net/20161108210652438?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

重复点击button之后，得到的结果如下：

![](https://img-blog.csdn.net/20161108210656994?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)





## 原理解析

### view.post()

* ~~~java
  public boolean post(Runnable action) {
          final AttachInfo attachInfo = mAttachInfo;
          if (attachInfo != null) {
              //看到这里用 handler 来 post，这个 attachInfo.handler 哪里来的呢？
              return attachInfo.mHandler.post(action);
          }
  
          // Postpone the runnable until we know on which thread it needs to run.
          // Assume that the runnable will be successfully placed after attach.
          getRunQueue().post(action);
          return true;
  }
  ~~~

* 这个 attachInfo.handler 哪里来的呢？

  ~~~java
  void dispatchAttachedToWindow(AttachInfo info, int visibility) {
          mAttachInfo = info;//就是这里,传参数传进来的，看这个方法名，传递关联window，应该就是 viewRootImpl 操作完成的，去 viewRootImpl 找找看
          if (mOverlay != null) {
              mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
          }
      ......
  }
  ~~~

* 去 viewRootImpl 找找看

  ~~~java
  private void performTraversals() {
          // cache mView since it is used so much below...
          final View host = mView;
  
          ........
              .......//现在就该去找 mAttachInfo 的实现是啥
              host.dispatchAttachedToWindow(mAttachInfo, 0);
  }
  
   final View.AttachInfo mAttachInfo;
  public ViewRootImpl(Context context, Display display) {
      ...//在构造函数就对其初始化，看到初始化的参数了没有，就有我们关心的 handler
          mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                  context);
      ...
  }
  final ViewRootHandler mHandler = new ViewRootHandler();
  //这个 VIewRootHandler 继承于handler，没有复写构造方法，也就是说，在哪个线程 new ,就是用的哪个线程的 looper 构建，我们可以知道我们的 ViewRootImpl 是在 主线程调用 WMI 的 addWindow 时被创建
  final class ViewRootHandler extends Handler {}
  
  //第一个 ViewRootImpl 怎么创建的呢？在 Activity 的创建过程中，是在 WindowManegerGlobal 的 addWindow 时被创建，此时还在主线程中，所以此时的是用 主线程的 looper 初始化 handler 的，所以View.post()会将消息投进主线程的消息队列中
  root = new ViewRootImpl(view.getContext(), display);
  ~~~

*  ViewRootImpl 怎么创建的呢？在ActivityThread.java 中performLaunchActivity() 创建Activity()

*  ```Java
activity = mInstrumentation.newActivity(
                      cl, component.getClassName(), r.intent);
  //然后 activity调用Activity中的attach创建Window
  
  activity.attach(appContext, this, getInstrumentation(), r.token,
          r.ident, app, r.intent, r.activityInfo, title, r.parent,
          r.embeddedID, r.lastNonConfigurationInstances, config,
          r.referrer, r.voiceInteractor, window, r.configCallback);
  
  final void attach(Context context, ActivityThread aThread,
              Instrumentation instr, IBinder token, int ident,
              Application application, Intent intent, ActivityInfo info,
              CharSequence title, Activity parent, String id,
              NonConfigurationInstances lastNonConfigurationInstances,
              Configuration config, String referrer, IVoiceInteractor voiceInteractor,
              Window window, ActivityConfigCallback activityConfigCallback) {
          attachBaseContext(context);
  
          mFragments.attachHost(null /*parent*/);
  
      //在这里创建了 PhoneWindow 
          mWindow = new PhoneWindow(this, window, activityConfigCallback);
          mWindow.setWindowControllerCallback(this);
          mWindow.setCallback(this);
          mWindow.setOnWindowDismissedCallback(this);
          mWindow.getLayoutInflater().setPrivateFactory(this);
          if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
              mWindow.setSoftInputMode(info.softInputMode);
          }
          if (info.uiOptions != 0) {
              mWindow.setUiOptions(info.uiOptions);
          }
          mUiThread = Thread.currentThread();
  
          mMainThread = aThread;
          mInstrumentation = instr;
          mToken = token;
          mIdent = ident;
          mApplication = application;
          mIntent = intent;
          mReferrer = referrer;
          mComponent = intent.getComponent();
          mActivityInfo = info;
          mTitle = title;
          mParent = parent;
          mEmbeddedID = id;
          mLastNonConfigurationInstances = lastNonConfigurationInstances;
          if (voiceInteractor != null) {
              if (lastNonConfigurationInstances != null) {
                  mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
              } else {
                  mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                          Looper.myLooper());
              }
          }
  
          mWindow.setWindowManager(
                  (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                  mToken, mComponent.flattenToString(),
                  (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
          if (mParent != null) {
              mWindow.setContainer(mParent.getWindow());
          }
          mWindowManager = mWindow.getWindowManager();
          mCurrentConfig = config;
  
          mWindow.setColorMode(info.colorMode);
  
          setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
          enableAutofillCompatibilityIfNeeded();
      }
  //当 activity 生命周期到达 Resume 阶段的时候
  public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
              String reason) {
      。。。//这里调用了获取了 decorView
      View decor = r.window.getDecorView();
      。。。//调用了wm的 addView
          wm.addView(decor, l);
      。。。
  }
  public void addView(View view, ViewGroup.LayoutParams params,
              Display display, Window parentWindow) {
      。。。//ViewRootImpl 其实就是在添加进窗口的时候new的，此时是在ActivityThread中回调的哦，也就是说此时是在主线程，拿到的 looper 也就是主线程的 looper
          root = new ViewRootImpl(view.getContext(), display);
      。。。
  }
  ```

#### runOnUiThread

~~~java
//这个是 activity 的方法
final Handler mHandler = new Handler();
public final void runOnUiThread(Runnable action) {
    
        if (Thread.currentThread() != mUiThread) {//当此方法不在主线程调用时，直接调用activity创建的handler 来 post 消息即可
          
            mHandler.post(action);
        } else {
            //如果是在主线程调用，直接执行 run 方法，不丢消息队列
            action.run();
        }
}
~~~



