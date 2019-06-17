[TOC]

# 了解Android设计架构

* 之前看 Android 源码设计模式最后面就有讲到这些设计架构，之前有做笔记，不过看了下面这篇文章讲的是Android 中架构写的很好，直接参考下面博客上的

* 参考至`<https://www.tianmaying.com/tutorial/AndroidMVC>`

* 为什么要架构设计？

  通过设计使程序模块化，做到模块内部的高聚合和模块之间的低耦合。这样做的好处是使得程序在开发的过程中，开发人员只需要专注于一点，提高程序开发的效率，并且更容易进行后续的测试以及定位问题。但设计不能违背目的，对于不同量级的工程，具体架构的实现方式必然是不同的，切忌犯为了设计而设计，为了架构而架构的毛病。

## MVC

* MVC全名是Model View Controller，如图，是模型(model)－视图(view)－控制器(controller)的缩写，一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑。其中M层处理数据，业务逻辑等；V层处理界面的显示结果；C层起到桥梁的作用，来控制V层和M层通信以此来达到分离视图显示和业务逻辑层。

* ![](http://assets.tianmaying.com/md-image/0f7bfc3b4bb89562737510b0af1f7ad0.png)

* Android 中界面部分也采用了当前比较流行的MVC框架，XML（即Activity中的试图） 即 View，Activity 为 Controller （可以看出 Activity身兼两职，所以我们的 Activity 可能会代码量爆炸）,M针对业务模型，建立的数据结构和相关的类；还有例子：

* 我手画了一张图简单举例

  ![](https://wx2.sinaimg.cn/mw690/007duwLrgy1g41nlabl45j30u014045h.jpg)

* Android MVC 的缺点

  * 在Android开发中，Activity并不是一个标准的MVC模式中的Controller，它的首要职责是加载应用的布局和初始化用户 界面，并接受并处理来自用户的操作请求，进而作出响应。随着界面及其逻辑的复杂度不断提升，Activity类的职责不断增加，以致变得庞大臃肿。

## MVP

* Activity本身需要担负与用户之间的操作交互，界面的展示，不是单纯的Controller或View。而且**现在大部分的Activity还对整个App起到类似IOS中的【ViewController】的作用**，这又带入了大量的逻辑代码，造成Activity的臃肿。为了解决这个问题，让我们引入MVP框架。

* MVP框架由3部分组成：View负责显示，Presenter负责逻辑处理，Model提供数据。在MVP模式里通常包含3个要素（加上View interface是4个）：

  - View:负责绘制UI元素、与用户进行交互(在Android中体现为Activity)

  - Model:负责存储、检索、操纵数据(有时也实现一个Model interface用来降低耦合)

  - Presenter:作为View与Model交互的中间纽带，处理与用户交互的负责逻辑。

  - View interface:需要View实现的接口，View通过View interface与Presenter进行交互，降低耦合，方便进行单元测试

    * **Tips：**View interface的必要性

      回想一下你在开发Android应用时是如何对代码逻辑进行单元测试的？是否每次都要将应用部署到Android模拟器或真机上，然后通过模拟用 户操作进行测试？然而由于Android平台的特性，每次部署都耗费了大量的时间，这直接导致开发效率的降低。而在MVP模式中，处理复杂逻辑的Presenter是通过interface与View(Activity)进行交互的，这说明我们可以通过自定义类实现这个interface来模拟Activity的行为对Presenter进行单元测试，省去了大量的部署及测试的时间。

* 当我们将Activity复杂的逻辑处理移至另外的一个类（Presenter）中时，Activity其实就是MVP模式中的View，它负责UI元素的初始化，建立UI元素与Presenter的关联（Listener之类），同时自己也会处理一些简单的逻辑（复杂的逻辑交由 Presenter处理）。

* 与MVC的区别

  两种模式的**主要区别**：

  - **（最主要区别）View与Model并不直接交互**，而是通过与Presenter交互来与Model间接交互。而在MVC中View可以与Model直接交互
  - 通常View与Presenter是一对一的，但复杂的View可能绑定多个Presenter来处理逻辑。而Controller是基于行为的，并且可以被多个View共享，Controller可以负责决定显示哪个View
  - Presenter与View的交互是通过**接口**来进行的（这里很重要），更有利于添加单元测试。

  [![MVC与MVP区别](http://pic001.cnblogs.com/images/2012/1/2012040113391482.jpg)](http://pic001.cnblogs.com/images/2012/1/2012040113391482.jpg)

* 举个简单的例子，UI层通知逻辑层（Presenter）用户点击了一个Button，逻辑层（Presenter）自己决定应该用什么行为进行响应，该找哪个模型（Model）去做这件事，最后逻辑层（Presenter）将完成的结果更新到UI层。

* MVP的变种有很多，其中使用最广泛的是Passive View模式，即被动视图。在这种模式下，View和Model之间不能直接交互，View通过Presenter与Model打交道。Presenter接受View的UI请求，完成简单的UI处理逻辑，并调用Model进行业务处理，并调用View将相应的结果反映出来。View直接依赖Presenter，但是Presenter间接依赖View，它直接依赖的是View实现的接口。

  ![](http://assets.tianmaying.com/md-image/f7002cd0e8951e46fd963bff0a0081d8.jpg)

  相对于View的被动，那Presenter就是主动的一方。对于Presenter的主动，有如下的理解：

  - Presenter是整个MVP体系的控制中心，而不是单纯的处理View请求的人；
  - View仅仅是用户交互请求的汇报者，对于响应用户交互相关的逻辑和流程，View不参与决策，真正的决策者是Presenter；
  - View向Presenter发送用户交互请求应该采用这样的口吻：“我现在将用户交互请求发送给你，你看着办，需要我的时候我会协助你”，不应该是这样：“我现在处理用户交互请求了，我知道该怎么办，但是我需要你的支持，因为实现业务逻辑的Model只信任你”；
  - 对于绑定到View上的数据，不应该是View从Presenter上“拉”回来的，应该是Presenter主动“推”给View的；
  - View尽可能不维护数据状态，因为其本身仅仅实现单纯的、独立的UI操作；Presenter才是整个体系的协调者，它根据处理用于交互的逻辑给View和Model安排工作。

## MVVM

MVVM可以算是MVP的升级版，其中的VM是ViewModel的缩写，ViewModel可以理解成是View的数据模型和Presenter的合体，ViewModel和View之间的交互通过Data Binding完成，而Data Binding可以实现双向的交互，这就使得视图和控制层之间的耦合程度进一步降低，关注点分离更为彻底，同时减轻了Activity的压力。

在比较之前，先从图上看看三者的异同。

[![2f9e4ee7d9616257ab41de204c06ffd5_b.jpg](http://assets.tianmaying.com/md-image/bb8f3106230c33063ab53393dfe1876a.jpg)](http://assets.tianmaying.com/md-image/bb8f3106230c33063ab53393dfe1876a.jpg)

刚开始理解这些概念的时候认为这几种模式虽然都是要**将view和model解耦**，但是非此即彼，没有关系，一个应用只会用一种模式。后来慢慢发现世界绝对不是只有黑白两面，中间最大的一块其实是灰色地带，同样，这几种模式的边界并非那么明显，可能你在自己的应用中都会用到。实际上也根本没必要去纠结自己到底用的是MVC、MVP还是MVVP，不管黑猫白猫，捉住老鼠就是好猫。