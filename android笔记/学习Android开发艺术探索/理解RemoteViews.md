[TOC]



# 理解RemoteViews

## 简述

* 顾名思义，reMoteView就是一种远程view,即可以在其他进程中显示（这个进程其实就是系统的SystemServer进程），为了能够更新它的界面，remoteViews提供了一簇基础的操作于跨进程更新它的界面；

## remoteViews的应用

* 在实际开发中，主要用在通知栏和桌面小部件；通知栏不用说就是用NotificationManager的notify方法来实现；而桌面小部件则是通过AppWidgetProvider来实现（AppWidgetProvider本质上时一个广播）；
* 通知栏和小部件分别由NOtificationManager和AppWidgetManager管理，而NOtification和AppWidgetManager通过Binder分别和SystenServer中的NotifacationManagerservice以及AppwidgetService进行通信；由此可见，通知栏和小部件的view实际上是在notificationManagerService以及AppWidgetService中被加载的，而它们运行在系统SystemServer中,这就和我们的进程构成了跨进程通信的场景；
* 由于跨进程更新界面，view在更新界面时无法向在Activity中那样去直接更新view，remoteViews提供了一系列set方法，并且这些方法只是view全部方法的子集，另外remoteView支持的view类型也是有限的（自定义view和一些view不行）;

### 通知栏

~~~java
//直接给代码好理解,这里是通知栏的代码
RemoteViews remoteViews = new RemoteViews(getPackageName(),R.layout.layout_notification);
//就是这里与在Activity中那样去直接更新view不同；并且只能这么来更新view
remoteViews.setTextViewText(R.id.msg,"文本内容");
remoteViews.setImageViewResource(R.id.icon,R.drawable.icon);
//控件的点击事件要用PendingIntent来搞
//关于PendingIntent，它表示的是一种待定的Intent，这个Intent中所包含的意图必须由用户来触发；
PendingIntent pendingIntent = PendingIntent.getActivity(this,0,new Intent(this,DemoActivity2),PendingIntent.FLAG_UPDATE_CURRENT);
remoteViews.setOnClickPendingIntent(R.id.open_activity2,pendingIntent);

Notification notification = new NOtification();
notification.contentView = remoteViews;
(NotifacationManager)getSystemService(COntext.NOTIFICATION_SERVICE).notify(2,notification);
~~~

### remoteViews在桌面小部件的应用

AppWidgetProvider是Android中提供的同于实现桌面小部件的类，其本质是一个广播，即BroadcastReceiver；

要点

1. 定义小部件界面

2. 定义小部件配置信息

   在res/xml/下新建appwidget_provider_info.xml，名称随意选择；

   ```java
   <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
     //initalLayout就是指小工具所使用的初始化布局
     android:initialLayout="@layout/widget"
     //minHeight和minWidth定义小工具的最小尺寸和最大尺寸
     android:minHeight="84dp"
     android:minWidth = "64dp"
     //updatePeriodMillis定义小工具的自动更新周期，毫秒为单位
     android:updatePeriodMillis = "86400000">
   </appwidget-provider>
   ```

3. 定义小部件的实现类（继承AppWidgetProvider）

   ​

   ```java
   public class MyAppWidgetProvider extends AppWidgetProvider {
       public static final String TAG = "MyAppWidgetProvider";
       public static final String CLICK_ACTION = "com.example.remoteviews";

       public MyAppWidgetProvider() {
           super();
       }

       @Override
       public void onEnabled(Context context) {
           super.onEnabled(context);
           Toast.makeText(context, "第一次添加小部件会调用我", Toast.LENGTH_SHORT).show();
       }

       @Override
       public void onDisabled(Context context) {
           super.onDisabled(context);
           Toast.makeText(context, "最后一个小部件已被删除！", Toast.LENGTH_SHORT).show();
       }

       @Override
       public void onDeleted(Context context, int[] appWidgetIds) {
           super.onDeleted(context, appWidgetIds);
           Toast.makeText(context, "小部件已被删除！", Toast.LENGTH_SHORT).show();
       }
   //当广播到来以后，AppWidgetProvider会自动很具广播的Action通过onReceive()方法来自动分发广播；也就是根据情况来调用几个方法:onUpdate(),onEnabled(),onDisabled(),onDeleted()
       @Override
       public void onReceive(final Context context, Intent intent) {
           super.onReceive(context, intent);
           Log.i(TAG, "onReceive:action=" + intent.getAction());
           // 这里判断自己的action，做自己的事情
           if (intent.getAction().equals(CLICK_ACTION)) {
               Toast.makeText(context, "我在旋转", Toast.LENGTH_SHORT).show();
               new Thread(new Runnable() {
                   @Override
                   public void run() {
                       Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(),
                               R.drawable.ic_launcher);
     				  AppWidgetManager appWidgetManager=AppWidgetManager.getInstance(context);				          
                       for (int i = 0; i < 37; i++) {
                           float degree = (i * 10) % 360;
                           RemoteViews remoteViews = new RemoteViews(context.getPackageName(),
                                   R.layout.widget);
                           remoteViews.setImageViewBitmap(R.id.iv_img,rotateBitmap(context, 					   bitmap, degree));

                           Intent intent = new Intent();
                           intent.setAction(CLICK_ACTION);
                           PendingIntent pendingIntent = PendingIntent.getBroadcast(
                                   context, 0, intent, 0);

                           remoteViews.setOnClickPendingIntent(R.id.iv_img, pendingIntent);
                           appWidgetManager.updateAppWidget(
                                   new ComponentName(context, MyAppWidgetProvider.class), 						  	  remoteViews);
                           SystemClock.sleep(30);
                       }
                   }
               }).start();
           }
       }
       /**
        * 每次桌面小部件被添加时或者更新时都调用一次该方法,小部件的更新时机由配置文件中的updatePeriodMillis来指定；
        */
       @Override
       public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[]  appWidgetIds) {
           super.onUpdate(context, appWidgetManager, appWidgetIds);
           Toast.makeText(context, "每次更新小部件会调用我！", Toast.LENGTH_SHORT).show();

           final int counter = appWidgetIds.length;
           Log.i(TAG, "counter = "+ counter);
           for (int i = 0; i < counter; i++) {
               int appWidgetId = appWidgetIds[i];
               onWidgetUpdate(context, appWidgetManager, appWidgetId);
           }
       }

       /**
        * 桌面小部件更新
        * @param context
        * @param appWidgetManager
        * @param appWidgetId
        */
       private void onWidgetUpdate(Context context, AppWidgetManager appWidgetManager, int appWidgetId) {
           Log.i(TAG, "appWidgetId = " + appWidgetId);
           RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.widget);

           // 桌面小部件单击事件发送的Intent广播
           Intent intent = new Intent();
           intent.setAction(CLICK_ACTION);
           PendingIntent pendingIntent = PendingIntent.getBroadcast(context, 0, intent, 0);
           remoteViews.setOnClickPendingIntent(R.id.iv_img, pendingIntent);
           appWidgetManager.updateAppWidget(appWidgetId, remoteViews);
       }

       /**
        * 旋转
        * @param context
        * @param bitmap
        * @param degree
        * @return
        */
       private Bitmap rotateBitmap(Context context, Bitmap bitmap, float degree) {
           Matrix matrix = new Matrix();
           matrix.reset();
           matrix.setRotate(degree);
           Bitmap tmpBitmap = Bitmap.createBitmap(bitmap, 0, 0,
                   bitmap.getWidth(), bitmap.getHeight(),
                   matrix, true);
           return tmpBitmap;
       }
   }
   ```

   ​

4. 在AndroidManifest.xml中声明小部件

   ```java
    <receiver android:name=".MyAppWidgetProvider" >
               <meta-data
                   android:name="android.appwidget.provider"
                     //resource是小部件的配置信息
                   android:resource="@xml/appwidget_provider_info" />
               <intent-filter>
                     //这个action用于识别小部件的单击行为
                   <action android:name="com.example.remoteviews" />
                     //这个action作为小部件的标识必须存在，是系统规范
                   <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
               </intent-filter>
    </receiver>
   ```

## remoteViews的内部机制(以下copy和修改而来)

* 首先remoteViews会通过binder传递到systenServer进程中，因为RemoteViews实现了Parcelable接口 所以它可以跨进程传输(跨进程传输其实就是序列化和反序列化的过程。。。) 系统会根据RemoteViews中的包名等信息去获取该应用中的资源，然后通过LayoutInFlater加载RemoteViews中的布局文件。
* 在SystemServer进程中加载后的布局文件是一个普通的view，只不过相对于我们的进程他是一个remoteViews而已。接着系统就会对view执行一系列界面更新任务，这些任务就是我们之前通过setXXX()来提交的；这里可以看出，我们在代码中setXXX()方法对view所做的更新并不是立即执行的，在remoteView内部会记录所有的更新操作，具体的执行时间要等到remoteViews被加载以后才能执行，这样remoteViews才可以在SystemServer进程中显示了；
* 所以，我们需要更新remoteViews时，我们需要调用一系列set方法，然后通过notificationManager和AppWidgetManager来***提交更新任务***，之后的具体更新操作时在SystemServer中完成的；

**RemoteViews支持的layout和View：**

```java
//layout：  
FrameLayout  LinearLayout  RelativeLayout  GridLayout
//View：
AnalogClock button Chronometer ImageButton ImageView ProgressBar TextView ViewFlipper ListView GridView StackView AdapterViewFlipper ViewStub 
```

上述就是RemoteViews所支持的所有layout和View 如果使用了这些之外的View 将会抛出异常 表明不可使用该View

**RemoteViews设置view内容的方法：**

```
由于RemoteViews没有findViewById的方法  因为它是远程的View,即使findViewById(),我们也不知道远程app的资源文件id  所以如果想要更新View的内容 就要使用RemoteViews提供的一系列set方法  如下 
```

| 方法名                                      | 作用                                       |
| ---------------------------------------- | ---------------------------------------- |
| setTextViewText(int viewId,CharSequence text) | 设置TextView的文本内容 第一个参数是TextView的id 第二个参数是设置的内容 |
| setTextViewTextSize(int viewId,int units,float size) | 设置TextView的字体大小 第二个参数是字体的单位              |
| setTextColor(int viewId,int color)       | 设置TextView字体颜色                           |
| setImageViewResource(int viewId,int srcId) | 设置ImageView的图片                           |
| setInt(int viewId,String methodName,int value) | 反射调用View对象的参数类型为Int的方法 比如上述的setImageViewResource的方法内部就是这个方法实现 因为srcId为int型参数 |
| setLong setBoolean                       | 类似于setInt                                |
| setOnClickPendingIntent(int viewId,PendingIntent pendingIntent) | 添加点击事件的方法                                |

* 上述是常用的一些RemoteViews的方法 还有很多没有列举 但是都是差不多的调用方式 大部分是通过反射调用View的方法


* 这个过程大体是这个样子的: 首先RemoteViews每调用一个set方法都会向自身添加一个Action， Action中封装了一些处理布局的方法 ，也是序列化对象 ,之后当RemoteViews需要加载的时候会调用自身的apply方法，这个方法会遍历所有的action一个一个执行action的apply方法，在action的apply方法中会反射调用View的方法 从而实现布局的更新；


* 首先我们要知道这是进程间通信 所以一般是基于Binder实现的 在使用RemoteViews的时候 它会通过Binder传递到SystemServer进程 因为RemoteViews实现了Parcelable接口 所以它可以跨进程传输(跨进程传输其实就是序列化和反序列化的过程。。。) 系统会根据RemoteViews中的包名等信息去获取该应用中的资源 然后通过LayoutInFlater加载RemoteViews中的布局 显示出来 然后还会调用RemoteViews中的action的apply方法 更新布局的信息 


* 理论上Binder可以支持View的所有操作 但是那样太麻烦了 需要提供很多Ipc操作 效率势必会降低 所以没有那样实现 所以就使用了Action ，Action代表一个view操作，action同样实现了Parcelable接口。系统首先将view操作封装到Action对象并将这些对象跨进程传输到远程进程，接着在远程进程中执行Action对象中的具体操作。在我们通过NotificationManager和AppWidgetManager来提交我们的更新时，这些Action对象就会传输到远程进程并在远程进程中依次执行；







