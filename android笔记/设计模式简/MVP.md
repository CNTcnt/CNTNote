[TOC]

# MVP

* MVP 是 MVC 的一个演化版本，全称：Model-View-Presenter；
* MVP 有效地降低 View 的复杂度；避免业务逻辑被塞进View 中；解除了 View 与 Model 的耦合，同时带来了良好的可扩展性，可测试性；
* MVP 模式分离显示层和逻辑层，它们之间通过接口进行通信，降低耦合。理想化的 MVP 模式可以实现同一份逻辑代码搭配不同的显示界面，因为它们之间并不依赖于具体，而是依赖于抽象；这使得 Presenter 可以运用于任何实现了 View 逻辑接口的 UI ，使之具有更广泛的实用性，保障了灵活度；
* 为什么要用到 MVP 这个框架思想呢？其实，对于一个可扩展的，稳定的应用来说，我们需要定义分离各个层，主要是 UI 层，业务逻辑层和数据层。例如，随着产品的升级，UI 可能会被重新设计，此时 UI 发生变化，但如果你把业务逻辑代码耦合在 View 中，那么你就得去到原来的 View 中去抽离 业务逻辑代码，你看到头来你还是得去做这步工作，为什么不听老人言呢？
* MVP 没有一个标准的模式，实现方式多种，但其实，只要保证是通过 Presenter 将 VIew 和 Model  解耦和，降低类型复杂度，各个模块可以独立此时，独立变化，这就是正确的方向；

## 简述各部分

* View——用户界面：是指显示数据并且和用户交互的层。它含有一个 Presenter 成员变量。通常的做法是：View 实现一个逻辑接口，将View 上的操作转交给 Presenter 进行实现，最后 Presenter 调用 View 逻辑接口将结果返回给 View 元素去做 UI 显示；在安卓中，它们可以是一个 Activity，一个 Fragment，一个android.view.View 或者是一个 Dialog。
* Model——数据源层： 提供数据的存取功能。Presenter  需要通过 Model 层存储，获取数据，Model 就像一个数据仓库。比如封装了数据库 DAO 或 数据库接口 或者远程服务器的 api。
* Presenter——交互中间人：是从 Model 中获取数据并返回给 View 的层（View 和 Model 解耦），将业务逻辑从 View 角色上抽离出来 ，Presenter 还得负责处理后台任务。
* MVP 是一个将业务逻辑，后台任务和 activities/views/fragment 分离的方法，让它们独立于绝大多数跟生命周期相关的事件。这样应用就会变得更简单，整个应用的稳定性提高，代码也变得更短，可维护性增强；

## NavigationView源码分析

* 首先，NavigationView 不是使用经典的 MVP ，是一个变种，但是不影响我们学习；

* 顾名思义就是一个 导航View，视图菜单 ，其实就是要导航菜单，一般结合 抽屉布局：DrawerLayout 使用 Navigation ，当用户从屏幕的左边界向右滑动时，菜单视图就会显示出来；

* NavigationView 就是一个分为2个部分的 View，上半部分为头部，比如显示个头像什么鬼的，下半部分为菜单选项（菜单选项就是一个个类似的View）；你想想是不是用 RecycleView就能完成，其实它内部就是一个 RecycleView ，只是封装成 NavigationView 而已，封装了 子View  的添加和 子View 的点击事件，使用起来极为简单；

* 从 使用方法入手去分析源码

  ~~~java
  <android.support.design.widget.NavigationView
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          app:menu="@menu/slide_menu"//菜单布局
          app:headerLayout="@layout/slide_header"//头布局
            />
  ~~~

* 设置点击事件

  ~~~java
  public class NavigationActivity extends AppCompatActivity
      implements NavigationView.OnNavigationItemSelectedListener {
      
      navigationView.setNavigationItemSelectedListener(this);
      //。。。。。。。。。。。。
  	public boolean onNavigationItemSelected(MenuItem item) {
          // Handle navigation view item clicks here.
          int id = item.getItemId();
          if (id == R.id.nav_camera) {
              // Handle the camera action
          } else if (id == R.id.nav_gallery) {
          } else if (id == R.id.nav_slideshow) {
          } 
          mDrawerLayout.closeDrawer(GravityCompat.START);
          return true;
      }
      //。。。。。。。。。。。。
  }
  ~~~

* 用 MVP 来自定义一个 RecycleView ，Model 就是 子View 的集合，View 就是 NavigationView 本身啊，Presenter 来处理子View布局和处理子View 的点击事件即可；

* 源码

  ~~~java
  public class NavigationView extends ScrimInsetsFrameLayout {...}
  public class ScrimInsetsFrameLayout extends FrameLayout {...}
  //从上面可以看出，NavigationView 就是一个 FrameLayout
  ~~~

  ~~~java
  public class NavigationView extends ScrimInsetsFrameLayout {
  
      private final NavigationMenu mMenu;
      //Presenter
      private final NavigationMenuPresenter mPresenter = new NavigationMenuPresenter();
  
    //View的监听接口
      OnNavigationItemSelectedListener mListener;
      //看构造方法
    public NavigationView(Context context, AttributeSet attrs, int defStyleAttr) {
          super(context, attrs, defStyleAttr);
          // Create the menu
          mMenu = new NavigationMenu(context);
  	 //省略了很多代码
       //注册菜单的点击回调事件
          mMenu.setCallback(new MenuBuilder.Callback() {
              @Override
              public boolean onMenuItemSelected(MenuBuilder menu, MenuItem item) {
                  //给菜单设置点击回调事件，回调我们覆写的onNavigationItemSelected
                  return mListener != null && mListener.onNavigationItemSelected(item);
              }
  
              @Override
              public void onMenuModeChange(MenuBuilder menu) {}
          });
          mPresenter.setId(PRESENTER_NAVIGATION_VIEW_ID);
        //传入 menu 菜单数据参数传给 presenter 操作
          mPresenter.initForMenu(context, mMenu);
          mPresenter.setItemIconTintList(itemIconTint);
          if (textAppearanceSet) {
              mPresenter.setItemTextAppearance(textAppearance);
          }
          mPresenter.setItemTextColor(itemTextColor);
          mPresenter.setItemBackground(itemBackground);
          mMenu.addMenuPresenter(mPresenter);
        //1. View 只负责页面显示，调用 addView 即可，至于 View 的具体不关心，由 presenter 指定即可； 
          addView((View) mPresenter.getMenuView(this));
  
          if (a.hasValue(R.styleable.NavigationView_menu)) {
              //2. 看到这里，加载我们自定义的菜单文件
              inflateMenu(a.getResourceId(R.styleable.NavigationView_menu, 0));
          }
  
          if (a.hasValue(R.styleable.NavigationView_headerLayout)) {
              inflateHeaderView(a.getResourceId(R.styleable.NavigationView_headerLayout, 0));
          }
  
          a.recycle();
      }
      //可以看到上面，NavigationView 的业务逻辑都由 mPresenter 包办了，NavigationView 就负责view 层该负责的显示即可；分工明确；
  }
  1. 
  @Override
      public MenuView getMenuView(ViewGroup root) {
          //private NavigationMenuView mMenuView;这个 mMenuView 其实就是一个 ReycleView 
          //public class NavigationMenuView extends RecyclerView implements MenuView {}
          if (mMenuView == null) {
              mMenuView = (NavigationMenuView) mLayoutInflater.inflate(
                      R.layout.design_navigation_menu, root, false);
              if (mAdapter == null) {
                  //创建 RecycleView 的适配器
                  mAdapter = new NavigationMenuAdapter();
              }
              mHeaderLayout = (LinearLayout) mLayoutInflater
                      .inflate(R.layout.design_navigation_item_header,
                              mMenuView, false);
              mMenuView.setAdapter(mAdapter);
          }
          return mMenuView;
      }
  //再看 RecycleView 的适配器
  NavigationMenuAdapter() {
      //构造函数就准备数据源，现在先不分析这里
              prepareMenuItems();
  }
  
  2.
  //接下来加载我们自定义的菜单文件
  public void inflateMenu(int resId) {
          mPresenter.setUpdateSuspended(true);
      //拿到加载器加载，把 我们的数据源 NavigationMenu 传进去，解析菜单项资源
          getMenuInflater().inflate(resId, mMenu);
          mPresenter.setUpdateSuspended(false);
      //更新菜单视图；
          mPresenter.updateMenuView(false);
  }
  
  
  //下面先分析解析菜单项资源
  private MenuInflater getMenuInflater() {
          if (mMenuInflater == null) {
              //自定义的加载器，为什么要自定义呢，我们去看看
              mMenuInflater = new SupportMenuInflater(getContext());
          }
          return mMenuInflater;
  }
  // SupportMenuInflater 类，把菜单布局参数映射成 Java 对象
  public SupportMenuInflater(Context context) {
          super(context);
          mContext = context;
          mActionViewConstructorArguments = new Object[] {context};
          mActionProviderConstructorArguments = mActionViewConstructorArguments;
  }
  @Override
      public void inflate(int menuRes, Menu menu) {
          // If we're not dealing with a SupportMenu instance, let super handle
          if (!(menu instanceof SupportMenu)) {
              super.inflate(menuRes, menu);
              return;
          }
  
          XmlResourceParser parser = null;
          try {
              parser = mContext.getResources().getLayout(menuRes);
              AttributeSet attrs = Xml.asAttributeSet(parser);
  			//这里，解析菜单文件，我们去里面看看
              parseMenu(parser, attrs, menu);
          } catch (XmlPullParserException e) {
              throw new InflateException("Error inflating menu XML", e);
          } catch (IOException e) {
              throw new InflateException("Error inflating menu XML", e);
          } finally {
              if (parser != null) parser.close();
          }
      }
  private void parseMenu(XmlPullParser parser, AttributeSet attrs, Menu menu)
              throws XmlPullParserException, IOException {
          MenuState menuState = new MenuState(menu);
  
          int eventType = parser.getEventType();
          String tagName;
          boolean lookingForEndOfUnknownTag = false;
          String unknownTagName = null;
  
          // This loop will skip to the menu start tag
          do {
              if (eventType == XmlPullParser.START_TAG) {
                  tagName = parser.getName();
                  if (tagName.equals(XML_MENU)) {
                      // Go to next tag
                      eventType = parser.next();
                      break;
                  }
  
                  throw new RuntimeException("Expecting menu, got " + tagName);
              }
              eventType = parser.next();
          } while (eventType != XmlPullParser.END_DOCUMENT);
  
          boolean reachedEndOfMenu = false;
          while (!reachedEndOfMenu) {
              switch (eventType) {
                  case XmlPullParser.START_TAG:
                      if (lookingForEndOfUnknownTag) {
                          break;
                      }
  
                      tagName = parser.getName();
                      if (tagName.equals(XML_GROUP)) {
                          menuState.readGroup(attrs);
                      } else if (tagName.equals(XML_ITEM)) {
                          menuState.readItem(attrs);
                      } else if (tagName.equals(XML_MENU)) {
                          // A menu start tag denotes a submenu for an item
                          SubMenu subMenu = menuState.addSubMenuItem();
                          //我们去看一下这方法
                          //public SubMenu addSubMenuItem() {
              			//itemAdded = true;
                          //就是下面这步，把菜单数据存储到刚开始传进来的 数据源 menu 中
              			//SubMenu subMenu = menu.addSubMenu(groupId, itemId, itemCategoryOrder, itemTitle);
              			//setItem(subMenu.getItem());
              			//return subMenu;
          				//}
  
                          // Parse the submenu into returned SubMenu
                          parseMenu(parser, attrs, subMenu);
                      } else {
                          lookingForEndOfUnknownTag = true;
                          unknownTagName = tagName;
                      }
                      break;
  
                  case XmlPullParser.END_TAG:
                      tagName = parser.getName();
                      if (lookingForEndOfUnknownTag && tagName.equals(unknownTagName)) {
                          lookingForEndOfUnknownTag = false;
                          unknownTagName = null;
                      } else if (tagName.equals(XML_GROUP)) {
                          menuState.resetGroup();
                      } else if (tagName.equals(XML_ITEM)) {
                          // Add the item if it hasn't been added (if the item was
                          // a submenu, it would have been added already)
                          if (!menuState.hasAddedItem()) {
                              if (menuState.itemActionProvider != null &&
                                      menuState.itemActionProvider.hasSubMenu()) {
                                  menuState.addSubMenuItem();
                              } else {
                                  menuState.addItem();
                              }
                          }
                      } else if (tagName.equals(XML_MENU)) {
                          reachedEndOfMenu = true;
                      }
                      break;
  
                  case XmlPullParser.END_DOCUMENT:
                      throw new RuntimeException("Unexpected end of document");
              }
  
              eventType = parser.next();
          }
      }
  //NavigationMenu 中，addSubMenu 方法中把菜单数据放进传进来的中的NavigationMenu mMenu对象中，mMenu 对象中的 mItems 中
  @Override
      public SubMenu addSubMenu(int group, int id, int categoryOrder, CharSequence title) {
          final MenuItemImpl item = (MenuItemImpl) addInternal(group, id, categoryOrder, title);
          final SubMenuBuilder subMenu = new NavigationSubMenu(getContext(), this, item);
          item.setSubMenu(subMenu);
          return subMenu;
      }
  //NavigationMenu 的父类 MenuBuilder 
  protected MenuItem addInternal(int group, int id, int categoryOrder, CharSequence title) {
          final int ordering = getOrdering(categoryOrder);
  
          final MenuItemImpl item = createNewMenuItem(group, id, categoryOrder, ordering, title,
                  mDefaultShowAsAction);
  
          if (mCurrentMenuInfo != null) {
              // Pass along the current menu info
              item.setMenuInfo(mCurrentMenuInfo);
          }
  
          mItems.add(findInsertIndex(mItems, ordering), item);
          onItemsChanged(true);
  
          return item;
  }
  
  
  //现在分析 更新菜单视图的过程,在  inflateMenu() 方法中，刚刚说了要分析它的，上面第2点看
  mPresenter.updateMenuView(false);
  //不就是更新 RecycleView 的视图吗，那就更新数据源后再通知 recycelView 更新即可呀
  @Override
      public void updateMenuView(boolean cleared) {
          if (mAdapter != null) {
              mAdapter.update();
          }
      }
  
  public void update() {
              prepareMenuItems();
              notifyDataSetChanged();
  }
  ~~~

* 这里重新看回 Adapter 适配器的构造函数中调用的 prepareMenuItems() 方法

  ~~~java
  
  private void prepareMenuItems() {
              if (mUpdateSuspended) {
                  return;
              }
              mUpdateSuspended = true;
              mItems.clear();
      //RecycleView 数据源为 mItems
              mItems.add(new NavigationMenuHeaderItem());
  
              int currentGroupId = -1;
              int currentGroupStart = 0;
              boolean currentGroupHasIcon = false;
              for (int i = 0, totalSize = mMenu.getVisibleItems().size(); i < totalSize; i++) {
                  //从 mMenu 拿数据放到数据源 mItems 中，MenuItemImpl 就是 Model，由此推测 mMenu 就是 Model 的集合
                  MenuItemImpl item = mMenu.getVisibleItems().get(i);
                  if (item.isChecked()) {
                      setCheckedItem(item);
                  }
                  if (item.isCheckable()) {
                      item.setExclusiveCheckable(false);
                  }
                  if (item.hasSubMenu()) {
                      SubMenu subMenu = item.getSubMenu();
                      if (subMenu.hasVisibleItems()) {
                          if (i != 0) {
                              mItems.add(new NavigationMenuSeparatorItem(mPaddingSeparator, 0));
                          }
                          mItems.add(new NavigationMenuTextItem(item));
                          boolean subMenuHasIcon = false;
                          int subMenuStart = mItems.size();
                          for (int j = 0, size = subMenu.size(); j < size; j++) {
                              MenuItemImpl subMenuItem = (MenuItemImpl) subMenu.getItem(j);
                              if (subMenuItem.isVisible()) {
                                  if (!subMenuHasIcon && subMenuItem.getIcon() != null) {
                                      subMenuHasIcon = true;
                                  }
                                  if (subMenuItem.isCheckable()) {
                                      subMenuItem.setExclusiveCheckable(false);
                                  }
                                  if (item.isChecked()) {
                                      setCheckedItem(item);
                                  }
                                 // menuItem是菜单的 Java 对象，我们需要的是 View 对象，所以把 MenuItem 转化为 NavigationMenuTextItem 对象，添加到数据源中
                                  mItems.add(new NavigationMenuTextItem(subMenuItem));
                              }
                          }
                          if (subMenuHasIcon) {
                              appendTransparentIconIfMissing(subMenuStart, mItems.size());
                          }
                      }
                  } else {
                      int groupId = item.getGroupId();
                      if (groupId != currentGroupId) { // first item in group
                          currentGroupStart = mItems.size();
                          currentGroupHasIcon = item.getIcon() != null;
                          if (i != 0) {
                              currentGroupStart++;
                              mItems.add(new NavigationMenuSeparatorItem(
                                      mPaddingSeparator, mPaddingSeparator));
                          }
                      } else if (!currentGroupHasIcon && item.getIcon() != null) {
                          currentGroupHasIcon = true;
                          appendTransparentIconIfMissing(currentGroupStart, mItems.size());
                      }
                      NavigationMenuTextItem textItem = new NavigationMenuTextItem(item);
                      textItem.needsEmptyIcon = currentGroupHasIcon;
                      mItems.add(textItem);
                      currentGroupId = groupId;
                  }
              }
              mUpdateSuspended = false;
  }
  ~~~

* 总结一下这些组件的逻辑：NavigationView 就是 View 角色，通过 Presenter 处理解析，构造各类菜单项的业务逻辑，将自身从复杂的逻辑中解耦出来，因此，NavicationView 的职责就减轻，而 Model 就是 MenuItemImpl；

## 简单实现一个 MVP 例子

* 需求：界面一个 recycleView 和 进度条，加载文章数据时显示进度条，加载完之后隐藏；

* 首先是 界面接口，界面需要显示文章，显示进度条，隐藏进度条；(文章的对应的 javabean 就不写了)

  ~~~java
  public interface ArticleViewInterface {
      public void showArticles(List<Article> articles){}
      public void showLoading(){}
      public void hideLoading(){}
  }
  ~~~

* 接下来是 Model 层，Model 也给出接口，子类实现即可,模型层需要储存数据，向外提供数据

  ~~~java
  public class ArticleModelImpl implements ArticleModel{
      List<Article> list = new  Array<Article>();
      public void saveArticles(List<Article> articles){
          this.list = articles;
      }
      //从缓存中获取数据方法参数设置为回调接口，调用时直接编写获取数据完成后的操作即可
      public void loadArticlesFromCache(DataListener<List<Article>> listener){
  		list.onComplete(list);
      }
  }
  ~~~

* 接下来自然是  Presenter 层，负责获取文章数据再显示界面上；

  ~~~java
  public class ArticlePresenter{
      //首先让 presenter层持有 View 层和model 层的引用；
      private ArticleViewInterface articleView;
      private ArticleModel articleModel  = new ArticleModelImpl();
      //获取文章的工具类
      private ArticleApi articleApi = new  ArticleApiImpl();
      public ArticlePresenter(ArticleViewInterface view){
          this.articleView = view;
      }
      //获取文章数据
      public void fetchAticle(){
          //第一步先界面显示加载进度条
          articleView.showLoading();
          //接下来就是利用工具类获取数据咯
          articleApi.fetchAticles(new DataListener<List<Article>>(){
              public void onComplete(List<Article> list){
                  //获取完后先界面显示
                  articleView.showArticles(lsit);
                  articleView.hideLoading();
                  //然后保存在 model 中；
                  articleModel.saveArticles(list);
              }
          });        
      }
      public void loadArticlesFromDB(){
          articleModel.loadArticlesFromCache(new DataListener<List<Article>>(){
              public void onComplete(List<Article> list){
                  articleView.showArticles(lsit);
              }
          });
      }
  }
  ~~~

* 接下来是 Activity 实现 View  接口，并与 Presenter 产生联系；

  ~~~java
  public class ArticlesActivity extends Activity  implements ArticleViewInterface{
      //省略不重要的代码
      List<Article> articleList = new Array<Article>();
      
      @Override
      protected void onCreate(Bundle saveInstanceState){
          super.onCreate();
          setContentView(R.layout.activity_main);
          initViews();
          //此时 activity 将自身传递给 Presenter ，而 Presenter 中又有 ArticleModel 类型的成员变量，所以，Model-View-Presenter 形成；
          mPresenter = new ArticlePresenter(this);
          //业务就是 presenter 完成；
          mPresenter.fetchArticles();
      }
      
      @Override
      public void showArticles(List<Article> articles){
          articleList.addAll(articles);
          mAdapter.notifyDataSetChanged();
      }
      /*进度条代码省略
      public void showLoading(){}
      public void hideLoading(){}*/
  }
  ~~~



## MVP 与 Activity，Fragment 的生命周期

* MVP 很多优点，如易于维护，易于测试，复用性高，健壮稳定，易于扩展；

* 在平时的编码中，MVP 中的 View 角色一般由 Activity 或 Fragment 扮演，此时，由于业务逻辑是由 Presenter 处理，耗时操作比如网络请求，又由于 Presenter 持有了  Activity 或 Fragment 的强引用，如果在请求结束前，Activity 被销毁，那么由于请求还未结束，导致 Presenter 一直持有 Activity 对象，使得 Activity 对象无法被回收，此时发生了内存泄漏。那么如何处理这个问题呢？

* 通过弱引用和结合 Activity,Fragment的生命周期来解决这个问题；

  1. 首先让  Presenter 不持有的  Activity 或 Fragment 的强引用，而为弱引用：被弱引用关联的对象只能生存到下一次垃圾回收之前，垃圾收集器工作之后，无论当前内存是否足够，都会回收掉只被弱引用关联的对象；

     ~~~java
     public abstract class BasePresenter<T>{
         //View 层的引用
         protected Reference<T> mViewRef;
         
         public void attachView(T view){
             //转化为 弱引用对象；
             mViewRef = new WeakReference<T>(view);
         }
         public void detachView(){
             if(mViewRef != null){
                 mViewRef.clear();
                 mViewRef = null;
             }
         }
         public T getView(){
             //未被处理时，WeakReference#get方法可以返回目标对象的引用
             return mViewRef.get();
         }
         //具体的，GC时会清除掉所有WeakReference中对目标对象的引用，即将referent字段置为null，这样会导致WeakReference#get方法将返回null
         public boolean isViewAttached(){
             return mViewRef!=null && mViewRef.get() !=null;
         }
     }
     ~~~

  2. 再结合 Activity,Fragment的生命周期，创建出一个可复用的 Activity 的抽象类；以下创建的MVPBaseActivity 类封装了 Presenter 与 View 联系的代码

     ~~~java
     //View 层要实现 View 层的接口，Presenter 层的接口；自此，这些关乎内存泄漏的问题就会被抽象到这个MVPBaseActivity
     public abstract class MVPBaseActivity<V,T extends BasePresenter<T>> extends Activity{
         protected T mPresenter;
         @SuppressWarnings("unchecked")
         @Override
         protected void onCreate(Bundle savedInstanceState){
             super.onCreate(savedInstanceState);
             mPresenter = createPresenter();
             //在 onCreate 方法中 presenter 与 view 层发生联系
             mPresenter.attachView((V)this);
         }
         @Override
         protected void onDestroy(){
             super.onDestroy();
             //在 onDestroy 中 presenter 与 view 层断开联系
             mPresenter.detachView();
         }
         //使用时直接实现 createPresenter()方法即可，无序理会刚刚提到的内存泄漏问题了；
         protected abstract T createPresenter();
     }
     //我看到这里就想，其实结合 Activity,Fragment的生命周期就可以解决问题了，为什么还要使用弱引用呢？书里后面给了我答案，就是其实某些特殊情况下， onDestroy()并不会被调用，使用 弱引用是为了保证不会造成内存泄漏，加多一层保障；
     ~~~

  3. 我们实际使用时就很方便

     ~~~java
     public class  ArticlesActivity extends MVPBaseActivity<ArticleViewInterface,ArticlePresenter> implements ArticleViewInterface{
         @Override
         protected ArticlePresenter createPresenter(){
             return new ArticlePresenter();
         }
         @Override
         protected void onCreate(Bundle savedInstanceState){
             super.onCreate(savedInstanceState);
             setContentView(R.layout.activity_main);
             //直接使用即可，太舒服了，不用去理会presenter与view 层之间的联系；
             mPresenter.fetchArticles();
         }   
     }
     ~~~

## 小结

* MVP 是开发过程中非常值得推荐的架构模式，他的思想非常好的体现了面向对象的设计原则，即 抽象，单一职责，最小化，低耦合；
* MVP 可以将各组件进行解耦，并且带来良好的可扩展性，可测试性，稳定性，可维护性，同时使得每个类型的职责相对单一，简单；

