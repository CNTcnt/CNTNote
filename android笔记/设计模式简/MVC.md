[TOC]

# MVC

* 参考至《Android 源码设计模式解析与实战》


* Model-View-Controller：数据模型-视图-控制器，是一个经典的框架模式（历史悠久）；
* 优点：代码的维护性更好；为了实现 Model 的复用；代码可读性大幅提高，逻辑清晰；
* 缺点：会增加代码量，设计要花点心思；好的，这缺点就是鸡蛋里挑骨头的感觉；
* 为了让你对各个部分有个初步的了解，引用 Reenskaug（创造者） 的介绍，下面是典型的 MVC 的各部分介绍，注意，典型的 MVC 并没有大规模的普及，所以不是你现在印象中的 MVC ，你现在印象中的 MVC 是二十一世纪的 MVC 呀！！！毕竟典型的 MVC 是1979 年被首次提出来，几十年前呀，要是 MVC 没点进化你现在都不可能听过 MVC 这个框架思想；典型的 MVC 的各部分介绍如下：
  * Model：Model 可以是一个独立的对象，也可以是一系列对象的集合体，可提供数据的对象；
  * View：View 是 Model 中一些重要数据的 UI 体现；
  * Controller：Controller 用于连接 user 和 系统，这句话有点难理解，但是结合一下 MVC 刚提出来的背景就可以理解，当时没有什么事件驱动系统，做一个应用都需要自己去实现一套事件输入输出系统，自己去从零绘制各种控件，然后再将对应的事件分配给对应的 View 去处理，所以，你现在理解 Controller 用于连接 user 和 系统这句话的意思了没有？
* 现在由于 GUI （图形用户界面）操作系统的流行，让典型的 MVC 陷入了一个比较尴尬的境界，GUI 一般都会提供一套完善的 View 框架，而这套框架中，本身就集成了捕获用户操作事件的行为，比如监听鼠标，键盘或者触摸事件，现在你看上面 Controller 的定义，你问它尴不尴尬，工作被抢了咋子搞哦
* 如今我们主要使用的是改进后的MVC，而经典MVC水土不服；

## 典型的 MVC 

* 先说一句，典型的 MVC 并不完美，也不适用于较为复杂的逻辑，我认为是有些部分间的耦合较高，功能不够专一明确；等等写个例子就很容易理解；

* Model：用来保持程序的数据状态，比如 bean, 数据储存，网络请求等；还会通过某种事件机制（比如观察者模式）通知 View 状态的改变来让 View 更新，Model 还会接收来自 Controller 的事件，Model 也会允许 View 查询相关数据以显示数据状态；看到了没有，View 和 Controller 都可以直接对 Model 有点想法；

* Controller ： 控制器由 View 根据用户行为触发并响应来自 View 的用户交互，Controller 并不关心 View 如何展示，Controller 只是去通过修改 Model 并由 Model 的事件机制来触发 View 的刷新；也就是 Controller 是间接地去改变 View 的状态，不是直接，Controller 只需要去关心事务及数据Model 即可，事务代码与界面展示解耦；

* 下面用文字描述简单的写个例子；需求：一个界面2个按钮，一个按钮用于加载图片，一个按钮用于清除图片（用设置空白页代替）；

  ~~~java
  //你是不是首先想到以下的做法
  main{
    1.视图布局包括2个按钮；
    2.为2个按钮注册点击事件，加载图片(数据)和清除图片的业务流程直接写在点击事件内   
  }
  //这样做只是一时之爽，几分钟做好任务，但是没有为后来的需求及维护考虑，把所有代码高度耦合地写在一个函数内对后期极不负责任；比如叫你修改一下某个业务，你就要先去找到用到这些业务的地方，检查每个view 的点击事件，但是 view 的工作只是展示业务成果数据呀，你还把业务丢给他处理，view 责任混乱，体积臃肿；

  //此时用 MVC 来做上述事情；
  1. 把图片数据抽象成 Model 类（只关注业务数据）内含image对象，但是类中还得提供监听接口给 View 以便让 Model 对象数据变化时刷新 View；（观察者模式即可实现）
  2. View 中包括了一个ImageView和2个按钮；View 实现 Model 监听接口，跟 Model 对象拿数据呈现（只关注 Model 数据呈现），提供按钮一的点击事件接口（加载图片）和按钮二点击事件接口（清除图片）；你想想，view 获取到点击事件后，自己独自处理吗？当然不是，view 应该只关注展示数据，点击事件应该由业务控制器 Controller 指定具体的事件呀；
  3. 再把业务封装成 Controller ，为 View 中的按钮指定业务事件；业务事件为加载图片和清除图片，这些应该由 PictureModel 来提供，Controller 通过改变 PictureModel 对象的数据间接刷新 View 的展示；

  main(){
    Model model = new Model();
    View view = new View(model);//可以看出，典型的 MVC 中 Model 和 view 存在一定的耦合，
    model.setOnStateChangeListener(view);//可以看出，典型的 MVC 中 Model 和 view 存在一定的耦合，这种耦合其实并不适用于如今的 GUI 系统；
    new Controller(model,view);
  }
  ~~~

## 流行的MVC

* 真正让 MVC 流行起来的原因是由于 Web 的出现，在 web 时代，基于 Http  的web 使用的是 Request/Response 模型，没有请求就没有回应，在这种情况下，你想想，典型的MVC 中Model数据改变时就主动去通知 View 去改变（即想要不发起 http 请求），现在在 web 中实现不了这种操作呀；那么，典型的 MVC 遇到了人生中最大的坎；

* 为了解决这个问题，Model2 的概念首次被提及，马上被作为一种新的 MVC 框架；Java 使用一套自己成熟的系统，将 JavaBean ，jsp ，servlet 对应 MVC 3个部分；也改进了事件流向，当用户作出交互请求时，把这个交互事件交给 Controller ，Controller 根据实际情况完成对 Model 的操作，Model 将自身改变的结果告诉 Controller ，再由 Controller 传递给 View 让 View 反馈给用户；通过上面的介绍，你有没有发现，Controller 真正地变成一个中间人了，View 和 Model 没有了耦合，对，很爽！！事件流向为：Controller->Model->Controller->View

* 缺点：日积月累下 Controller 的职责很重量级；

* 再随着移动操作系统的兴起，MVC 也进一步演化，由于 GUI 系统本身就带有一套强大的 控件系统，这些控件都有捕获交互事件甚至处理交互事件的能力；那么，Controller 的地位就显得很尴尬，本身捕获用户事件理应单独放在 Contoller 中处理，但是在 GUI 系统中特别是移动系统中，view 和事件的关联性实在太强，甚至到了不可分割的地步，gg ，难道要抛弃 MVC 了吗？其实，既然你 View 要捕获交互就让你捕获呀，把 View 和原本 Controler 负责用户交互的那部分打包成一个新 View ，把其他 Controller 负责的部分再打包成一个新的 Controller 即可；这样做了之后，MVC 中事件流向就由 View 处理；事件流向：View->Controller->Model->Controller->View；其实这种结构你看看，是不是和上面的典型的 MVC 有太大的区别了，但是，这种结构的 MVC 的影响却远远大于典型的 MVC ；

* 举个例子：还是上面那个例子

  ~~~java
  1. View 只关注数据的展示，不再去监听 Model 的变化，只需提供接口：改变需要显示的图片即可；
  2. Controller 则承担起监听 Model 状态改变的大梁，当 Model 改变时通知 Controller ，再由 Controller 去决定刷新 View ；
  main(){
    Model model = new  Model();
    View view = new View();
    new  Controller(model,view);
  }
  ~~~



## MVC 在 Android 中的实现

* 其实一开始在 Android 编写代码的时候，我们已经在不知情的情况下用到了 MVC，不信你想想，把 XML 布局和 View 类当作 View 层；Model 则由相关数据操作类承担，比如 JavaBean；而我们是不是把业务代码写在 Activity 上，那么这个 Activity 是不是就是对应 Controller ，是不是恍然大悟！！而在开发中，运用到 MVC 思想的地方非常多，比如最经典的 ListView 与 Adapter ，ListView  看作 View 层，Model 层就由我们外部需要提供的实体类数据，而Controller 就是 Adapter ，你想想，业务逻辑是不是在 Adapter 中。像这种例子还有非常多，没有最好的框架思想，只有适合的才是最好的！

* 继续上面那个例子，在 Android 的实现，使用 流行的MVC；View 即是 XML 文件，Model 为图片数据，提供监听接口由 Activity 实现，Activity 即是那个 Controller；

* view 

  ~~~xml
     <Button
          android:id="@+id/image_load_button"
          android:layout_width="match_parent"
          android:layout_marginTop="10dp"
          android:gravity="center"
          android:layout_height="60dp"
          android:text="image_load"/>
      <Button
          android:id="@+id/image_clean_button"
          android:text="image_clean"
          android:gravity="center"
          android:layout_marginTop="10dp"
          android:layout_width="match_parent"
          android:layout_height="60dp"/>
      <ImageView
          android:id="@+id/image"
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          />
  ~~~

* model

  ~~~java

  /**
   * Created by cnt on 2018/10/30.
   */

  public class ImageModel {
      private Handler handler = new Handler(Looper.getMainLooper());

    //监听接口，由 Activity 实现，并监听 Model 的数据变化
      private OnStateChangeListener mOnStateChangeListener;

      private Bitmap mBitmap;
      private Context mContext;

      public ImageModel(Context context){
          mContext = context;
      }

      public Bitmap getmBitmap() {
          return mBitmap;
      }

      void setOnStateChangeListener(OnStateChangeListener onStateChangeListener){
          this.mOnStateChangeListener = onStateChangeListener;
      }

      public void loadImage(){//这里可以设置图片链接参数，我懒了不想加
          new Thread(new Runnable() {
              @Override
              public void run() {
                  //加载图片一般由网络上加载，耗时的任务要在子线程
                  Drawable vectorDrawable = mContext.getDrawable(R.drawable.ic_launcher_background);
                  mBitmap = Bitmap.createBitmap(vectorDrawable.getIntrinsicWidth(),
                          vectorDrawable.getIntrinsicHeight(), Bitmap.Config.ARGB_8888);
                  Canvas canvas = new Canvas(mBitmap);
                  vectorDrawable.setBounds(0, 0, canvas.getWidth(), canvas.getHeight());
                  vectorDrawable.draw(canvas);

                  handler.post(new Runnable() {
                      @Override
                      public void run() {
                          if(null != mOnStateChangeListener){
                            //此时外部调用本方法，Model中数据 bitmap 发生改变，就要去通知 Controller，Controller 已经是观察者了，观察到数据发生改变后想做什么就是什么，Model 不用去干涉 ，只需通知 Activity 即可； 
                              mOnStateChangeListener.OnStateChange(mBitmap);
                          }
                      }
                  });
              }
          }).start();
      }

      public void cleanImage(){
          mBitmap = null;

          handler.post(new Runnable() {
              @Override
              public void run() {
                  if(null != mOnStateChangeListener){
                      mOnStateChangeListener.OnStateChange(mBitmap);
                  }
              }
          });
      }
    //监听接口
      public interface OnStateChangeListener{
          void OnStateChange(Bitmap image);
      }
  }

  ~~~

* Controller

  ~~~java
  //实现 ImageModel.OnStateChangeListener 接口，并注册监听 ImageModel 的数据变化
  public class MainActivity extends AppCompatActivity implements ImageModel.OnStateChangeListener {

      @BindView(R.id.image_load_button)
      Button image_load_button;
      @BindView(R.id.image_clean_button)
      Button image_clean_button;
      @BindView(R.id.image)
      ImageView imageView;

      ImageModel mImageModel;
    
    	@Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          ButterKnife.bind(this);

          mImageModel = new ImageModel(this);
        //注册监听者，Activity 去监听 mImageModel 
          mImageModel.setOnStateChangeListener(this);
    }
    @Override
    public void OnStateChange(Bitmap image) {   
          imageView.setImageBitmap(image);
    }
    @OnClick({R.id.image_load_button, R.id.image_clean_button})
    public void onViewClicked(View view) {
          switch (view.getId()) {
              case R.id.image_load_button:
                  mImageModel.loadImage();
                  break;
              case R.id.image_clean_button:
                  mImageModel.cleanImage();
                  break;
          }
    }
  }
  ~~~

## 总结一下

* MVC 是一种框架模式，不是设计模式，不过 GOF 把 MVC 看作是 观察者模式，策略模式，组合模式的混合体，核心其实就是观察者模式，也就是一个基于发布/订阅者模型的框架；
* 通常来说，框架是对代码的重用，设计模式通常是对设计的重用，简单地来说，就是框架面向一系列相同行为代码的复用，而设计模式则面向一系列相同结构代码的复用；
* 框架是大智慧，用来对软件设计进行分工；设计模式是雄安技巧，对具体问题提出解决方案，以提高代码复用率，降低耦合度；



