[TOC]



# View的事件分发

## 简述

* 关于View的事件分发，我已经看了很多篇笔记，可能是由于看的别人的笔记所以总时觉得没看懂，所以打算自己看书学习一遍，写一篇自己看得懂的笔记；（书是：Android开发艺术探索（之前看郭霖的公众号中留言有人问然后大神推荐的））

## 事件分发的基本规则

* 接下来以一段伪代码来简单的说明事件分发

  ~~~java
  //通过一段伪代码很舒服的就把事件分发表示出来（任玉刚大神真的叼）
  //点击事件产生后，首先会传递给根ViewGroup，此时它的dispatchTouchEvent()就会被调用
  public boolean dispatchTouchEvent(MontionEvent ev){
    boolean consume = false;
    if(onInterceptTouchEvent(ev)){
    	//onInterceptTouchEvent返回true就表示当前ViewGroup要拦截当前事件，接着事件就会交给它处理，即它的
      //onTouchEvent()就会被调用
      consume = onTouchEvent(ev);
    }else{
      //onInterceptTouchEvent返回false就表示当前ViewGroup要不拦截当前事件，此时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理；
      consume = child.dispatchTouchEvent(ev);
    }
  }
  ~~~

  * 现在解释一下3个重要的方法

    ~~~java
    1. public boolean dispatchTouchEvent(MontionEvent ev)
      用来进行事件的分发，如果事件能够传递当前的View，那么这个方法绝壁会被调用，返回布尔值值表示是否消耗当前事件；
    2. public boolean onInterceptTouchEvent(MontionEvent ev)
      用来判断是否拦截某个事件（ViewGroup才有这个事件，因为最末端的View是不需要拦截事件的，因为它就是最末端的，就算不拦截，那事件分发给谁，你说啊）（还有，一点有点击事件传递给这个最末尾的View,那么他的onTouchEvent()方法就会被调用）；如果当前的View拦截了某个事件，那么在同一事件序列中，该方法将不会再被调用；返回布尔值表示是都拦截当前事件；
    3. public boolean onTouchEvent(MontionEvent ev)
      用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，那么在同一事件序列中，当前View无法再次接受到事件；
    ~~~

  ​

### 当View设置了OnTouch Listener

* 当一个View需要处理事件时，如果它设置了OnTouch Listener，那么OnTouch Listener中的onTouch方法就会被回调；此时事件如何处理还要看onTouch的返回值，如果返回false，那么当前View的onTouchEvent方法就会被调用；如果返回true，那么当前view的onTouchEvent不会被调用到。嗯，可以看出来给view设置的 OnTouch Listener的优先级比onTouchEvent要高；还有，在onTouchEvent中，如果当前设置的有OnClickListener,那么它的onClick方法会被调用，可以看出来，平时我们常用的OnClickListener的优先级时最低的，即处于事件传递的尾端；

## 事件传递的机制的结论

* 最重要：！！！正常情况下，一个事件序列只能被一个view拦截且消耗（因为当一个view决定拦截一个事件后，那么系统 就会把同一个事件序列内的其他方法都直接叫给它来处理，因此就不会再调用这个view的onInterceptTouchEvent去询问它是否拦截；）
* 某个view一旦开始处理事件（即已经到了onTouchEvent事件），如果它不消耗ACTION_MOWN事件（即onTouchEvent返回的是false），那么同一事件序列中的其他比如ACTION_MOVE事件,ACTION_UP事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理（即去调用父元素的onTouchEvent）。意思就是事件一旦交给一个view处理，那么他就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了；
* 某个view一旦开始处理事件，如果它不消耗除ACTION_MOWN以外的事件（即只处理了ACTION_MOWN事件），那么这个点击事件就会消失，此时父元素的onTouchEvent()并不会调用（嗯，对的），并且当前view可以持续收到后续的事件，最终这些消失的点击事件会传递给Avtivity处理；
* view的onTouchevent默认都会消耗事件(返回true)，除非它是不可点击的（clickable和longClickale同时为false），view的longClickable属性默认为false;
* viewGroup默认不拦截任何事件，返回false；
* view的enable属性并不影响onTouchEvent的默认返回值；
* onClick会发生的前提时当前的view是可点击的，并且收到了down和up的事件；
* 最后一点，事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View,通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是！！！ACTION_DOWN事件除外；
  * 这一点是由于当viewgroup决定拦截事件后，那么后续的点击事件将默认交给它处理，并且不再调用它的onInterceptTouchEvent方法；
  * 而requestDisallowInterceptTouchEvent方法可以设置FLAG_DISALLOW_INTERCEPT这个标志位，这个标志位的作用是让viewGroup不再拦截事件，当然前提是viewGroup不拦截ACTION_DOWN事件；所以，要想让子View通过requestDisallowInterceptTouchEvent方法干预父元素的事件分发过程，要让父容器不拦截ACTION_DOWN事件，然后子View通过requestDisallowInterceptTouchEvent（boolean ）方法设置FLAG_DISALLOW_INTERCEPT这个标志位来去干预父元素的事件分发过程，可以在子view控制父元素是否拦截事件;


## 当事件到达ViewGroup时事件的分发过程

1. 点击事件到达顶级View(一般是一个ViewGroup)，会调用dispatchTouchEvent（）方法；
2. 然后，如果viewGroup拦截事件onInterceptTouchEvent返回true，则事件由viewgroup处理；
   * 此时，如果viewgroup的mONTouchListener被设置的话，那么onTouch会被调用(此时onTouchEvent不会被调用)，否则onTouchEvent会被调用；
   * 当调用了onTouchEvent时，在onTouchEvent的中，如果由设置了mOnClickListener，则onClick()会被调用；
3. 回到第2点的开头，如果viewGroup拦截事件onInterceptTouchEvent返回false,那么事件会传递给它所在的点击使劲按链上的字VIew，此时子View的dispatchTouchEvent会被调用；接下来的传递过程和顶级View是一致的，如此循环以完成事件分发；


