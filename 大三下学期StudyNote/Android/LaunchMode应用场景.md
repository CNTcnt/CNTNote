[TOC]

# LaunchMode应用场景

* 首先当然是得知道有多少种　LaunchMOde
* 默认情况下，所有的 Activity 所需的 任务栈的名字为应用的包名，为 TaskAffinity 属性

## Standard

* standard，标准模式，默认模式，srandard模式的Activity默认会进入启动它的Activity所属的任务栈中，现在就有一个问题，如果用 ApplicicationContext 来启动Activity,由于它不是 Activity 类型的 Context ,并没有所谓的任务栈，所以就会报错，所以如果真的要使用ApplicicationContext 来启动Activity，就要为它指定启动类型为 FLAG_ACTIVITY_NEW_TASK，这样实际上是以 singleTask（栈内复用模式） 模式启动的。

## SingTop

* singtop：栈顶复用模式，栈顶不是该类型的Activity，创建一个新的Activity。否则，回调onNewIntent（）方法。

* singleTop适合接收通知启动的内容显示页面。

  例如，某个新闻客户端的新闻内容页面，如果收到10个新闻推送，每次都打开一个新闻内容页面是很烦人的。

## SingleTask

* singleTask：栈内复用模式。回退栈中没有该类型的Activity，创建Activity，否则，ClearTop＋onNewIntent，即清除在它之上的其他　Activity 然后回调 onNewIntent。

* singleTask适合作为程序入口点。

  例如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。

## SingleInstance

* singleInstance：单实例模式。回退栈中，只有这一个Activity，没有其他Activity（系统会新创建一个新的任务栈单独放这个 acitivty ）。其余特性就和 singTask 一样了

* singleInstance应用场景：

  闹铃的响铃界面。 你以前设置了一个闹铃：上午6点。在上午5点58分，你启动了闹铃设置界面，并按 Home 键回桌面；在上午5点59分时，你在微信和朋友聊天；在6点时，闹铃响了，并且弹出了一个对话框形式的 Activity(名为 AlarmAlertActivity) 提示你到6点了(这个 Activity 就是以 SingleInstance 加载模式打开的)，你按返回键，回到的是微信的聊天界面，这是因为 AlarmAlertActivity 所在的 Task 的栈只有他一个元素， 因此退出之后这个 Task 的栈空了。如果是以 SingleTask 打开 AlarmAlertActivity，那么当闹铃响了的时候，按返回键应该进入闹铃设置界面。

## TaskAffinity

* taskAffinity 用来指定 activity 所需要的任务栈的名字。

* **在启动模式为Standard下，单独使用TaskAffinity属性是无效的**。

* allowTaskReparenting用来标记Activity能否从启动的Task移动到taskAffinity指定的Task，默认是继承至application中的allowTaskReparenting=false，如果为true，则表示可以更换；false表示不可以。

* taskAffinity 和singleTask 配合使用或者和 allowTaskReparenting 配合使用

  * taskAffinity 和singleTask 配合使用：待启动的Activity 会启动在名字和 taskAffinity 相同的任务栈中

  * taskAffinity 和 allowTaskReparenting 配合使用：从一个与该Activity TaskAffinity属性不同的任务栈中迁移到与它TaskAffinity相同的任务栈中

    比如：现在有两个应用A和B，A启动了B的一个Activity C，然后按Home键回到桌面，然后再单击B的桌面图标，这个时候不是启动了B的主Activity，而是重新显示了已经被应用A启动的Activity C。我们也可以理解为，C从A的任务栈转移到了B的任务栈中。
     可以这么理解，由于A启动了C，这个时候C只能运行在A的任务栈中，但是C属于B应用，正常情况下，它的TaskAffinity值肯定不可能和A的任务栈相同，所以当B启动后，B会创建自己的任务栈，这个时候系统发现C原本想要的任务栈已经创建了，所以就把C从A的任务栈中转移过来了。