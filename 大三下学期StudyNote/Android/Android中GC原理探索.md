[TOC]

# Android中GC原理探索

* 参考至`<https://mp.weixin.qq.com/s/CUU3Ml394H_fkabhNNX32Q>`
* 对于 JVM 的 GC 机制之前已经有所了解，这里就不再说明，对于 Android 的 Dalvik 虚拟机和 ART 虚拟机的 GC 只是看过而已，看过上面这篇文章整理一次，大部分是原文搬过来的，小部分修改

## Dalvik 的 GC 

* COW是copy on write的缩写。意思是当写的时候拷贝一份新的，在新的上面进行写入，写入完成后，再将新的结果赋值给旧的数据。

### Dalvik 的 Java堆的组成

* Dalvik 中的 Java 堆实际上是由一个Active堆和一个Zygote堆组成的（为什么这么分看下面（为了COW）），其中，Zygote堆用来管理Zygote进程在启动过程中预加载和创建的各种对象，而Active堆是在Zygote进程fork第一个子进程之前创建的。以后启动的所有应用程序进程是被Zygote进程fork出来的，并都持有一个自己的Dalvik虚拟机。**在创建应用程序的过程中，Dalvik虚拟机采用COW策略复制Zygote进程的地址空间**。
* **COW策略**（写时复制策略）：一开始的时候（未复制Zygote进程的地址空间的时候），应用程序进程和Zygote进程共享了同一个用来分配对象的堆。当Zygote进程或者应用程序进程对该堆进行**写操作**时，内核就会执行真正的拷贝操作，使得Zygote进程和应用程序进程分别拥有自己的一份拷贝，这就是所谓的COW。因为copy是十分耗时的，所以必须尽量避免copy或者尽量少的copy。
* 为了实现这个目的（尽可能避免 copy ），当创建第一个应用程序进程时，会将已经使用了的那部分堆内存划分为一部分，还没有使用的堆内存划分为另外一部分。前者就称为 Zygote 堆，后者就称为 Active 堆。这样只需把 zygote 堆中的内容复制给应用程序进程就可以了(但是还没有复制)。以后无论是 Zygote 进程，还是应用程序进程，当它们需要分配对象的时候，都在Active堆上进行。这样就可以使得Zygote堆尽可能少地被执行写操作，因而就可以减少执行写时拷贝的操作。在 Zygote 堆里面分配的对象其实主要就是Zygote进程在启动过程中预加载的类、资源和对象了。这意味着这些预加载的类、资源和对象可以在Zygote 进程和应用程序进程中做到长期共享。这样既能减少拷贝操作，还能减少对内存的需求。

### 和 GC 有关的参数

* 在启动Dalvik虚拟机的时候，我们可以分别通过**-Xms**、**-Xmx**和**-XX:HeapGrowthLimit**三个选项来指定三个值，以上三个值分别表示表示

  - **Starting Size：** Dalvik虚拟机启动的时候，会先分配一块**初始的堆内存**给虚拟机使用。
  - **Maximum Size：**设置的最大堆内存大小
  - **Growth Limit：**是系统给每一个程序的**最大堆上限**,超过这个上限，程序就会OOM

  同时除了上面的这个三个指标外，还有几个指标也是值得我们关注的，那就是堆最小空闲值（Min Free）、堆最大空闲值（Max Free）和堆目标利用率（Target Utilization）。假设在某一次GC之后，存活对象占用内存的大小为LiveSize，那么这时候堆的理想大小应该为(LiveSize / U)。但是(LiveSize / U)必须大于等于(LiveSize + MinFree)并且小于等于(LiveSize + MaxFree)，**每次GC后垃圾回收器都会尽量让堆的利用率往目标利用率靠拢**。所以当我们尝试手动去生成一些几百K的对象，**试图去扩大可用堆大小的时候**，反而会导致频繁的GC，因为这些对象的分配会导致GC，而**GC后会让堆内存回到合适的比例**，而我们使用的局部变量很快会被回收，理论上存活对象还是那么多，我们的堆大小也会缩减回来无法达到扩充的目的。 

### GC 的类型

malloc 的全称是memory allocation，中文叫动态内存分配

- **GC_FOR_MALLOC**: 表示是在堆上分配对象时内存不足触发的GC。（最容易触发的GC）
- **GC_CONCURRENT**: 当我们应用程序的堆内存达到一定量，或者可以理解为快要满的时候，系统会自动触发GC操作来释放内存。
- **GC_EXPLICIT**: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。
- **GC_BEFORE_OOM**: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。

实际上，GC_FOR_MALLOC、GC_CONCURRENT和GC_BEFORE_OOM三种类型的GC都是在分配对象的过程触发的。而并发和非并发GC的区别主要在于前者在GC过程中，有条件地挂起和唤醒非GC线程，而后者在执行GC的过程中，**一直都是挂起**非GC线程的。并行GC通过有条件地挂起和唤醒非GC线程，就可以使得应用程序获得更好的响应性。但是同时并行GC需要多执行一次标记根集对象以及递归标记那些在GC过程被访问了的对象的操作，所以也需要花费更多的CPU资源。后文在Art的并发和非并发GC中我们也会着重说明下这两者的区别。

### 对象的分配过程中GC的触发时机

1. 一开始调用函数dvmHeapSourceAlloc在Java堆上分配指定大小的内存。如果分配成功，那么就将分配得到的地址直接返回给调用者了。函数dvmHeapSourceAlloc在不改变Java堆当前大小的前提下进行内存分配，这是属于轻量级的内存分配动作。
2. 如果上一步内存分配失败，这时候就需要执行一次GC（GC_FOR_MALLOC类型）了。不过如果GC线程已经在运行中，即gDvm.gcHeap->gcRunning的值等于true，那么就直接调用函数dvmWaitForConcurrentGcToComplete等到GC执行完成就是了。否则的话，就需要调用函数 gcForMalloc 来执行一次GC了，此时参数false表示不要回收软引用对象引用的对象。
3. GC执行完毕后，再次调用函数dvmHeapSourceAlloc尝试轻量级的内存分配操作。如果分配成功，那么就将分配得到的地址直接返回给调用者了。
4. 如果上一步内存分配失败，这时候就得考虑先将Java堆的当前大小设置为Dalvik虚拟机启动时指定的Java堆最大值，再进行内存分配了。这是通过调用函数dvmHeapSourceAllocAndGrow来实现的。
5. 如果调用函数dvmHeapSourceAllocAndGrow分配内存成功，则直接将分配得到的地址直接返回给调用者了。
6. 如果上一步内存分配还是失败，这时候就得出狠招了。再次调用函数gcForMalloc来执行GC（GC_BEFORE_OOM 类型）。此时参数true表示要回收软引用对象引用的对象。
7. GC执行完毕，再次调用函数dvmHeapSourceAllocAndGrow进行内存分配。这是最后一次努力了，成功与事都到此为止。

### 回收算法和内存碎片

主流的**大部分Davik**采取的都是标注与清理（Mark and Sweep）回收算法，也有实现了拷贝GC的，这一点和HotSpot是不一样的，具体使用什么算法是在编译期决定的，无法在运行的时候动态更换。如果在编译dalvik虚拟机的命令中指明了”WITH_COPYING_GC”选项，则编译”/dalvik/vm/alloc/Copying.cpp”源码 – 此是Android中拷贝GC算法的实现，否则编译”/dalvik/vm/alloc/HeapSource.cpp” – 其实现了标注与清理GC算法。

由于Mark and Sweep算法的缺点，容易导致内存碎片，所以在这个算法下，当我们有大量不连续小内存的时候，再分配一个较大对象时，还是会非常容易导致GC。

所以对于Dalvik虚拟机的手机来说，我们首先要尽量避免掉频繁生成很多临时小变量（比如说：getView，onDraw等函数），另一个又要尽量去避免产生很多长生命周期的大对象。

## ART 的 GC

### ART的Java堆组成

* ART运行时内部使用的Java堆的主要组成包括Image Space、Zygote Space、Allocation Space和Large Object Space四个Space，Image Space用来存在一些预加载的类， Zygote Space和Allocation Space与Dalvik虚拟机垃圾收集机制中的Zygote堆和Active堆的作用是一样的，Large Object Space就是一些离散地址的集合，用来分配一些大对象从而提高了GC的管理效率和整体性能

### GC 的类型

- **kGcCauseForAlloc** ，当要分配内存的时候发现内存不够的情况下引起的GC，这种情况下的GC会stop world
- **kGcCauseBackground**，当内存达到一定的阀值的时候会去出发GC，这个时候是一个后台gc，不会引起stop world
- **kGcCauseExplicit**，显式调用的时候进行的gc，如果art打开了这个选项的情况下，在system.gc的时候会进行gc

### 对象的分配过程中GC的触发时机

* 和 Davilk 一样

### 并发和非并发GC

Art在GC上不像Dalvik仅有一种回收算法，Art在不同的情况下会选择不同的回收算法，比如Alloc内存不够的时候会采用非并发GC，而在Alloc后发现内存达到一定阀值的时候又会触发并发GC。同时在前后台的情况下GC策略也不尽相同，后面我们会一一给大家说明。

##### 非并发GC

​	步骤1. 调用子类实现的成员函数InitializePhase执行GC初始化阶段。

​	步骤2. 挂起所有的ART运行时线程。（非并发GC在stopworld 过程中执行GC标记阶段和回收阶段，单次stopworld耗时比并发GC久）

​	步骤3. 调用子类实现的成员函数MarkingPhase执行GC标记阶段。

​	步骤4. 调用子类实现的成员函数ReclaimPhase执行GC回收阶段。

​	步骤5. 恢复第2步挂起的ART运行时线程。

​	步骤6. 调用子类实现的成员函数FinishPhase执行GC结束阶段。

##### 并发GC

​	步骤1. 调用子类实现的成员函数InitializePhase执行GC初始化阶段。

​	步骤2. 获取用于访问Java堆的锁。

​	步骤3. 调用子类实现的成员函数MarkingPhase执行GC并行标记阶段。（并发GC的标记阶段不需要stopworld）

​	步骤4. 释放用于访问Java堆的锁。

​	步骤5. 挂起所有的ART运行时线程（并发GC也会stopWorld 但是在stopWorld中只处理在GC并行标记阶段被修改的对象,不需要执行标记回收过程）

​	步骤6. 调用子类实现的成员函数HandleDirtyObjectsPhase处理在GC并行标记阶段被修改的对象。

​	步骤7. 恢复第4步挂起的ART运行时线程。

​	步骤8. 重复第5到第7步，直到所有在GC并行阶段被修改的对象都处理完成。

​	步骤9. 获取用于访问Java堆的锁。

​	步骤10. 调用子类实现的成员函数ReclaimPhase执行GC回收阶段。（并行GC回收阶段不需要stopworld）

​	步骤11. 释放用于访问Java堆的锁。

​	步骤12. 调用子类实现的成员函数FinishPhase执行GC结束阶段。

* 所以不论是并发还是非并发，都会引起stopworld的情况出现，并发的情况下单次stopworld的时间会更短

### 前后台GC

- 前台Foreground指的就是应用程序在前台运行时，而后台Background就是应用程序在后台运行时。因此，Foreground GC就是应用程序在前台运行时执行的GC，而Background就是应用程序在后台运行时执行的GC。

- 应用程序在前台运行时，响应性是最重要的，因此也要求执行的GC是高效的。相反，应用程序在后台运行时，响应性不是最重要的，这时候就适合用来解决堆的内存碎片问题。因此，Mark-Sweep （标记回收）GC适合作为Foreground GC，而Mark-Compact （标记压缩）GC适合作为Background GC。

- 由于有Compact的能力存在，碎片化在Art上可以很好的被避免，这个也是Art一个很好的能力。

### Art并发和Dalvik并发GC的差异

* 通过阅读相关文档可以了解到Art并发GC对于Dalvik来说主要有三个优势点：

##### 1、标记自身

* Art在对象分配时会将新分配的对象压入到Heap类的成员变量allocation*stack*描述的Allocation Stack中去，从而可以一定程度缩减对象遍历范围。

##### 2、预读取

* 对于标记Allocation Stack的内存时，会预读取接下来要遍历的对象，同时再取出来该对象后又会将该对象引用的其他对象压入栈中，直至遍历完毕。

##### 3、减少Pause时间

* 在Mark阶段是不会Block其他线程的，这个阶段会有脏数据，比如Mark发现不会使用的但是这个时候又被其他线程使用的数据，在Mark阶段也会处理一些脏数据而不是留在最后Block的时候再去处理，这样也会减少后面Block阶段对于脏数据的处理的时间。

###  Art大法好

* 总的来看，art在gc上做的比dalvik好太多了，不光是gc的效率，减少pause时间，而且还在内存分配上对大内存的有单独的分配区域，同时还能有算法在后台做内存整理，减少内存碎片。对于开发者来说art下我们基本可以避免很多类似gc导致的卡顿问题了。另外根据谷歌自己的数据来看，Art相对Dalvik内存分配的效率提高了10倍，GC的效率提高了2-3倍。

## 小结

* 这篇文章帮助理解了Andorid 中 Dalvik 和 art 的 Java 堆组成，各自使用的回收算法，及他们两者间的若干区别。了解了 GC 的触发时机和GC 类型，