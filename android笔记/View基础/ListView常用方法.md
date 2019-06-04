[TOC]



# ListView常用方法解析

## getChildAt(int position)

* 获取当前点击或者选中的View(即position对应的View)，当ListView、GridView没有滑动的时候，可以正常地获取到index对应的View;但是当ListView滑动之后，却获取到null或者一个存在偏移量的View，而并不是想要获取的View。

* 原因是我们容易被这个方法名误导，ListView采用了View回收机制。简单来说就是如果屏幕最多可以显示N个ChildView，那么在内存中也就只存在这N个ChildView,当屏幕滚动的时候,第(N+1)个ChildView就会复用第一个View,以此类推。

* 所以，在ListView中，getChildAt(int position)方法的参数position指的是当前可见区域中的ChildView位置。如果要ListView的第N个ChildView,则position就是N减去第一个可见的ChildView的位置，代码实现如下：

  ~~~java
  View view = listView.getChildAt(i - listView.getFirstVisiblePosition());
  ~~~

## pointToPosition(int x,int y)

* 可以用屏幕上的坐标来获取在这个坐标上的ListView的子项的位置

  ~~~java
  int item=mListView.pointToPosition((int) event.getX(), (int) event.getY());
  ~~~

