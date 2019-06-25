[TOC]

# Java实现线程安全总结

//[https://www.iteye.com/topic/806990](https://www.iteye.com/topic/806990)

* 所谓线程安全无 非是要控制多个线程对某个资源的有序访问或修改。

## 多线程引起的问题

* 由于 CPU 的执行速度是要远高于主存的读写速度，所以直接从主存中读写数据会降低 CPU 的效率。为了解决这个问题，就有了高速缓存的概念，在每个 CPU 中都有高速缓存，它会事先从主存中读取数据，在 CPU 运算之后在合适的时候刷新到主存中。
* 现在的计算机，cpu在计算的时候，并不总是从内存读取数据，它的数据读取顺序优先级 是：寄存器－高速缓存－内存。线程耗费的是CPU，线程计算的时候，原始的数据来自内存，在计算过程中，有些数据可能被频繁读取，这些数据被存储在寄存器 和高速缓存中，当线程计算完后，这些缓存的数据在适当的时候应该写回内存。当个多个线程同时读写某个内存数据时，就会产生多线程并发问题。
* 我们看看JVM，JVM是一个虚拟 的计算机，它也会面临多线程并发问题，java程序运行在java虚拟机平台上，java程序员不可能直接去控制底层线程对寄存器高速缓存内存之间的同 步，那么java从语法层面，应该给开发人员提供一种解决方案，这个方案就是诸如**synchronized,** **volatile,锁机制（如同步块，就绪队 列，阻塞队列）等等。这些方案只是语法层面的，但我们要从本质上去理解它，不能仅仅知道一个** **synchronized 可以保证同步就完了。**
* 处理线程安全：要解决两个主要的问题：可见性和有序性。

## 可见性

* 多个线程之间是不能互相传递数据通信的，它们之间的沟通只能通过**共享变量**来进行。Java内存模型（JMM）规定了jvm有主内存，主内存是多个线程共享 的。当new一个对象的时候，也是被分配在主内存中，每个线程都有自己的工作内存，工作内存存储了主存的某些对象的副本，当然线程的工作内存大小是有限制 的。当线程操作某个对象时，执行顺序如下：

   (1) 从主存复制变量到当前工作内存 (read and load)
     (2) 执行代码，改变共享变量值 (use and assign)
     (3) 用工作内存数据刷新主存相关内容 (store and write)

  JVM规范定义了线程对主存的操作指 令：read，load，use，assign，store，write。当一个共享变量在多个线程的工作内存中都有副本时，如果一个线程修改了这个共享 变量，那么其他线程应该能够看到这个被修改后的值，这就是多线程的可见性问题。


## 有序性

* 当某些顺序被打破时会影响到我们期待的结果
* 多线程运行时打破了我们所期待的原子性操作
* 因为线程的执行顺序是不可预见的。这是java同步产生的根 源

### synchronized关键字

* java用synchronized关键字做为多线程并发环境的执行有序性的保证手段之一。当一段代码会修改共享变量，这一段代码成为互斥区或 临界区，为了保证共享变量的正确性，synchronized标示了临界区。典型的用法如下：

  ~~~java
  synchronized(锁){  
       临界区代码  
  }   
  ~~~

* 再举一个例子：为了保证银行账户的安全，可以操作账户的方法如下：

  ~~~java
  public synchronized void add(int num) {  
       balance = balance + num;  
  }  
  public synchronized void withdraw(int num) {  
       balance = balance - num;  
  } 
  //这种情况下，跟上面的比较，怎么没有看见锁，那么这种使用方法时锁是什么？其实是这个方法的 this 对象,可想而知如果是 static 方法的锁就应该是 class 对象
  ~~~


* 理论上，每个对象都可以做为锁，但一个对象做为锁时，应该被多个线程共享，这样才显得有意义，在并发环境下，一个没有共享的对象作为锁是没有意义的。假如 有这样的代码：

  ~~~java
  public class ThreadTest{
    // 此时如果不同线程进入这个方法时，都会新建一个 lock 对象，每个线程都有自己的lock，根本不存在锁竞争。 锁有什么意义呢？都不存在竞争锁这种东西了怎么实现线程同步
    public void test(){  
       Object lock=new Object();  
       synchronized (lock){  
          //do something  
       }  
    }  
  }  
  ~~~

* 每个锁对象都有两个队列，一个是就绪队列，一个是阻塞队列，就绪队列存储了将要获得锁的线程，阻塞队列存储了被阻塞的线程，当一个被线程被唤醒 (notify)后，才会进入到就绪队列，等待cpu的调度。当一开始线程a第一次执行account.add方法时，jvm会检查锁对象account 的就绪队列是否已经有线程在等待，如果有则表明account的锁已经被占用了，由于是第一次运行，account的就绪队列为空，所以线程a获得了锁， 执行account.add方法。如果恰好在这个时候，线程b要执行account.withdraw方法，因为线程a已经获得了锁还没有释放，所以线程 b要进入account的就绪队列，等到得到锁后才可以执行。

* 一个线程执行临界区代码过程如下：

  ~~~java
  1 获得同步锁
  2 清空工作内存
  3 从主存拷贝变量副本到工作内存
  4 对这些变量计算
  5 将变量从工作内存写回到主存
  6 释放锁
  ~~~

  可见，synchronized既保证了多线程的并发有序性，又保证了多线程的内存可见性。

### volatile关键字

* volatile是java提供的一种同步手段，只不过它是轻量级的同步，为什么这么说，因为volatile只能保证多线程的内存可见性，不能保证多线 程的执行有序性。而最彻底的同步要保证有序性和可见性，例如synchronized。任何被volatile修饰的变量，都不拷贝副本到工作内存，任何 修改都及时写在主存。因此对于Valatile修饰的变量的修改，所有线程马上就能看到，但是volatile不能保证对变量的修改是有序的。

* 要使 volatile 变量提供理想的线程安全,必须同时满足下面两个条件:

  1)对 变量的写操作不依赖于当前值。

  2)该变量没有包含在具有其他变量的不变式中

  volatile只保证了可见性，所以Volatile适合直接赋值的场景

  ~~~java
  public class VolatileTest{
      public volatile int a;
      public void setA(int A){
          this.a = A;
      }
  }
  //在没有volatile声明时，多线程环境下，a的最终值不一定是正确的，因为this.a=a;涉及到给a赋值和将a同步回主存的步骤，这个顺序可能被打乱。如果用volatile声明了，读取主存副本到工作内存和同步a到主存的步骤，相当于是一个原子操作。
  ~~~

* 解决双锁单例在极端模式下的失败场景

  ~~~java
  private volatile static Singleton instace;   //这里如果不是用 volatile 声明会有问题，如果不用volatile，则因为内存模型允许所谓的“无序写入”，可能导致失败。——某个线程可能会获得一个未完全初始化的实例。
    
  public static Singleton getInstance(){   
      //第一次null检查     
      if(instance == null){            
          synchronized(Singleton.class) {    //1     
              //第二次null检查       
              if(instance == null){          //2  
                  instance = new Singleton();//3  
              }  
          }           
      }  
      return instance;    
  }
  //问题发生的情况举例：
      线程 1 进入 getInstance() 方法。
      由于 instance 为 null，线程 1 在 //1 处进入synchronized 块。
      线程 1 前进到 //3 处，但在构造函数执行之前，使实例成为非null。
      线程 1 被线程 2 预占。
      线程 2 检查实例是否为 null。因为实例不为 null，线程 2 将instance 引用返回，返回一个构造完整但部分初始化了的Singleton 对象。
      线程 2 被线程 1 预占。
      线程 1 通过运行 Singleton 对象的构造函数并将引用返回给它，来完成对该对象的初始化。
  
  ~~~

* 造成上诉情况的原因：一个对象的完整创建简单来说是三步走：1分配并获取内存地址，2初始化对象，3将引用指向初始化对象。因为编译器或处理器的重排序，可能导致1,3,2这样的步骤，那么有一个线程发现对象不为空就去使用，可能会导致因为初始化未完成而出错。

* ### 可以做开销较低的“读－写锁”策略

~~~java
//如果读操作远远超过写操作，可以结合使用 synchronized 内部锁和 volatile 变量来减少公共代码路径的开销。
@ThreadSafe
public class CheesyCounter {
    // Employs the cheap read-write lock trick
    // All mutative operations MUST be done with the 'this' lock held
    @GuardedBy("this") private volatile int value;
 
    //读操作，没有synchronized，提高性能
    public int getValue() { 
        return value; 
    } 
 
    //写操作，必须synchronized。因为x++不是原子操作
    public synchronized int increment() {
        return value++;
    }
~~~



## 线程的关键方法

* ### Thread.sleep()

  ~~~java
  public static native void sleep(long millis) throws InterruptedException;
  //使当前所在线程进入阻塞
  //只是让出 CPU ，并没有释放对象锁
  //由于休眠时间结束后不一定会立即被 CPU 调度，因此线程休眠的时间可能大于传入参数
  //如果被中断会抛出 InterruptedException
  //由于 sleep 是静态方法，它的作用时使当前所在线程阻塞。因此最好在线程内部直接调用 Thread.sleep()，如果你在主线程调用某个线程的 sleep() 方法，其实阻塞的是主线程！(极有可能发生这件事)
  ~~~

* ### **Object.wait()**

  ~~~java
  //与 Thread.sleep() 容易混淆的是 Object.wait() 方法。
  
  Object.wait() 方法：
  
  //让出 CPU，释放对象锁
  //在调用前需要先拥有某对象的锁，所以一般在 synchronized 同步块中使用
  //使该线程进入该对象监视器的等待队列
  ~~~

* ### **Thread.yield()**

  ~~~java
  public static native void yield();
  //Thread.yield() 表示暂停当前线程，让出 CPU 给优先级与当前线程相同，或者优先级比当前线程更高的就绪状态的线程。
  //和 sleep() 方法不同的是，它不会进入到阻塞状态，而是进入到就绪状态。
  //yield() 方法只是让当前线程暂停一下，重新进入就绪的线程池中。
  ~~~

* ### **Thread.join()**

  ~~~java
  //表示线程合并，调用线程会进入阻塞状态，需要等待被调用线程结束后才可以执行。
  Thread thread = new Thread(new Runnable() {
      @Override
      public void run() {
          System.out.println("thread is running!");
          try {
              Thread.sleep(5000);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  });
  thread.start();
  thread.join();
  System.out.println("main thread ");
  //我们在主线程调用了 thread.join() 方法，该线程会在输出"thread is running!"一句话后休眠 5 秒，等该线程结束后主线程才可以继续执行，输出"main thread "
  ~~~

  * 源码分析

    ~~~java
    public final void join() throws InterruptedException {
        join(0);
    }
    //主线程执行到这里的时候，先获取该方法锁，然后进入
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
    
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
    
        //由于millis参数传进来的为 0
        if (millis == 0) {
            while (isAlive()) {//如果当前this对象即上面的thread对象的isAlive()是否存活
                wait(0);//如果存活，那么调用wait(0)使主线程进入该对象监视器的等待队列，直到当前this 对象即上面的 thread 对象任务执行完毕run方法结束后，isAlive() 返回false，即退出此循环；
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
    ~~~

    