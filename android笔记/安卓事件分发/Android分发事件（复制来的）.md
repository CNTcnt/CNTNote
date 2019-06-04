# 1. 基础认知

### 1.1 事件分发的对象是谁？

**答：事件**

- 当用户触摸屏幕时（View或ViewGroup派生的控件），将产生点击事件（Touch事件）。

  > Touch事件相关细节（发生触摸的位置、时间、历史记录、手势动作等）被封装成MotionEvent对象

- 主要发生的Touch事件有如下四种：

  - MotionEvent.ACTION_DOWN：按下View（所有事件的开始）
  - MotionEvent.ACTION_MOVE：滑动View
  - MotionEvent.ACTION_CANCEL：非人为原因结束本次事件
  - MotionEvent.ACTION_UP：抬起View（与DOWN对应）

- 事件列：从手指接触屏幕至手指离开屏幕，这个过程产生的一系列事件 
  任何事件列都是以DOWN事件开始，UP事件结束，中间有无数的MOVE事件，如下图： 
  ![事件列](http://upload-images.jianshu.io/upload_images/944365-79b1e86793514e99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即当一个MotionEvent 产生后，系统需要把这个事件传递给一个具体的 View 去处理,

### 1.2 事件分发的本质

**答：将点击事件（MotionEvent）向某个View进行传递并最终得到处理**

> 即当一个点击事件发生后，系统需要将这个事件传递给一个具体的View去处理。**这个事件传递的过程就是分发过程。**

### 1.3 事件在哪些对象之间进行传递？

**答：Activity、ViewGroup、View**

> 一个点击事件产生后，传递顺序是：Activity（Window） -> ViewGroup -> View

- Android的UI界面是由Activity、ViewGroup、View及其派生类组合而成的 
  ![UI界面](http://upload-images.jianshu.io/upload_images/944365-ece40d4524784ffa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- View是所有UI组件的基类

  > 一般Button、ImageView、TextView等控件都是继承父类View

- ViewGroup是容纳UI组件的容器，即一组View的集合（包含很多子View和子VewGroup），

  > 1. 其本身也是从View派生的，即ViewGroup是View的子类
  > 2. 是Android所有布局的父类或间接父类：项目用到的布局（LinearLayout、RelativeLayout等），都继承自ViewGroup，即属于ViewGroup子类。
  > 3. 与普通View的区别：ViewGroup实际上也是一个View，只不过比起View，它多了可以包含子View和定义布局参数的功能。

### 1.4 事件分发过程由哪些方法协作完成？

**答：dispatchTouchEvent() 、onInterceptTouchEvent()和onTouchEvent()**

![事件分发相关方法](http://upload-images.jianshu.io/upload_images/944365-a5eeeae6ee27682a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 下文会对这3个方法进行详细介绍

### 1.5 总结

- Android事件分发机制的本质是要解决：

  点击事件由哪个对象发出，经过哪些对象，最终达到哪个对象并最终得到处理。

   

  ​

  > 这里的对象是指Activity、ViewGroup、View

- Android中事件分发顺序：**Activity（Window） -> ViewGroup -> View**

- 事件分发过程由dispatchTouchEvent() 、onInterceptTouchEvent()和onTouchEvent()三个方法协助完成

经过上述3个问题，相信大家已经对Android的事件分发有了感性的认知，接下来，我将详细介绍Android事件分发机制。

------

# 2. 事件分发机制方法&流程介绍

- 事件分发过程由dispatchTouchEvent() 、onInterceptTouchEvent()和onTouchEvent()三个方法协助完成，如下图：

![方法详细介绍](http://upload-images.jianshu.io/upload_images/944365-74bdb5c375a37100.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Android事件分发流程如下：（

  必须熟记

  ）

   

  ​

  > Android事件分发顺序：**Activity（Window） -> ViewGroup -> View**

其中：

- super：调用父类方法
- true：消费事件，即事件不继续往下传递
- false：不消费事件，事件也不继续往下传递 / 交由给父控件onTouchEvent（）处理

**接下来，我将详细介绍这3个方法及相关流程。**

### 2.1 dispatchTouchEvent()

| 属性   | 介绍                         |
| ---- | -------------------------- |
| 使用对象 | Activity、ViewGroup、View    |
| 作用   | 分发点击事件                     |
| 调用时刻 | 当点击事件能够传递给当前View时，该方法就会被调用 |
| 返回结果 | 是否消费当前事件，详细情况如下：           |

**1. 默认情况：根据当前对象的不同而返回方法不同**

| 对象        | 返回方法                       | 备注                                  |
| --------- | -------------------------- | ----------------------------------- |
| Activity  | super.dispatchTouchEvent() | 即调用父类ViewGroup的dispatchTouchEvent() |
| ViewGroup | onIntercepTouchEvent()     | 即调用自身的onIntercepTouchEvent()        |
| View      | onTouchEvent（）             | 即调用自身的onTouchEvent（）                |

![流程解析](http://upload-images.jianshu.io/upload_images/944365-673eacf259c8b50c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2. 返回true**

- 消费事件
- 事件不会往下传递
- 后续事件（Move、Up）会继续分发到该View
- 流程图如下：

![流程图](http://upload-images.jianshu.io/upload_images/944365-919f10d7d671cdea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**3. 返回false**

- 不消费事件

- 事件不会往下传递

- 将事件回传给父控件的onTouchEvent()处理

  > Activity例外：返回false=消费事件

- 后续事件（Move、Up）会继续分发到该View(与onTouchEvent()区别）

- 流程图如下： 
  ![流程图](http://upload-images.jianshu.io/upload_images/944365-8b45e9551e833955.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 onTouchEvent()

| 属性   | 介绍                        |
| ---- | ------------------------- |
| 使用对象 | Activity、ViewGroup、View   |
| 作用   | 处理点击事件                    |
| 调用时刻 | 在dispatchTouchEvent()内部调用 |
| 返回结果 | 是否消费（处理）当前事件，详细情况如下：      |

> 与dispatchTouchEvent()类似

**1. 返回true**

- 自己处理（消费）该事情
- 事件停止传递
- 该事件序列的后续事件（Move、Up）让其处理；
- 流程图如下： 
  ![流程图](http://upload-images.jianshu.io/upload_images/944365-cd63e2c3b7f89b47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2. 返回false（同默认实现：调用父类onTouchEvent()）**

- 不处理（消费）该事件
- 事件往上传递给父控件的onTouchEvent()处理
- 当前View不再接受此事件列的其他事件（Move、Up）；
- 流程图如下： 
  ![流程图](http://upload-images.jianshu.io/upload_images/944365-9ce60722ac0a9a36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.3 onInterceptTouchEvent()

| 属性   | 介绍                                  |
| ---- | ----------------------------------- |
| 使用对象 | ViewGroup（注：Activity、View都没该方法）     |
| 作用   | 拦截事件，即自己处理该事件                       |
| 调用时刻 | 在ViewGroup的dispatchTouchEvent()内部调用 |
| 返回结果 | 是否拦截当前事件，详细情况如下：                    |

![返回结果](http://upload-images.jianshu.io/upload_images/944365-6b34c1017b14104d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 流程图如下：

![流程图](http://upload-images.jianshu.io/upload_images/944365-37be4474ef7a1741.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.4 三者关系

下面将用一段伪代码来阐述上述三个方法的关系和点击事件传递规则

```
// 点击事件产生后，会直接调用dispatchTouchEvent（）方法
public boolean dispatchTouchEvent(MotionEvent ev) {

    //代表是否消耗事件
    boolean consume = false;


    if (onInterceptTouchEvent(ev)) {
    //如果onInterceptTouchEvent()返回true则代表当前View拦截了点击事件
    //则该点击事件则会交给当前View进行处理
    //即调用onTouchEvent (）方法去处理点击事件
      consume = onTouchEvent (ev) ;

    } else {
      //如果onInterceptTouchEvent()返回false则代表当前View不拦截点击事件
      //则该点击事件则会继续传递给它的子元素
      //子元素的dispatchTouchEvent（）就会被调用，重复上述过程
      //直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    return consume;
   }1234567891011121314151617181920212223
```

### 2.5 总结

- 对于事件分发的3个方法，你应该清楚了解
- 接下来，我将开始介绍Android事件分发的常见流程

# 3. 事件分发场景介绍

下面我将利用例子来说明常见的点击事件传递情况

### 3.1 背景描述

我们将要讨论的布局层次如下： 
![布局层次](http://upload-images.jianshu.io/upload_images/944365-ecac6247816a3db1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 最外层：Activiy A，包含两个子View：ViewGroup B、View C
- 中间层：ViewGroup B，包含一个子View：View C
- 最内层：View C

假设用户首先触摸到屏幕上View C上的某个点（如图中黄色区域），那么Action_DOWN事件就在该点产生，然后用户移动手指并最后离开屏幕。

### 3.2 一般的事件传递情况

一般的事件传递场景有：

- 默认情况
- 处理事件
- 拦截DOWN事件
- 拦截后续事件（MOVE、UP）

### 3.2.1 默认情况

- 即不对控件里的方法(dispatchTouchEvent()、onTouchEvent()、onInterceptTouchEvent())进行重写或更改返回值

- 那么调用的是这3个方法的默认实现：调用父类的方法

- 事件传递情况：（如图下所示）

   

  ​

  - 从Activity A—->ViewGroup B—>View C，从上往下调用dispatchTouchEvent()
  - 再由View C—>ViewGroup B —>Activity A，从下往上调用onTouchEvent()

![流程图](http://upload-images.jianshu.io/upload_images/944365-69229fd4a804c0f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注：虽然ViewGroup B的onInterceptTouchEvent方法对DOWN事件返回了false，后续的事件（MOVE、UP）依然会传递给它的onInterceptTouchEvent()

> 这一点与onTouchEvent的行为是不一样的。

#### 3.2.2 处理事件

假设View C希望处理这个点击事件，即C被设置成可点击的（Clickable）或者覆写了C的onTouchEvent方法返回true。

> 最常见的：设置Button按钮来响应点击事件

事件传递情况：（如下图）

- DOWN事件被传递给C的onTouchEvent方法，该方法返回true，表示处理这个事件
- 因为C正在处理这个事件，那么DOWN事件将不再往上传递给B和A的onTouchEvent()；
- 该事件列的其他事件（Move、Up）也将传递给C的onTouchEvent()

![流程图](http://upload-images.jianshu.io/upload_images/944365-26290b0d33370853.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.2.3 拦截DOWN事件

假设ViewGroup B希望处理这个点击事件，即B覆写了onInterceptTouchEvent()返回true、onTouchEvent()返回true。 
事件传递情况：（如下图）

- DOWN事件被传递给B的onInterceptTouchEvent()方法，该方法返回true，表示拦截这个事件，即自己处理这个事件（不再往下传递）

- 调用onTouchEvent()处理事件（DOWN事件将不再往上传递给A的onTouchEvent()）

- 该事件列的其他事件（Move、Up）将直接传递给B的onTouchEvent()

  > 该事件列的其他事件（Move、Up）将不会再传递给B的onInterceptTouchEvent方法，该方法一旦返回一次true，就再也不会被调用了。

![流程图](http://upload-images.jianshu.io/upload_images/944365-d8acd35d3a50e091.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.2.4 拦截DOWN的后续事件

假设ViewGroup B没有拦截DOWN事件（还是View C来处理DOWN事件），但它拦截了接下来的MOVE事件。

- DOWN事件传递到C的onTouchEvent方法，返回了true。

- 在后续到来的MOVE事件，B的onInterceptTouchEvent方法返回true拦截该MOVE事件，但该事件并没有传递给B；这个MOVE事件将会被系统变成一个CANCEL事件传递给C的onTouchEvent方法

- 后续又来了一个MOVE事件，该MOVE事件才会直接传递给B的onTouchEvent()

  > 1. 后续事件将直接传递给B的onTouchEvent()处理
  > 2. 后续事件将不会再传递给B的onInterceptTouchEvent方法，该方法一旦返回一次true，就再也不会被调用了。

- C再也不会收到该事件列产生的后续事件。 
  ![流程图](http://upload-images.jianshu.io/upload_images/944365-edb138a07cbb990c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

特别注意：

- 如果ViewGroup A 拦截了一个半路的事件（如MOVE），这个事件将会被系统变成一个CANCEL事件并传递给之前处理该事件的子View；
- 该事件不会再传递给ViewGroup A的onTouchEvent()
- 只有再到来的事件才会传递到ViewGroup A的onTouchEvent()