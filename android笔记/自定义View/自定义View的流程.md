## 概述

Android已经为我们提供了大量的View供我们使用，但是可能有时候这些组件不能满足我们的需求，这时候就需要自定义控件了。自定义控件对于初学者总是感觉是一种复杂的技术。因为里面涉及到的知识点会比较多。但是任何复杂的技术后面都是一点点简单知识的积累。通过对自定义控件的学习去可以更深入的掌握android的相关知识点，所以学习android自定义控件是很有必要的。记得以前学习总是想着去先理解很多知识点，然后再来学着自定义控件，但是每次写自定义控件的时候总是不知道从哪里下手啊。后来在学习的过程中发现自己跟着去写一些简单的自定义控件，然后在这个过程中遇到了没有掌握的知识点再去学习。不仅自定义控件的能力有所提高。其它的知识也有了很好的巩固和认识。所以，今天写的是怎么去自定义一个控件。而不是里面涉及到的细化知识点。一个东西我们先知道怎么用，再去问为什么。

## 自定义控件需要考虑的点

根据Android Developers官网的介绍，自定义控件你需要以下的步骤。（根据你的需要，某些步骤可以省略）

### 1、创建View

### 2、处理View的布局

### 3、绘制View

### 4、与用户进行交互

### 5、优化已定义的View

上面列出的五项就是android官方给出的自定义控件的步骤。每个步骤里面又包括了很多细小的知识点。我们可以记住这五个点，并且了解每个点里包含的小知识点。再加上一些自定义控件的练习。不断的将这些知识熟练于心，相信我们每个人都能够定义出优秀的自定义控件。接下来我们开始对上面列出的5个要点进行细化解说

## 自定义控件5个要点详细说明

### 1、创建View

#### 继承View

对Android有一些了解的朋友都知道，android为我们提供的很多View都是继承与View的。所以我们自定义的View当然也是继承于View,当然如果你要自定义的View拥有某些android已经提供的控件的功能，你可以直接继承于已经提供的控件。

我们在使用android提供的控件的时候，我们在.xml文件中编辑了一个控件，在运行的时候就能够看到和获得这个控件。我们自定义的控件当然也要支持配置和一些自定义属性，所以下面的构造方法就必须有了。这个构造方法允许我们在.xml文件中创建和编辑我们自定义控件的实例。

上面说了那么多其实就是下面一段代码。

```
class PieChart extends View {//继承View

    public PieChart(Context context, AttributeSet attrs) {
        super(context, attrs);
    }//为了支持.xml中进行创建和编辑
}123456
```

#### 定义自定义属性

大部分情况我们的自定义View需要有更多的灵活性,比如我们在xml中指定了颜色大小等属性，在程序运行时候控件就能展示出相应的颜色和大小。所以我们需要自定义属性

自定义属性通常写在在res/values/attrs.xml文件中 下面是自定义属性的标准写法

```
 <declare-styleable name="PieChart">
       <attr name="showText" format="boolean" />
       <attr name="labelPosition" format="enum">
           <enum name="left" value="0"/>
           <enum name="right" value="1"/>
       </attr>
   </declare-styleable>1234567
```

这段代码声明了两个自定义属性，showText和labelPosition，它们都是属于styleable PieChart的，为了方便，一般styleable的name和我们自定义控件的类名一样。自定义控件定义好了之后就是使用了。 
使用代码示例

```
 <?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:custom="http://schemas.android.com/apk/res/com.example.customviews">
 <com.example.customviews.charting.PieChart
     custom:showText="true"
     custom:labelPosition="left" />
</LinearLayout>1234567
```

使用自定义属性的时候需要指定命名空间，固定写法就是<http://schemas.android.com/apk/res/>你的包名。如果你是在android studio，也可以用<http://schemas.android.com/apk/res/res-auto>

#### 获取自定义属性

在xml中设置了控件自定义属性，我们就需要拿到属性做一些事情。否则定义自定义属性就没有意义了。 
固定的获取自定义属性代码如下

```
public PieChart(Context context, AttributeSet attrs) {
   super(context, attrs);
   TypedArray a = context.getTheme().obtainStyledAttributes(
        attrs,
        R.styleable.PieChart,
        0, 0);

   try {
       mShowText = a.getBoolean(R.styleable.PieChart_showText, false);
       mTextPos = a.getInteger(R.styleable.PieChart_labelPosition, 0);
   } finally {
       a.recycle();
   }
}1234567891011121314
```

当我们在 xml中创建了一个view时，所有在xml中声明的属性都会被传入到view的构造方法中的AttributeSet类型的参数当中。 
通过调用Context的obtainStyledAttributes()方法返回一个TypedArray对象。然后直接用TypedArray对象获取自定义属性的值。 
由于TypedArray对象是共享的资源，所以在获取完值之后必须要调用recycle()方法来回收。

#### 添加设置属性事件

在xml中指定的自定义属性只有在view被初始化的时候能够获取到，有时候我们可能在运行时做一些操作，这种情况就需要我们为自定义属性设置getter和setter方法,以下代码展示了自定义控件暴露的set 和get方法

```
public boolean isShowText() {
   return mShowText;
}

public void setShowText(boolean showText) {
   mShowText = showText;
   invalidate();
   requestLayout();
}123456789
```

重点看setShowText方法，在为mShowText赋值之后，调用了invalidate()和requestLayout()方法，我们自定义控件的属性发生改变之后，控件的样子也可能发生改变，在这种情况下就需要调用invalidate()方法让系统去调用view的onDraw()重新绘制。同样的，控件属性的改变可能导致控件所占的大小和形状发生改变，所以我们需要调用requestLayout()来请求测量获取一个新的布局位置。

### 2、处理View的布局.

#### 测量

一个View是在展示时总是有它的宽和高，测量View就是为了能够让自定义的控件能够根据各种不同的情况以合适的宽高去展示。提到测量就必须要提到onMeasure方法了。onMeasure方法是一个view确定它的宽高的地方。

```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    }1234
```

onMeasure方法里有两个重要的参数, widthMeasureSpec, heightMeasureSpec。在这里你只需要记住它们包含了两个信息:mode和size 
我们可以通过以下代码拿到mode和size

```
int specMode = MeasureSpec.getMode(measureSpec);
int specSize = MeasureSpec.getSize(measureSpec);
123
```

那么获取到的mode和size又代表了什么呢？ 
mode代表了我们当前控件的父控件告诉我们控件，你应该按怎样的方式来布局。 
mode有三个可选值：EXACTLY, AT_MOST, UNSPECIFIED。它们的含义是：

EXACTLY：父控件告诉我们子控件了一个确定的大小，你就按这个大小来布局。比如我们指定了确定的dp值和macth_parent的情况。 
AT_MOST：当前控件不能超过一个固定的最大值，一般是wrap_content的情况。 
UNSPECIFIED:当前控件没有限制，要多大就有多大，这种情况很少出现。

size其实就是父布局传递过来的一个大小，父布局希望当前布局的大小。

下面是一个重写onMeasure的固定伪代码写法：

```
if mode is EXACTLY{
     父布局已经告诉了我们当前布局应该是多大的宽高, 所以我们直接返回从measureSpec中获取到的size 
}else{
     计算出希望的desiredSize
     if mode is AT_MOST
          返回desireSize和specSize当中的最小值
     else:
          返回计算出的desireSize
}
12345678910
```

上面的代码虽然基本都是固定的，但是需要写的步骤还是有点多，如果你不想自己写，你也可以用android为我们提供的工具方法:resolveSizeAndState，该方法需要传入两个参数：我们测量的大小和父布局希望的大小，它会返回根据各种情况返回正确的大小。这样我们就可以不需要实现上面的模版，只需要计算出想要的大小然后调用resolveSizeAndState。之后在做自定义View的时候我会展示用这个方法来确定view的大小。

计算出height和width之后在onMeasure中别忘记调用setMeasuredDimension()方法。否则会出现运行时异常。

#### 计算一些自定义控件需要的值 onSizeChange()

onSizeChange() 方法在view第一次被指定了大小值、或者view的大小发生改变时会被调用。所以一般用来计算一些位置和与view的size有关的值。

### 3、绘制View(Draw)

一旦自定义控件被创建并且测量代码写好之后，接下来你就可以实现onDraw()来绘制View了,onDraw方法包含了一个Canvas叫做画布的参数，onDraw()简单来说就两点： 
Canvas决定要去画什么 
Paint决定怎么画

比如，Canvas提供了画线方法，Paint就来决定线的颜色。Canvas提供了画矩形，Paint又可以决定让矩形是空心还是实心。

在onDraw方法中开始绘制之前，你应该让画笔Paint对象的信息初始化完毕。这是因为View的重新绘制是比较频繁的，这就可能多次调用onDraw，所以初始化的代码不应该放在onDraw方法里。

Canvas和Paint提供的很多方法在本文中就不一一列举了。大家可以自己去查看api，之后的文章中我们也会用到，现在你只需要理解定义的大体步骤，然后再慢慢锻炼加深理解。

### 4、与用户进行交互

也许某些情况你的自定义控件不仅仅只是展示一个漂亮的内容，还需要支持用户点击，拖动等等操作，这时候我们的自定义控件就需要做用户交互这一步骤了。

在android系统中最常见的事件就是触摸事件了，它会调用view的onTouchEvent(android.view.MotionEvent).重写这个方法去处理我们的事件逻辑

```
  @Override
   public boolean onTouchEvent(MotionEvent event) {
    return super.onTouchEvent(event);
   }
12345
```

对与onTouchEvent方法相信大家都有一定了解，如果不了解的话，你就先记住这是处理Touch的地方。

现在的触控有了更多的手势，比如轻点，快速滑动等等，所以在支持特殊用户交互的时候你需要用到android提供的GestureDetector.你只需要实现GestureDetector中相对应的接口，并且处理相应的回调方法。

除了手势之外，如果有移动之类的情况我们还需要让滑动的动画显示得比较平滑。动画应该是平滑的开始和结束，而不是突然消失突然开始。在这种情况下，我们需要用到属性动画 property animation framework

由于与用户进行交互中涉及到的知识举例子会比较多，所以我在之后的自定义控件文章中再讲解。

### 5、优化你的自定义View

在上面的步骤结束之后，其实一个完善的自定义控件已经出来了。接下来你要做的只是确保自定义控件运行得流畅，官方的说法是：为了避免你的控件看得来迟缓，确保动画始终保持每秒60帧.

下面是官网给出的优化建议：

1、避免不必要的代码 
2、在onDraw()方法中不应该有会导致垃圾回收的代码。 
3、尽可能少让onDraw()方法调用，大多数onDraw()方法调用都是手动调用了invalidate()的结果，所以如果不是必须，不要调用invalidate()方法。

## 总结

到这里基本上自定义控件的大致步骤和可能涉及到的知识点都说完了。看一张图。

![这里写图片描述](http://img.blog.csdn.net/20160412183737502)

图片基本描述了自定义控件的大致流程，右边是相对应的流程所涉及到的一些知识点。可以看到自定义控件包括了很多android知识。所以学习android自定义控件是很有必要的。当你能够轻松的定义出一个完美的自定义控件的时候，相信你已经是大牛了。

本篇文章只对自定义控件的步骤进行了大体的介绍，如果你对自定义的流程还不清楚，请你记住上面所说的步骤，至少也要有个大致的印象。在之后的文章中，我在按照这个步骤去讲解一些自定义控件，用这里列出的步骤去一步步实现我们想要的自定义控件。

#### 基本图形

~~~java
protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //把整张画布绘制成白色
        canvas.drawColor(Color.WHITE);
        Paint paint = new Paint();
        //去锯齿
        paint.setAntiAlias(true);
        paint.setColor(Color.BLUE);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeWidth(3);
        //绘制圆形
        canvas.drawCircle(40, 40, 30, paint);
        //绘制正方形
        canvas.drawRect(10, 80, 70, 140, paint);
        //绘制矩形
        canvas.drawRect(10, 150, 70, 190, paint);
        RectF rel = new RectF(10,240,70,270);
        //绘制椭圆
        canvas.drawOval(rel, paint);
        //定义一个Path对象，封闭一个三角形
        Path path1 = new Path();
        path1.moveTo(10, 340);
        path1.lineTo(70, 340);
        path1.lineTo(40, 290);
        path1.close();
        //根据Path进行绘制，绘制三角形
        canvas.drawPath(path1, paint);
        //定义一个Path对象，封闭一个五角星
        Path path2 = new Path();
        path2.moveTo(27, 360);
        path2.lineTo(54, 360);
        path2.lineTo(70, 392);
        path2.lineTo(40, 420);
        path2.lineTo(10, 392);
        path2.close();
        //根据Path进行绘制，绘制五角星
        canvas.drawPath(path2, paint);
        //设置填丛风格后进行绘制
        paint.setStyle(Paint.Style.FILL);
        paint.setColor(Color.RED);
        canvas.drawCircle(120, 40, 30, paint);
        //绘制正方形
        canvas.drawRect(90, 80, 150, 140, paint);
        //绘制矩形
        canvas.drawRect(90, 150, 150, 190, paint);
        //绘制圆角矩形
        RectF re2 = new RectF(90,200,150,230);
        canvas.drawRoundRect(re2, 15, 15, paint);
        //绘制椭圆
        RectF re21 = new RectF(90, 240, 150, 270);
        canvas.drawOval(re21, paint);
        Path path3 = new Path();
        path3.moveTo(90, 340);
        path3.lineTo(150, 340);
        path3.lineTo(120, 290);
        path3.close();
        //绘制三角形
        canvas.drawPath(path3,paint);
        //绘制五角形
        Path path4 = new Path();
        path4.moveTo(106, 360);
        path4.lineTo(134, 360);
        path4.lineTo(150, 392);
        path4.lineTo(120, 420);
        path4.lineTo(90, 392);
        path4.close();
        canvas.drawPath(path4, paint);
        //设置渐变器后绘制
        //为Paint设置渐变器
        Shader mShasder = new LinearGradient(0, 0, 40, 60, new int[]{Color.RED,Color.GREEN,Color.BLUE,Color.YELLOW}, null, Shader.TileMode.REPEAT);
        paint.setShader(mShasder);
        //设置阴影
        paint.setShadowLayer(45, 10, 10, Color.GRAY);
        //绘制圆形
        canvas.drawCircle(200, 40, 30, paint);
        //绘制正方形
        canvas.drawRect(170, 80, 230, 140, paint);
        //绘制矩形
        canvas.drawRect(170, 150, 230, 190, paint);
        //绘制圆角的矩形
        RectF re31 = new RectF();
        canvas.drawRoundRect(re31, 15, 15, paint);
        //绘制椭圆
        RectF re32 =new RectF();
        canvas.drawOval(re32, paint);
        //根据Path,绘制三角形
        Path path5 = new Path();
        path5.moveTo(170, 340);
        path5.lineTo(230, 340);
        path5.lineTo(200, 290);
        path5.close();
        canvas.drawPath(path5, paint);
        //根据PAth,进行绘制五角形
        Path path6 = new Path();
        path6.moveTo(186, 360);
        path6.lineTo(214, 360);
        path6.lineTo(230, 392);
        path6.lineTo(200, 420);
        path6.lineTo(170, 392);
        path6.close();
        canvas.drawPath(path6, paint);
    }
~~~

