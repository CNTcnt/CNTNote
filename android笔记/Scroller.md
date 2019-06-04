# Scroller

上一个方法中我们说到： 
scrollTo和scrollBy都是瞬间移动的，为了它们达到平滑移动的效果，我们可以通过motionEvent来多次处理， 
但是，如果是像ViewPager一样的操作，当用户滑动并抬起时，Viewpager会自己进行回弹或者进入下一个page，期间没有任何MotionEvent， 
如果使用scrollBy和To则会直接移动，没有动画效果，这又该怎么办呢？

我们可以试想以下， 
要想实现这个效果，无非就是让view使用scrollTo/By每次移动一小段距离，多次形成一个动画效果（即动画的原理） 
所以我们需要明确以下几点： 
\1. 每次移动多少距离？ 
\2. 移动从什么位置移动到什么位置？ 
\3. 每次移动后都要进行一次重绘（invalidate()）,重绘后，再移动，再重绘 
\4. 那么什么时候结束呢？即移动到目标点后，即可完成重绘。

所以google为了实现这个功能，提供了Scroller类，这是一个工具类。

常用方法如下：

```java
mScroller.getCurrX() //获取mScroller当前水平滚动的位置
mScroller.getCurrY() //获取mScroller当前竖直滚动的位置
mScroller.getFinalX() //获取mScroller最终停止的水平位置
mScroller.getFinalY() //获取mScroller最终停止的竖直位置

mScroller.setFinalX(int newX) //设置mScroller最终停留的水平位置，没有动画效果，直接跳到目标位置
mScroller.setFinalY(int newY) //设置mScroller最终停留的竖直位置，没有动画效果，直接跳到目标位置

mScroller.startScroll(int startX, int startY, int dx, int dy) //滚动，startX, startY为开始滚动的位置，dx,dy为滚动的偏移量
mScroller.startScroll(int startX, int startY, int dx, int dy, int duration) //滚动，startX, startY为开始滚动的位置，dx,dy为滚动的偏移量, duration为完成滚动的时间

mScroller.computeScrollOffset() //返回值为boolean，true说明滚动尚未完成，false说明滚动已经完成。这是一个很重要的方法，通常放在View.computeScroll()中，用来判断是否滚动是否结束
```

![这里写图片描述](http://img.blog.csdn.net/20161116124936247)

使用这个类实现效果，有以下步骤： 
\1. 初始化Scroller 
\2. 重写View的computeScroll()：用来计算滑动的一些数值，以及滑动的逻辑 
\3. 调用startScroll():开始执行滑动过程

code如下：

```java
import android.content.Context;
import android.graphics.Color;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.widget.Scroller;

public class ScrollerTestView extends View {

    private int lastX;
    private int lastY;
    private Scroller mScroller;

    public ScrollerTestView(Context context) {
        this(context, null);
    }

    public ScrollerTestView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ScrollerTestView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        ininView(context);
    }

    private void ininView(Context context) {
        setBackgroundColor(Color.BLUE);
        // 1. 初始化Scroller
        mScroller = new Scroller(context);
    }

    // 2. 重写该方法，用来计算一次滑动的距离，并重绘
    @Override
    public void computeScroll() {
        super.computeScroll();
        // 如果完成了滑动，则不再进行重绘，否则继续重绘
        if (mScroller.computeScrollOffset()) {
            ((View) getParent()).scrollTo(
                    mScroller.getCurrX(),
                    mScroller.getCurrY());
            // 通过重绘来不断调用computeScroll
            // invalidate()重绘会调用draw() draw()则会调用computeScroll()方法，从而实现间接的递归
            invalidate();
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = (int) event.getX();
                lastY = (int) event.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                int offsetX = x - lastX;
                int offsetY = y - lastY;
                ((View) getParent()).scrollBy(-offsetX, -offsetY);
                break;
            case MotionEvent.ACTION_UP:
                // 3. 手指离开时，执行滑动过程
                View viewGroup = ((View) getParent());
                mScroller.startScroll(
                        viewGroup.getScrollX(),
                        viewGroup.getScrollY(),
                        -viewGroup.getScrollX(),
                        -viewGroup.getScrollY());
                invalidate();
                break;
        }
        return true;
    }
}
```