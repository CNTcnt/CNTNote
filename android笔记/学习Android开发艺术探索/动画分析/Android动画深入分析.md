[TOC]

# Android动画深入分析

## 笼统介绍

* Android 的动画可以分为三种：View动画，帧动画和属性动画；

  View ；通过对场景里的对象不断做图像变换（平移,缩放,旋转,透明度）从而产生动画效果，他是一种渐进式动画；

  帧动画：通过顺序播放放一系列图像从而产生动画效果，可以理解为图片切换动画，很显然图片过大容易导致OOM

  属性动画：通过动态的改变对象的属性从而达到动画效果，属性动画为AIP11的新特性，在低版本要通过兼容库来使用它；

## View动画

* View动画，支持4种动画效果，分别是平移动画，缩放动画，旋转动画，透明度动画；（不改变View的位置即属性）

  除了这四种典型的变换效果外，帧动画也属于View动画，但是帧动画的表现形式和上面的四种变换效果不太一样；

### View动画的种类

* View动画的四种变换效果对应着 Animation 的四个子类

  * TranslateAnimation：移动动画
  * ScaleAnimation：缩放动画
  * RotateAnimation：旋转动画
  * AlphaAnimation：屏幕度动画

* 这四种动画可以通过XML来定义，也可以通过代码来动态创建；对于View动画来说，建议采用XML来定义动画，因为XML格式的动画可读性更好；

* 要使用View 动画,首先要创建动画的XML文件，这个 文件的路径为：res/anim/filename.xml，view动画有固定的语法

  ~~~java
  //随便写个例子，大概就这样
  <?xml version="1.0" encoding="utf-8"?>
  <set xmlns:android="http://schemas.android.com/apk/res/android"
      android:fillAfter="true"
      android:zAdjustment="top">
      <alpha
      android:fromAlpha="1"
      android:toAlpha="0" />
      <translate
          android:fromXDelta="300"
          android:fromYDelta="110"
          android:toXDelta="500"
          android:toYDelta="500"
          android:duration="5000"
          android:interpolator="@android:anim/linear_interpolator"/>
  </set>
  ~~~

  | xml属性                   | java方法                        | 解释                                       |
  | ----------------------- | ----------------------------- | :--------------------------------------- |
  | android:detachWallpaper | setDetachWallpaper(boolean)   | 是否在壁纸上运行                                 |
  | android:duration        | setDuration(long)             | 动画持续时间，毫秒为单位                             |
  | android:fillAfter       | setFillAfter(boolean)         | 控件动画结束时是否保持动画最后的状态                       |
  | android:fillBefore      | setFillBefore(boolean)        | 控件动画结束时是否还原到开始动画前的状态                     |
  | android:fillEnabled     | setFillEnabled(boolean)       | 与android:fillBefore效果相同                  |
  | android:interpolator    | setInterpolator(Interpolator) | 设定插值器（指定的动画效果，譬如回弹等）                     |
  | android:repeatCount     | setRepeatCount(int)           | 重复次数                                     |
  | android:repeatMode      | setRepeatMode(int)            | 重复类型有两个值，reverse表示倒序回放，restart表示从头播放     |
  | android:startOffset     | setStartOffset(long)          | 调用start函数之后等待开始运行的时间，单位为毫秒               |
  | android:zAdjustment     | setZAdjustment(int)           | 表示被设置动画的内容运行时在Z轴上的位置（top/bottom/normal），默认为normal |

* 有两个方式启用动画

  ~~~java
  	    TextView textView = (TextView) findViewById(R.id.text1);
          Animation animation = AnimationUtils.loadAnimation(this,R.anim.animation1);
          //animation.setDuration(5000);
          textView.setAnimation(animation);
  //第二种通过代码
          AlphaAnimation alphaAnimation = new AlphaAnimation(0,1);
          alphaAnimation.setDuration(5000);
          textView.setAnimation(alphaAnimation);
  ~~~

* 当你启动动画后，你可以想要在动画过程中或开始或结束执行一些逻辑配合操作，Android细心地给我们提供了接口

  ~~~java
  //例子最清楚
  alphaAnimation.setAnimationListener(new Animation.AnimationListener() {
              @Override
              public void onAnimationStart(Animation animation) {                
              }
              @Override
              public void onAnimationEnd(Animation animation) {
              }
              @Override
              public void onAnimationRepeat(Animation animation) {
              }
          });
  ~~~

* View动画也可以自定义，只需要继承Animation这个抽象类，然后重写它的 initialize 和 applyTransformation 方法，在initialize 中做一些初始化地操作，applyTransformation 中进行相应的矩阵变换即可；例子在这里就不写了（就是不想写，数学差，写矩阵变换，不可能写，这辈子都不可能写）；

### View动画的特殊使用场景

- View动画可以在一些特殊场景下使用，比如

  1. 在ViewGroup 中可以控制子元素的出场效果
  2. 在Activity中可以实现不同Activity 之间的切换效果

  ```java
  //第一种，在ViewGroup可以控制子元素的出场效果
  //第一步，先在res/anim/目录下定义一个xml文件:anim_layout.xml
  <layoutAnimation 
  	xmlns:android = "http://schemas.android.com/apk/res/android"
      android:delay = "0.5"//设置子元素开始动画的时间延迟，比如子元素入场动画的时间走起为300ms，那么0.5表示每个子元素都需要延迟150ms才能播放入场动画。总体来说，第一个子元素延迟150ms开始播放入场动画，第二个延迟300ms，以此来推；
      androiid:animationOrder = "normal"//表示子元素动画的顺序，有3种选项：normal（顺序）,reverse（逆向显示） 和 random（随机），
      android:animation = "@anim/anim_item"//为子元素指定具体的进场动画
          />
          
  //第二步当然是写下子元素的进场动画啊，这里就不写了
  //第三步就可以直接为ViewGroup指定android:layoutAnimation属性：android:layoutAnimation="@anim/anim_layout",这种方式适合于所有ViewGroup
  <ListView
  	。。。
  	android:layoutAnimation="@anim/anim_layout"/>
      
          
          
  //也可以在代码中实现
  Animation animation = AnimationUtils.loadAnimation(this,R.anim.anim_item);
  LayoutAnimationController controller = new LayoutAnimationController(animation);
  controller.setDelay(0.5f);
  controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
  viewGroup.setLayoutAnimation(controller)；
  ```

- 第二种应用：在Activity中可以实现不同Activity 之间的切换效果，这个很简单，效果也很明显；

  Activity 有默认的切换效果，但是这个效果我们也可以去自定义，主要是用到overridePendingTransition(int enterAnim,int exitAnim) 这个方法，但是有一个要求，就是必须在startActivity(intent) 或者  finish()之后被调用才能生效；

  ```java
  overridePendingTransition(int enterAnim,int exitAnim) ；
  //enterAnim是指Activity 被打开时所需的动画资源id
  //exitAnim  是指Activity 被暂停时所需的动画资源id
  使用时会出现黑背景情况，此时在style中修改即可
  <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
      <!-- 设置没有标题 -->
      <item name="android:windowNoTitle">true</item>
      <!-- 也可以在这里设置activity切换动画 -->
      <item name="android:windowAnimationStyle">@style/activityAnimation</item>
      <!-- 就是这里重点 -->
      <item name="android:windowIsTranslucent">true</item>
  </style>
  ```



## 帧动画

* 定义：显而易见，帧动画是顺序播放一组预先定义好的图片，就是电影嘛！

* Android 对于帧动画提供了 另外一个类来操作：AnimationDrawable ；使用起来很简单

  ~~~java
  //通过XML来定义一个AnimationDrawable：res/drawable/frame_animation.xml
  <?xml version="1.0" encoding="utf-8"?>
  <animation_list xmlns:android="http://schemas.android.com/apk/res/android"
      android:oneshot="false">			//帧动画的自动执行：oneshot  。 如果为true，表示动画只										播放一次停止在最后一帧上，如果设置为false表示动画循环播放。
        <item android:drawable="@drawable/image1"
              android:duration="500"/>
        <item android:drawable="@drawable/image2"
              android:duration="500"/>        
  </animation_list>
               
  ~~~

  在代码里使用

  ~~~java
  Button button = (Button) findViewById(R.id.button1);
  button.setBackGroundResource(R.drawable.frame_animation);//将帧动画设置为View的背景
  AnimationDrawable drawable = (AnimationDrawable)button.getBackground();//然后通过AnimationDrawable 来播放动画即可；
  drawable.shart();
  ~~~


* 显而易见，使用起来很简单，但是！很容易引起OOM，所以尽量不用最好咯，不然就在使用帧动画时尽量避免使用多尺寸且较大的图片；

## 属性动画

* 属性动画时API 11 新加入的特性，和 View 动画不同，它对作用对象进行了扩展，可以对任意对象的属性进行动画，动画默认时间间隔 300 ms ，默认帧率 10ms/帧。其可以达到的效果是：在一个时间间隔内完成对象从一个属性值到另一个属性值的改变，因此，属性动画天下第一！！但是。。由于他是从API 11 才有，可以用开源动画库 nineoldandroids 来兼容以前的版本；

* 比较常用的几个动画类是：ValueAnimator,ObjectAnimator： 继承于ValueAnimator 和 AnimatorSet （动画集合）；

  举几个例子简单明了

  ~~~java
  //改变myObject的translation属性，让其沿着Y轴上升一个它本身高度距离，该动画在默认时间内完成，时间和插值器和估值算法是可以自定义的；
  ObjectAnimator.ofFloat( myObject, "translationY", -myObject.getHeight() ).start();

  //改变一个对象的背景属性：
  ValueAnimator colorAnim = ObjectAnimator.ofInt(对象引用，"backgroundColor",0xFFFF8080，0xFF8080FF);
  colorAnim.setDuration(3000);
  colorAnim.setEvaluator(new ArgbEvaluator());//设置颜色专用估值器
  colorAnim.setRepeatCount(ValueAnimator.INFINITE);//重复次数
  colorAnim.setRepeatMode(ValueAnimator.REVERSE);//重复模式
  colorAnim.start();

  //动画集合，5秒内对View的旋转，平移，缩放，和透明度
  AnimatorSet set = new AnimatorSet();
  set.playTogether(
      ObjectAnimaor.ofFloat(myView,"rotationX",0,360),
  	ObjectAnimaor.ofFloat(myView,"rotationY",0,360),
  	ObjectAnimaor.ofFloat(myView,"rotation",0,-90),
  	ObjectAnimaor.ofFloat(myView,"translationX",0,90),
  	ObjectAnimaor.ofFloat(myView,"translationY",0,90),
  	ObjectAnimaor.ofFloat(myView,"scaleX",1,1.5f),
  	ObjectAnimaor.ofFloat(myView,"scaleY",1,0.5f),
  	ObjectAnimaor.ofFloat(myView,"alpha",1,0.25f,1));
  set.setDuration(5*1000).start();
  ~~~

* 属性动画还可以通过XML来实现，文件需要定义在res/animator/目录下

  ~~~xml

  ~~~

  ~~~java
  //在代码中引用时
  AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator(myContext,R.anim.property_animator);
  set.setTarget(mButton);
  set.start();
  ~~~



### 属性动画的监听器

* 属性动画提供了监听器用于监听动画的播放过程，主要有2个接口：AnimatorUpdateListener和AnimatorListener；

  ~~~java
  public static interface AnimatorListener{
       void onAnimationStart(Animator animation);//监听开始
       void onAnimationEnd(Animator animation);//结束
       void onAnimationCancel(Animator animation);//取消
       void onAnimationRepeat(Animator animation);//重复播放
  }
  //为了方便开发，系统提供了 AnimationListenerAdapter 这个类，看名字就知道时AnimatorListener 的适配器类，这样我们就可以有选择地实现上面的四个方法；
  public static interface AnimatorUpdateListener{//这个比较特殊，它会监听这个动画的过程，每播放一帧就调用一次；
      void onAnimationUpdate(ValueAnimator animation);
  }
  ~~~

  ​

* 当需要对一个属性（不是系统已经写好了动画的平移啊，旋转啊属性），如width属性进行动画操作，此时，就算你按上面的方式来执行属性动画，也没什么用，因为

* 属性动画的原理：属性动画要求动画作用的对象提供该属性的 get 和 set 方法，属性动画根据外界传递的该属性的初始值和最终值，以动画的效果多次去调用 set 方法，每次传递给 set 方法的值都不一样，确切来说时随着时间的推移，所传递的值越来越接近最终值。总结：对object的属性 abc 做动画，如果想让动画生效，要同时满足2个条件：

  1. object 必须要提供 setAbc 方法，如果动画的时候没有传递初始值，那么还要提供 getAcb 方法，因为系统要去取 abc 属性的初始值；
  2. object 的 setAbc 对属性 abc 所用的改变必须能够通过某种方法反映出来，比如会带来UI的改变之类；
  3. 例子部分在安卓开发艺术的282页；

## 理解插值器和估值器

* TimeInterpolator：时间插值器，它的作用是根据**时间流逝的百分比**来计算出当前属性值改变的百分比，系统预设的有LinearInterpolator(线性插值器：匀速动画)，AcceleratrDecelerateInterpolator(动画两头慢中间快)和DecelerateInterpoltor（减速插值器：动画越来越慢）等

* TypeEvaluator：类型估值算法，也叫估值器，它的作用是根据**当前属性改变的百分比**来计算改变后的属性值，系统预置的有 IntEvaluator(针对振兴属性)，Float Evaluator（针对浮点型属性），ArgbEvaluator(针对Color属性)

* 举个例子更容易理解

  ~~~java
  //假设采用线性插值器和整形估值算法，在40ms内，View的x属性实现从0到40的变换
  //当时间 t=20ms时，时间流逝的百分比时0.5，即时间过去了一半，那么View的x应该改变到多少呢？
  //这个就该由插值器和估值算法来一起确定；拿线性插值器来说，当时间流逝一般的时候，x的变换也应该是一半，因为她是匀速动画，看一下源码
  public class LinearInterpoltor implements Interpolator{
      public LinearInterpolator(){}
      public LinearInterpolator(Conrext context,AttributeSet attrs){}
      puvlic float getInterpolation(float input){
       return input;//返回值和输入值一样，因此插值器返回的值是0.5，这意味这View的x属性的变化百分比是0.5
      }
  }
  //从上面知道，在线性插值器的作用下，时间过去一半后View的x属性的变化百分比是0.5，那么x属性具体变成了什么呢？此时就是估值算法大显神威的时候了；
  ~~~

  ​