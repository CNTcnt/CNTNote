[TOC]



# CrashHandler

## 简述

* 当我们的程序在用户手中的发生了预料不到的 Crash 抛出了异常，而在开发时我们没有预想到这个异常然后没有进行捕获处理,此时会导致程序崩溃，此时我们需要去知道这个 crash 才能去修复它；我们需要了解到的是，当crash 发生时，程序最后会调用 CrashHandler 的 uncaughtException 方法，所以在这个方法中我们可以获取 crash 的具体信息然后储存在本地有网络是上传服务器即可；Android 提供了 CrashHandler 来监视应用的 crash 信息；具体做法是我们给程序设置一个我们自定义的 CrashHandler;

* 主要是覆写 CrashHandler 的 uncaughtException 方法；

* 需要申请相关权限

  ~~~java
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
      <uses-permission android:name="android.permission.INTERNET" />
      <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
      <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
      <uses-permission android:name="android.permission.READ_PHONE_STATE" />
      <uses-permission android:name="android.permission.READ_LOGS" />
  ~~~

  ​

## CrashHandler对象实现

~~~java
/**
 * UncaughtException处理类,当程序发生Uncaught异常的时候,由该类来接管程序,并记录发送错误报告(没有实现上传服务器的功能).
 *
 * @author cnt
 */
public class CrashHandler implements UncaughtExceptionHandler {

    public static final String TAG = "CrashHandler";

    //系统默认的UncaughtException处理类
    private Thread.UncaughtExceptionHandler mDefaultHandler;
    //CrashHandler实例,单例实现
    private static CrashHandler INSTANCE = new CrashHandler();
    //程序的Context对象
    private Context mContext;
    //用来存储设备信息和异常信息
    private Map<String, String> infos = new HashMap<String, String>();

    //用于格式化日期,作为日志文件名的一部分
    private DateFormat formatter = new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss");

    /**
     * 保证只有一个CrashHandler实例
     */
    private CrashHandler() {
    }

    /**
     * 获取CrashHandler实例 ,单例模式
     */
    public static CrashHandler getInstance() {
        return INSTANCE;
    }

    /**
     * 初始化
     *
     * @param context
     */
    public void init(Context context) {
        mContext = context;
        //获取系统默认的UncaughtException处理器
        mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();
        //设置该CrashHandler为程序的默认处理器
        Thread.setDefaultUncaughtExceptionHandler(this);
    }

    /**
     * 当UncaughtException发生时会转入该函数来处理
     */
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        if (!handleException(ex) && mDefaultHandler != null) {
            //如果用户没有处理则让系统默认的异常处理器来处理
            mDefaultHandler.uncaughtException(thread, ex);
        } else {

            Log.e(TAG, "uncaughtException: " + "几次");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            //这个绕过了生命周期的顺序，属于强制关闭，一旦执行这句话，后面的再也不会执行了
            android.os.Process.killProcess(android.os.Process.myPid());
            System.exit(1);
        }
    }

    /**
     * 自定义错误处理,收集错误信息 发送错误报告等操作均在此完成.
     *
     * @param ex
     * @return true:如果处理了该异常信息;否则返回false.
     */
    private boolean handleException(Throwable ex) {
        if (ex == null) {
            return false;
        }
        /*
		 * 使用Toast来显示异常信息,
		 * 由于在主线程会阻塞,
		 * 不能实时出现 Toast 信息,
		 * 这里我们在子线程中处理 Toast 信息
		 */
        new Thread() {
            @Override
            public void run() {
                Looper.prepare();
                Toast.makeText(mContext, "很抱歉,程序出现异常,即将退出.", Toast.LENGTH_LONG).show();
                Looper.loop();
            }
        }.start();
        //收集设备参数信息
        collectDeviceInfo(mContext);
        //保存日志文件
        saveCrashInfo2File(ex);
        return true;
    }

    /**
     * 收集设备参数信息
     *
     * @param ctx
     */
    public void collectDeviceInfo(Context ctx) {
        try {
            //获取包管理器
            PackageManager pm = ctx.getPackageManager();
            //获取包信息
            PackageInfo pi = pm.getPackageInfo(ctx.getPackageName(), PackageManager.GET_ACTIVITIES);
            if (pi != null) {
                String versionName = pi.versionName == null ? "null" : pi.versionName;
                String versionCode = pi.versionCode + "";
                infos.put("versionName", versionName);
                infos.put("versionCode", versionCode);
            }
        } catch (NameNotFoundException e) {
            Log.e(TAG, "an error occured when collect package info", e);
        }
        //使用反射获取 Build 类成员变量,并遍历获取这些变量内容:
        Field[] fields = Build.class.getDeclaredFields();
        for (Field field : fields) {
            try {
                //设置 Build 成员变量可访问
                field.setAccessible(true);
                infos.put(field.getName(), field.get(null).toString());
                Log.d(TAG, field.getName() + " : " + field.get(null));
            } catch (Exception e) {
                Log.e(TAG, "an error occured when collect crash info", e);
            }
        }
    }

    /**
     * 保存错误信息到文件中
     *
     * @param ex
     * @return 返回文件名称, 便于将文件传送到服务器
     */
    private String saveCrashInfo2File(Throwable ex) {

        //存储相关的字符串信息
        StringBuffer sb = new StringBuffer();
        ///将成员变量 Map<String, String> infos  中的数据存储到 StringBuffer sb 中
        for (Map.Entry<String, String> entry : infos.entrySet()) {
            String key = entry.getKey();
            String value = entry.getValue();
            sb.append(key + "=" + value + "\n");
        }

        //拿到所抛异常添加进 sb 中 再将 sb 写入文件
        Writer writer = new StringWriter();
        PrintWriter printWriter = new PrintWriter(writer);
        //printStackTrace(PrintWriter pw)，输出到名为pw的PrintWriter。
        ex.printStackTrace(printWriter);
        Throwable cause = ex.getCause();
        while (cause != null) {
            cause.printStackTrace(printWriter);
            cause = cause.getCause();
        }
        printWriter.close();
        String result = writer.toString();
        sb.append(result);
        try {
            long timestamp = System.currentTimeMillis();
            String time = formatter.format(new Date());
            String fileName = time + timestamp + ".txt";
            //创建文件夹和文件
            if (Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)) {
                String path = Environment.getExternalStorageDirectory().getPath() + "/sdcardcrash/";
                File dir = new File(path);
                if (!dir.exists()) {
                    dir.mkdirs();
                }
                //创建输出流
                FileOutputStream fos = new FileOutputStream(path + fileName);
                //向文件中写出数据
                fos.write(sb.toString().getBytes());
                fos.close();
            }
            return fileName;
        } catch (Exception e) {
            Log.e(TAG, "an error occured while writing file...", e);
        }
        return null;
    }
}
~~~

## 使用时

~~~java
CrashHandler.getInstance().init(this);
~~~



