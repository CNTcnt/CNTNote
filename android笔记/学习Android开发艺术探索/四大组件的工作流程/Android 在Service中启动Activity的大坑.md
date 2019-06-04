[TOC]



# Android 在Service中启动Activity的大坑

2017年07月05日 10:43:05阅读数：1729

在Activity中其中startActivity这个大家应该是非常熟悉的；那么从service里面调用startActivity话，会怎么样呢？

会出现下面的异常：

```java
android.util.AndroidRuntimeException: Calling startActivity() from outside of an Activity  context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?1
```

也就是在service里面启动Activity的话，必须添加FLAG_ACTIVITY_NEW_TASK flag。

> 那么下面的话，我们将从下面几个方面分析这个问题。

- 1．    这个异常怎么产生的？
- 2．    解决这个异常后会出现问题？
- 3．    为什么Activity.startActivity()不会出现这个问题？
- 4．    Android 为什么要这么设计？

下面，一一分析

## 一.    Context的继承关系图

首先来看一张图， 这张图表示了Context里面的基本继承关系。

​ ![img](http://s3.51cto.com/wyfs02/M01/54/6A/wKioL1SBnNHSOicYAACm7622Oog549.jpg)

1.    最上面的是Context.java，它其实是一个抽象类，它有两个重要的子类ContextImpl和ContextWrapper

2.    ContextImpl，是Context功能实现的主要类，

3.    ContextWrapper，顾名思义，它只是一个包装而已。主要功能实现都是通过调用ContextImpl去实现的。

4.    ContextThemeWrapper，包括一些主题的包装，由于Service没有主题，所以直接继承ContextWrapper；但是Activity就需要继承ContextThemeWrapper

## 二.    异常如何产生

**找到报错的代码**

文件：

> frameworks\base\core\Java\Android\app\ContextImpl.java

代码：

```java
public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity)null, intent, -1, options);
}
```

在下面的if条件判断，如果不包含FLAG_ACTIVITY_NEW_TASK就会报这个错误

```java
if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
    ...
}   
```

那么service.startActivity(Intent intent)怎么会调用这里来的呢？

要回答这个问题，我们分析下service.startActivity()做了什么，其实，service.startActivity调用的是ContextWrapper.startActivity()，因为service继承自ContextWrapper

2.  代码文件

> frameworks\base\core\java\android\content\ContextWrapper.java

代码：

```java
public void startActivity(Intent intent, Bundle options) {
    mBase.startActivity(intent, options);
}
```

ContextWrapper.startActivity的话，是直接调用的

mBase.startActivity(intent, options);

那么这个mBase是什么呢？又是什么时候赋值的呢？其实mBase是在ContextWrapper的attachBaseContext的时候初始化的。如下：

```java
protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
 }
```

那又是谁调用attachBaseContext的呢？

是在service创建的时候，在ActivityThread里面调用，如下：

1. 代码文件

> frameworks\base\core\java\android\app\ActivityThread.java

代码：

```java
private void handleCreateService(CreateServiceData data) {
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            ....
        }
        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
            mServices.put(data.token, service);
            ....
        } catch (Exception e) {
            ...
        }
    }
```

抽出主要代码分析ActivityThread. handleCreateService()方法里面主要做这几件事

- 1 通过pms找到要启动的Service配置信息，然后通过反射生成Service对象
- 2 创建ContextImpl对象，然后调用service.attach方法设置到ContextWrapper.java的mBaseContext变量里面。

那现在就明白了，service.startActivity()->ContextWrapper.startActivity()->ContextImpl.startActivity()

然后再ContextImpl.startActivity里面会检查Intent的参数是否包含FLAG_ACTIVITY_NEW_TASK，从而出现这个异常。

## 三.    解决这个异常后会出现问题？

有些同学就会说了，在Service里面启动Activity必须要有FLAG_ACTIVITY_NEW_TASK参数，那么我们添加上不就可以了？如下：

intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

那么这样会带来什么问题呢？

这样带来的问题就是在最近任务列表里面会出现两个相同的应用程序，比如你是在电话本里面启动的，那么最近任务列表就会出现两个电话本；因为有两个Task嘛！

那怎么解决呢？其实也非常好解决，只要在新的Task里面的Activity里面配置android:excludeFromRecents=”true”就可以了。表示这个Activity不会显示在最近列表里面。

## 四.    Activity.startActivity()为什么不出现这个异常呢？

要回答这个问题，需要看下Activity.startActivity()调用到哪里去了

代码文件：

> frameworks\base\core\java\android\app\Activity.java

代码：

```java
public void startActivity(Intent intent) {
   this.startActivity(intent, null);
}
```

接下来会调用startActivityForResult()->然后一路调用到Ams去启动Activity;

原来如此，Activity重写了startActivity()方法…

## 五.    Android 为什么要这么设计?

那现在来回答这个问题，为什么Android在Service 里面启动Activity要强制规定使用参数FLAG_ACTIVITY_NEW_TASK呢？

我们可以来做这样一个假设，我们有这样一个需求：

我们在电话本里面启动一个Service，然后它执行5分钟后，启动一个Activity

那么很有可能用户在5分钟后已经不在电话本程序里面操作了，有可能去上网，打开浏览器程序了。

5分钟后，此时当前的Task是浏览器的task，那么弹出Activity，如果这个Activity在当前Task的话，也就是浏览器的Task；那么用户就会觉得莫名其妙；因为弹出的Activity和浏览器在一个Task，本来这个Activity应该属于电话本的。

所以，对于Service而言，干脆强制定义启动的Activity要创建一个新的Task.

这种设计，我觉得还是比较合理的。

## 六. 样例代码:

```java
Intent dialogIntent = new Intent(getBaseContext(), YourActivity.class);   
dialogIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);   
getApplication().startActivity(dialogIntent);
```

# Service 启动 Dialog

由于 Dialog 是依赖于 Activity 存在的，所以对于从 Service 启动 Dialog 主要有两种方法：

1. 首先启动一个 半透明的 Activity，然后在 Activity 里启动 Dialog。（或者直接使用 Activity 来仿写一个 Activity）
2. 使用 WindowManager 实现

使用 WindowManager 时需要注意，此时的 Dialog 是 SYSTEM 级别的，如果程序在后台时启动这个 Dialog，Dialog 会浮在桌面上。（使用小米等有自己权限管理的系统时，需要申请一定权限才可以在桌面显示这个 Dialog，否则只能在自己 APP 前台时才显示）

## 使用 WindowManager 实现

```java
AlertDialog.Builder builder = new AlertDialog.Builder(this);
builder.setMessage("是否接受文件?")
.setPositiveButton("是", new DialogInterface.OnClickListener() {
@Override
publicvoid onClick(DialogInterface dialog, int which) {
}
}).setNegativeButton("否", new OnClickListener() {
@Override
publicvoid onClick(DialogInterface dialog, int which) {
}
});
AlertDialog ad = builder.create();
// ad.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_DIALOG); //系统中关机对话框就是这个属性
ad.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
ad.setCanceledOnTouchOutside(false); //点击外面区域不会让dialog消失
ad.show();
```

权限

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

## 使用 Activity 实现

Activity 半透明主题

```xml
<style name="DialogTransparent" parent="@android:style/Theme.Dialog">
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowAnimationStyle">@android:style/Animation</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowContentOverlay">@null</item>
        <item name="android:windowIsFloating">false</item>
        <item name="android:windowIsTranslucent">true</item>
    </style>
```

或者直接使用

```java
@android:style/Theme.Dialog
```

