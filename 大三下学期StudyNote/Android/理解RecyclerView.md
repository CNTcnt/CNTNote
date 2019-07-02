[TOC]

# 理解 RecyclerView

* 参考至`https://blog.csdn.net/weixin_43130724`



## 简述

* 在郭神的公众号看到这篇文章，解决了我之前对 RecyclerView 的困惑
* 比如 RecyclerView 中的 Recycler 是怎么缓存回收 View 的？我们可以如何优化等

## 了解一下 RecyclerView

* RecyclerView 有五虎上将：
* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5mstH6U0lEmicrdtgP7nNpQ1SnbhKf0gFibOD3CZIPPuKrpjibOnufnhSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##  pre-layout 与 post-layout 是什么

* 要理解这是什么，我们先知道它用在哪里，才理解它有什么作用？

* 有这样的一个场景：我们有3个item【a, b, c】，其中a与b显示在屏幕上，当我们删除b的时候，c会显示出来。
* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt5wTacxdzsnH2zaxuIbEeDW1yqaB6KsZh3I3JJibQJdmGJlf9ERu6s2oxrLEhhrZ9cN1pWibysXnPwg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 我们希望看到的是 c 从底部顺利滑动到它的新位置。但这是如何发生呢？怎么实现让 RecyclerView 自动知道我们的期望让c 从底部顺利滑动到它的新位置而不是从别的位置呢？？？

* RecyclerView 虽然知道新布局中 c 的最终位置，但它怎么知道应该从何处开始往哪个方向进行滑动呢？？？
* 谷歌的解决方案提供如下：
  * 在adapter发生更改后，RecyclerView会从LayoutManager请求两个布局。
  * 第一个 —— pre-layout，因为我们可以收到适配器的变化，所以这里我们可以做一些特殊的处理。在我们的例子中，因为我们现在知道b被删除了，通过计算我们会额外的显示出c，尽管它已经超出界限。
  * 第二个 —— post-layout，一个正常的布局，对应于更改后的适配器状态。
  * ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5s8LZUARoAY3ibL9nz1Y8dSRK72GnIx5nTU5McibyuzVBF9M1QrK5TIew/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 现在，通过比较 pre-layout 和 post-layout 中 c 的位置，RecyclerView 就知道如何可以正确地为其设置动画了。

* 再思考一个场景：如果b只是发生了变化，而不是被删除了，那么会怎么样呢？
* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5ia5lf8Ie74iagu6NlTdf2SkBeJj5HaXRwdmtMDBPODiaCgqDc9CKc775g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 答案是，仍然会在 pre-layout 阶段将 C 放置到后面！为什么呢？因为无法预测C的动画是什么动画，也许动画使b的高度变小了呢，那么c就需要显示出来，如果没有，那么 C 就会被放到缓存里去。

##   缓存介绍 

* 现在我们就要开始我们最关心也是最重要的缓存了

* 先看源码中 缓存内有什么

  ~~~java
  public final class Recycler {
      // 2个 Scrap,用于 在 Layout 期间发挥作用
      final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
      ArrayList<ViewHolder> mChangedScrap = null;
  	// CacheViews 很重要，用于直接缓存，mCachedViews 满了之后，它会将最先存入的元素移除，放入到pool中
      final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
  
      private final List<ViewHolder>
              mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);
  	//设置 CacheViews 的大小，可以看出为 2
      private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
      int mViewCacheMax = DEFAULT_CACHE_SIZE;
  	// 收回池：recyclePool
      RecycledViewPool mRecyclerPool;
  	// 用户可以自定义的一个回收扩展
      private ViewCacheExtension mViewCacheExtension;
  	
      static final int DEFAULT_CACHE_SIZE = 2;
  ~~~

* 接下来我们一个一个解析上面各个 缓存 的内容

### Scrap

* scrap 缓存是 recyclerView 最先搜索的缓存,仅仅在 layout 期间不为空(也就是说在 Layout 期间才发挥作用)。当LayoutManager 开始  layout 的时候（有可能是重新Layout的时候）（pre-layout 或 post-layout），会将所有的 viewHolder 都放到scrap中。等等需要View的时候一个个再取回来（缓存可用就用缓存的），除非有些view发生了变化（有些需要重新创建 Viewholder）。

* mAttachScrap 添加的时机，在LinearLayoutManager的onLayoutChildren方法里面：

  ~~~java
  public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
      ...
          onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
          // 这里调用了 mAttachedScrap.add(holder);
          // 这里也调用了 mChangedScrap.add(holder);
          detachAndScrapAttachedViews(recycler);
      ...
  }
  ~~~

* 这里的调用时机很值得思考，它在布局的时候就放到缓存里面了，这里说明这个缓存针对的是屏幕上显示的View.

* 那么问题就来了，屏幕上显示的为什么要缓存起来呢？我的想法倾向于减少 layout 方法调用带来的影响,复用View。

  比如说，当我们调用 notifyItemRangeChanged 方法的时候，会触发 requestLayout 方法，就会重新布局，重新布局的话，就会先将 viewHolder 放到 scrap 中（屏幕上变化的放入 mChangedScrap 中，没变化的的放入mAttachedScrap 中），然后 fill 布局的时候，再从 mAttachedScrap里面取出来直接使用。而 mChangedScrap 中的 viewHolder 会被移动到 RecycledViewPool 中，所以 mChangedScrap 对应的 item 需要从 pool 中取对应的 viewHolder，然后重新绑定。因为 mChangedScrap 表示item变化了，有可能是数据变化，有可能是类型变化，所以它的 viewHolder 无法重用，只能去RecycledViewPool中重新取对应的，然后再重新绑定。

### View 中的 detach 和 remove

* Scrap View指的是在RecyclerView中，经历了detach操作的缓存。RecyclerView源码中部分代码注释的detach其实指代的是remove，此类缓存是通过position匹配的，不需要重新bindView。

* Recycled View指代的就是真正的移除操作 remove 后的缓存，取出时需重新 bindView 使用。

### Recycled View 中的 Scrap View

* Scrap View指的是在 RecyclerView 中，经历了 detach 操作的缓存。此类缓存是通过position匹配的，不需要重新bindView。

* Recycled View指代的就是真正的移除操作remove后的缓存，取出时需重新bindView使用。

### cache 与 pool

* 这里顺便说一下，各个缓存的使用上的区别，也好对各个缓存池有一个大概的了解：
  - 如果在所有缓存中都没有找到 viewHolder，那才会调用 create 和 bind 方法。
  - 如果在 pool （RecycledViewPool ） 中找到了，那么会调用 bind 方法。
  - 如果在 cache （mCachedViews）中找到了，啥都不用做，直接显示就好了。

* 一个 viewHolder 进入到 cache 与进入到 pool 中是不一样的。进入规则不一样

#### cache

* mCachedViews，这个比较简单。
* 它是一个 ArrayList 类型，不区分 viewHolder 的类型，大小限制为2，但是你可以使用 setItemViewCacheSize()这个方法调整它的大小。
* 由于它不区分 viewHolder 的类型，所以只能根据 position 来获取 viewHolder 。

#### pool

* RecycledViewPool，它储存了各个类型的 viewHolder
* 每种类型的 Viewholder 最大数量为5，可以通过 setMaxRecycledViews() 方法来设置每个类型储存的容量。还有一个重要的点就是，可以多个列表公用一个 RecycledViewPool，使用 setRecycledViewPool() 方法。

#### 什么情况进入 cache 或 Pool 呢？怎么针对性地调优呢？

* 现在，我们来思考下一个问题：mCachedViews 的大小是有限制的，如果存不下了，怎么办？

* 实际上，mCachedViews虽然是一个 ArrayList，但是它的工作方式却和链表有点类似。**当 mCachedViews 满了之后，它会将最先存入的元素移除，放入到pool中**，如下图：

* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h536lWMkNK7mgxauIYFoxjav3ZqHSW8PFiauMK9YBxHibKIiaB70reiaL9pQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 当我们滑动列表的时候，一旦item超出了屏幕，那么就会被放入到mCachedViews 中，如果满了，就会将“尾部”的元素移动到 pool 中，如果 pool 也满了，那么就会被丢弃，等待回收。

* 下面，用几个场景来巩固一下我们学到的知识：

  * **场景一：**
  * ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5J2h3TIHgurgsIQ6bYz9fJD5OdpiaDw1oAZkg5t5zxK738VfVnPMmPBw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
  * 先看图的左边（此时假设 cache 与 pool 中没有东西），当向下滑动时，3 最先进入 mCachedViews，随后是 4 与 5，5 会将3挤出来，3就会跑到 pool 中去了。
  * 再看图的右边，继续向下滑动时，4 被 6 挤出来，放到了 pool 中，同时，8需要显示，那么就会先从 pool 中取，发现正好有一个 3，那么就会取出来复用，执行bindView后将 3 重新显示到屏幕上。

  * **场景二：**
* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5WVxqoHsoXDrZ8qS6oVdfWRRNibo8m4Xkzh7Flp9DpagoWCJneQ5GS6Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
  * 如果，向下滑倒7显示出来之后，不再继续向下，而是往上滑动，那么又会怎么样呢？看图的右边，很明显，5 从 cache 中被取出来直接复用，不用重新绑定，7 被放入了 cache 中。（**这就是 cacheView的用处**）
  * 思考一下，对于这种情况，我们应该如何加以利用呢？
    * 比如，我们有一个壁纸库的列表，用户经常会上下（左右）滑动，那么我们**增加 cache 的容量**，就会获得更好得性能。然而对于feed流之类得列表，用户很少返回，所以增加 cache 容量意义不大。
  * 再深入一下，我们继续向上滑动，那么，4 与 7 会放入到 cache 中，3 会从 pool 中取出来，但是，这里需要注意，因为 3 是从 pool 中取出来的，所以它需要重新绑定，但是从逻辑上来说，如果 3 位置的数据没有发生变化，它不需要重新绑定，也是有效的。所以，你也可以把这里当作一个优化点，在 onBindViewHolder() 方法中，检查一下。
  * 再再深入一下，在我们滑动的过程中，一个类型的 viewHolder 在 pool 中应该一直只会存在一个（除非你使用了 GridLayoutManager），所以，如果你的 pool 中存在多个 viewHolder 的话，他们在滚动过程中基本上是无用的。
  
  * **场景三：**
* ![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4iciasKiaPrFk69TMxh9DB0h5TwuRd7fF13a06PdZQ74kMH5QBqEnx9b2x7yqf6PaONeiau6MtmYaicqA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
  * 当我们调用 notifyDataSetChanged() 或者 notifyItemRangeChanged(i, c) （c这个范围非常大的时候），那么很多 viewHolder 都会最终被放入到 pool 中，因为 pool 只能放置 5 个，那么多余的就会被丢弃，等待回收。最重要的是会重新 create 与 bind 对性能影响比较大。如果你的列表能够容纳很多行，而且使用 notifyDataSetChanged 方法比较频繁，那么你应该考虑设置一下容量大小。