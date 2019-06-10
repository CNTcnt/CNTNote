[TOC]

# View的绘制流程、

* 之前做的笔记不知道哪里去了懒得找了，现在再做一次好了

## 绘制从哪里开始的

* View的绘制肯定是 ViewRoot 来管理的啊，我们将 View 添加进 Window 的时候，里面会创建 ViewRootImpl 对象，然后调用 它的 setVIew 方法，我们直接去里面看一下

  ~~~java
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      ......
      int res; /* = WindowManagerImpl.ADD_OKAY; */
  
                  // Schedule the first layout -before- adding to the window
                  // manager, to make sure we do the relayout before receiving
                  // any other events from the system.在添加到窗口管理器之前安排第一个布局，以确保在从系统接收到任何其他事件之前进行重新布局。就是这里了，第一次绘制
                  requestLayout();
                  if ((mWindowAttributes.inputFeatures
                          & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                      mInputChannel = new InputChannel();
                  }
                  mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                          & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                  try {
                      mOrigWindowType = mWindowAttributes.type;
                      mAttachInfo.mRecomputeGlobalAttributes = true;
                      collectViewAttributes();
                      res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                              getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                              mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                              mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                  } catch (RemoteException e) {
                      mAdded = false;
                      mView = null;
                      mAttachInfo.mRootView = null;
                      mInputChannel = null;
                      mFallbackEventHandler.setView(null);
                      unscheduleTraversals();
                      setAccessibilityFocus(null, null);
                      throw new RuntimeException("Adding window failed", e);
                  } finally {
                      if (restore) {
                          attrs.restore();
                      }
                  }    
      ......
  }
  ~~~

* requestLayout();

  ~~~java
  @Override
      public void requestLayout() {
          if (!mHandlingLayoutInLayoutRequest) {
              checkThread();
              mLayoutRequested = true;
              scheduleTraversals();
          }
      }
  //小插曲，这里调用了checkThread 方法，也就是说，绘制前是会检查线程的，那怎么检查就直接看下面代码里
  void checkThread() {
          if (mThread != Thread.currentThread()) {
              throw new CalledFromWrongThreadException(
                      "Only the original thread that created a view hierarchy can touch its views.");//只有创建视图层次结构的原始线程才能触摸其视图。
          }
  }
  public ViewRootImpl(Context context, Display display) {     
          mThread = Thread.currentThread();//就是在创建 ViewRootImpl 的线程
  }
  ~~~

* scheduleTraversals();

  ~~~java
  void scheduleTraversals() {
          if (!mTraversalScheduled) {
              mTraversalScheduled = true;
              //设置同步障碍，确保mTraversalRunnable优先被执行，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息。
          mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          //内部通过Handler发送了一个异步消息(我们如果不指定，那么我们发送的是同步消息)
          mChoreographer.postCallback(
                  Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
              if (!mUnbufferedInputDispatch) {
                  scheduleConsumeBatchedInput();
              }
              notifyRendererOfFramePending();
              pokeDrawLockIfNeeded();
          }
  }
  
  final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
  
  final class TraversalRunnable implements Runnable {
          @Override
          public void run() {
              doTraversal();
          }
  }
  ~~~

* doTraversal()

  ~~~java
  void doTraversal() {
          if (mTraversalScheduled) {
              mTraversalScheduled = false;
              mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
  
              if (mProfile) {
                  Debug.startMethodTracing("ViewAncestor");
              }
  
              //看名字，执行遍历
              performTraversals();
  
              if (mProfile) {
                  Debug.stopMethodTracing();
                  mProfile = false;
              }
          }
  }
  ~~~

* performTraversals();

  ~~~java
  //这个方法太长了，没必要全部写出来，把重点我们需要讨论的绘制流程写出,就是依次执行这3个重要的方法
  ......
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  ......
  if (didLayout) {
              performLayout(lp, mWidth, mHeight);
      ......
  }
  ......
  if (!cancelDraw && !newSurface) {
              if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                  for (int i = 0; i < mPendingTransitions.size(); ++i) {
                      mPendingTransitions.get(i).startChangingAnimations();
                  }
                  mPendingTransitions.clear();
              }
  
              performDraw();
  }
  ~~~

* 小结：View的绘制流程就是从 ViewRootImpl 的 performTraversals 开始的，下面我们就一个一个去解析这个方法里的绘制流程

## Measure 过程

* performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

  ~~~java
  private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
          if (mView == null) {
              return;
          }
          Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
          try {
              //调用 View 的 measure ,一般我们这个的 View 是一个 继承了VeiwGroup的Layout，这里的参数是什么呢，我们去看一下
              mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_VIEW);
          }
  }
  
  ~~~

### MeasureSpec 是啥子

`<https://www.jianshu.com/p/5a71014e7b1b>`

* MeasureSpec 是两个单词组成，翻译过来“测量规格”或者“测量参数”，官方文档对他的说明是“一个MeasureSpec封装了从父容器传递给子容器的布局要求”，更精确的说法应该这个 MeasureSpec 是由父View的 MeasureSpec 和子View的 LayoutParams 通过简单的计算得出一个针对子View的测量要求，这个测量要求就是 MeasureSpec 。
* MeasureSpec 是一个大小跟模式的组合值, MeasureSpec 中的值是一个整型（32位）将 size 和 mode 打包成一个 Int型，其中高两位是 mode，后面30位存的是 size，是为了减少对象的分配开支。

* **UPSPECIFIED** : 父容器对于子容器没有任何限制,子容器想要多大就多大
  **EXACTLY**: 父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大的空间。
  **AT_MOST**：子容器可以是声明大小内的任意大小

* ~~~java
  public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();
  mWinFrame = new Rect();
  private void performTraversals() {
      Rect frame = mWinFrame;
      if (mWidth != frame.width() || mHeight != frame.height()) {
                  mWidth = frame.width();
                  mHeight = frame.height();
      }
      
      ......
      WindowManager.LayoutParams lp = mWindowAttributes;
      ......
  	int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
      int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  }
  public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
      public LayoutParams() {
          //可以看到这里是 match——parent
              super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
              type = TYPE_APPLICATION;
              format = PixelFormat.OPAQUE;
      }
  }
  private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
          if (mView == null) {
              return;
          }
          Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
          try {
              mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_VIEW);
          }
  }
  ~~~

* getRootMeasureSpec(mWidth, lp.width);

  ~~~java
  private static int getRootMeasureSpec(int windowSize, int rootDimension) {
          int measureSpec;
          switch (rootDimension) {
  
          case ViewGroup.LayoutParams.MATCH_PARENT:
              // Window can't resize. Force root view to be windowSize.
              measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
              break;
          case ViewGroup.LayoutParams.WRAP_CONTENT:
              // Window can resize. Set max size for root view.
              measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
              break;
          default:
              // Window wants to be an exact size. Force root view to be that size.
              measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
              break;
          }
          return measureSpec;
  }
  ~~~

* mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);

  ~~~java
  //在View.java中，并且不可复写
  public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
          boolean optical = isLayoutModeOptical(this);
          if (optical != isLayoutModeOptical(mParent)) {
              Insets insets = getOpticalInsets();
              int oWidth  = insets.left + insets.right;
              int oHeight = insets.top  + insets.bottom;
              widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
              heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
          }
  
          // Suppress sign extension for the low bytes
          long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
          if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
  
          final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
  
          // Optimize layout by avoiding an extra EXACTLY pass when the view is
          // already measured as the correct size. In API 23 and below, this
          // extra pass is required to make LinearLayout re-distribute weight.
          final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                  || heightMeasureSpec != mOldHeightMeasureSpec;
          final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                  && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
          final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                  && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
          final boolean needsLayout = specChanged
                  && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
  
          if (forceLayout || needsLayout) {
              // first clears the measured dimension flag
              mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
  
              resolveRtlPropertiesIfNeeded();
  
              int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
              if (cacheIndex < 0 || sIgnoreMeasureCache) {
                  // measure ourselves, this should set the measured dimension flag back
                  //这里就是我们复写的方法
                  onMeasure(widthMeasureSpec, heightMeasureSpec);
                  mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
              } else {
                  long value = mMeasureCache.valueAt(cacheIndex);
                  // Casting a long to int drops the high 32 bits, no mask needed
                  setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                  mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
              }
  
              // flag not set, setMeasuredDimension() was not invoked, we raise
              // an exception to warn the developer
              if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                  throw new IllegalStateException("View with id " + getId() + ": "
                          + getClass().getName() + "#onMeasure() did not set the"
                          + " measured dimension by calling"
                          + " setMeasuredDimension()");
              }
  
              mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
          }
  
          mOldWidthMeasureSpec = widthMeasureSpec;
          mOldHeightMeasureSpec = heightMeasureSpec;
  
          mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                  (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
  }
  ~~~

* 父 View 的 measure 的过程会先测量子 View，等子 View 测量结果出来后，再来测量自己，我们可以查看 LinearLayout的源码看一下

  ~~~java
  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          if (mOrientation == VERTICAL) {
              measureVertical(widthMeasureSpec, heightMeasureSpec);
          } else {
              measureHorizontal(widthMeasureSpec, heightMeasureSpec);
          }
  }
  
  void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
      //ViewGroup需要测量 子View 的布局大小
      ......
      measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                          heightMeasureSpec, usedHeight);
      ......
  }
  
  void measureChildBeforeLayout(View child, int childIndex,
              int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
              int totalHeight) {
          measureChildWithMargins(child, widthMeasureSpec, totalWidth,
                  heightMeasureSpec, totalHeight);
  }
  
  //measureChildWithMargins 就是用来测量某个子 View 的，我们来分析是怎样测量的，具体看注释：
  protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) { 
  
  	// 子View的LayoutParams，你在xml的layout_width和layout_height,
  	// layout_xxx的值最后都会封装到这个个LayoutParams。先拿到这个 layoutParams 
  	final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();   
  
  	//根据父View的测量规格和父View自己的Padding，
  	//还有子View的Margin和已经用掉的空间大小（widthUsed），就能算出子View的MeasureSpec,具体计算过程看getChildMeasureSpec方法。
  	final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,            
  	mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);    
  
  	final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,           
  	mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin  + heightUsed, lp.height);  
  
  	//通过父View的MeasureSpec和子View的自己LayoutParams的计算，算出子View的MeasureSpec，然后父容器传递给子容器的
  	// 然后让子View用这个MeasureSpec（一个测量要求，比如不能超过多大）去测量自己，如果子View是ViewGroup 那还会递归往下测量。
  	//就是说 测量前就要先得到 ＭｅａｓｕｒｅＳｐｅｃ 才可以测量，而这个  ＭｅａｓｕｒｅＳｐｅｃ 就需要父View和子View来共同得到，而怎么得到这个 MeasureSppec 就看下面   
  	child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
  }
  
  
  // spec参数   表示父View的MeasureSpec 
  // padding参数    父View的Padding+子View的Margin，父View的大小减去这些边距，才能精确算出
  //               子View的MeasureSpec的size
  // childDimension参数  表示该子View内部LayoutParams属性的值（lp.width或者lp.height）
  //                    可以是wrap_content、match_parent、一个精确指(an exactly size),
  //举例：根据父View的 parentWidthMeasureSpec 和 子View 的 lp.width 就可以得出此时 子View 的MeasureSpec 
  public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  
      int specMode = MeasureSpec.getMode(spec);  //获得父View的mode  
      int specSize = MeasureSpec.getSize(spec);  //获得父View的大小  
  
     //父View的大小-自己的Padding+子View的Margin，得到值才是子View的大小。
      int size = Math.max(0, specSize - padding);   
    
      int resultSize = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
      int resultMode = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
    
      switch (specMode) {  
      // Parent has imposed an exact size on us  
      //1、父View是 EXACTLY 的 ！  
      case MeasureSpec.EXACTLY:   
          //1.1、子View的width或height是个精确值 (an exactly size)  
          if (childDimension >= 0) {            
              resultSize = childDimension;         //size为精确值  
              resultMode = MeasureSpec.EXACTLY;    //mode为 EXACTLY 。  
          }   
          //1.2、子View的width或height为 MATCH_PARENT/FILL_PARENT   
          else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size. So be it.  
              resultSize = size;                   //size为父视图大小  
              resultMode = MeasureSpec.EXACTLY;    //mode为 EXACTLY 。  
          }   
          //1.3、子View的width或height为 WRAP_CONTENT  
          else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;                   //size为父视图大小  
              resultMode = MeasureSpec.AT_MOST;    //mode为AT_MOST 。  
          }  
          break;  
    
      // Parent has imposed a maximum size on us  
      //2、父View是AT_MOST的 ！      
      case MeasureSpec.AT_MOST:  
          //2.1、子View的width或height是个精确值 (an exactly size)  
          if (childDimension >= 0) {  
              // Child wants a specific size... so be it  
              resultSize = childDimension;        //size为精确值  
              resultMode = MeasureSpec.EXACTLY;   //mode为 EXACTLY 。  
          }  
          //2.2、子View的width或height为 MATCH_PARENT/FILL_PARENT  
          else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size, but our size is not fixed.  
              // Constrain child to not be bigger than us.  
              resultSize = size;                  //size为父视图大小  
              resultMode = MeasureSpec.AT_MOST;   //mode为AT_MOST  
          }  
          //2.3、子View的width或height为 WRAP_CONTENT  
          else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;                  //size为父视图大小  
              resultMode = MeasureSpec.AT_MOST;   //mode为AT_MOST  
          }  
          break;  
    
      // Parent asked to see how big we want to be  
      //3、父View是UNSPECIFIED的 ！  
      case MeasureSpec.UNSPECIFIED:  
          //3.1、子View的width或height是个精确值 (an exactly size)  
          if (childDimension >= 0) {  
              // Child wants a specific size... let him have it  
              resultSize = childDimension;        //size为精确值  
              resultMode = MeasureSpec.EXACTLY;   //mode为 EXACTLY  
          }  
          //3.2、子View的width或height为 MATCH_PARENT/FILL_PARENT  
          else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size... find out how big it should  
              // be  
              resultSize = 0;                        //size为0！ ,其值未定  
              resultMode = MeasureSpec.UNSPECIFIED;  //mode为 UNSPECIFIED  
          }   
          //3.3、子View的width或height为 WRAP_CONTENT  
          else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size.... find out how  
              // big it should be  
              resultSize = 0;                        //size为0! ，其值未定  
              resultMode = MeasureSpec.UNSPECIFIED;  //mode为 UNSPECIFIED  
          }  
          break;  
      }  
      //根据上面逻辑条件获取的mode和size构建MeasureSpec对象。  
      return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
  }    
  ~~~

  