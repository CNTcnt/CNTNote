###                                              Fragment（碎片）###

***

#### Fragment的认知####

* 可以认为是一个活动中的子活动
* 可以把Fragment当成Activity一个界面的一部分，甚至Activity的界面由完全不同的Fragment组成，更帅气的是Fragment有自己的声明周期和接收、处理用户的事件，这样就不必要在一个Activity里面写一堆事件、控件的代码了。增强了代码的可读性，减轻Activity的负担，更为重要的是，你可以动态的添加、替换、移除某个Fragment。

#### Fragment的方法####

* onAttach(Activity);　　**//当Activity与Fragment发生关联时调用**

  onCreateView(LayoutInflater,ViewGroup,Bundle);　　**//创建该Fragment的视图**

  onActivityCreate(bundle);　　**//当Activity的onCreate()；方法返回时调用**

  onDestoryView();　　**//与onCreateView相对应，当改Fragment被移除时调用**

  onDetach();　　**//与onAttach()相对应，当Fragment与Activity的关联被取消时调用**

  注意：除了onCreateView，其他的所有方法如果你重写了，必须调用父类对于该方法的实现。

* **获取FragmentManager的方式：**

  getFragmentManager() // v4中，getSupportFragmentManager（）

* **主要的操作都是FragmentTransaction的方法**

  FragmentTransaction transaction = fm.benginTransatcion();//开启一个事务，这里的fm已经是一个FragManager对象(这个对象是调用getFragmentManager()获得)

  1. **transaction.add() **

  ​       往Activity中添加一个Fragment

  2. **transaction.remove()**

  ​       从Activity中移除一个Fragment，如果被移除的Fragment没有添加到回退栈（回退栈后面       会详细说），这个Fragment实例将会被销毁。

  3. **transaction.replace()**

  ​       使用另一个Fragment替换当前的fragment，实际上就是remove()然后add()的合体.

  4. **transaction.commit()**

     执行事务。

* **transaction.hide()**

  隐藏当前的Fragment，仅仅是设为不可见，并不会销毁

  **transaction.show()**

  显示之前隐藏的Fragment

  **detach()**

  将此Fragment从Activity中分离，会销毁其布局，但不会销毁该实例

  **attach()**

  将从Activity中分离的Fragment，重新关联到该Activity，重新创建其视图层次

