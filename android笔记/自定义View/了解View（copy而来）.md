[TOC]



###了解View（copy而来）

####LayoutInflater

* 相信接触Android久一点的朋友对于LayoutInflater一定不会陌生，都会知道它主要是用于加载布局的。而刚接触Android的朋友可能对LayoutInflater不怎么熟悉，因为加载布局的任务通常都是在Activity中调用setContentView()方法来完成的。其实setContentView()方法的内部也是使用LayoutInflater来加载布局的，只不过这部分源码是internal的，不太容易查看到。那么今天我们就来把LayoutInflater的工作流程仔细地剖析一遍，也许还能解决掉某些困扰你心头多年的疑惑。

* 先来看一下LayoutInflater的基本用法吧，它的用法非常简单，首先需要获取到LayoutInflater的实例，有两种方法可以获取到，第一种写法如下：

  ~~~java
  LayoutInflater layoutInflater = LayoutInflater.from(context);  
  ~~~

* 当然，还有另外一种写法也可以完成同样的效果：

  ~~~java
  LayoutInflater layoutInflater = (LayoutInflater) context  

  				getSystemService(Context.LAYOUT_INFLATER_SERVICE);  
  			
  ~~~

  其实第一种就是第二种的简单写法，只是Android给我们做了一下封装而已。得到了LayoutInflater的实例之后就可以调用它的inflate()方法来加载布局了，如下所示：

  ~~~java
  layoutInflater.inflate(resourceId, root);  
  ~~~

  inflate()方法一般接收两个参数，第一个参数就是要加载的布局id，第二个参数是指给该布局的外部再嵌套一层父布局，如果不需要就直接传null。这样就成功成功创建了一个布局的实例，之后再将它添加到指定的位置就可以显示出来了。

  举例：

  ~~~java
  super.onCreate(savedInstanceState);  
          setContentView(R.layout.activity_main);  
          mainLayout = (LinearLayout) findViewById(R.id.main_layout);  
          LayoutInflater layoutInflater = LayoutInflater.from(this);//拿到LayoutInflater对象  
          View buttonLayout = layoutInflater.inflate(R.layout.button_layout, null);//加载一个布局，可直接指定父布局加入即可，这里不要，要在下面动态加入；
          mainLayout.addView(buttonLayout); //在主布局中把加载好的布局加入；
  ~~~

#### VIew的大小（Layout_Width/height）

* 它们其实是用于设置View在布局中的大小的，也就是说，首先View必须存在于一个布局中，之后如果将layout_width设置成match_parent表示让View的宽度填充满布局，如果设置成wrap_content表示让View的宽度刚好可以包含其内容，如果设置成具体的数值则View的宽度会变成相应的数值。这也是为什么这两个属性叫作layout_width和layout_height，而不是width和height。
* 平时在Activity中指定布局文件的时候，最外层的那个布局是可以指定大小的呀，layout_width和layout_height都是有作用的。确实，这主要是因为，在setContentView()方法中，Android会自动在布局文件的最外层再嵌套一个FrameLayout，所以layout_width和layout_height属性才会有效果。

