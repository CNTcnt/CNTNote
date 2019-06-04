[TOC]

# RecyclerView 面试



`https://juejin.im/post/5cce410551882541e40e471d`

## 要点

* 

## 面试19问

### **1. 请说一下RecyclerView？adapter的作用是什么，几个方法是做什么用的？如何理解adapter订阅者模式？**

* * 各部分各司其职
  * RecyclerView.Adapter - 处理数据集合并负责绑定视图
  * ViewHolder - 持有所有的用于绑定数据或者需要操作的View
  * LayoutManager - 负责摆放视图等相关操作
  * ItemDecoration - 负责绘制Item的装饰
  * ItemAnimator - 为Item的一般操作添加动画效果，如，增删条目等

* **adapter的作用是什么？**

  * 根据不同ViewType创建与之相应的的Item-Layout
  * 访问数据集合并将数据绑定到正确的View上

* **几个方法是做什么用的？**

  ~~~java
  public VH onCreateViewHolder(ViewGroup parent, int viewType)
  //创建Item视图，并返回相应的ViewHolder
  
  public void onBindViewHolder(VH holder, int position)
  //绑定数据到正确的Item视图上。
  
  public int getItemCount()
  //返回该Adapter所持有的Itme数量
  
  public int getItemViewType(int position)
  //用来获取当前项Item(position参数)是哪种类型的布局
  ~~~
  
* **如何理解adapter订阅者模式？**

  * 当时据集合发生改变时，我们通过调用.notifyDataSetChanged()，来刷新列表，因为这样做会触发列表的重绘。这里我们根据名字就可以猜测出这里使用到了观察者模式
  * 观察者：RecyclerView对象的 mObserver 对象
  * 被观察者：RecyclerView.Adapter类中的mObservable，可以发送更新数据的通知：全局更新、局部更新、插入更新、删除更新等，每次调用 setAdapter （）方法都会去将 RecyclerView对象的 mObserver 对象 注册为 Adapter的 mObservable的观察者；

### **2. ViewHolder的作用是什么？如何理解ViewHolder的复用？什么时候停止调用onCreateViewHolder？**

* ViewHolder的作用是什么？
  * ViewHolder内部应当存储一些子view对象，避免时间代价很大的findViewById操作
  * 持有所有的用于绑定数据或者需要操作的View
* ViewHolder的复用机制是怎么样的？
  * 模拟场景：只有一种ViewType，上下滑动的时候需要的ViewHolder种类是只有一种，但是需要的ViewHolder对象数量并不止一个。所以在后面创建了9个ViewHolder之后，需要的数量够了，无论怎么滑动，都只需要复用以前创建的对象就行了。
  * 在这个ViewHolder对象的数量“够用”之后就停止调用onCreateViewHolder方法，但是onBindViewHolder方法每次都会调用的，因为虽然 View 的结构是一样的，但是View 所绑定的内容是不同的；
  *  如果共享池有可用于正确的视图类型，则回收程序可以重用共享池中的废视图或分离视图。如果适配器没有指示给定位置上的数据已更改，则回收程序将尝试发回一个以前为该数据初始化的报废视图，而不进行重新绑定（减少一次调用 onBindViewHolder ,因为Adapter没有说数据源修改了）。

### **3. ViewHolder中为何使用SparseArray替代HashMap存储viewId？**

* 相较于HashMap,我们舍弃了HashMap.Entry和Object类型的key（HashMap.Entry 对象本身是一层额外需要被创建以及被垃圾回收的对象，而 Object 类型的 key ，则需要经过自动装箱的处理）放弃了HashCode并依赖于二分法查找。在添加和删除操作的时候有更好的性能开销。
* SparseArray比HashMap更省内存，在某些条件下性能更好，主要是因为它避免了对key的自动装箱（int转为Integer类型）,SparseArray只能存储key为int类型的数据，同时，SparseArray在存储和读取数据时候，使用的是二分查找法
* 在Android中，当涉及到快速响应的应用时，内存至关重要，因为持续地分发和释放内存会触发垃圾回收机制，这会拖慢应用运行。

### **4. LayoutManager作用是什么？LayoutManager样式有哪些？setLayoutManager源码里做了什么？**

* LayoutManager作用是什么？

  * LayoutManager的职责是摆放Item的位置，并且负责决定何时回收和重用Item。
* RecyclerView 允许自定义规则去放置子 view，这个规则的控制者就是 LayoutManager。一个 RecyclerView 如果想展示内容，就必须设置一个 LayoutManager
  
* LayoutManager样式有哪些？

  * LinearLayoutManager 水平或者垂直的Item视图。
  * GridLayoutManager 网格Item视图。
  * StaggeredGridLayoutManager 交错的网格Item视图。

* setLayoutManager(LayoutManager layout)源码

  * 当之前设置过 LayoutManager 时，移除之前的视图，并缓存视图在 Recycler 中，将新的 mLayout 对象与 RecyclerView 绑定，更新缓存 View 的数量。最后去调用 requestLayout ，重新请求 measure、layout、draw。

### **5. SnapHelper主要是做什么用的？SnapHelper是怎么实现支持RecyclerView的对齐方式？**

* RecyclerView在24.2.0版本中新增了SnapHelper这个辅助类，用于辅助RecyclerView在滚动结束时将Item对齐到某个位置。特别是列表横向滑动时，很多时候不会让列表滑到任意位置，而是会有一定的规则限制，这时候就可以通过SnapHelper来定义对齐规则了。

* **SnapHelper是怎么实现支持RecyclerView的对齐方式**

  ~~~java
  //其实，SnapHelper辅助RecyclerView实现滚动对齐就是通过给RecyclerView设置OnScrollerListenerh和OnFlingListener这两个监听器实现的。其实，就是在滚动结束之后做出动作，先找到停留在我们设置好的坐标（比如recyvleView 的中心）最近的一个 View，然后再计算出这个view 离我们设置好的坐标的距离，然后再调用 mRecyclerView.smoothScrollBy 即可，整个框架 SnapHelper 已经为我们搭建好了，我们只需要复写其中关键方法即可；
  //这部分https://www.jianshu.com/p/e54db232df62这篇文章写的很好
  
  new LinearSnapHelper().attachToRecyclerView(mRecyclerView);
  
  public void attachToRecyclerView(@Nullable RecyclerView recyclerView)
              throws IllegalStateException {
        //如果SnapHelper之前已经附着到此RecyclerView上，不用进行任何操作
          if (mRecyclerView == recyclerView) {
              return;
          }
        //如果SnapHelper之前附着的RecyclerView和现在的不一致，清理掉之前RecyclerView的回调
          if (mRecyclerView != null) {
              destroyCallbacks();
          }
        //更新RecyclerView对象引用
          mRecyclerView = recyclerView;
          if (mRecyclerView != null) {
            //设置当前RecyclerView对象的回调,这里很重要，里面调用了mRecyclerView.addOnScrollListener(mScrollListener);，这个mScrollListener 作用就是只是在正常滚动停止的时候调用了snapToTargetExistingView()方法对targetView进行滚动调整，以确保停止的位置是在对应的坐标上。这个回调就是最终要的部分
              setupCallbacks();
            //创建一个Scroller对象，用于辅助计算fling的总距离，后面会涉及到
              mGravityScroller = new Scroller(mRecyclerView.getContext(),
                      new DecelerateInterpolator());
            //调用snapToTargetExistingView()方法以实现对SnapView的对齐滚动处理
              snapToTargetExistingView();
          }
      }
  ~~~

  ~~~java
  void snapToTargetExistingView() {
          if (mRecyclerView == null) {
              return;
          }
          LayoutManager layoutManager = mRecyclerView.getLayoutManager();
          if (layoutManager == null) {
              return;
          }
        //找出SnapView
          View snapView = findSnapView(layoutManager);
          if (snapView == null) {
              return;
          }
        //计算出SnapView需要滚动的距离
          int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView);
        //如果需要滚动的距离不是为0，就调用smoothScrollBy（）使RecyclerView滚动相应的距离
          if (snapDistance[0] != 0 || snapDistance[1] != 0) {
              mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
          }
      }
  ~~~

### 6.SpanSizeLookup的作用是干什么的？SpanSizeLookup如何使用？

* SpanSizeLookup的作用是干什么的？

  * RecyclerView 可以通过 GridLayoutManager 实现网格布局， 但是很少有人知道GridLayoutManager 还可以用来设置网格中指定的Item的所占列数量，类似于合并单元格的功能，而所有的这些我们仅仅只需通过定义一个RecycleView列表就可以完成.

  * SpanSizeLookup如何使用？

    ~~~java
    GridLayoutManager manager = new GridLayoutManager(this, 6);
    manager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
        @Override
        public int getSpanSize(int position) {
            SpanModel model = mDataList.get(position);
            if (model.getType() == 1) {
                return 6;
            } else if(model.getType() == 2){
                return 3;
            }else if (model.getType() == 3){
                return 2;
            }else if (model.getType() == 4){
                return 2;
            } else {
                return 1;
            }
        }
    });
    ~~~

### 7. ItemDecoration的用途是什么？自定义ItemDecoration有哪些重写方法？分析一下addItemDecoration()源码？

* ItemDecoration的用途是什么？

  * ItemDecoration允许从adapter的数据集合中为特定的item视图添加特性的绘制以及布局间隔。它可以用来实现item之间的分割线，高亮，分组边界等。

  * 通过设置recyclerView.addItemDecoration();
  * 当然，你也可以对RecyclerView设置多个ItemDecoration，列表展示的时候会遍历所有的ItemDecoration并调用里面的绘制方法，对Item进行装饰。

* 自定义ItemDecoration有哪些重写方法？

  ~~~java
  //getItemOffests可以通过outRect.set(l,t,r,b)设置指定itemview的paddingLeft，paddingTop， paddingRight， paddingBottom
  
  //onDraw可以通过一系列c.drawXXX()方法在绘制itemView之前绘制我们需要的内容。
  
  //onDrawOver与onDraw类似，只不过是在绘制itemView之后绘制，具体表现形式，就是绘制的内容在itemview上层。
  
  ~~~

* 分析一下addItemDecoration()源码？

  ~~~java
  public void addItemDecoration(ItemDecoration decor) {
      addItemDecoration(decor, -1);
  }
  
  
  public void addItemDecoration(ItemDecoration decor, int index) {
      if (mLayout != null) {
          mLayout.assertNotInLayoutOrScroll("Cannot add item decoration during a scroll  or"
                  + " layout");
      }
      if (mItemDecorations.isEmpty()) {
          setWillNotDraw(false);
      }
      if (index < 0) {
          mItemDecorations.add(decor);
      } else {
          // 指定decor在集合中的索引
          mItemDecorations.add(index, decor);
      }
      markItemDecorInsetsDirty();
      // 重新请求 View 的测量、布局、绘制
      requestLayout();
  }
  ~~~

### 9. 上拉加载更多的功能是如何做的？添加滚动监听事件需要注意什么问题？网格布局上拉加载如何优化？

* **上拉加载更多的功能是如何做的？**

  * 01.添加recyclerView的滑动事件
    * 首先给recyclerView添加滑动监听事件。那么我们知道，上拉加载时，需要具备两个条件。第一个是监听滑动到最后一个item，第二个是滑动到最后一个并且是向上滑动。
    * 设置滑动监听器，RecyclerView自带的ScrollListener，获取最后一个完全显示的itemPosition，然后判断是否滑动到了最后一个item，

  

  * 02.上拉加载分页数据
    * 然后开始调用更新上拉加载更多数据的方法。注意这里的刷新数据，可以直接用notifyItemRangeInserted方法，不要用notifyDataSetChanged方法。

  * 03.设置上拉加载的底部footer布局
    * 在adapter中，可以上拉加载时处理footerView的逻辑
    * 在getItemViewType方法中设置最后一个Item为FooterView
    * 在onCreateViewHolder方法中根据viewType来加载不同的布局
    * 最后在onBindViewHolder方法中设置一下加载的状态显示就可以
    * 由于多了一个FooterView，所以要记得在getItemCount方法的返回值中加上1。

  * 04.显示和隐藏footer布局
    * 一般情况下，滑动底部最后一个item，然后显示footer上拉加载布局，然后让其加载500毫秒，最后加载出下一页数据后再隐藏起来。

* **网格布局上拉加载如何优化**

  * 如果是网格布局，那么上拉刷新的view则不是居中显示，到加载更多的进度条显示在了一个Item上，如果想要正常显示的话，进度条需要横跨两个Item，这该怎么办呢？

    ~~~java
    
    @Override
    public void onAttachedToRecyclerView(@NonNull RecyclerView recyclerView) {
        super.onAttachedToRecyclerView(recyclerView);
        RecyclerView.LayoutManager manager = recyclerView.getLayoutManager();
        if (manager instanceof GridLayoutManager) {
            final GridLayoutManager gridManager = ((GridLayoutManager) manager);
            gridManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
                @Override
                public int getSpanSize(int position) {
                    // 如果当前是footer的位置，那么该item占据2个单元格，正常情况下占据1个单元格
                    return getItemViewType(position) == footType ? gridManager.getSpanCount() : 1;
                }
            });
        }
    }
    ~~~

### **10. RecyclerView的Recyler是如何实现ViewHolder的缓存？如何理解recyclerView三级缓存是如何实现的？**

* 这个问题很重要

* ![](https://img-blog.csdnimg.cn/20181202211850718.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lvb25lcmxvb3A=,size_16,color_FFFFFF,t_70)

* 我们首先要知道是什么时候需要将某个viewHolder根据情况存进某个缓存的,在查找一个缓存的ViewHolder的时候,会按照mCachedViews -> ViewCacheExtension -> RecycledViewPool的顺序来查找

* 滑出屏幕表项对应的`ViewHolder`会被回收到`mCachedViews`+`mRecyclerPool`结构中，`mCachedViews`是`ArrayList`，默认存储最多2个`ViewHolder`，当它放不下的时候，按照先进先出原则将最先进入的`ViewHolder`存入回收池的方式来腾出空间。`mRecyclerPool`是`SparseArray`，它会按`viewType`分类存储`ViewHolder`，默认每种类型最多存5个。

* 1. View Cache（不清除ViewHolder的各种状态）(返回布局和内容都都有效的ViewHolder)

     ~~~java
     //View Cache也叫第一级缓存,主要指的是RecyclerView.Recycler中的mCachedViews字段,它是一个ArrayList,不区分view type,默认容量是2,但是可以通过RecyclerView的setItemViewCacheSize()方法来设置,主要用于解决RecyclerView滑动抖动时的情况，还有用于保存Prefetch的ViewHoder
     public final class Recycler {
             final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
             ArrayList<ViewHolder> mChangedScrap = null;
             final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
     }
     //当Item被移出屏幕区域时,先是缓存进了mCachedViews中,ViewHolder相关的和重要的position,flag等标志都一并被缓存了
     //(mAttachedScrap用于布局过程中屏幕可见表项的回收和复用。)如果一个Item比如已经滑动出可视范围，但还没有被移除掉,那么就会被放进mAttachedScrap中,如果调用了notifXXX()之类的方法,那么需要改变的ViewHolder就被放进mChangedScrap中
     ~~~

  2. ### ViewCacheExtension(返回View)

     ~~~java
     //开发者可自定义的一层缓存,也是第二级缓存
     ~~~

  3. ### RecycledViewPool() (返回布局有效，内容无效的ViewHolder)

     ~~~java
     //RecycledViewPool也叫第三级缓存
     //其中有一个 SparseArray<ScrapData> 类型的mScrap来缓存ViewHolder,每一个View Type 类型的Item都会有一个该缓存(源码如下),默认最大容量为5(这里为什么选用 SparseArray参考上文第三点)
     static class ScrapData {
           ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
           int mMaxScrap = DEFAULT_MAX_SCRAP; //每个View Type默认容量为5
           long mCreateRunningAverageNs = 0;
           long mBindRunningAverageNs = 0;
       }
      SparseArray<ScrapData> mScrap = new SparseArray<>();
     ~~~

     * 由于viewHolder 在存进 RecycledViewPool 时，除了其自身的View Type以外,其自身与外界的绑定关系,flag标志,与原来RecyclerView的联系等信息都被清除了,在RecycledViewPool中缓存的ViewHolder之间是依靠 View Type 来区分的,也就是说,同一个View Type之间的ViewHolder缓存在RecycledViewPool中是没有区别的;

     * 在什么情况下一个ViewHolder 会被扔进 Pool 中呢？

       * 在View Cache中的Item被更新或者被删除时(存满溢出时)（即从Cache中移出的ViewHolder会进入pool中）

       * LayoutManager在pre_layout过程中添加View,但是在post_layout过程中没有添加该View(数据集改变,如删除)

         ~~~java
         //RecyclerView的onMeasure()方法,经过了dispatchLayoutStep1()和dispatchLayoutStep2()两个主要的过程,前一个负责预布局(pre_layout),后一个负责真正的布局(post_layout)
         ~~~

### 11. 如何解决RecyclerView使用Glide加载图片导致图片错乱问题？

* RecyclerView用的是我们自定义的内部类ViewHolder来复用的，也就是复用的是ViewHoler

* 当屏幕下滑，item1滑出可视区域，将item1的ViewHolder对象给item8复用，那么此时item1中ViewHolder对象中持有的变量都是item1的。

* tem1中的ViewHolder对象，在onBindViewHolder(MyViewHolder holder, int position)方法中对holder进行更新，但是如果在这里调用glide去从url加载图片到holder中的imageView对象的话，就有可能因为网络延迟，导致图片加载不出来，那么item8就会先显示item1的图片，过一会延迟之后，显示正确的item8该显示的图片

* 解决方法一

  ~~~java
  Object tag=holder.ivImage.getTag(R.id.image_tag);
  if (tag!=null&&(int) tag!= position) {
              //如果tag不是Null,并且同时tag不等于当前的position，则我们需要处理这个图片显示
              Glide.with(mContest).clear(holder.ivImage);
   }
  String url = imagesUrl.get(0);
  Glide.with(fragment)
                  .load(url + "?imageView2/0/w/100")
                  .apply(options)
                  .into(holder.ivImage);
  //给ImageView设置唯一标记。
  holder.ivImage.setTag(R.id.image_tag, position);
  ~~~

* 解决方法二

  ~~~java
  //在view被回收的时候那么直接清理 ImageView的视图
  @Override
  public void onViewRecycled(Myholder holder)
  {
      if (holder != null)
      {
          Glide.clear(holder.mImvShow);
      }
      super.onViewRecycled(holder);
  
  }
  
  ~~~


### 12. 关于item条目点击事件在onCreateViewHolder中写和在onBindViewHolder中写有何区别？如何优化

* 由于 RecyclerView 的复用机制，在onBindViewHolder() 中频繁创建新的 onClickListener 实例没有必要，建议实际开发中应该在 onCreateViewHolder() 中每次为新建的 View 设置一次就行。



