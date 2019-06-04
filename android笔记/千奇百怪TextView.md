[TOC]

# 千奇百怪TextView

## 1.AutoSizeableTextView

* 简介：公要求在一个列表中的每个条目中展示字数不限个数的文本。而且每个条目的宽度都是固定的，展示的文本如果过长，不可以用省略号显示，只能动态的调整(缩小)文本的字号来达到文本能完全显示的效果，而且要限一行展示。关于这个效果，其实目前android官方已经提供了实现方式。那就是AutoSizeableTextView。

* 其实 AutoSizeableTextView 是一个接口，我们可以使用一个它的一个实现类 AppCompatTextView ；

* 之所以不使用 TextView ，其实是为了兼容老版本，如果你的项目不针对27以下的版本进行兼容，你完全可以直接在xml中声明TextView控件，而且在xml中也可以直接用android声明的那几个属性进行设置，无须再引入app空间下的属性。而当低于27的时候，这个TextView必须属于AutoSizeableTextView类型的，而前面已经说过AppCompatTextView 实现了 AutoSizeableTextView 接口，因此，为了兼容老版本，我们在xml声明的时候需要声明为AppCompatTextView。

* xml 使用：

  ~~~java
  <android.support.v7.widget.AppCompatTextView
          android:layout_width="match_parent"
          android:layout_height="40dp"
          android:maxLines="1"
          android:textSize="35dp"
          android:textColor="#000000"                                    
          app:autoSizeTextType="uniform"//设置TextView大小设置样式为支持改变(none时为不支持改变)
          app:autoSizeStepGranularity="1dp"//每次改变的尺寸阶梯
          app:autoSizeMinTextSize="10dp" //最小字体号
          app:autoSizeMaxTextSize="40dp" //最大字体号
          android:text="房价快速犯得上发生大夫士大夫房贷首付"/>
  ~~~

* 代码中动态设置

  ~~~java
  TextViewCompat.setAutoSizeTextTypeUniformWithConfiguration(
                  textView, 10, 40, 1, TypedValue.COMPLEX_UNIT_SP);
  ~~~

* 其次，要说到一个特别要注意的事情，那就是**控件的宽度和高度必须要有具体的值**，不能设置为wrap_content。这一点估计也好理解，**如果宽高不固定，也就不会有根据宽高改变字号这一问题了。**

  ​