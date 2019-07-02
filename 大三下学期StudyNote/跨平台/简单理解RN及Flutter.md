[TOC]

# 简单理解ReactNative及Flutter

* 参考至`https://juejin.im/post/5d0bac156fb9a07ec56e7f15`

## 移动端跨平台的优势

* 跨平台的市场优势不在于性能或学习成本，甚至平台适配会更耗费时间，但是它最终能让代码逻辑（特别是业务逻辑），无缝的复用在各个平台上，降低了重复代码的维护成本，保证了各平台间的统一性， 如果这时候还能保证一定的性能，那就更完美了。

| 类型           | React Native                          | Flutter                    |
| :------------- | :------------------------------------ | :------------------------- |
| 语言           | JavaScript                            | dart                       |
| 环境           | JSCore                                | Flutter Engine             |
| 发布时间       | 2015                                  | 2017                       |
| star           | 78k+                                  | 67k+                       |
| 对比版本       | 0.59.9                                | 1.6.3                      |
| 空项目打包大小 | Android 20M(可调整至 7.3M) / IOS 1.6M | Android 5.2M / IOS 10.1M   |
| GSY项目大小    | Android 28.6M / IOS 9.1M              | Android 11.6M / IOS 21.5M  |
| 代码产物       | JS Bundle 文件                        | 二进制文件                 |
| 维护者         | Facebook                              | Google                     |
| 风格           | 响应式，Learn once, write anywhere    | 响应式，一次编写多平台运行 |
| 支持           | Android、IOS、(PC)                    | Android、IOS、(Web/PC)     |
| 使用代表       | 京东、携程、腾讯课堂                  | 闲鱼、美团B端              |

---

## 环境搭建

无论是 React Native 还是 Flutter ，都需要 Android 和 IOS 的开发环境，也就是 JDK 、Android SDK、Xcode 等环境配置，而不同点在于 ：

- React Native 需要 npm 、node 、react-native-cli 等配置 。
- Flutter 需要 flutter sdk 和 Android Studio / VSCode 上的 Dart 与 Flutter插件。

从配置环境上看， Flutter 的环境搭配相对简单，而 React Native 的环境配置相对复杂，而且由于 node_module 的“黑洞”属性和依赖复杂度等原因，目前在个人接触的例子中，**首次配置运行成功率 Flutter 是高于 React Native 的，且 Flutter 失败的原因则大多归咎于网络。**

## 实现原理

* 在 Android 和 IOS 上，默认情况下 **Flutter** 和 **React Native 都需要一个原生平台的Activity / ViewController 支持，且在原生层面属于一个“单页面应用”**，而它们之间最大的不同点其实在于 **UI 构建**:

**React Native ：**

* React Native 是一套 UI 框架，默认情况下 React Native 会在 Activity 下加载 JS 文件，然后运行在 JavaScriptCore 中解析 Bundle 文件布局，最终堆叠出一系列的原生控件进行渲染。

* 简单来说就是 **通过写 JS 代码配置页面布局，然后 React Native 最终会解析渲染成原生控件**，如 <View> 标签对应 ViewGroup/UIView ，<ScrollView> 标签对应 ScrollView/UIScrollView ，<Image> 标签对应 ImageView/UIImageView 等。

![img](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwMeltiaqkGib9w9miboc8ezicDKZIiaCvzbP0icvXcxtPDuFvsHluwdlW8Fec7aEsXJAllkGN7sEywQA9Zg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

所以相较于如 Ionic 等框架而言， React Native 让页面的性能能得到进一步的提升。

---

**Flutter ：**

* **如果说 React Native 是为开发者做了平台兼容，那 Flutter 则更像是为开发者屏蔽平台的概念。**

> Flutter 中只需平台提供一个 Surface 和一个 Canvas ，剩下的 Flutter说：“你可以躺下了，我们来自己动”。

* Flutter 中绝大部分的 Widget 都与平台无关， 开发者基于 Framework 开发 App ，而 Framework 运行在 Engine 之上，由 Engine 进行适配和跨平台支持。这个跨平台的支持过程，其实就是**将 Flutter UI 中的 Widget “数据化” ，然后通过 Engine 上的 Skia 直接绘制到屏幕上 ,**当需要更新UI的时候，Framework通知Engine，Engine会等到下个Vsync信号到达的时候，会通知 Framework，然后 Framework 会进行 animations, build，layout，compositing，paint，最后生成 layer 提交给 Engine。Engine会把 layer 进行组合，生成纹理，最后通过 Open Gl接口提交数据给GPU， GPU经过处理后在显示器上面显示。在Flutter界面渲染过程分为三个阶段：布局、绘制、合成；布局和绘制在Flutter框架中完成，合成则交由引擎负责。
* ![](https://img-blog.csdn.net/2018102310541525?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzY2Nzc3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

* ![img](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwMeltiaqkGib9w9miboc8ezicDKBEV2GhZPm07KicE6mTz72Zz05Q3wmYThQXTQWbVGwqiceD8R16Aicaotw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 所以从以上可以看出：**React Native 的 Learn once, write anywhere 的思路，就是只要你会 React ，那么你可以用写 React 的方式，再去开发一个性能不错的App；而 Flutter 则是让你忘掉平台，专注于 Flutter UI 就好了。**

**DOM：**

额外补充一点，React 的虚拟 DOM 的概念相信大家都知道，这是 React 的性能保证之一，而 Flutter 其实也存在类似的虚拟  DOM 概念。为什么要提出虚拟ＤＯＭ的概念？

* React 的 虚拟DOM：React实现了一个Virtual DOM，组件的真实DOM结构和Virtual DOM之间有一个映射的关系，React在虚拟DOM上实现了一个diff算法，当render()去重新渲染组件的时候，diff会找到需要变更的DOM，然后再把修改更新到浏览器上面的真实DOM上，所以，React并不是渲染了整个DOM树，Virtual DOM就是JS数据结构，所以比原生的DOM快得多。
* 原生的DOM不是慢，DOM全称文档对象模型，本质也是一个JS对象，所以操作DOM对象其实也是很快的，慢就慢在操作了DOM对象之后，浏览器之后所做出的行为，比如布局layout、绘制paint。比如原生代码中，我们需要操作10个DOM，我们的理想状态是一次性构建出全部的DOM树，然后渲染，但是实际上，浏览器收到第一个DOM操作请求之后，它并不知道你后面还有9个操作，它就会走一遍完整的渲染流程，显然像计算元素坐标这些操作都是白白浪费的，因为下一次DOM操作可能会改变这些坐标，前面的计算就白费了。而优化原生的DOM操作，其实是去试图减少`layout`的次数，中心思想都是一样的，我们在一个不在`Render Tree(渲染树)`的元素统一操作，操作完之后，再把这个节点加入到`Render Tree`上，这样的效果就是无论多么复杂的DOM操作，最终都只会进行一次`layout`。
* 回到正题：为什么要使用虚拟DOM？**`virtual dom`比原生dom快，就它的`batching` 和和独特的 `diff`算法，`batching`就是把所有的DOM操作搜集起来，一次性提交给真实DOM，`diff`算法通过对比新旧虚拟DOM树，记录之间的差异。**
* 而在Flutter 中，我们写的 Widget ， 其实并非真正的渲染控件，这一点和 React Native 中的标签类似，Widget 更像配置文件， 由它组成的 Widget 树并非真正的渲染树。

* **Widget 在渲染时会经过 Element 变化， 最后转化为 RenderObject 再进行绘制， 而最终组成的 RenderObject 树才是 “真正的渲染 Dom”** ， 每次 Widget 树触发的改变，并不一定会导致RenderObject 树的完全更新。

### 小结

* **所以在实现原理上 React Native 和 Flutter 是完全不同的思路，虽然都有类似“虚拟 DOM 的概念” ，但是React Native 带有较强的平台关联性，而 Flutter UI 的平台关联性十分薄弱。**

---

## 编程开发

### 两者的开发语言区别

*  **JS 是动态语言，而 Dart 是伪动态语言的强类型语言。**
* **动态语言和非动态语言都有各种的优缺点，比如 JS 开发便捷度明显会高于 Dart ，而 Dart 在类型安全和重构代码等方面又会比 JS 更稳健。**
* 额外补充一点，***JS* 和 *Dart* 都是单线程应用**，利用了协程的概念实现异步效果，而在 **Flutter** 中 *Dart* 支持的 `isolate` ，却是属于完完全全的异步线程处理，可以通过 Port 快捷地进行异步交互，这大大拓展了 **Flutter** 在 *Dart* 层面的性能优势。
* 协程：协程，又称微线程，纤程。英文名Coroutine。
* 协程和子程序的区别：子程序或者称为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，最后是A执行完毕。所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。
* 即协程在执行到一半的时候可以中断执行，然后CPU执行别的子程序，这里不是协程的方法内部调用，而是中断当前程序，然后转而执行别的子程序，在适当的时候再返回来接着就刚才的中断的位置继续执行。
*  看起来A、B的执行有点像多线程，但协程的特点在于是一个线程执行，那和多线程比，协程有何优势？
   * 最大的优势就是协程极高的执行效率。因为子程序切换不是线程切换，而是由程序自身控制，因此，没有线程切换的开销，和多线程比，线程数量越多，协程的性能优势就越明显。
   * 第二大优势就是不需要多线程的锁机制，因为只有一个线程，也不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。
   * 因为协程是一个线程执行，那怎么利用多核CPU呢？最简单的方法是多进程+协程，既充分利用多核，又充分发挥协程的高效率，可获得极高的性能。

### 界面开发

* React Native 在界面开发上延续了 React 的开发风格，**支持 scss/sass 、样式代码分离、在 0.59 版本开始支持 React Hook 函数式编程** 等等，而不同  React 之处就是更换标签名，并且样式和属性支持因为平台兼容做了删减。
* 一个普通 React Native 组件常见实现方式，**继承 Component类，通过 props 传递参数，然后在 render 方法中返回需要的布局，布局中每个控件通过 style 设置样式** 等等，这对于前端开发者基本上没有太大的学习成本。 

* Flutter 最大的特点在于： **Flutter 是一套平台无关的 UI 框架，在 Flutter 宇宙中万物皆 Widget。**
* Flutter 开发中一般是通过继承 **无状态 StatelessWidget** 控件或者 **有状态 StatefulWidget**控件 来实现页面，然后在**对应的 Widget build(BuildContext context) 方法内实现布局，利用不同 Widget 的 child / children 去做嵌套，通过控件的构造方法传递参数，最后对布局里的每个控件设置样式等。**
* Flutter 中把一切皆为 Widget 贯彻得很彻底，**所以 Widget 的颗粒度控制得很细** ，如 Padding 、Center 都会是一个单独的 Widget，甚至**状态共享都是通过 InheritedWidget 共享 Widget 去实现的**，而这也是被吐槽的代码嵌套样式难看的原因。 **事实上正是因为颗粒度细，所以你才可以通过不同的 Widget ， 自由组合出多种业务模版**， 比如 Flutter 中常用的 Container ，它就是官方帮你组合好的模板之一， **Container 内部其实是由 Align、 ConstrainedBox、DecoratedBox 、Padding 、Transform 等控件组合而成** ，所以嵌套深度等问题完全是可以人为控制，甚至可以在帧率和绘制上做到更细致的控制。

### 原生控件

* 在跨平台开发中，就不得不说到接入原有平台的支持，比如 在 Android 平台上接入 x5 浏览器 、接入视频播放框架、接入 Lottie 动画框架等等。
* 这一需求 React Native 先天就支持，甚至在社区就已经提供了类似 lottie-react-native的项目。  **因为 React Native 整个渲染过程都在原生层中完成，所以接入原有平台控件并不会是难事** ，同时因为发展多年，虽然各类第三方库质量参差不齐，但是数量上的优势还是很明显的。
* 而 Flutter 在就明显趋于弱势，甚至官方在开始的时候，连 WebView 都不支持，这其实涉及到  Flutter 的实现原理问题。因为  **Flutter 的整体渲染脱离了原生层面，直接和 GPU 交互，导致了原生的控件无法直接插入其中** ，而在视频播放实现上， Flutter 提供了外界纹理的设计去实现，但是这个过程需要的数据转换，很明显的限制了它的通用性， **所以在后续版本中 Flutter提供了 PlatformView 的模式来实现集成。**

> 以 Android 为例子，在原生层 Flutter 通过 Presentation 副屏显示的原理，利用 VirtualDisplay 的方式，让 Android 控件在内存中绘制到 Surface 层。VirtualDisplay 绘制在 Surface 的 textureId ，之后会通知到 Dart 层，在 Dart 层利用 AndroidView 定义好的 Widget 并带上 textureId ，那么 Engine 在渲染时，就会在内存中将 textureId 对应的数据渲染到  AndroidView 上。

* PlatformView 的设计必定导致了性能上的缺陷，最大的体现就是内存占用的上涨，同时也引导了诸如键盘无法弹出#19718和黑屏等问题，甚至于在 Android 上的性能还可能不如外界纹理。
* *https://github.com/flutter/flutter/issues/19718*
* 所以目前为止， Flutter 原生控件的接入上是仍不如 React Native 稳定。