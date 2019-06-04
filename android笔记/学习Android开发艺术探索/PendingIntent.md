[TOC]



# pendingIntent

## 简述

* pendintIntent表示的是一种等待发生中的Intent，也就是一个Intent在一个不确定的时间会被触发，可以在程序外部进行执行，即使是程序已经退出了；会被经常使用在remoteView的点击事件中；
* PendingIntent主要持有的信息是它所包装的Intent和当前App Context，即使当前App已经不存在了，也能通过存在于PendingIntent里的 Context来执行Intent。当你把PendingIntent递交给别的程序进行处理时,PendingIntent仍然拥有PendingIntent原程序所拥有的权限，当你从系统取得一个PendingIntent时，一定要非常小心才行，比如，通常，如果Intent目的地是你自己的component（Activity/Service/BroadcastReceiver）的话，你最好采用在Intent中显示指定目的component名字的方式，以确保Intent最终能发到目的，否则Intent最后可能不知道发到哪里了。

## 使用

* 得到PendingIntent有几个方法，情况不同对应不同

  ~~~java
  public static PendingIntent getActivity(Context context, int requestCode,Intent intent, @Flags int flags)  //意图发生时，相当于context.startActivity(intent);下面2个以此类推
  public static PendingIntent getActivity(Context context, int requestCode,@NonNull Intent intent, @Flags int flags, @Nullable Bundle options)  
  public static PendingIntent getActivities(Context context, int requestCode,@NonNull Intent[] intents, @Flags int flags)  
    
  //requestCode一般为0，但是requestCode会影响到flags的效果
  //flags有4个，
   1. FLAG_CANCEL_CURRENT ：如果PendingIntent已经存在，那么将会取消当前的 PendingIntent，从而创建一个新的PendingIntent，对于通知栏来说，那些被取消的消息单击后将无法执行；
   2. FLAG_UPDATE_CURRENT：如果PendingIntent已经存在，可以让新的Intent更新之前PendingIntent中的Intent对象数据，例如更新Intent中的Extras，（即如果已经存在，那么全部都更新为最新的Intent）另外，我们也可以在PendingIntent的原进程中调用PendingIntent的cancel ()把其从系统中移除掉
   3. FLAG_NO_CREATE ：（开发中一般不用）如果PendingIntent已经存在，那么将不进行任何操作，直接返回已经存在的PendingIntent，如果PendingIntent不存在了，那么返回nul
   4. FLAG_ONE_SHOT:当前的pendingTntent只能被使用一次，然后他就会被自动的cancel掉，如果后续还有相同的pendingIntent，那么他们的send方法都会调用失败，对于通知栏来说，对于同类的通知只能使用一次，后续的通知单击后将无法打开；
     
  举个例子，如果多个通知，当pendingIntent不匹配时，那么这些通知之间不会互相干扰；当dendingIntent处于匹 配状态时，这个时候要根据上面几个参数来做出不同的措施：
     当FLAG_ONE_SHOT时，那么后续通知中的pendingIntent会和第一天通知保持一致，包括其中的Extras，单击任何    一条通知后，其他通知均无法打开，当所有通知都被清楚后，会再次重复这个过程；
     当FLAG_CANCEL_CURRENT时，那么只有最新的通知可以打开，之前的通知均无法打开；
     当FLAG_UPDATE_CURRENT时，那么之前弹出的通知中的pendingIntent会被更新（包括Extras），和最新一条通    知保持一致，并且这些通知都是可以打开的；
     
     
  //有一个好问题，怎么判断pending Intent是否存在了，这里就要明白pending Intent的匹配规则：
  //如果2个pendingIntent的requestCode相同，里面的intent也相同，那么这2个pendingIntent就是相同的
  //那么又怎么判断intent是相同的呢，如果2个intent的ComponentName（目的地）是相同的，intent-filter（AndroidManifest有）也是相同的，那么它们才是相同的;不包括extras
  ~~~

  ​