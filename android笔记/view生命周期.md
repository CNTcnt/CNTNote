任何一个视图都是要经过非常科学的绘制流程后才能显示出来的，每一个视图的绘制过程其实就是一个完整的生命周期，我们从这里开始入手，一起学习自定义View。

**一.准备工作**

布局文件：

```
    <org.daliang.xiaohehe.androidartstudy.MyView
        android:id="@+id/my_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    </org.daliang.xiaohehe.androidartstudy.MyView>12345
```

Activity代码：

```
public class FiveActivity extends AppCompatActivity {

    private MyView myView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.e("log", "Activity生命周期:onCreate");
        setContentView(R.layout.activity_five);
        initView();
    }

    private void initView() {
        myView = (MyView) findViewById(R.id.my_view);
    }

    @Override
    protected void onStart() {
        super.onStart();
        Log.e("log", "Activity生命周期:onStart");
    }


    @Override
    protected void onResume() {
        super.onResume();
        Log.e("log", "Activity生命周期:onResume");
    }

    @Override
    protected void onRestart() {
        super.onRestart();
        Log.e("log", "Activity生命周期:onRestart");
    }


    @Override
    protected void onPause() {
        super.onPause();
        Log.e("log", "Activity生命周期:onPause");
    }

    @Override
    protected void onStop() {
        super.onStop();
        Log.e("log", "Activity生命周期:onStop");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.e("log", "Activity生命周期:onDestroy");
    }
}123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354
```

自定义View代码：

```
public class MyView extends View {
    public MyView(Context context, AttributeSet attrs) {
        super(context, attrs);
        Log.e("log", "onCreate");
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        Log.e("log", "onFinishInflate");
    }


    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        Log.e("log", "onAttachedToWindow");
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.e("log", "onMeasure");

    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        Log.e("log", "onSizeChanged");
    }


    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        Log.e("log", "onLayout");

    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.e("log", "onDraw");
    }


    @Override
    public void onWindowFocusChanged(boolean hasWindowFocus) {
        super.onWindowFocusChanged(hasWindowFocus);
        Log.e("log", "onWindowFocusChanged" + "  " + hasWindowFocus);

    }
}123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354
```

很简单的代码，整个布局文件里面一个自定义View，进入这个界面，然后返回。依次打印各个生命周期，我们看一下打印的结果：

![这里写图片描述](http://img.blog.csdn.net/20161107151935240)

**二.结果分析**

1.Activity生命周期:onCreate

Activity生命周期的第一个方法，表示此Activity正在被创建，这里我们主要进行一些初始化的工作。比如调用setContentView()方法去加载界面布局资源。

2.View生命周期:onCreate

当Activity在onCreate加载界面布局资源的时候，我们自定义的View会在xml文件中被加载，并且调用构造函数 MyView(Context context, AttributeSet attrs)来加载自定义View。

3.View生命周期:onFinishInflate

源码是这样告诉我们的：

![这里写图片描述](http://img.blog.csdn.net/20161107160445481)

自己蹩脚的英语水平加上网上的一些解释，觉得这样理解还是比较靠谱的： 
onFinishInflate是在自定义View中所有的子控件均被映射成xml后调用。这样View就完成了初始准备工作，Activity也完成了对应布局资源的加载。

4.View生命周期:onAttachedToWindow

再来瞅一眼源码：

![这里写图片描述](http://img.blog.csdn.net/20161107161242302)

onAttachedToWindow是在将view绑定到activity所在window时调用，附加到window后，程序开始进行自定义View的绘制。

5.View生命周期:onMeasure

参考资料： 
[Android视图绘制流程完全解析，带你一步步深入了解View(二)](http://blog.csdn.net/sinyu890807/article/details/16330267)

[自定义View系列教程02–onMeasure源码详尽分析](http://blog.csdn.net/lfdfhl/article/details/51347818)

作为自定义View三部曲的第一步，onMeasure方法有着极其重要的作用，所以深入理解其中原理对我们以后自定义View开发有着很大的帮助。

MeasureSpec测量规范： 
系统显示一个View，首先需要通过测量(measure)该View来获取其长和宽从而确定显示该View时需要多大的空间。在测量的过程中MeasureSpec贯穿全程，发挥着不可或缺的作用，先瞅一眼源码：

![这里写图片描述](http://img.blog.csdn.net/20161107164013433)

从上面这张图片中我们可以获取到许多重要的信息：

**1.MeasureSpec封装了父布局传递给子View的布局要求**

**2. MeasureSpec可以表示宽和高**

**3 MeasureSpec由size(大小)和mode(模式)组成**

MeasureSpec是一个32位的int数据，其中高2位代表SpecMode即某种测量模式，低30位为SpecSize代表在该模式下的规格大小. 
可以通过如下方式分别获取这两个值：

获取SpecSize

> int specSize = MeasureSpec.getSize(measureSpec)

获取specMode

> int specMode = MeasureSpec.getMode(measureSpec)

通过这两个值生成新的MeasureSpec

> int measureSpec=MeasureSpec.makeMeasureSpec(size, mode);

SpecMode一共有三种: 
MeasureSpec.UNSPECIFIED，MeasureSpec.EXACTLY ， MeasureSpec.AT_MOST ， 我们依次看看每种测量模式代表什么意思：

MeasureSpec.UNSPECIFIED：父视图不对子视图施加任何限制，子视图可以得到任意想要的大小，表示子布局想要多大就多大。这种模式一般用作Android系统内部，或者ListView和ScrollView等滑动控件，很少使用。

MeasureSpec.EXACTLY ：父视图希望子视图的大小是specSize中指定的大小，在该模式下，View的测量大小即为SpecSize。

MeasureSpec.AT_MOST：父容器未能检测出子View所需要的精确大小，但是指定了一个可用大小即specSize ,在该模式下，View的测量大小不能超过SpecSize

可用表格来规整各一下MeasureSpec的生成:

![这里写图片描述](http://img.blog.csdn.net/20161107173932933)

**子View的MeasureSpec由其父容器的MeasureSpec和该子View本身的布局参数LayoutParams共同决定。**

当设置View的参数等于MATCH_PARENT的时候，MeasureSpec的specMode就等于EXACTLY，当设置View的参数等于WRAP_CONTENT的时候，MeasureSpec的specMode就等于AT_MOST。并且MATCH_PARENT和WRAP_CONTENT时的specSize都是等于windowSize的，也就意味着根视图总是会充满全屏的。**所以自定义View在重写onMeasure()的过程中应该手动处理View的宽或高为wrap_content的情况。**

经过测量得出的子View的MeasureSpec是系统给出的一个期望值(参考值)，我们也可摒弃系统的这个测量流程，直接调用setMeasuredDimension( )设置子View的宽和高的测量值。

需要注意的是，在setMeasuredDimension()方法调用之后，我们才能使用getMeasuredWidth()和getMeasuredHeight()来获取视图测量出的准确的宽度与高度。

由此可见，视图大小的控制是由父视图、布局文件、以及视图本身共同完成的，父视图会提供给子视图参考的大小，开发人员可以在XML文件中指定视图的大小，最后视图本身才会确定最终的大小。

6.View生命周期:onSizeChanged

继续看看源码是怎么解释的：

![这里写图片描述](http://img.blog.csdn.net/20161108111202970)

onSizeChanged是在布局文件中自定义View的大小发生改变时被调用。四个参数依次代表变化之后的宽高以及变化之前的宽高。在onMeasure方法结束之后第一次进行调用，将测量的宽高作为前两个参数，0作为后两个默认参数。

7.View生命周期:onLayout

测量过程结束后，视图的大小就已经测量好了，接下来就是layout的过程了。正如其名字所描述的一样，这个方法是用于给视图进行布局的，也就是确定视图的具体位置。

看看源码怎么解释的：

![这里写图片描述](http://img.blog.csdn.net/20161108112042293)

onLayout是在layout方法中指定子View的大小和位置。其实就是ViewGroup会调用onLayout(）决定子View的显示位置。其中四个参数l, t, r, b分别表示子View相对于父View的左、上、右、下的坐标。

**getMeasureWidth()方法在measure()过程结束后就可以获取到了，而getWidth()方法要在layout()过程结束后才能获取到。在某些复杂或者极端的情况下系统会多次执行measure过程，所以在onMeasure()中去获取View的测量大小得到的是一个不准确的值。为了避免该情况，最好在onMeasure()的下一阶段即onLayout()中去获取View的宽高。**

8.View生命周期:onDraw

看看源码的解释：

![这里写图片描述](http://img.blog.csdn.net/20161108151231050)

onDraw是真正地开始对视图进行绘制。 
在Andorid官方文档中将该过程概况成了六步：

**第一步**：绘制背景：Draw the background

**第二步：**保存当前画布的堆栈状态并在该画布上创建Layer用于绘制View在滑动时的边框渐变效果，通常情况下我们不需要处理这一步：If necessary, save the canvas’ layers to prepare for fading。

**第三步：绘制View的内容：Draw view’s content。这一步是整个draw阶段的核心，在此会调用onDraw()方法绘制View的内容。 之前我们在分析layout的时候发现onLayout()方法是一个抽象方法，具体的逻辑由ViewGroup的子类去实现。与之类似，在此onDraw()是一个空方法；因为每个View所要绘制的内容不同，所以需要由具体的子View去实现各自不同的需求。**

**第四步：**调用dispatchDraw()绘制View的子View：Draw children

**第五步**：绘制当前视图在滑动时的边框渐变效果，通常我们是不需要处理这一步的：If necessary, draw the fading edges and restore layers

**第六步**：绘制View的滚动条：Draw decorations (scrollbars for instance)

在绘图时需要明确四个核心的东西(basic components)：

**用什么工具画？** 
这个小问题很简单，我们需要用一支画笔(Paint)来绘图。 
当然，我们可以选择不同颜色的笔，不同大小的笔。 
**把图画在哪里呢？** 
我们把图画在了Bitmap上，它保存了所绘图像的各个像素(pixel)。 
也就是说Bitmap承载和呈现了画的各种图形。 
**画的内容？** 
根据自己的需求画圆，画直线，画路径。 
**怎么画？** 
调用canvas执行绘图操作。 
比如，canvas.drawCircle()，canvas.drawLine()，canvas.drawPath()将我们需要的图像画出来。

关于绘制View的内容，这里就不多做讨论，以后实战中根据实际情况来进行详细的分析，继续下一个生命周期。

9.View生命周期:onWindowFocusChanged

瞅一眼源码：

![这里写图片描述](http://img.blog.csdn.net/20161108164459263)

判断view是否获取焦点，参数hasWindowFocus 对应返回true 和false 可以用来判断view是否进出后台。第一次进入当前Activity的时候，返回true，返回上一个Activity，返回了false。

至此，我们已经经历了一次完整的自定义View的生命周期，接下来就是应用到实际项目中，希望能对你有所帮助，下一篇再见~~~