### Selector



上一篇详细讲了shape的用法，讲解了怎么用shape自定义矩形、圆形、线形和环形，以及有哪些需要注意的地方。不过，shape只能定义单一的形状，而实际应用中，很多地方比如按钮、Tab、ListItem等都是不同状态有不同的展示形状。举个例子，一个按钮的背景，默认时是一个形状，按下时是一个形状，不可操作时又是另一个形状。有时候，不同状态下改变的不只是背景、图片等，文字颜色也会相应改变。而要处理这些不同状态下展示什么的问题，就要用selector来实现了。

selector标签，可以添加一个或多个item子标签，而相应的状态是在item标签中定义的。定义的xml文件可以作为两种资源使用：drawable和color。作为drawable资源使用时，一般和shape一样放于**drawable**目录下，item必须指定**android:drawable**属性；作为color资源使用时，则放于**color**目录下，item必须指定**android:color**属性。

那么，看看都有哪些状态可以设置呢：

- **android:state_enabled**: 设置触摸或点击事件是否可用状态，一般只在false时设置该属性，表示不可用状态
- **android:state_pressed**: 设置是否按压状态，一般在true时设置该属性，表示已按压状态，默认为false
- **android:state_selected**: 设置是否选中状态，true表示已选中，false表示未选中
- **android:state_checked**: 设置是否勾选状态，主要用于CheckBox和RadioButton，true表示已被勾选，false表示未被勾选
- **android:state_checkable**: 设置勾选是否可用状态，类似state_enabled，只是state_enabled会影响触摸或点击事件，而state_checkable影响勾选事件
- **android:state_focused**: 设置是否获得焦点状态，true表示获得焦点，默认为false，表示未获得焦点
- **android:state_window_focused**: 设置当前窗口是否获得焦点状态，true表示获得焦点，false表示未获得焦点，例如拉下通知栏或弹出对话框时，当前界面就会失去焦点；另外，ListView的ListItem获得焦点时也会触发true状态，可以理解为当前窗口就是ListItem本身
- **android:state_activated**: 设置是否被激活状态，true表示被激活，false表示未激活，API Level 11及以上才支持，可通过代码调用控件的setActivated(boolean)方法设置是否激活该控件
- **android:state_hovered**: 设置是否鼠标在上面滑动的状态，true表示鼠标在上面滑动，默认为false，API Level 14及以上才支持

接下来，看看示例代码，以下是bg_btn_selector.xml的代码，用于按钮的背景：

```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 当前窗口失去焦点时 -->
    <item android:drawable="@drawable/bg_btn_lost_window_focused" android:state_window_focused="false" />
    <!-- 不可用时 -->
    <item android:drawable="@drawable/bg_btn_disable" android:state_enabled="false" />
    <!-- 按压时 -->
    <item android:drawable="@drawable/bg_btn_pressed" android:state_pressed="true" />
    <!-- 被选中时 -->
    <item android:drawable="@drawable/bg_btn_selected" android:state_selected="true" />
    <!-- 被激活时 -->
    <item android:drawable="@drawable/bg_btn_activated" android:state_activated="true" />
    <!-- 默认时 -->
    <item android:drawable="@drawable/bg_btn_normal" />
</selector>

```

而下面则是text_btn_selector.xml的代码，用于按钮的文本颜色：

```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 当前窗口失去焦点时 -->
    <item android:color="@android:color/black" android:state_window_focused="false" />
    <!-- 不可用时 -->
    <item android:color="@android:color/background_light" android:state_enabled="false" />
    <!-- 按压时 -->
    <item android:color="@android:color/holo_blue_light" android:state_pressed="true" />
    <!-- 被选中时 -->
    <item android:color="@android:color/holo_green_dark" android:state_selected="true" />
    <!-- 被激活时 -->
    <item android:color="@android:color/holo_green_light" android:state_activated="true" />
    <!-- 默认时 -->
    <item android:color="@android:color/white" />
</selector>

```

最后，则是在控件中的引用：

```
<Button 
    android:id="@+id/btn_default"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    android:background="@drawable/bg_btn_selector"
    android:text="默认按钮"
    android:textColor="@color/text_btn_selector" />

```

那么，在使用过程中，有几点还是需要注意和了解的：

1. selector作为drawable资源时，item指定**android:drawable**属性，并放于**drawable**目录下；
2. selector作为color资源时，item指定**android:color**属性，并放于**color**目录下；
3. color资源也可以放于drawable目录，引用时则用**@drawable**来引用，但不推荐这么做，drawable资源和color资源最好还是分开；
4. **android:drawable**属性除了引用**@drawable**资源，也可以引用**@color**颜色值；但**android:color**只能引用**@color**；
5. item是从上往下匹配的，如果匹配到一个item那它就将采用这个item，而不是采用最佳匹配的规则；所以设置默认的状态，一定要写在最后，如果写在前面，则后面所有的item都不会起作用了。

另外，selector标签下有两个比较有用的属性要说一下，添加了下面两个属性之后，则会在状态改变时出现淡入淡出效果，但必须在API Level 11及以上才支持：

- **android:enterFadeDuration** 状态改变时，新状态展示时的淡入时间，以毫秒为单位
- **android:exitFadeDuration** 状态改变时，旧状态消失时的淡出时间，以毫秒为单位

最后，关于ListView的ListItem样式，有两种设置方式，一种是在ListView标签里设置**android:listSelector**属性，另一种是在ListItem的布局layout里设置**android:background**。但是，这两种设置的结果却有着不同。同时，使用ListView时也有些其他需要注意的地方，总结如下：

1. **android:listSelector**设置的ListItem默认背景是透明的，不管你在selector里怎么设置都无法改变它的背景。所以，如果想改ListItem的默认背景，只能通过第二种方式，在ListItem的布局layout里设置**android:background**。

2. 当触摸点击ListItem时，第一种设置方式下，**state_pressed**、**state_focused**和**state_window_focused**设为true时都会触发，而第二种设置方式下，只有**state_pressed**会触发。

3. 当ListItem里有Button或CheckBox之类的控件时，会抢占ListItem本身的焦点，导致ListItem本身的触摸点击事件会无效。那么，要解决此问题，有三种解决方案：

   - 将Button或CheckBox换成TextView或ImageView之类的控件
   - 设置Button或CheckBox之类的控件设置**focusable**属性为false
   - 设置ListItem的根布局属性**android:descendantFocusability="blocksDescendants"**

   第三种是最方便，也是推荐的方式，它会将ListItem根布局下的所有子控件都设置为不能获取焦点。**android:descendantFocusability**属性的值有三种，其中，ViewGroup是指设置该属性的View，本例中就是ListItem的根布局：

   - **beforeDescendants**：ViewGroup会优先其子类控件而获取到焦点
   - **afterDescendants**：ViewGroup只有当其子类控件不需要获取焦点时才获取焦点
   - **blocksDescendants**：ViewGroup会覆盖子类控件而直接获得焦点