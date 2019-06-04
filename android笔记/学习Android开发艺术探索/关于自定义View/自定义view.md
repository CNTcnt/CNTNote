[TOC]



# 自定义view

## 自定义view须知

1. 让view支持wrap_content

   这是因为直接继承于view或者view Group的控件，如果不再onMeasure中队wrap_content做特殊处理，那么当外界在布局中使用wrap_content时就无法达到预期的结果；

2. 如果有必要，让view支持padding

   这是来直接继承view的控件，如果不在draw方法中处理padding，那么padding属性无法起作用；

   直接继承自viewGroup的控件需要在onMeasure和onLayout中考虑padding和子元素的margin对其造成的影响，不然它们不起作用；

3. 不要在view中使用Handler，因为没必要

   因为view内部就提供了post系列的方法，完全可以替代Handler的作用；

4. 如果view中有线程或者动画，需要及时停止，可在View #onDetachedFromWindow中操作；

   当包含此view的Activity推出或者当前view被remove时，view的onDetachedFromWindow方法会被调用，和此方法对应的是，view的onAttachedToWindow方法会被调用；

   当view变得不可见时我们也需要停止线程和动画，如果不及时处理这种问题，有可能会造成内存泄漏；

5. view带有滑动冲突时要处理好；

## 自定义view示例

1. 继承view重写onDraw方法

   * 这种方法主要用于实现一些不规则的效果，一般需要重写onDraw方法；采用这种方法需要做自己支持wrap_content,并且padding也需要自己处理。

   * 为了让view更加容易使用，很多情况下我们需要为其提供自定义属性；

     1. 在values目录下面创建自定义属性的xml，比如 attrs.xml,名字随便取（格式自行百度）；

     2. 在view的构造方法中解析自定义属性的值并做出相应处理，如：

        ~~~java
        public CircleView(Vontect context, AttributeSet attrs, int defStyleAttr){
           super(context,attrs, defStyleAttr);
          //首先加载自定义属性集合CircleView
           TypedAttay a = context.obtainStyledAttributes(attrs, R.styleable.CitcleView)
        //接着解析CircleView属性集合中的Circle_color属性，id为R.styleable.CitcleView_vitcle_color
        //如果使用时没有指定circle_color，那么就会选择红色作为默认的颜色值
           mColor = a.getColot(R.styleable.CitcleView_vitcle_color, Color.RED);
          //解析完自定义属性后，通过recycle方法来释放资源
           a.recycle();
           init();
        }
        ~~~

        ​

   ​