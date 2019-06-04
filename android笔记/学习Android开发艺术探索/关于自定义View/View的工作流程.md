[TOC]



# View的工作流程

## 简述

* view的工作流程主要是指measure.layout,draw,这3大流程，即测量，布局和绘制；其中measure是确定view的测量宽和高，layout确定view的最终宽高和4个顶点的位置，draw则将view绘制到屏幕上；

## measure过程中需要注意的地方

1. 直接继承view的自定义控件需要重写onMeasure方法并设置wrap_content时的自身大小（可以在view中指定一个默认的宽高，然后再wrap_content时设置此宽高即可），否则在布局中使用wrap_content就相当于使用了match_parent。

2. view的measure过程是三大流程中最复杂的一个，measure完成之后，通过getMeasuredWidth/Height方法就可以正确地获取到view地测量宽/高，需要注意的是，再某些极端的情况下，系统可能需要多次measure才能确定最终的测量，在这种情形下，在onMeasure方法中拿到的是测量宽/高很可能是不准确的；所以，所以！！一个好的习惯是在onLayout方法中去获取view的测量宽高；

3. 第3点，考虑一种情况，如果在Activity已经启动的时候就做一件任务，但是这个任务需要获取某个view的宽高。当我看到这个需求的时候，嗯，直接在onCreate或者onResume里面去获取这个view的宽/高不就行了？？机制如我，但是，失败了。。。咋子搞哦。实际上在onCreate，onStart,onResume中均无法正确得到某个view的宽高信息，这是因为view的measure过程和activity的生命周期方法不是同步执行的，如果view还没有测量完毕，那么获取的宽高就是0；草，那我该怎么办，感觉脑几被掏空；别急，嗯，书里叫我别急，它还给出4种解决方法给我

   1. Activity / view# onWindowFocusChanged（）

      onWindowFocusChanged()这个方法的含义是：view已经初始化完毕了，宽高已经准备好了，这个时候去获取宽高是没问题的；需要注意的是，这个方法会被调用多次哟，当Activity的窗口得到焦点和失去焦点时均会被调用一次，具体来说，当Activity继续执行和暂停执行时，onWindowFocusChanged()均会被调用，如果频繁的进行onResume和onPause，那么onWindowFocusChanged()也会被频繁的调用；所以，注意放在这个方法中的代码处理次数逻辑；

   2. view.post(runnable)

      通过post可以将一个runnable投递到消息队列的尾部，然后等待looper调用此runnable的时候，view也已经初始化好了；这里说明一下，等调用的时候并不是用开启线程的方式去调用这个 run 方法，而是把它转化成Message的Callback对象，然后像调用普通类的一个普通方法调用run方法；

      ~~~java
      protected void onStart(){
        super.onStart();
        view.post(new Runnable(){
        	int width = view.getMeasuredWidth();
          int height = view.getMeasuredHeight():
        });
      }
      ~~~

   3. ViewTreeObserver

      使用ViewTreeObserver的众多回调可以完成这个功能，比如使用OnGlobalLayoutListener这个接口，当view树内部的view的可见性发生改变时，onGlobalLayout方法将被回调，因此这是获取view的宽高一个很多的时机，需要注意的是，伴随着view树的状态改变等，onGlobaLayout会被调用多次。

      ~~~java
      protected void onStart(){
        super.onStart();
        ViewTreeObserver observer = view.getViewTreeObserver():
        observer.addOnGlobalLayoutListener(){
        	@SuppressWarnings("deprecation")
          @Override
          public void onGlobalLayout(){
        		view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
            	int width = view.getMeasuredWidth();
          	int height = view.getMeasuredHeight():
      	}
         }
      }
      ~~~

   4. view.measure(int widthMeasureSpev,int heightMeasureSpec)

      通过手动对view进行measure出具体的宽高，这里有问题，要分情况处理，根据view的layoutParams来分；

      - match_parent

        直接放弃，无法measure出具体的宽高


      - 具体的数值（dp/px）
    
        比如宽高都是100px，如下measure:
    
        ```
        int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);view.measure(widthMeasureSpec,heightMeasureSpec);
    
        ```
    
      - wrap_content
    
        如下measure
    
        ```
        int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30）-1,MeasureSpec.AT_MOST);view.measure(widthMeasureSpec,heightMeasureSpec);
    
        ```
    
        解释：(1<<30)-1：通过MeasureSpec的实现可以知道，view的尺寸使用30位二进制表示，也就是说最大是30个1（即2^30-1）,也就是(1<<30)-1，在最大化模式下，我们用view理论上能支持的最大值去构造MeasureSpec是合理的；

   ​

   ​

## Layout过程

### 简述

* layout的作用是viewGroup用来确定子元素的位置，当viewgroup的位置被确定后，它在onLayout中会遍历所有的子元素并调用其layout方法，在layout方法中onLayout方法又会被调用；
* layout方法确定view本身的位置，而onLayout方法则会确定所有子元素的位置。

### 小要点

* 在view的默认实现中，view的测量宽高和最终宽高是相等的，只不过测量宽高形成于view的measure过程，而最终宽高形成于view的layout过程；

## Draw过程

### 简述

* draw过程的作用是将view绘制到屏幕上面， view的绘制过程遵循如下几步：
  1. 绘制背景：background.draw(canvas)
  2. 绘制自己：onDraw
  3. 绘制children：dispatchDraw
  4. 绘制装饰：onDrawScrollBars

### 小要点

* view有一个特殊的方法：setWillNotDraw（boolean willNotDraw）

  ~~~java
  public void setWillNotDraw(bool willNotDraw){
     setFlags(willNotDraw ? WILL_NOT_DRAW:0, DRAW_MASK);
  }
  ~~~

  * 如果一个view不需要绘制任何内容，那么设置这个标志位为true后，系统会惊醒相应的优化。
  * 默认情况下，view没有启用这个优化标记位，但是viewgroup会默认启用这个优化标记位；
  * 对于实际开发的意义：当我们的自定义控件继承于viewGroup并且本身不具备绘制功能时，就可以开启这个标记为从而便于系统进行后续的优化（viewgroup会默认启用这个优化标记位）；当然，当明确知道一个viewGroup需要通过onDraw来绘制内容时，我们需要显示地关闭WILL_NOT_DRAW这个标志位；


