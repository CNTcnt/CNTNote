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

`关于MeasureSpec的解释参考至<https://www.jianshu.com/p/5a71014e7b1b>`

* MeasureSpec 是两个单词组成，翻译过来“测量规格”或者“测量参数”，官方文档对他的说明是“一个MeasureSpec封装了从父容器传递给子容器的布局要求”，更精确的说法应该这个 MeasureSpec 是由父View的 MeasureSpec 和子View的 LayoutParams 通过简单的计算得出一个针对子View的测量要求，这个测量要求就是 MeasureSpec 。

* MeasureSpec 是一个大小跟模式的组合值, MeasureSpec 中的值是一个整型（32位）将 size 和 mode 打包成一个 Int型，其中高两位是 mode，后面30位存的是 size，是为了减少对象的分配开支。

* **UPSPECIFIED** : 父容器对于子容器没有任何限制,子容器想要多大就多大
  **EXACTLY**: 父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大的空间。
  **AT_MOST**：子容器可以是声明大小内的任意大小

* 现在我们看看是怎么获得这个 MeasureSpec 的
  
  ~~~java
  public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();
  mWinFrame = new Rect();
  private void performTraversals() {
      Rect frame = mWinFrame;
      if (mWidth != frame.width() || mHeight != frame.height()) {
                  mWidth = frame.width();//到这里时，frame已经被赋值了，宽度为 Window 宽度
                  mHeight = frame.height();
      }
      
      ......
      WindowManager.LayoutParams lp = mWindowAttributes;
      ......
          //可以看到，这里就是获取 根View 的 MeasureSpec ，这个参数是怎么获取到的呢
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
  ~~~
  
* getRootMeasureSpec(mWidth, lp.width);

  ~~~java
  private static int getRootMeasureSpec(int windowSize, int rootDimension) {
          int measureSpec;
          switch (rootDimension) {
  
          case ViewGroup.LayoutParams.MATCH_PARENT:
              // Window can't resize. Force root view to be windowSize.
              //可以看到我们上面传的是 LayoutParams.MATCH_PARENT ，就是进入这里，测量规格设置为 MeasureSpec.EXACTLY
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
  
  //得到 根View 的 MeasureSpec 后就需要进入真正的测量
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
  
  //如果 子View不是ViewGroup ，那么就没有必要测量它的子View（因为没有），如果不复写就是以下实现方式 
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      //就是直接调用 setMeasuredDimension 设置 View 的大小了，所以我们自定义View的时候，可以复写onMeasure 方法，在内部直接调用 setMeasuredDimension 设置我们指定的大小即可
          setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                  getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
  }
  public static int getDefaultSize(int size, int measureSpec) {
          int result = size;
          int specMode = MeasureSpec.getMode(measureSpec);
          int specSize = MeasureSpec.getSize(measureSpec);
  
          switch (specMode) {
          case MeasureSpec.UNSPECIFIED:
              result = size;
              break;
          //从下面可以看出，如果不复写 View 的 onMeasure方法,让它默认调用 getDefaultSize 方法来获取默认大小，那么当父View为Match_parent，子View为 wrap_content 时，结果大小都是 match_parent，即这里只响应子View设置成 Match_parent 或者 具体某个值的情况，当设置成 wrap_content 时没有用，如果我们需要处理 当子View 为 wrap_content 的情况，那么很简单，我们需要复写 onMeasure 方法了
          case MeasureSpec.AT_MOST:
          case MeasureSpec.EXACTLY:
              result = specSize;
              break;
          }
          return result;
  }
  //现在我们去看看得到测量结果（即得到此时View的具体长宽后怎么处理）
  protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
          boolean optical = isLayoutModeOptical(this);
          if (optical != isLayoutModeOptical(mParent)) {
              Insets insets = getOpticalInsets();
              int opticalWidth  = insets.left + insets.right;
              int opticalHeight = insets.top  + insets.bottom;
  
              measuredWidth  += optical ? opticalWidth  : -opticalWidth;
              measuredHeight += optical ? opticalHeight : -opticalHeight;
          }
          setMeasuredDimensionRaw(measuredWidth, measuredHeight);
  }
  private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
      //其实就是把 measure 测量结果 简单地保存在 View 的属性中，等 layout 过程中取用吧
          mMeasuredWidth = measuredWidth;
          mMeasuredHeight = measuredHeight;
  
          mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
  }
  ~~~

* 如果是 ViewGroup，父 View 的 measure 的过程会先测量子 View，等子 View 测量结果出来后，再来测量自己(因为有时候父 View 的大小不是固定的（比如 wrap_content），而是根据 子View 的大小来最终确定的)，我们可以查看 LinearLayout的源码看一下

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
      //ViewGroup是测量好子View后才可以知道自身大小的
      setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                  heightSizeAndState);
      ......
  }
  
  void measureChildBeforeLayout(View child, int childIndex,
              int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
              int totalHeight) {
      //这里就是去递归测量 子View 了
          measureChildWithMargins(child, widthMeasureSpec, totalWidth,
                  heightMeasureSpec, totalHeight);
  } 
  ~~~
  
* measureChildWithMargins

  ~~~java
  //measureChildWithMargins 就是用来测量某个子 View 的，定义在 ViewGroup 中，我们来分析是怎样测量的，具体看注释：
  protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) { 
  
  	// 子View的LayoutParams，你在xml的layout_width和layout_height 和一切 layout_***的值最后都会封装到这个个LayoutParams。先拿到 子View 的 layoutParams 
  	final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();   
  
  	//根据父View的测量规格和父View自己的Padding，还有子View的Margin和已经用掉的空间大小（widthUsed），再结合 子View的 lp.width 就能算出子View的 MeasureSpec,具体计算过程看getChildMeasureSpec方法。
  	final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,       
  	mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);    
  
  	final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,     
  	mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin  + heightUsed, lp.height);  
  
  	//通过父View的MeasureSpec和子View的自己LayoutParams的计算，算出子View的MeasureSpec
  	// 然后让子View用这个MeasureSpec（一个测量要求，比如不能超过多大）去测量自己，如果子View是ViewGroup 那还会递归往下测量。
  	//就是说 测量前就要先得到 ＭｅａｓｕｒｅＳｐｅｃ 才可以测量，而这个  ＭｅａｓｕｒｅＳｐｅｃ 就需要父View和子View来共同得到，而怎么得到这个 MeasureSppec 就看下面   
  	child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
  }
  
  
  // spec参数   表示父View的MeasureSpec 
  // padding参数   这个参数不只代表 padding ,代表的是 父View的Padding+子View的Margin，父View的大小减去这些边距，才能精确算出子 View的MeasureSpec 的size,这里需要知道，如果我们的 最底层子View（不是ViewGroup） 没有在 onDraw 方法里处理它本身的 padding ，那么我们设置的 padding 无效化,这里可能会有疑问，为什么不是在 onMeasure 方法里，因为如果在 onMeasure 处理就会影响控件的大小，而padding表示的是控件内容离控件边缘的距离（不影响控件大小），所以在onDraw处理，那又有一个疑问，为什么 ViewGroup 可以在 onMeasure 中处理，因为 ViewGroup 只代表一个容器，它的padding会影响到它的子View的大小（即测量过程），所以需要在onMeasure方法里处理
  // childDimension参数  表示该子View内部LayoutParams属性的值（lp.width或者lp.height）
  //                    可以是wrap_content、match_parent、一个精确指(an exactly size),
  //举例：根据父View的 parentWidthMeasureSpec 和 子View 的 lp.width 就可以得出此时 子View 的MeasureSpec 
  public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  
      int specMode = MeasureSpec.getMode(spec);  //获得父View的mode  
      int specSize = MeasureSpec.getSize(spec);  //获得父View的大小  
  
     //父View的大小-自己的Padding+子View的Margin，得到值才是子View的大小
      int size = Math.max(0, specSize - padding);   
    
      int resultSize = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
      int resultMode = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
    
      switch (specMode) {  //在 父View 的限制之下 子View 的测量
      // Parent has imposed an exact size on us，父View 的 规格 基础下来 测量子View
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
          //1.3、子View的width或height为 WRAP_CONTENT 时，那么此时最大Size就是父View 提供的size
          else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;                   //size为父视图大小，子View不能超过这个size  
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
              // ConstraViewGroupin child to not be bigger than us.  
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
      //根据上面逻辑条件获取的mode和size构建MeasureSpec对象。此时 子VIew 的MeasureSpec已经得出  
      return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
  }   
  ~~~

### 小结

* 到这里 onMeasure 就到达一段落，其实就是需要知道 MeasureSpec 的使用。其实就是先获取 MeasureSpec 再进行onMeasure 测量工作，而 MeasureSpec 就是 子View 在自身的限制下和在 父View 的限制下然后定下来的 子View 的 测量规格。

## layout 过程

* 在measure测量过程结束之后，就要到layout 布局过程，就是确定控件的摆放位置，

  ~~~java
  private void performTraversals() {
      ......
      if (didLayout) {
              performLayout(lp, mWidth, mHeight);
      ......
    	}
  }
  ~~~

* performLayout(lp, mWidth, mHeight);

  ~~~java
  private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
              int desiredWindowHeight) {
          mLayoutRequested = false;
          mScrollMayChange = true;
          mInLayout = true;
  
          final View host = mView;
      	......
          try {
              host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
              ......
          }
  }
  ~~~

* host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());//host肯定是个ViewGroup，不会是View,我们直接看下ViewGroup 的 layout 函数

  ~~~java
  @Override
  public final void layout(int l, int t, int r, int b) {
          if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
              if (mTransition != null) {
                  mTransition.layoutChange(this);
              }
              //调用父类即View的Layout方法
              super.layout(l, t, r, b);
          } else {
              // record the fact that we noop'd it; request layout when transition finishes
              mLayoutCalledWhileSuppressed = true;
          }
      //LayoutTransition是用于处理ViewGroup增加和删除子视图的动画效果，也就是说如果当前ViewGroup未添加LayoutTransition动画，或者LayoutTransition动画此刻并未运行，那么调用super.layout(l, t, r, b)，继而调用到ViewGroup中的onLayout，否则将mLayoutSuppressed设置为true，等待动画完成时再调用requestLayout()。
  }
  ~~~

* super.layout(l, t, r, b);

  ~~~java
  public final void layout(int l, int t, int r, int b) {
         .....
        //这里就来设置 子View 位于父视图的位置,这里的具体位置都是相对与父视图的位置
         boolean changed = setFrame(l, t, r, b); 
         //判断View的位置是否发生过变化或者有没有指定要求布局，看有必要进行重新layout吗
         if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {
             if (ViewDebug.TRACE_HIERARCHY) {
                 ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);
             }
             //调用onLayout(changed, l, t, r, b); 函数
             onLayout(changed, l, t, r, b);
             mPrivateFlags &= ~LAYOUT_REQUIRED;
         }
         mPrivateFlags &= ~FORCE_LAYOUT;
         .....
     }
  }
  //最底层 View 的 onLayout 是个空实现，我们自定义View（不是ViewGroup）的时候极大部分情况下不需要复写它
   protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
  }
  
  ~~~

  * 自定义 ViewGroup  自然需要复写它，因为 ViewGroup 需要管理它的 子View 的布局位置，现在就到 LinearLayout 中看看 onLayout 方法

  ~~~java
  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
          if (mOrientation == VERTICAL) {
              layoutVertical(l, t, r, b);
          } else {
              layoutHorizontal(l, t, r, b);
          }
  }
  ~~~

* layoutVertical(l, t, r, b);

  ~~~java
  //其实很简单，线性布局就是把子View线性地排列即可
  void layoutVertical(int left, int top, int right, int bottom) {
          final int paddingLeft = mPaddingLeft;
  
          int childTop;
          int childLeft;
  
          // Where right end of child should go
          final int width = right - left;
          int childRight = width - mPaddingRight;
  
          // Space available for child
          int childSpace = width - paddingLeft - mPaddingRight;
  
          final int count = getVirtualChildCount();
  
          final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
          final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
  
          switch (majorGravity) {
             case Gravity.BOTTOM:
                 // mTotalLength contains the padding already
                 childTop = mPaddingTop + bottom - top - mTotalLength;
                 break;
  
                 // mTotalLength contains the padding already
             case Gravity.CENTER_VERTICAL:
                 childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
                 break;
  
             case Gravity.TOP:
             default:
                 childTop = mPaddingTop;
                 break;
          }
  
      //一个for循环，按线性顺序来一个一个来确定子View 的布局位置
          for (int i = 0; i < count; i++) {
              final View child = getVirtualChildAt(i);
              if (child == null) {
                  childTop += measureNullChild(i);
              } else if (child.getVisibility() != GONE) {
                  //这里就取出View在测量过程中测量好的 MeasureSpec 测量规程参数
                  final int childWidth = child.getMeasuredWidth();
                  final int childHeight = child.getMeasuredHeight();
  
                  final LinearLayout.LayoutParams lp =
                          (LinearLayout.LayoutParams) child.getLayoutParams();
  
                  int gravity = lp.gravity;
                  if (gravity < 0) {
                      gravity = minorGravity;
                  }
                  final int layoutDirection = getLayoutDirection();
                  final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                  switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                      case Gravity.CENTER_HORIZONTAL:
                          childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                  + lp.leftMargin - lp.rightMargin;
                          break;
  
                      case Gravity.RIGHT:
                          childLeft = childRight - childWidth - lp.rightMargin;
                          break;
  
                      case Gravity.LEFT:
                      default:
                          childLeft = paddingLeft + lp.leftMargin;
                          break;
                  }
  
                  if (hasDividerBeforeChildAt(i)) {
                      childTop += mDividerHeight;
                  }
  
                  childTop += lp.topMargin;
                  //前面根据View本身的测量好的大小还有设置好的属性来处理好 子View 的布局位置后就可以让子View 去布局了，这里我们可以看到，我们这里可以随便指定4个布局位置给他，那么我们这里随便指定有什么影响呢？对于我们的View的呈现一点问题都没有，但是对于我们使用一些方法时如果我们不清楚会导致我们出现错误，比如看下面
                  setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                          childWidth, childHeight);
                  childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
  
                  i += getChildrenSkipCount(child, i);
              }
          }
  }
  
  private void setChildFrame(View child, int left, int top, int width, int height) {
      //递归调用子View 的 Layout方法
          child.layout(left, top, left + width, top + height);
  }
  ~~~

* 这里注意

  ~~~java
  //如果我们没有随便指定布局位置，那么下面这2个方法返回的结果是一样的，不然就要好好考虑使用哪个方法
  public final int getMeasuredWidth() {  
          return mMeasuredWidth & MEASURED_SIZE_MASK;  
  }  
  public final int getWidth() {  
          return mRight - mLeft;  
  }
  ~~~

## Draw过程

* ```java
  private void performTraversals() {
      ......
      performDraw();
      ......
  }
  private void performDraw() {
   	try {
          ......
              boolean canUseAsync = draw(fullRedrawNeeded);
          ......
      }
  }
  private boolean draw(boolean fullRedrawNeeded) {
      ......
      if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                          scalingRequired, dirty, surfaceInsets)) {
                      return false;
      }
      ......
  }
  private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
      try {
                  canvas.translate(-xoff, -yoff);
                  if (mTranslator != null) {
                      mTranslator.translateCanvas(canvas);
                  }
                  canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                  attachInfo.mSetIgnoreDirtyState = false;
  
                  mView.draw(canvas);
  
                  drawAccessibilityFocusedDrawableIfNeeded(canvas);
      }
  }
  ```

* draw方法

  ~~~java
  public void draw(Canvas canvas) {
      ...
          /*
           * Draw traversal performs several drawing steps which must be executed
           * in the appropriate order:
           *
           *      1. Draw the background
           *      2. If necessary, save the canvas' layers to prepare for fading
           *      3. Draw view's content
           *      4. Draw children
           *      5. If necessary, draw the fading edges and restore layers
           *      6. Draw decorations (scrollbars for instance)
           */
  
          // Step 1, draw the background, if needed
      ...
          background.draw(canvas);
      ...
          // skip step 2 & 5 if possible (common case)
      ...
          // Step 2, save the canvas' layers
      ...
          if (solidColor == 0) {
              final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;
  
              if (drawTop) {
                  canvas.saveLayer(left, top, right, top + length, null, flags);
              }
      ...
          // Step 3, draw the content,绘制内容就是调用 onDraw 方法，我们自定义View的时候一般会复写这个方法，在这个方法里绘制内容，onDraw(canvas) 方法是view用来draw 自己的，具体如何绘制，颜色线条什么样式就需要子View自己去实现，View.java 的onDraw(canvas) 是空实现，ViewGroup 也没有实现，每个View的内容是各不相同的，所以需要由子类去实现具体逻辑。
          if (!dirtyOpaque) onDraw(canvas);
  
          // Step 4, draw the children//这里就是去绘制子View了，其实就是去递归调用子View的Draw,这里就不写下去了
          dispatchDraw(canvas);
  
          // Step 5, draw the fade effect and restore layers
  
          if (drawTop) {
              matrix.setScale(1, fadeHeight * topFadeStrength);
              matrix.postTranslate(left, top);
              fade.setLocalMatrix(matrix);
              canvas.drawRect(left, top, right, top + length, p);
          }
      ...
          // Step 6, draw decorations (scrollbars)
          onDrawScrollBars(canvas);
  }
  ~~~

### 小结

* View 的绘制流程到这里就结束了。View 的绘制其实是根据一个固定的模式去绘制的，就是测量完后布局，布局后绘制的过程。