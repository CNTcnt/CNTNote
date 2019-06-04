#### 全局获取Context的技巧

- 我们在很多地方都会要用到Context，在Activity中，由于Activity本身就是一个Context，所以没有对我们形成困扰；但是，当我们的逻辑代码不是写在Activity中时，当我们需要到Context时，这个怎么办呢？
- Android提供了一个Application类，当应用程序启动的时候，系统会自动将这个类进行初始化。也就是说，这个类可以管理一些全局的状态信息，那么我们可以自定义一个继承Application的类来获取全局Context，那么问题就解决了；
- 当然不是定义了这个类，系统就只找你这个类，系统是找给它指定的Application类，所以我们还要指定即告知系统应该要初始化我们自定义的类；在AndroidManifest中指定

1. ~~~java
   public class MyApplication extends Application {
       private static Context context;
       @Override
       public void onCreate() {
           context = getApplicationContext();//获取一个应用程序级别的Context来使用}
       public static android.content.Context getContext() {//获取Context的方法
           return context;}
   ~~~

2. ~~~java
   <application
           android:name="com.cnt.lbstest.MyApplication"//在application中指定即可，但每个项目只能配置一个Application
   ~~~

3. 需要注意的是，当我们使用了外部的包，外部的包中也可以要用到Context，那么此时外部的包中也有一个自定义的Application要来指定，例如之前的LitePal，我们需要指定

   ~~~java
   <application
           android:name="org.litepal.LitePalApplication"
   ~~~

   但是，当我们自己的应用已经配置了一个Application，那么，此时应该怎么办呢？对于这种情况，LitePal有一个很简单的解决方案，那就是我们自己的Applicatio的onCreate()n中去调用LitePal的初始化方法就可以；

   ~~~java
   public void onCreate() {
           context = getApplicationContext();
   		LitePal.initialize(context);}
   ~~~

   ​