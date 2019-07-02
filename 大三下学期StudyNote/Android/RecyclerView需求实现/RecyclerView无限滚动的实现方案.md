[TOC]

# RecyclerView无限滚动的实现方案

* 参考至`https://juejin.im/post/5cfa198ff265da1b8c197c2f`

## 背景

* 实现横向的无限滚动的轮播图

## 方案思想选择

### 1.用几乎无限代替无限

* 方案实现：
  * 我们让 Adapter 的 getItemCount() 方法返回 Integer.MAX_VALUE，使得position数据达到很大很大；这里就是让RecyclerView 拥有近乎无限个数的 子项View。
  * 在 onBindViewHolder() 方法里对position参数取余运算，拿到position对应的真实数据索引，然后对itemView绑定数据；
  * 在初始化RecyclerView的时候，让其滑动到指定位置，如 Integer.MAX_VALUE/2，这样就不会滑动到边界了，如果用户一根筋，真的滑动到了边界位置，再加一个判断，如果当前索引是0，就重新动态调整到初始位置
    这个方案是挺简单，但并不完美。一是对我们的数据和索引做了计算操作，二是如果滑动到边界，再动态调整到中间，会有一个不明显的卡顿操作，使得滑动不是很顺畅。

### 2.真正无限实现

* 方案实现：我们都知道，RecyclerView的数据绑定是通过Adapter来处理的，而排版方式以及View的回收控制等，则是通过LayoutManager来实现的，因此我们直接修改 itemView 的排版方式就可以实现我们的目标，即滑动到我们可见的最后一个子项的边界时，就拿一个新 ItemView （可以是回收来的也可以是新创建的）接上以实现无限循环。即实现：自定义LayoutManager，修改RecyclerView的布局方式。

* **1.创建自定义LayoutManager**
  首先，自定义 LooperLayoutManager 继承自 RecyclerView.LayoutManager，然后需要实现抽象方法 generateDefaultLayoutParams()，这个方法的作用是给 itemView 设置默认的LayoutParams，直接返回如下就行。

  ```java
  public class LooperLayoutManager extends RecyclerView.LayoutManager {
      @Override
      public RecyclerView.LayoutParams generateDefaultLayoutParams() {
          return new RecyclerView.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT,
                  ViewGroup.LayoutParams.WRAP_CONTENT);
      }
  }
  ```

  **2.打开滚动开关**
  接着，对滚动方向做处理，重写canScrollHorizontally()方法，打开横向滚动开关。注意我们是实现横向无限循环滚动，所以实现此方法，如果要对垂直滚动做处理，则要实现canScrollVertically()方法。

  ```java
  @Override
  public boolean canScrollHorizontally() {
      return true;
  }
  ```

  **3.对RecyclerView进行初始化布局**

  * 好了，以上两部是基础工作，接下来，重写 onLayoutChildren() 方法，开始对itemView初始化布局。

  ```java
  //对所有的 itemView 进行布局，一般会在初始化和调用 Adapter 的 notifyDataSetChanged() 方法时调用。
  @Override
  public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
          if (getItemCount() <= 0) {
              return;
          }
          //标注1.如果当前时准备状态，直接返回
          if (state.isPreLayout()) {
              return;
          }
          //标注2.将视图分离放入scrap缓存中，以准备重新对view进行排版,从View树上detach的View会放入scrap缓存里，调用removeView()删除的View会放入recycler缓存中。
          detachAndScrapAttachedViews(recycler);
  
          int autualWidth = 0;
          for (int i = 0; i < getItemCount(); i++) {
              //标注3.初始化，将在屏幕内的view填充,这个方法内部会先从 scrap 缓存中取 itemView，如果没有则从 recycler 缓存中取，如果还没有则调用 adapter 的 onCreateViewHolder() 去创建 itemView。
              View itemView = recycler.getViewForPosition(i);
              //把View加入RV中
              addView(itemView);
              //标注4.测量itemView的宽高
              measureChildWithMargins(itemView, 0, 0);
              int width = getDecoratedMeasuredWidth(itemView);
              int height = getDecoratedMeasuredHeight(itemView);
              //标注5.根据itemView的宽高进行布局
              layoutDecorated(itemView, autualWidth, 0, autualWidth + width, height);
  
              autualWidth += width;
              //标注6.如果当前布局过的itemView的宽度总和大于RecyclerView的宽，则不再进行布局,这一步很重要，只布局占满一个RecyclerView的宽度的子项，后面滑动超过边界就新加载
              if (autualWidth > getWidth()) {
                  break;
              }
          }
  }
```
  
**4.对RecyclerView进行滚动和回收itemView处理**
  
  * 对RecyclerView的子item进行排版布局后，运行一下效果就会出现了，此时我们的列表还不支持移动，因为需要复写 scrollHorizontallyBy() 方法，在其内部拿到需要移动的距离，然后使 RV 的全部子项移动相应的距离即可，即调用 offsetChildrenHorizontal ，横向地移动全部子项 offset 距离。
* 前面说过,我们打开了横向滚动的开关，所以对应的，最重要的是我们需要重写 scrollHorizontallyBy()方法进行横向滑动操作。
  
  ```java
  @Override
  public int scrollHorizontallyBy(int dx, RecyclerView.Recycler recycler, RecyclerView.State state) {
          //标注1.横向滑动的时候，对左右两边按顺序填充itemView，itemView 可以从缓存中取也可以新New,即调用 recycler.getViewForPosition（int position）；
          int travl = fill(dx, recycler, state);
          if (travl == 0) {
              return 0;
          }
  
          //2.滑动
          offsetChildrenHorizontal(-travl);
  
          //3.回收已经不可见的itemView
          recyclerHideView(dx, recycler, state);
          return travl;
  }
```
  
可以看到，滑动逻辑很简单，总结为三步：
  
  - 横向滑动的时候，对左右两边按顺序填充itemView
  - 滑动itemView
- 回收已经不可见的itemView
  
下面一步一步介绍：
  
* 首先第一步，滑动的时候调用自定义的 fill() 方法，对左右两边进行填充。还没忘了，我们是来实现循环滑动的，所以这一步尤其重要，先看代码：
  
  ```java
  /**
  * 左右滑动的时候，填充
  */
  private int fill(int dx, RecyclerView.Recycler recycler, RecyclerView.State state) {
          if (dx > 0) {//标注1.向左滚动         
              View lastView = getChildAt(getChildCount() - 1);
              if (lastView == null) {
                  return 0;
              }
              int lastPos = getPosition(lastView);
              //标注2.可见的最后一个itemView完全滑进来了，需要补充新的
              if (lastView.getRight() < getWidth()) {
                  View scrap = null;
                  //标注3.判断可见的最后一个itemView的索引，如果是数据列表中的最后一个，则将下一个itemView设置为第一个，否则设置为当前索引的下一个
                  if (lastPos == getItemCount() - 1) {
                      if (looperEnable) {
                          scrap = recycler.getViewForPosition(0);
                      } else {
                          dx = 0;
                      }
                  } else {
                      scrap = recycler.getViewForPosition(lastPos + 1);
                  }
                  if (scrap == null) {
                      return dx;
                  }
                 //标注4.将新的itemViewadd进来并对其测量和布局，测量到 width和 height 后就可以布局了
                  addView(scrap);
                  measureChildWithMargins(scrap, 0, 0);
                  int width = getDecoratedMeasuredWidth(scrap);
                  int height = getDecoratedMeasuredHeight(scrap);
                  layoutDecorated(scrap,lastView.getRight(), 0,
                          lastView.getRight() + width, height);
                  return dx;
              }
          } else {
              //向右滚动
              View firstView = getChildAt(0);
              if (firstView == null) {
                  return 0;
              }
              int firstPos = getPosition(firstView);
  
              if (firstView.getLeft() >= 0) {
                  View scrap = null;
                  if (firstPos == 0) {
                      if (looperEnable) {
                          scrap = recycler.getViewForPosition(getItemCount() - 1);
                      } else {
                          dx = 0;
                      }
                  } else {
                      scrap = recycler.getViewForPosition(firstPos - 1);
                  }
                  if (scrap == null) {
                      return 0;
                  }
                  addView(scrap, 0);
                  measureChildWithMargins(scrap,0,0);
                  int width = getDecoratedMeasuredWidth(scrap);
                  int height = getDecoratedMeasuredHeight(scrap);
                  layoutDecorated(scrap, firstView.getLeft() - width, 0,
                          firstView.getLeft(), height);
              }
          }
          return dx;
  }
  ```

  * 代码是有点长，不过逻辑很清晰。首先分为两部分，往左填充或是往右填充，dx为将要滑动的距离，如果 dx > 0，则是往左边滑动，则需要判断右边的边界，如果最后一个itemView完全显示出来后，在右边填充一个新的itemView。
  * 第二步：填充完新的itemView后，就开始进行滑动了，这里直接调用 LayoutManager 的 offsetChildrenHorizontal() 方法滑动-travl 距离，travl 是通过fill方法计算出来的，通常情况下都为 dx，只有当滑动到最后一个itemView，并且循环滚动开关没有打开的时候才为0，也就是不滚动了。

  ```java
  //2.滚动具体实现
  offsetChildrenHorizontal(travl * -1);
  
  //3.回收已经不可见的itemView,才能做到回收利用，防止内存爆增。
  recyclerHideView(dx, recycler, state);
  
  /**
  * 回收界面不可见的view
  */
  private void recyclerHideView(int dx, RecyclerView.Recycler recycler, RecyclerView.State state) {
      //循环遍历来回收不可见的View
          for (int i = 0; i < getChildCount(); i++) {
              View view = getChildAt(i);
              if (view == null) {
                  continue;
              }
              if (dx > 0) {
                  //标注1.向左滚动，移除左边不在内容里的view
                  if (view.getRight() < 0) {
                      removeAndRecycleView(view, recycler);
                      Log.d(TAG, "循环: 移除 一个view  childCount=" + getChildCount());
                  }
              } else {
                  //标注2.向右滚动，移除右边不在内容里的view
                  if (view.getLeft() > getWidth()) {
                      removeAndRecycleView(view, recycler);
                      Log.d(TAG, "循环: 移除 一个view  childCount=" + getChildCount());
                  }
              }
          }
  }
  ```

  * 代码也很简单，遍历所有添加进 RecyclerView 里的item，然后根据 itemView 的顶点位置进行判断，移除不可见的item。移除 itemView 调用 removeAndRecycleView(view, recycler) 方法，会对移除的item进行回收，然后存入 RecyclerView 的缓存里。
    至此，一个可以实现左右无限循环的LayoutManager就实现了，调用方式跟通常我们用RrcyclerView没有任何区别，只需要给 RecyclerView 设置 LayoutManager 时指定我们的LayoutManager，如下：

    ```java
    recyclerView.setAdapter(new MyAdapter());
    LooperLayoutManager layoutManager = new LooperLayoutManager();
    layoutManager.setLooperEnable(true);
    recyclerView.setLayoutManager(layoutManager);
    ```

    源码地址：
    https://github.com/Xiasm/LooperLayoutManager