[TOC]

## AIDL例子及注意事项

* https://blog.csdn.net/u010132993/article/details/72632458#%E7%BC%96%E5%86%99%E6%9C%8D%E5%8A%A1%E7%AB%AF%E4%BB%A3%E7%A0%81


* AIDL是Android Interface Definition Language的缩写，即Android接口描述语言。设计这门语言的主要目的是为了实现进程间通信，**只有当你允许不同的客户端访问你的服务并且服务需要处理多线程问题时你才必须使用AIDL**。利用AIDL可以定义不同进程间通信所必备的通信边界(即通信协议)，通过这些通信边界，不同进程间可以互相共享数据，甚至调用一些特定方法。AIDL分同步实现和异步实现两种，同步实现是一种阻塞形式，就想我们在上篇中介绍IBinder的描述一样，客户端需等待服务的返回以唤醒自己的线程继续执行；异步就是回调的形式，客户端发出请求传到服务并不需要等待服务返回，服务执行后后主动告知客户端，引起客户端处理请求结果，就像setXXXListener的过程一样。

* 对于AIDL而言，其新建的文件为.aidl后缀并非.java后缀，而且在使用基本数据类型的时候是不需要导包的，但是除了这些基本类型之外，其他类型在使用之前必须导包，就算目标文件与当前正在编写的 .aidl 文件在同一个包下，而在 Java 中，这种情况是不需要导包的。

  了解了AIDL的语法，乘热打铁，赶紧来编写一个同步AIDL的demo吧，AIDL项目编写的一般步骤如下：

  * 在服务端创建`AIDL`文件并编译(编译的目的是生成中间文件)[PS：也可以在客户端创建`AIDL`文件]
  * 拷贝`AIDL`文件及实体类到客户端(或者服务端，取决于你首次编写`AIDL`在那一端，编译成功后拷贝至另一端即可)
  * 编写服务端代码
  * 编写客户端代码，发送请求到服务器

* 例子：

  https://blog.csdn.net/u010132993/article/details/72632458#%E7%BC%96%E5%86%99%E6%9C%8D%E5%8A%A1%E7%AB%AF%E4%BB%A3%E7%A0%81

* 启动服务

  ~~~java
  intent.setClassName("com.example.cnt.aidltest1","com.example.cnt.aidltest1.BookManagerService");
  ~~~

* 针对AIDL开发，我总结了几条注意事项，简称为M3T，分别代表：

  MUST：Package Name Of JAVA Bean MUST Be Same To AIDL
  MUST：AIDL File Of Client MUST Be Same To Server
  MUST：Non Primitive type MUST implement Parcelable Interface
  TAG：Directional TAG For Non-primitive Parameters
  Package Name Of Java Bean MUST Be Same To AIDL
  ​

  * 这个Must的意思是AIDL文件中的Book.aidl文件的包名必须和Java 实体Bean的包名一致。 

  ​

  * 此时多半是因为你拷贝过来的AIDL路径没有被当作源码路径导致的，此时可在build.gradle的android域中添加如下代码：

    ~~~java
      sourceSets {
            main {
                java.srcDirs = ['src/main/java', 'src/main/aidl']
            }
        }
    ~~~

  AIDL File Of Client MUST Be Same To Server
  这个MUST的意思是在aidl文件从客户端向服务器(或者服务器向客户端)拷贝的过程中，AIDL文件在两端必须保持一致，不止每个文件的内容，还包含每个文件的包名类名等，如下图所示： 

  ​

  Client和Server必须完全一致，此时就要求我们适时调整JAVA Beans 目录。

  这个MUST的意思是说，用于Binder传递的实体类必须实现Parcelable 接口。在跨进程过程中，数据必须从Application传入Framework进而传入Kernel，在共享内存中share给另一个进程，要实现这种操作，数据必须序列化，Parcelable是一个用于序列化的接口，Android特有，效率比Serializable高，不会引起频繁GC,不适用于持久化数据，通常用于IPC通信，实现Parcelable接口必须实现Parcelable.Creator接口且实现该接口的对象名必须为CREATOR，而Serializable是一个标记接口，实现方式简单，JAVA SE本身支持，依赖于反射实现,通常用于持久化数据，这里一定要注意一点，Parcelable实体类中默认是没有生成readFromParcel(Parcel dest)函数的，切记切记，随后在讲TAG的时候会用到。

## 定向标记in,out,inout

* 这些标记针对于 服务端 in（读）,out（写）,inout

或许细心的朋友已经发现了，在上篇的demo中，我们编写的BookManager.aidl文件中有如下接口： 

Book前面有一个in，就像数据类型一样的标记，那么这个东东是干什么的呢？这里的in被叫做定向标记，对于非基本类型的数据我们必须使用in指定其数据流向，一共有三种标记可以使用：in，out，inout。 这里的In Out标记并不是指的单纯流向，而应该指的是数据的相对流向： 

- in表示数据流是从客户端传入对象流向服务器的 
- out表示数据流是从服务器流入客户端传入对象的 
- inout指的是针对传入对象双向传递 

上述运行结果充分验证我们的猜想是完全正确的，对于out TAG型的变量而已，其不会传递客户端任何属性值到服务器，其属性值完全受服务器操作影响，服务器修改其值，客户端同时发生改变； 在AIDL开发中，所有的非基本参数都需要一个定向TAG来指出数据流 通的方式，基本参数的定向TAG默认是并且只能是 in，

- 其中 in 表示数据只能由客户端流向服务端， 
- out 表示数据只能由服务端流向客户端，
- 而 inout 则表示数据可在服务端与客户端之间双向流通。
- 其中，数据流向是针对在客户端中传入的对象而言的。in 为定向 TAG的话表现为服务端将会接收到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端对传参的修改而发生变动；
- out 的话表现为服务端将会接收到那个对象的参数为空的对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；
- inout 为定向 TAG的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动.


  