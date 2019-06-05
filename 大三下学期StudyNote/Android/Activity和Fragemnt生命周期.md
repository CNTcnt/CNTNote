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
  //可以看出，当回到 Activity ,里面的 Fragemnt 只回调了 onStart 和 onResume 方法
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