[TOC]

# 简单解析 Glide

* 参考至`<https://blog.csdn.net/guolin_blog/article/details/53939176/>`

* `implementation 'com.github.bumptech.glide:glide:3.7.0'`

* Glide 是我们常用的图片加载库，我们简单通过源码去解析它，说简单但是篇幅应该不是很短，我们可以想如果要我们自己实现一个图片加载框架怎么实现，我想的是：因为网络加载是耗时的操作所以需要线程池的参与，handler 机制的参与，随意一个网络请求框架，还要有一些缓存策略的参与，至少应该要这4方才能算一个图片加载框架把，不过Glide的实现可远远不止这些，我们可以往下看 Glide 是怎么实现的

* Glide是一款快速高效的Android开源媒体管理和图像加载框架，它将媒体解码，内存和磁盘缓存以及资源池包装成简单易用的界面。

* 使用我们就不说了,随便举个例子

  ~~~java
  Glide.with(this).load(url).into(imageView);
  ~~~

* 接下来我们根据这个例子来解析 Glide 是怎么将我们的图片显示出来的。

## Glide.with(this)

* Glide 提供了多个重载的 with 方法，参数可以为 Context，Activity，Fragment，FragmentActivity，参数类型虽然不同，但是方法实现是一样的，返回的是 RequestManager 对象

  ~~~java
  //返回一个 RequestManager 对象
  public static RequestManager with(Context context) {
      //1. 先获取一个单例的 RequestManagerRetriever 对象
          RequestManagerRetriever retriever = RequestManagerRetriever.get();
      //2. 调用 get 方法获取 RequestManager 对象
          return retriever.get(context);
  }
  
  //1.先获取一个单例的 RequestManagerRetriever 对象
  public class RequestManagerRetriever implements Handler.Callback {
  	private static final RequestManagerRetriever INSTANCE = new RequestManagerRetriever();
  	RequestManagerRetriever() {
      //创建一个主线程的 handler 
          handler = new Handler(Looper.getMainLooper(), this /* Callback */);
  	}
  	public static RequestManagerRetriever get() {
          return INSTANCE;
      }
      @Override
      public boolean handleMessage(Message message) {
          boolean handled = true;
          Object removed = null;
          Object key = null;
          switch (message.what) {
              case ID_REMOVE_FRAGMENT_MANAGER:
                  android.app.FragmentManager fm = (android.app.FragmentManager) message.obj;
                  key = fm;
                  removed = pendingRequestManagerFragments.remove(fm);
                  break;
              case ID_REMOVE_SUPPORT_FRAGMENT_MANAGER:
                  FragmentManager supportFm = (FragmentManager) message.obj;
                  key = supportFm;
                  removed = pendingSupportRequestManagerFragments.remove(supportFm);
                  break;
              default:
                  handled = false;
          }
          if (handled && removed == null && Log.isLoggable(TAG, Log.WARN)) {
              Log.w(TAG, "Failed to remove expected request manager fragment, manager: " + key);
          }
          return handled;
      }
  }
  
  //2. 调用 get 方法获取 RequestManager 对象
  public RequestManager get(Context context) {
          if (context == null) {
              throw new IllegalArgumentException("You cannot start a load on a null Context");
          } else if (Util.isOnMainThread() && !(context instanceof Application)) {
              //如果此时在主线程并且 Context不为Application类型，则进行以下判断
              //传入非Application参数的情况。不管你在Glide.with()方法中传入的是Activity、FragmentActivity、v4包下的Fragment、还是app包下的Fragment，最终的流程都是一样的，那就是会向当前的Activity当中添加一个隐藏的Fragment。那么这里为什么要添加一个隐藏的Fragment呢？因为Glide需要知道加载的生命周期。（很重要的一步）很简单的一个道理，如果你在某个Activity上正在加载着一张图片，结果图片还没加载出来，Activity就被用户关掉了，那么图片还应该继续加载吗？当然不应该。可是Glide并没有办法知道Activity的生命周期，于是Glide就使用了添加隐藏Fragment的这种小技巧，因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件并停止图片加载了。
              if (context instanceof FragmentActivity) {
                  return get((FragmentActivity) context);
              } else if (context instanceof Activity) {
                  return get((Activity) context);
              } else if (context instanceof ContextWrapper) {
                  return get(((ContextWrapper) context).getBaseContext());
              }
          }
  //如果此时不在主线程调用 或者 Context为Application类型就直接到这里调用这个方法，这是最简单的一种情况，因为Application对象的生命周期即应用程序的生命周期，因此Glide并不需要做什么特殊的处理，它自动就是和应用程序的生命周期是同步的，如果应用程序关闭的话，Glide的加载也会同时终止。
          return getApplicationManager(context);
  }
  ~~~

* 现在先解析 getApplicationManager(context)

  ~~~java
  private RequestManager getApplicationManager(Context context) {
          // Either an application context or we're on a background thread.
          if (applicationManager == null) {
              synchronized (this) {
                  if (applicationManager == null) {
            // 通常，暂停/恢复由我们添加到Fragment或Activity中的Fragemnt 处理。但是，在这种情况下，由于附加到 application 的 manager 将不接收别的生命周期事件，因此必须强制 manager 使用 Application 的 Lifecycle 开始 resumed。 创建 requestManager 对象   
              applicationManager = new RequestManager(context.getApplicationContext(),
                 new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
                  }
              }
          }
          return applicationManager;
  }
  
  class ApplicationLifecycle implements Lifecycle {
      @Override
      public void addListener(LifecycleListener listener) {
          listener.onStart();
      }
  }
  //RequestManager 实现了 生命周期监听者的接口
  public class RequestManager implements LifecycleListener {
      private final Context context;
      private final Lifecycle lifecycle;
      private final RequestManagerTreeNode treeNode;
      private final RequestTracker requestTracker;
      private final Glide glide;
      private final OptionsApplier optionsApplier;
      private DefaultOptions options;
  
      public RequestManager(Context context, Lifecycle lifecycle, RequestManagerTreeNode treeNode) {
          this(context, lifecycle, treeNode, new RequestTracker(), new ConnectivityMonitorFactory());
      }
  
      RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
              RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
          this.context = context.getApplicationContext();
          this.lifecycle = lifecycle;
          this.treeNode = treeNode;
          this.requestTracker = requestTracker;
          this.glide = Glide.get(context);
          this.optionsApplier = new OptionsApplier();
  
          ConnectivityMonitor connectivityMonitor = factory.build(context,
                  new RequestManagerConnectivityListener(requestTracker));
  
          //如果我们是应用程序级requestManager，那么我们可能在后台线程上创建。在这种情况下，我们不能冒险同步暂停或执行请求，因此我们通过向主线程发送消息来延迟作为生命周期监听器的加载项来绕过这个问题
          if (Util.isOnBackgroundThread()) {
              new Handler(Looper.getMainLooper()).post(new Runnable() {
                  @Override
                  public void run() {
                      lifecycle.addListener(RequestManager.this);
                  }
              });
          } else {
              lifecycle.addListener(this);
          }
          lifecycle.addListener(connectivityMonitor);
  }
  ~~~

* 我们现在解析一下如果context是activity的情况

  ~~~java
  @TargetApi(Build.VERSION_CODES.HONEYCOMB)
  public RequestManager get(Activity activity) {
          if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
              return get(activity.getApplicationContext());
          } else {
              //跑这里，先判断activity被销毁了没有
              assertNotDestroyed(activity);
              //然后拿到 activity 的碎片管理者
              android.app.FragmentManager fm = activity.getFragmentManager();
              //然后调用 fragmentGet(activity, fm);得到 RequestManager 对象，去看看
              return fragmentGet(activity, fm);
          }
  }
  
  RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
  //先拿到与Activity生命周期关联的RequestManagerFragment碎片(这种Fragment是不带有任何界面的，主要是用于同步生命周期。)，然后碎片中取出 requestManager 对象
          RequestManagerFragment current = getRequestManagerFragment(fm);
          RequestManager requestManager = current.getRequestManager();
          if (requestManager == null) {
              //如果为空就需要为这个 RequestManagerFragment 对象设置一个 requestManager,这里管理者创建需要 这个碎片对象的生命周期和 context，很重要
              requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
              current.setRequestManager(requestManager);
          }
          return requestManager;
  }
  
  RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {
          RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);//先从缓存里找，因为我们可能在同个Activity使用多次Glide，我们不可能每使用一次就创建一个Fragement
          if (current == null) {
              current = pendingRequestManagerFragments.get(fm);
              if (current == null) {
                  //这里就是创建碎片，创建 RequestManagerFragment()
                  current = new RequestManagerFragment();
                  pendingRequestManagerFragments.put(fm, current);
                  //将无UI的碎片添加进Activity
                  fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                  handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
              }
          }
          return current;
  }
  public class RequestManagerFragment extends Fragment {
  	public RequestManagerFragment() {
          this(new ActivityFragmentLifecycle());
      }
  }
  ~~~

  * 小结：Glide 将请求的管理封装成一个 请求管理者对象，将请求与context的生命周期对应以便重启请求或者销毁不必要的请求，这个类主要用于管理和启动Glide的所有请求，可以使用activity，fragment或者连接生命周期的事件去智能的停止，启动，和重启请求。也可以检索或者通过实例化一个新的对象，或者使用静态的Glide去利用构建在Activity和Fragment生命周期处理中。它的方法跟你的Fragment和Activity的是同步的。

## Glide.with(this).load(url)

* 现在看方法名字应该到请求网络的部分了吧。

* 我们以我们最经常使用的来分析，我们最经常传比如随便一张网络图片的网络链接路径

  ~~~java
  //那就会调用 string 参数的 load 方法
  public DrawableTypeRequest<String> load(String string) {
          return (DrawableTypeRequest<String>) fromString().load(string);
  }
  
  public DrawableTypeRequest<String> fromString() {
      //这里就按我们给的参数的类型返回对应的 ModelLoader ，由于我们刚才传入的参数是String.class，因此最终得到的是StreamStringLoader对象，它是实现了ModelLoader接口的。loadGeneric()方法是要返回一个DrawableTypeRequest 对象的，因此在loadGeneric()方法的最后又去new了一个DrawableTypeRequest对象，然后把刚才获得的ModelLoader对象，还有一大堆杂七杂八的东西都传了进去
          return loadGeneric(String.class);
  }
  
  @Override
  public DrawableRequestBuilder<ModelType> load(ModelType model) {
      //现在就到父类的 load 方法看看怎么操作的
          super.load(model);
      //此时返回 DrawableRequestBuilder<String> 对象，真正实现是 DrawableTypeRequest 对象
          return this;
  }
  
  public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> 	 load(ModelType model) {
      //。。。。。。什么鬼，只是简单地设置而已，没有网络访问，猜错了。。。那么网络访问的步骤就在 into 方法里了，真是操蛋
          this.model = model;
          isModelSet = true;
          return this;
  }
  ~~~

## Glide.with(this).load(url).into(imageView);

* 这是最后的 一步 into ，我们看看里面是怎么进行网络请求的

* 由于 DrawableTypeRequest 没有复写 into 方法，所以实现在父类中

  ~~~java
  @Override
  public Target<GlideDrawable> into(ImageView view) {
      //继续去它的父类看
          return super.into(view);
  }
  ~~~

* GenericRequestBuilder 的 into 方法

  ~~~java
  public Target<TranscodeType> into(ImageView view) {
      //先确定是否为 主线程
          Util.assertMainThread();
          if (view == null) {
              throw new IllegalArgumentException("You must pass in a non null View");
          }
  
          if (!isTransformationSet && view.getScaleType() != null) {
              switch (view.getScaleType()) {
                  case CENTER_CROP:
                      applyCenterCrop();
                      break;
                  case FIT_CENTER:
                  case FIT_START:
                  case FIT_END:
                      applyFitCenter();
                      break;
                  //$CASES-OMITTED$
                  default:
                      // Do nothing.
              }
          }
  //这里针对ImageView的填充方式做了筛选并对应设置到requestOptions上。最终的是通过ImageView和转码类型（transcodeClass）创建不通过的Target（例如Bitmap对应的BitmapImageViewTarget和Drawable对应的DrawableImageViewTarget）
          return into(glide.buildImageViewTarget(view, transcodeClass));
  }
  public <Y extends Target<TranscodeType>> Y into(Y target) {
        Util.assertMainThread();
          if (target == null) {
              throw new IllegalArgumentException("You must pass in a non null Target");
          }
          if (!isModelSet) {
              throw new IllegalArgumentException("You must first set a model (try #load())");
          }
  
          Request previous = target.getRequest();
  
          if (previous != null) {
              previous.clear();
              requestTracker.removeRequest(previous);
              previous.recycle();
          }
  		//现在创建 request 请求,这个请求的具体实现不指定一般是 GenericRequest
          Request request = buildRequest(target);
      	//target.setRequest(request)也是一个比较值得注意的地方，如果target是ViewTarget，那么request会被设置到View的tag上。这样其实是有一个好处，每一个View有一个自己的Request，如果有重复请求，那么都会先去拿到上一个已经绑定的Request，并且从RequestManager中清理回收掉。这应该是去重的功能。
          target.setRequest(request);
          lifecycle.addListener(target);
      	//这个方法非常的复杂，主要用于触发请求、编解码、装载和缓存这些功能。下面就一步一步来看吧：
          requestTracker.runRequest(request);
  
          return target;
  }
  ~~~
  
* 以下参考至`<http://frodoking.github.io/2015/10/10/android-glide/>`

  ~~~java
  public void runRequest(Request request) {
          requests.add(request);
          if (!isPaused) {
              //请求开始
              request.begin();
          } else {
              //否则挂起
              pendingRequests.add(request);
          }
  }
  ~~~

* 接下来就直接去请求开始的位置看，request 的具体实现是 GenericRequest

  ~~~java
  @Override
  public void begin() {
          startTime = LogTime.getLogTime();
          // 如果model空的，那么是不能执行的。 这里的model就是前面提到的RequestBuilder中的model
          if (model == null) {
              onException(null);
              return;
          }
  
          status = Status.WAITING_FOR_SIZE;
          // 如果当前的View尺寸已经加载获取到了，那么就会进入真正的加载流程。
          if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
              //再解析这里2
              onSizeReady(overrideWidth, overrideHeight);
          } else {
              //下面先解析这里1
              // 反之，当前View还没有画出来，那么是没有尺寸的。
  			// 这里会调用到ViewTreeObserver.addOnPreDrawListener。
  			// 等待View的尺寸都ok，才会继续
              target.getSize(this);
          }
  		// 如果等待和正在执行状态，那么当前会加载占位符Drawable，我们的占位符就是在这里设置的
          if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
              target.onLoadStarted(getPlaceholderDrawable());
          }
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
              logV("finished run method in " + LogTime.getElapsedMillis(startTime));
          }
  }
  ~~~

* 1.现在就到 ViewTarget 对象里的 getSize 方法

  ~~~java
  @Override
  public void getSize(SizeReadyCallback cb) {
          sizeDeterminer.getSize(cb);
  }
  public void getSize(SizeReadyCallback cb) {
              int currentWidth = getViewWidthOrParam();
              int currentHeight = getViewHeightOrParam();
              if (isSizeValid(currentWidth) && isSizeValid(currentHeight)) {
                  //如果尺寸已经获取到了就只接调用 onSizeReady ，负责就设置回调
                  cb.onSizeReady(currentWidth, currentHeight);
              } else {
                  //我们希望按照添加回调的顺序通知它们，并且一次只希望添加一个或两个回调，因此列表是一个合理的选择。
                  if (!cbs.contains(cb)) {
                      cbs.add(cb);
                  }
                  if (layoutListener == null) {
                      final ViewTreeObserver observer = view.getViewTreeObserver();
                      //创建一个监听者
                      layoutListener = new SizeDeterminerLayoutListener(this);
                      //设置进 View 的 ViewTreeObserver
                      observer.addOnPreDrawListener(layoutListener);
                  }
              }
  }
  private static class SizeDeterminerLayoutListener implements  ViewTreeObserver.OnPreDrawListener {
              private final WeakReference<SizeDeterminer> sizeDeterminerRef;
  
              public SizeDeterminerLayoutListener(SizeDeterminer sizeDeterminer) {
                  sizeDeterminerRef = new WeakReference<SizeDeterminer>(sizeDeterminer);
              }
  
              @Override
              public boolean onPreDraw() {
                  if (Log.isLoggable(TAG, Log.VERBOSE)) {
                      Log.v(TAG, "OnGlobalLayoutListener called listener=" + this);
                  }
                  SizeDeterminer sizeDeterminer = sizeDeterminerRef.get();
                  if (sizeDeterminer != null) {
                      sizeDeterminer.checkCurrentDimens();
                  }
                  return true;
              }
  }
  private void checkCurrentDimens() {
              if (cbs.isEmpty()) {
                  return;
              }
  
              int currentWidth = getViewWidthOrParam();
              int currentHeight = getViewHeightOrParam();
              if (!isSizeValid(currentWidth) || !isSizeValid(currentHeight)) {
                  return;
              }
  
      //重要的其实是这里，看名字应该是去通知调用了
              notifyCbs(currentWidth, currentHeight);
              //保留对布局侦听器的引用，并在此处删除它，而不是让观察者自己删除它，因为我们将侦听器添加到的观察者几乎会立即合并到另一个观察者中，因此永远不会存在。如果我们改为保留对侦听器的引用并在此处删除它，我们将得到当前视图树观察器，并且应该成功
              ViewTreeObserver observer = view.getViewTreeObserver();
              if (observer.isAlive()) {
                  observer.removeOnPreDrawListener(layoutListener);
              }
              layoutListener = null;
  }
  private void notifyCbs(int width, int height) {
              for (SizeReadyCallback cb : cbs) {
                  //就是这里，最终还是会调用 onSizeReady 方法，因为尺寸已经得到了
                  cb.onSizeReady(width, height);
              }
              cbs.clear();
   }
  ~~~

* 2.onSizeReady(overrideWidth, overrideHeight);

  ~~~java
  @Override
  public void onSizeReady(int width, int height) {
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
              logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
          }
          if (status != Status.WAITING_FOR_SIZE) {
              return;
          }
          status = Status.RUNNING;
  
          width = Math.round(sizeMultiplier * width);
          height = Math.round(sizeMultiplier * height);
  
          ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
          final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
  
          if (dataFetcher == null) {
              onException(new Exception("Failed to load model: \'" + model + "\'"));
              return;
          }
          ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
              logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
          }
          loadedFromMemoryCache = true;
      //看到这里很重要的异步，引擎发起请求了
          loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                  priority, isMemoryCacheable, diskCacheStrategy, this);
          loadedFromMemoryCache = resource != null;
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
              logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
          }
  }
  ~~~

* 接下来就是引擎发起请求的部分了engine.load

  ~~~java
  public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
              DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
              Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
          Util.assertMainThread();
          long startTime = LogTime.getLogTime();
  
          final String id = fetcher.getId();
      // 创建key，这是给每次加载资源的唯一标示。
          EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                  loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                  transcoder, loadProvider.getSourceEncoder());
      // 通过key查找缓存资源 （PS 这里的缓存主要是内存中的缓存，切记,可以查看MemoryCache）
          EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
          if (cached != null) {
              // 如果有，那么直接利用当前缓存的资源。
              cb.onResourceReady(cached);
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  logWithTimeAndKey("Loaded resource from cache", startTime, key);
              }
              return null;
          }
  	// 这是一个二级内存的缓存引用，很简单用了一个Map<Key, WeakReference<EngineResource<?>>>装载起来的。为何要两级内存缓存（loadFromActiveResources）。参考博客中的理解是一级缓存采用LRU算法进行缓存，并不能保证全部能命中，添加二级缓存提高命中率之用；
  	// 这个缓存主要是谁来放进去呢？ 可以参考上面一级内存缓存loadFromCache方法。
          EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
          if (active != null) {
              cb.onResourceReady(active);
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  logWithTimeAndKey("Loaded resource from active resources", startTime, key);
              }
              return null;
          }
  		// 根据key获取缓存的job。
          EngineJob current = jobs.get(key);
          if (current != null) {
              current.addCallback(cb);// 给当前job添加上回调Callback
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  logWithTimeAndKey("Added to existing load", startTime, key);
              }
              return new LoadStatus(cb, current);
          }
  		// 创建job,EngineJob充当了管理和调度者，主要负责加载和各类回调通知；DecodeJob是真正干活的劳动者，配合 EngineRunnable 
          EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
          DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                  transcoder, diskCacheProvider, diskCacheStrategy, priority);
          EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
          jobs.put(key, engineJob);
          engineJob.addCallback(cb);
      	// 放入线程池，执行
          engineJob.start(runnable);
  
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
              logWithTimeAndKey("Started new load", startTime, key);
          }
          return new LoadStatus(cb, engineJob);
  }
  public void start(EngineRunnable engineRunnable) {
          this.engineRunnable = engineRunnable;
          future = diskCacheService.submit(engineRunnable);
  }
  public Future<?> submit(Runnable task) {
          if (task == null) throw new NullPointerException();
          RunnableFuture<Void> ftask = newTaskFor(task, null);
          execute(ftask);
          return ftask;
  }
  ~~~

* EngineRunnable

  ~~~java
  @Override
  public void run() {
          if (isCancelled) {
              return;
          }
  
          Exception exception = null;
          Resource<?> resource = null;
          try {
              resource = decode();
          } catch (Exception e) {
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  Log.v(TAG, "Exception decoding", e);
              }
              exception = e;
          }
  
          if (isCancelled) {
              if (resource != null) {
                  resource.recycle();
              }
              return;
          }
  
          if (resource == null) {
              onLoadFailed(exception);
          } else {
              onLoadComplete(resource);
          }
  }
  //加载图片是通过decode()方法进行了，这里判断了是否要走缓存，我们直接不看缓存直接走decodeFromSource()，虽然EngineRunnable默认是走一次缓存的，但是第一次加载是没有缓存的，最后还是走decodeFromSource()
  private Resource<?> decode() throws Exception {
          if (isDecodingFromCache()) {
              return decodeFromCache();
          } else {
              return decodeFromSource();
          }
  }
  
  private Resource<?> decodeFromSource() throws Exception {
          return decodeJob.decodeFromSource();
  }
  
  public Resource<Z> decodeFromSource() throws Exception {
          Resource<T> decoded = decodeSource();
          return transformEncodeAndTranscode(decoded);
  }
  
  private Resource<T> decodeSource() throws Exception {
          Resource<T> decoded = null;
          try {
              long startTime = LogTime.getLogTime();
              //这个时候就是真正是网络中加载图片了,在DecodeJob中decodeSource()首先通过fetcher.loadData()获取data，这个fetcher是啥呢，下面下DecodeJob的生成去找到它
              final A data = fetcher.loadData(priority);
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  logWithTimeAndKey("Fetched data", startTime);
              }
              if (isCancelled) {
                  return null;
              }
              //fetcher.loadData()返回的内容会封装在ImageVideoWrapper对象中回到DecodeJob返回，接着会把他传递到decodeFromSourceData()处理：
              decoded = decodeFromSourceData(data);
          } finally {
              fetcher.cleanup();
          }
          return decoded;
  }
  //这里去看这个fetcher是啥呢
  @Override
  public void onSizeReady(int width, int height) {
      ......
      ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
      final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
      ......
      //fetcher就是从这里传的
      loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                  priority, isMemoryCacheable, diskCacheStrategy, this);
      ......
  }
  //loadProvider.getModelLoader()返回的是ImageVideoModelLoader ，直接去这里找方法
  @Override
  public DataFetcher<ImageVideoWrapper> getResourceFetcher(A model, int width, int height) {
          DataFetcher<InputStream> streamFetcher = null;
          if (streamLoader != null) {
              streamFetcher = streamLoader.getResourceFetcher(model, width, height);
          }
          DataFetcher<ParcelFileDescriptor> fileDescriptorFetcher = null;
          if (fileDescriptorLoader != null) {
              fileDescriptorFetcher = fileDescriptorLoader.getResourceFetcher(model, width, height);
          }
  
          if (streamFetcher != null || fileDescriptorFetcher != null) {
              //看这里可以看到返回的类型是 ImageVideoFetcher 对象，所以，上面的 fetcher 就是这个 ImagerVideoFetcher
              return new ImageVideoFetcher(streamFetcher, fileDescriptorFetcher);
          } else {
              return null;
          }
  }
  ~~~

* 现在就到ImageVideoFetcher里找 fetcher.loadData(priority);

  ~~~java
  @Override
  public ImageVideoWrapper loadData(Priority priority) throws Exception {
     // 随后进入了loadData()方法，其实在这里fileDescriptor获取为空，这是因为最开始初始化的原因，这里StreamStringLoader和FileDescriptorStringLoader生成太过复杂，我们只走主流程，这里就是去真正发起网络请求了，这里太复杂了，看不过来就不理他了，只知道去发起网络请求就好了，不理它了，能力达不到去解析这一部分,下面去看这么解析网络数据的
              InputStream is = null;
              if (streamFetcher != null) {
                  try {
                      is = streamFetcher.loadData(priority);
                  } catch (Exception e) {
                      if (Log.isLoggable(TAG, Log.VERBOSE)) {
                          Log.v(TAG, "Exception fetching input stream, trying ParcelFileDescriptor", e);
                      }
                      if (fileDescriptorFetcher == null) {
                          throw e;
                      }
                  }
              }
              ParcelFileDescriptor fileDescriptor = null;
              if (fileDescriptorFetcher != null) {
                  try {
                      fileDescriptor = fileDescriptorFetcher.loadData(priority);
                  } catch (Exception e) {
                      if (Log.isLoggable(TAG, Log.VERBOSE)) {
                          Log.v(TAG, "Exception fetching ParcelFileDescriptor", e);
                      }
                      if (is == null) {
                          throw e;
                      }
                  }
              }
            return new ImageVideoWrapper(is, fileDescriptor);
  }
  ~~~
  
* fetcher.loadData()返回的内容会封装在ImageVideoWrapper对象中回到DecodeJob返回，接着会把他传递到decodeFromSourceData()处理

  ~~~java
  private Resource<T> decodeFromSourceData(A data) throws IOException {
          final Resource<T> decoded;
          if (diskCacheStrategy.cacheSource()) {
              decoded = cacheAndDecodeSourceData(data);
          } else {
              //第一次走这里
              long startTime = LogTime.getLogTime();
              //loadProvider.getSourceDecoder()返回的是,到这里卡住了，有时间再搞好了，要期末考了
              decoded = loadProvider.getSourceDecoder().decode(data, width, height);
              if (Log.isLoggable(TAG, Log.VERBOSE)) {
                  logWithTimeAndKey("Decoded from source", startTime);
              }
          }
          return decoded;
  }
  ~~~

* 得到

  ~~~java
  @Override
  public Resource<GifBitmapWrapper> decode(ImageVideoWrapper source, int width, int height) throws IOException {
          ByteArrayPool pool = ByteArrayPool.get();
          byte[] tempBytes = pool.getBytes();
  
          GifBitmapWrapper wrapper = null;
          try {
              wrapper = decode(source, width, height, tempBytes);
          } finally {
              pool.releaseBytes(tempBytes);
          }
          return wrapper != null ? new GifBitmapWrapperResource(wrapper) : null;
  }
  private GifBitmapWrapper decode(ImageVideoWrapper source, int width, int height, byte[] bytes) throws IOException {
          final GifBitmapWrapper result;
          if (source.getStream() != null) {
              result = decodeStream(source, width, height, bytes);
          } else {
              result = decodeBitmapWrapper(source, width, height);
          }
          return result;
  }
  private GifBitmapWrapper decodeStream(ImageVideoWrapper source, int width, int height, byte[] bytes)
              throws IOException {
          InputStream bis = streamFactory.build(source.getStream(), bytes);
          bis.mark(MARK_LIMIT_BYTES);
          ImageHeaderParser.ImageType type = parser.parse(bis);
          bis.reset();
  
          GifBitmapWrapper result = null;
          if (type == ImageHeaderParser.ImageType.GIF) {
              result = decodeGifWrapper(bis, width, height);
          }
          // Decoding the gif may fail even if the type matches.
          if (result == null) {
              // We can only reset the buffered InputStream, so to start from the beginning of the stream, we need to
              // pass in a new source containing the buffered stream rather than the original stream.
              ImageVideoWrapper forBitmapDecoder = new ImageVideoWrapper(bis, source.getFileDescriptor());
              result = decodeBitmapWrapper(forBitmapDecoder, width, height);
          }
          return result;
  }
  private GifBitmapWrapper decodeBitmapWrapper(ImageVideoWrapper toDecode, int width, int height) throws IOException {
          GifBitmapWrapper result = null;
  
          Resource<Bitmap> bitmapResource = bitmapDecoder.decode(toDecode, width, height);
          if (bitmapResource != null) {
              result = new GifBitmapWrapper(bitmapResource, null);
          }
  
          return result;
  }
  ~~~

  