[TOC]

# Activity和Fragment 生命周期

* 对于生命周期这一部分主要是对 Fragemnt 的生命周期不熟悉，所以做这篇笔记
* 还有对于 Activity 中包含 Fragment 的生命周期的了解

## Activity和Fragemnt生命周期

* 我们看看两者结合初始化时的生命周期

  ~~~java
  //可以看出 Activity 的生命周期都是先于 Fragment,可以看出，fragment的 onAttach,onCreate,onCreateView都是在 Activity 的 onCreate 方法里回调的，以下同理可得。
  E/MainActivity: onCreate: 1 
  E/BlankFragment: onAttach: 
  E/BlankFragment: onCreate:
  E/BlankFragment: onCreateView: 
  E/MainActivity: onCreate: 2
      
  E/MainActivity: onStart: 1 
  E/BlankFragment: onActivityCreated: //原来onActivityCreated 是在Activity 的onStart执行中才执行的,可能是碎片与活动的绑定
  E/BlankFragment: onStart: 
  E/MainActivity: onStart: 2
      
  E/MainActivity: onResume: 
  E/BlankFragment: onResume: 
  ~~~

* 现在跳转到别的 Activity 看看

  ~~~java
  E/MainActivity: onPause:1
  E/BlankFragment: onPause: 
  E/MainActivity: onPause:2
      
  E/MainActivity: onStop:1
  E/BlankFragment: onStop:
  E/MainActivity: onStop:2
  ~~~

* 现在按返回键回到 原 Activity 看看

  ~~~java
  E/MainActivity: onRestart:1
  E/MainActivity: onRestart:2
      
  E/MainActivity: onStart:1
  E/BlankFragment: onStart:
  E/MainActivity: onStart:2
      
  E/MainActivity: onResume: 
  E/BlankFragment: onResume:
  //可以看出，当回到 Activity ,里面的 Fragemnt 只回调了 onStart 和 onResume 方法,这是很重要的一点
  ~~~

* 现在按 home 键看看

  ~~~java
  E/MainActivity: onPause:1
  E/BlankFragment: onPause: 
  E/MainActivity: onPause:2
      
  E/MainActivity: onStop:1
  E/BlankFragment: onStop:
  E/MainActivity: onStop:2
  ~~~

* 正常finish销毁看看

  ~~~java
  E/MainActivity: onPause:1
  E/BlankFragment: onPause: 
  E/MainActivity: onPause:2
  
  E/MainActivity: onStop: 1
  E/BlankFragment: onStop:
  E/MainActivity: onStop: 2
  
  E/MainActivity: onDestroy:1
  E/BlankFragment: onDestroyView: 
  E/BlankFragment: onDestroy: 
  E/BlankFragment: onDetach: 
  E/MainActivity: onDestroy:2
  ~~~

## 小结，以上就是两者的生命周期，通过日志即可了解。

## 还有一个要点

* 关于碎片的一个小要点我们很容易忽略：如果我们在xml 添加 fragment 的时候，没有给该碎片指定 id 或者,该碎片没有指定id的情况下它的父容器也没有指定 id ，那么 在setContentView 就会崩溃

* 原因是：在<fragment>元素中的android:name属性指定了在布局中要实例化的Fragment。

  当系统创建这个Activity布局时，它实例化在布局中指定的每一个Fragment，并且分别调用onCreateView()，来获取每个Fragment的布局。然后系统会在Activity布局中插入通过<fragment>元素中声明直接返回的视图。这里没有问题，问题在下面

  注：由于种种原因Android要求每个 Fragment 需要一个唯一的标识（只要是标识即可），这样可以做到很多功能，比如能够在Activity被重启时系统使用这个标识来恢复Fragment（或者你能够使用这个标识获取执行事务的Fragment，如删除）。有三种给Fragment提供标识的方法：

  A. 使用android:id属性来设置唯一ID；

  B. 使用android:tag属性来设置唯一的字符串；

  C. 如果没有设置前面两个属性，系统会使用容器视图的ID（如果AB都没有做，容器视图也没有设置ID那么在 setContentView 时就会崩溃）。