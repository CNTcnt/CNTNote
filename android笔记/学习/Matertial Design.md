### Material   Design

[TOC]

***

#### Toolbar

* 新建任何一个项目，都会默认显示ActionBar，但是现在更推荐使用ToolBar;

1. ~~~java
   <application
           android:allowBackup="true"
           android:icon="@mipmap/ic_launcher"
           android:label="@string/app_name"
           android:supportsRtl="true"
           android:theme="@style/AppTheme">//这里主题指定了定义在res/values/style.xml文件中的名为AppTheme的主题，ActionBar就是这个主题中的主题，而我们在需要用到ToolBar时，需要到这个文件中去修改它
   ~~~

   ~~~java
   <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">//这里是修改好了的，一般有2种形式可选择，Theme.AppCompat.Light.NoActionBar:页面主体设为淡色，陪衬颜色设置为深色；
     		//Theme.AppCompat.NoActionBar：页面主体设为深色，陪衬颜色设置为淡色；
           <!-- Customize your theme here. -->
           <item name="colorPrimary">@color/colorPrimary</item>//这里设置标题栏的背景色
           <item name="colorPrimaryDark">@color/colorPrimaryDark</item>//这里设置标题栏的上方背景色
           <item name="colorAccent">@color/colorAccent</item>//这里设置独立配件的自定义色，如按钮的选中状态的颜色
       </style>
   ~~~

2. 在页面中使用Toolbar时，为了系统版本兼容性，需要将一些新Material属性用新的命名空间定义出来使用，

   ~~~java
   <android.support.v7.widget.Toolbar
           android:id="@+id/toolbar"
           android:layout_width="match_parent"
           android:layout_height="?attr/actionBarSize"//高度指定为actionBarSize的高度，在?attr中
           android:background="@color/colorPrimary"
           android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"//之前我们将ToolBar主题设置为淡色主题，所以上面的元素的陪衬颜色是深色的，但是此时如果标题颜色变色深色会非常丑，所以需要将Toolbar单独使用深色主题，但是如果这样设定，那么我们的Toolbar如果里面包含菜单项，那么下拉菜单也会变成深色主题，又会非常丑，所以使用新属性popupTheme,将弹出的菜单项设置为淡色主题，使用时要根据Toolbar的主题对应使用；
           app:popupTheme="@style/Theme.AppCompat.Light"///>
   ~~~

3. ~~~java
   //使用时在主界面函数添加这2句即可
   Toolbar toolbar = (Toolbar)findViewById(R.id.toolbar);
   setSupportActionBar(toolbar);//将Toolbar传入即可；
   ~~~

4. 在Toolbar上，我们需要修改标题文字时，在AndroidManifest.xml中修改，如不修改，则默认使用application中指定的label内容（这个内容是我们的应用名称）；

   ~~~java
   <activity android:name=".MainActivity"
                   android:label="toolBar">
   ~~~

5. 现在使toolbar丰富一点，可以设置菜单项；现在res目录下创建一个menu文件夹，然后在这个文件夹下new一个Menu     resource file 创建toolbar.xml文件；写入如下代码

   ~~~java
    <item
           android:id="@+id/ic_backup"//菜单项名
           android:icon="@mipmap/picture1"//菜单项图标
           android:title="Backup"//菜单项文字
           app:showAsAction="always"//菜单项的表示位置，这里的always表示永远显示在Toolbar中，如果空间不够则不显示；换成ifRoom则表示如果Toolbar空间够就显示在toolbar中，如果不够就显示在下拉菜单中；换成never则表示永远显示在下拉菜单中；注意：在Toolbar表示的菜单项只显示图标，在下拉菜单的菜单项只显示文字
             />
   ~~~

6. 在页面使用时，我们需要告知activity:Toolbar的菜单位置，以便它加载到菜单文件；

   ~~~java
    @Override
       public boolean onCreateOptionsMenu(Menu menu) {
           getMenuInflater().inflate(R.menu.toolbar,menu);//R文件中的menu目录的toolbar文件
           return  true;}
   ~~~

7. 不可缺失注册菜单项的点击事件：

   ~~~java
   @Override
       public boolean onOptionsItemSelected(MenuItem item) {
           switch (item.getItemId()){
               case R.id.ic_backup:
                   Toast.makeText(this,"You Clicked Backup",Toast.LENGTH_SHORT).show();break;}
           return true; }
   ~~~

#### 滑动菜单

简介：滑动菜单是将一些菜单项隐藏起来，而不是放置在主屏幕上，需要时可以通过滑动的方式将菜单显示出来；

##### DrawLayout：拖拉布局（滑动菜单布局）

简介：这是一个允许放入2个直接子控件的滑动菜单布局，第一个控件是主屏幕页面（一般是主界面布局），第2个控件是滑动菜单中的显示内容；

注意：第2个控件一定要指定一个layout_gravity这个属性，因为我们要告诉DrawerLayout滑动菜单是在屏幕的左边还是右边，有left和right和start；一般指定为start，因为指定这个后会根据系统的语言进行判断，如果系统语言是从左往右的，那么活动菜单就在左边，比如英语汉语，阿拉伯语就在右边；

* 完成后，我们在主界面向右滑动菜单后就会出现滑动菜单页面；

* 但有时用户不知道有这个功能，所以我们需要在Toolbar最左端添加一个导航按钮，点击后也会出现菜单页面，这样就相当于有了2种方法来触发菜单；

  ~~~java
  ActionBar actionBar = getSupportActionBar();//得到Toolbar实例，
          if(actionBar!=null){
              actionBar.setDisplayHomeAsUpEnabled(true);//让导航按钮显示出来，这个按钮叫做HomeAsUp；
              actionBar.setHomeAsUpIndicator(R.mipmap.ic_launcher);//设置导航按钮的图标，不设置则为返回箭头 }
  ~~~

* 然后设置HomeAsUp按钮的点击事件

  ~~~java
  case android.R.id.home://这是HomeAsUp按钮的id
  drawerLayout.openDrawer(GravityCompat.START);//然后调用DrawerLayout的openDrawer()将滑动菜单展示出来，这里要传入一个Gravity参数，为了和刚才在.xml文件中设置的start一致，所以传入GravityCompat.START
  break;
  ~~~

##### NavigationView：简单实现滑动菜单的页面

* 滑动菜单内肯定要有各种控件；谷歌提供了一个控件来简单的实现：NavigationView：导航控件

* 使用前要添加依赖；

  ~~~java
   compile 'com.android.support:design:24.2.1'//这个是NavigationView的库依赖
      compile 'de.hdodenhof:circleimageview:2.1.0'//这个是后面要把正方形图片转化成圆形图片的库依赖
  ~~~

* NavigationView分为2个部分，一个是menu部分（菜单部分），一个是头部分（一个布局，在滑动菜单顶部的布局）

* ~~~java
  //把滑动菜单第二个部分（控件）修改为NavigationView；
  <android.support.design.widget.NavigationView android:id="@+id/nav_view"
          android:layout_height="match_parent"
         android:layout_width="match_parent"
         android:layout_gravity="start"//这个属性必须添加，功能前面有介绍
         app:menu="@menu/nav_menu"//把菜单主体部分添加进来
         app:headerLayout="@layout/nav_header"//把菜单的头部分添加进来
  ~~~

  ~~~java
  /这是菜单主体部分
  <menu xmlns:android="http://schemas.android.com/apk/res/android">/
      <group android:checkableBehavior="single">//group表示一个组，checkableBehavior="single"属性表示组中的菜单项只能单选
          <item android:id="@+id/nav_call"
              android:title="Call"
              android:icon="@mipmap/picture4"/>
          <item android:id="@+id/nav_friend"
              android:title="friend"
              android:icon="@mipmap/picture5"/></group>
  ~~~

  ~~~java
  //这是菜单的头部分
  <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_width="match_parent"
      android:layout_height="180dp"//头部分只需要高度为180dp就比价适合
      android:padding="10dp"//每个控件间隔10dp
      android:background="@color/colorPrimary">
      <de.hdodenhof.circleimageview.CircleImageView//这里就用到了之前添加的一个库，用来将图片圆形化
          android:id="@+id/icon_image"
          android:layout_width="70dp"
          android:layout_height="70dp"
          android:layout_centerInParent="true"
          android:src="@drawable/xpic5224" />
      头部分还可添加如用户名和邮箱之类的信息；
  ~~~

* 接下来还要为滑动菜单里的菜单项设置点击事件

  ~~~java
  NavigationView navigationView = (NavigationView) findViewById(R.id.nav_view);//先找到滑动菜单
          navigationView.setCheckedItem(R.id.nav_call);//这里是设置默认选中的菜单项
          navigationView.setNavigationItemSelectedListener(new/*这里就是设置菜单项的点击事件，先注册监听器，再覆写点击回调事件*/ NavigationView.OnNavigationItemSelectedListener(){
              @Override
              public boolean onNavigationItemSelected(@NonNull MenuItem item) {
                  drawerLayout.closeDrawers();//这里只是简单地把滑动菜单关闭而已
                  return true; }});
  ~~~


#### 悬浮按钮和可交互提示

##### 悬浮按钮FloatingActionButton

* 不属于主界面平面的一部分，位于另外一个维度

* FloatActionButton:是Design Support库中提供的一个控件，默认使用colorAccent来作为按钮的颜色；我们还可以通过给按钮指定一个图标来表示这个按钮的作用是什么；

  ~~~java
  <android.support.design.widget.FloatingActionButton android:id="@+id/floatingButton"
              android:layout_width="wrap_content"
              android:layout_height="wrap_content"
              android:layout_margin="16dp"
              android:layout_gravity="bottom|end"//end的意思和之前start意思一样
              app:elevation="8dp"//这个属性用于给按钮设置一个高度值，高度值越大，投影范围越大，但是投影效果越淡
              android:src="@drawable/xpic5224"//用于指定按钮上的图标
  ~~~

* 不可或缺的给按钮注册点击事件；步骤和普通按钮一致；

  ~~~java
  FloatingActionButton floatingActionButton = (FloatingActionButton)findViewById(R.id.floatingButton);
          floatingActionButton.setOnClickListener(new View.OnClickListener(){
              @Override
              public void onClick(View v) {
                  Toast.makeText(MainActivity.this,"emmmm",Toast.LENGTH_SHORT).show();} });
  ~~~

##### Snackbar

* 是由Design Support库提供的类似于Toast 的一个提示工具；但是和Toast的使用场景不同，Toast用于告诉用户现在发生了什么事件，但是用户只能被动地接收信息，没有交互，而Snackbar中还有一个按钮，可用于交互，整个Snackbar从底部滑出来让用户看到；

* Snackbar使用方法和Toast类似；make有3个参数，第一个参数是任意一个View，Snackbar会自动根据这个View找到最外层View，用来展示Snackbar，第二个参数就是Snackbar中显示的内容；紧急着调用一个setAction()动作，来设置按钮的名字和事件,最后show()即可；

  ~~~java
   Snackbar.make(v,"Data delete",Snackbar.LENGTH_SHORT).setAction("Undo",new View.OnClickListener(){
                      @Override
                      public void onClick(View v) {
                          Toast.makeText(MainActivity.this,"Data restored",Toast.LENGTH_SHORT).show();}}).show();
  ~~~



##### CoordinatorLayout:协调者布局

* 由Design Support库提供的一个加强版的**FrameLayout**布局，用法和FrameLayout布局类似，但是更加智能，因为CoordinatorLayout可以监听它自己内部的所有子控件的事件，然后自动地做出最合理的相应，但仅限于它自己的自控件；

* 举列子：当不用这个布局时，当悬浮按钮FloatingActionButton处于最底部时，如果Snackbar触发，则会把按钮遮挡住，此时要使用CoordinatorLayout，使用后，它会只能地做出响应，把按钮往上移动；

  ~~~java
  <android.support.design.widget.CoordinatorLayout
  ~~~

#### 卡片式布局：CardView

* 卡片式布局是Materials Design中提出的一个新的概念，他可以让页面中的元素看起来就像在卡片中一样，并且还可以拥有圆角和投影；

* CardView：用于实现卡片布局的控件，由appcompat-v7库提供，所以要添加库依赖，实际上，CardView也是一个FrameLayout，只是额外提供了圆角和阴影效果，看上去立体感；把一个布局弄成一张卡片；

  ~~~java
  <android.support.v7.widget.CardView 
  	xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      app:cardCornerRadius="4dp"//这里是卡片圆角的弧度，数值越大，弧度越大
      android:elevation="5dp"//指定卡片的高度，高度越大，投影范围越大，但是投影效果越淡；
      android:layout_margin="5dp">
  //这里添加卡片里的布局控件
  </android.support.v7.widget.CardView>
  ~~~

  ~~~java
  compile 'com.android.support:recyclerview-v7:24.2.1'//这是RecycleView的库依赖
      compile 'com.android.support:cardview-v7:24.2.1'//这就是CardView的库依赖
      compile 'com.github.bumptech.glide:glide:3.7.0'//这是Glide库库依赖，是一个超级强大的图片加载器，不仅可以用于加载本地图片，还可以加载网络图片，GIF图片，甚至是本地视屏；只需一行代码就可轻松实现复杂的图片加载功能；
  ~~~

  ~~~java
  <ImageView//这里是为了说明scaleType属性，这个属性指定照片的缩放模式，以让照片可以充满整个ImageView
              android:id="@+id/item_imageView"
              android:layout_width="match_parent"
              android:layout_height="100dp"
              android:scaleType="centerInside" />
  ~~~

* Glide的用法

  ~~~java
  //Glide.with()方法传入一个Context，Activity或Fragment参数，然后调用load()方法去加载图片，可以是一个URL地址，也可以是一个本地路径，或者是一个资源id，然后调用into()将图片设置到具体的一个ImageView中即可；
  //之所以要用到这个Glide而不是用传统的设置图片方式，是因为如果图片像素太高，不进行压缩就直接展示的话，很容易引起内存溢出。而Glide在内部实现了许多复杂的逻辑，其中就包括了图片压缩，所以推荐使用；
  Glide.with(context).load(fruit.getImageView()).into(holder.imageView);
  ~~~

* 将RecycleView的页面布局方式设置为：

  ~~~java
   GridLayoutManager layoutManager = new GridLayoutManager(this,2);//第二个参数是列数
   recyclerView.setLayoutManager(layoutManager);
  ~~~


#### AppBarLayout:智能ActionBar

* AppBarLayout又必须是CoordinatorLayout的自布局

- AppBarLayout实际上是一个垂直方向的LinearLayout，内部做了很多滚动事件的封装，并应用了MaterialDesign理念

- 当屏幕滚动时，已经将滚动事件通知给APPBarLayout了，当AppBarLayout接收到滚动事件时，它内部的子控件是可以指定如何去影响这些事件的，通过app:layout_scrollFlags属性就能实现；

  ~~~java
  <android.support.design.widget.AppBarLayout//把Toolbar放在这个AppBarLayout中,就能让Toolbar会随着屏幕的滚动而向上消失或向下出现；
              android:layout_width="match_parent"
              android:layout_height="wrap_content">
              <android.support.v7.widget.Toolbar
                  android:id="@+id/toolbar"
                  android:layout_width="match_parent"
                  android:layout_height="?attr/actionBarSize"
                  android:background="?attr/colorPrimary"
                  android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
                  app:layout_scrollFlags="scroll|enterAlways|snap"
                  app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />
          </android.support.design.widget.AppBarLayout>
          <android.support.v7.widget.RecyclerView
                  android:id="@+id/recyclerView"
                  android:layout_width="match_parent"
                  android:layout_height="match_parent"
  			   app:layout_behavior="@string/appbar_scrolling_view_behavior"//标题栏下面是主操作页面，这里设定的是RecycleView，要实现标题栏的上升和下降，就要在主操作页面添加这一行代码，让它相应AppBarLayout的相应；可以认为指定了一个跟随着标题栏的控件的布局行为
                  ></android.support.v7.widget.RecyclerView>
  ~~~

  ​

  ~~~java
  app:layout_scrollFlags="scroll|enterAlways|snap"//scroll表示当用户向上滚动屏幕时，一般是Toolbar控件会随着向上滑动隐藏；enterAlways表示向下滚动时，Toolbar会一起向下滚动并重新显现；snap表示当Toolbar还没有完全隐藏或显示的时候，会根据当前的滑动距离，自动选择是隐藏还是显示；
  ~~~

#### 下拉刷新：SwipeRefreshLayout

- SwipeRefreshLayout是用于实现下拉菜单的核心类，由support-v4库提供。

- 把要实现下拉刷新的控件放置到SwipeRefreshLayout中，就可以迅速让这个控件支持下拉刷新这个功能；

  ~~~java
  <android.support.v4.widget.SwipeRefreshLayout//将要实现下拉功能的控件放在这个布局里面
              android:id="@+id/swipeRefreshLayout"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              app:layout_behavior="@string/appbar_scrolling_view_behavior">//这个属性之前放在RecycleView中，为了对应AppBarLayout的智能排版，由于RecyclerView放进了SwipeRefreshLayout中，所以把这个属性放进SwipeRefreshLayout中；
              <android.support.v7.widget.RecyclerView//这里将RecyclerView实现下拉刷新的功能
                  android:id="@+id/recyclerView"
                  android:layout_width="match_parent"
                  android:layout_height="match_parent" ></android.support.v7.widget.RecyclerView>
          </android.support.v4.widget.SwipeRefreshLayout>
  ~~~

- 然后设置下拉刷新的具体逻辑

  ~~~java
  swipeRefreshLayout =(SwipeRefreshLayout)findViewById(R.id.swipeRefreshLayout);//先加载到刷新布局
        swipeRefreshLayout.setColorSchemeResources(R.color.colorPrimary);//这是设置刷新进度条的颜色
  //接下来写刷新布局的触发刷新的逻辑实现      
  swipeRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener(){
              @Override
              public void onRefresh() {
                  refreshFruits();//具体实现写在这个代码里}
          });
  ~~~

  ​


#### 可折叠式标题栏：CollapsingToolbarLayout

- CollapsingToolbarLayout是一个作用于**Toolbar基础之上**的布局，由Design Support库提供的。继承至FrameLayout；

- CoolapsingToolbaLayout可是不能单独存在的，它在设计的时候就被限定只能作为AppBarLayout的直接子布局来使用，而AppBarLayout又必须是CoordinatorLayout的子布局。

  ~~~java
  <android.support.design.widget.CoordinatorLayout 	/*协调者布局*/		      xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      xmlns:tools="http://schemas.android.com/tools"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      tools:context="com.cnt.lbstest.FruitActivity">
      <android.support.design.widget.AppBarLayout/*智能标题栏布局*/
          android:id="@+id/appBar"
          android:layout_width="match_parent"
          android:layout_height="wrap_content">
          <android.support.design.widget.CollapsingToolbarLayout/*折叠布局*/
              android:id="@+id/collapsingToolbarLayout"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              app:contentScrim="@color/colorPrimary"/*指定CollapsingToolbarLayout在趋于折叠状态后的背景色，其实它折叠后就是一个普通的Toolbar*/
              android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
              app:layout_scrollFlags="scroll|exitUntilCollapsed">/*这一行是指定Toolbar的行为的，为了实现更加高级的Toolbar效果，因此需要将这个主题的指定提到上一层来*/
              <ImageView/*标题栏拉升时就显示图片，标题栏回去即折叠完成时，图片是不显示的，此时，标题栏就自然会显示出来，(可能因为折叠布局是帧布局)*/
                  android:layout_width="match_parent"
                  android:layout_height="match_parent"
                  android:id="@+id/collapsingImageView"
                  android:scaleType="center"
                  app:layout_collapseMode="parallax"/*这个属性是指定当前控件在折叠过程中的折叠模式，指定成parallax表示在折叠过程中会产生错位偏移，这也是图片应有的动作*//>
              <android.support.v7.widget.Toolbar
                  android:id="@+id/collapsingToolbar"
                  android:layout_width="match_parent"
                  android:layout_height="?attr/actionBarSize"
                  app:layout_collapseMode="pin"/*这个属性是指定当前控件在折叠过程中的折叠模式，指定成pin表示在折叠的过程中位置始终保持不变，这也应该是标题栏应有的动作*/>
              </android.support.v7.widget.Toolbar>
          </android.support.design.widget.CollapsingToolbarLayout>
      </android.support.design.widget.AppBarLayout>
      <android.support.v4.widget.NestedScrollView/*这个控件允许用滚动的方式来查看屏幕以外的数据，由于我们要实现滚动屏幕的时候标题栏的折叠，所以要让标题栏下的布局是可以滚动的，所以要用滚动布局，这个布局内部只允许放一个子布局*/
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          android:id="@+id/fruit_nestedScrollView"
          app:layout_behavior="@string/appbar_scrolling_view_behavior"/*在使用了AppToolbar后，处于标题栏下的控件就要添加这一个属性，才能配合实现AppToolbar的智能标题栏动作*/>
          <LinearLayout
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:orientation="vertical">
              <android.support.v7.widget.CardView/*卡片式布局*/
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:layout_marginTop="35dp"
                  android:layout_marginBottom="15dp"
                  android:layout_marginRight="15dp"
                  android:layout_marginLeft="15dp"
                  app:cardCornerRadius="4dp"
                  android:elevation="5dp">
                  <TextView
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:id="@+id/collapdingText"
                      android:layout_margin="10dp"/>
              </android.support.v7.widget.CardView>
         </LinearLayout>
      </android.support.v4.widget.NestedScrollView>
      <android.support.design.widget.FloatingActionButton
          android:id="@+id/fab"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          app:layout_anchor="@id/appBar"/*这个属性是为了浮动按钮设置一个锚点，将锚点设置为APPBarLayout，这样悬浮按钮就会以水果标题栏为锚点出现在水果标题栏的范围内，但当折叠完成后，按钮会随着消失，因为要只显示Toolbar标题栏，增大用户阅读的控件，不要让按钮占地方*/
          app:layout_anchorGravity="bottom|end"
          android:layout_margin="16dp"/>
  </android.support.design.widget.CoordinatorLayout>
  ~~~

  ~~~java
  Toolbar toolbar = (Toolbar) findViewById(R.id.collapsingToolbar);
  setSupportActionBar(toolbar);//使用Toolbar的标准用法，将它作为ActionBar显示，并启用HomeAsUp返回按钮
   ActionBar actionBar = getSupportActionBar();
          if (actionBar != null) {
              actionBar.setDisplayHomeAsUpEnabled(true); }
          collapsingToolbarLayout.setTitle(name);//设置折叠标题栏的标题，折叠后也会显示这个标题，不需要在Toolbar中设置标题栏了，这里即可；
          Glide.with(this).load(imageId).into(imageView);//这里设置折叠标题的图片
  ~~~


#### 充分利用系统状态栏空间

* 在Android5.0即API  21之前，我们无法对状态栏的背景或颜色进行操作，在5.0之后，就可以操作；

* 通过指定状态栏的颜色即可，设置成透明色即可，设置成别的颜色就是指定状态栏的纯色；

* 可以指定某一Activity的theme为特定的自定义主题即可；

* 为了兼容性，我们要这样操作，在res目录下新建一个values-21目录，然后在这个目录下新建一个styles.xml文件；再在这个文件中编写将状态栏的颜色修改的代码的一主题，需要透明状态栏的Activity就可以使用这一主题即可；之所以要这样做，是因为在安卓5.0之后系统才会去读取这个目录；

  ~~~java
  <resources>
      <style name="FruitActivityTheme" parent="AppTheme">//z在values-21目录中，创建拥有透明状态栏的主题，Activity要用时直接指定这个主题即可；
      <item name="android:statusBarColor">@android:color/transparent</item>
      //@android:color/transparent表示透明色，指定别的色，则状态栏变成纯色状态栏；
      </style>
  </resources>
  ~~~

  如果要将图片显示在状态栏和Toolbar，则要把照片的ImageView和装载着照片的所有上层控件布局指定一个属性

  ~~~java
  android:fitsSystemWindows="true"
  ~~~

  然后在AndroidManifest中将需要自定义状态栏的Activity的theme改成自定义的Theme即可；

  ~~~java
  android:theme="@style/FruitActivityTheme">
  ~~~

  但是如果是安卓5.0之前的系统就不会去识别values-21这个目录中的styles，那么此时theme指定了FruitActivityTheme的Activity找不到主题怎么办呢？问题很好解决，只要在系统本来就会去的valuew/styles.xml中设置一个同名FruitActivi即可，

  ~~~java
  <style name="FruitActivityTheme" parent="AppTheme"></style>
  ~~~

  ​

