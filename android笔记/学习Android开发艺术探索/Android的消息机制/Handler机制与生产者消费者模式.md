[TOC]



# Handler机制与生产者消费者模式

[![9\](https://upload.jianshu.io/users/upload_avatars/1802251/b956a9582ae6?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96)](https://www.jianshu.com/u/fcc2f0577270) [zcbiner](https://www.jianshu.com/u/fcc2f0577270) [**关注](undefined)2018.06.02 18:07* 字数 1617 阅读 147评论 0喜欢 4

#### Handler机制

Handler机制在Android中通常用来更新UI。子线程执行任务，任务执行完毕后发送消息：Handler.sendMessage()，然后在UI线程Handler.handleMessage()就会调用，执行相应处理。

Handler机制在Android中通常用来更新UI。子线程执行任务，任务执行完毕后发送消息：Handler.sendMessage()，然后在UI线程Handler.handleMessage()就会调用，执行相应处理。
Handler机制有几个非常重要的类：

- Handler：用来发送消息：sendMessage等多个方法，并实现handleMessage()方法处理回调（还可以使用Message或Handler的Callback进行回调处理，具体可以看看源码）。
- Message：消息实体，发送的消息即为Message类型。
- MessageQueue：消息队列，用于存储消息。发送消息时，消息入队列，然后Looper会从这个MessageQueen取出消息进行处理。
- Looper：与线程绑定，不仅仅局限于主线程，绑定的线程用来处理消息。loop()方法是一个死循环，一直从MessageQueen里取出消息进行处理。

这几个类的作用还可以用下图解释：

![](https://upload-images.jianshu.io/upload_images/1802251-13dd4e7ef5df8c12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

Handler机制

Looper.loop()是一个死循环，为什么不会卡死主线程呢？简单的说，就是当MessageQueue为empty时，线程会挂起，一有消息，就会唤醒线程处理消息。关于这个问题可以看看这个回答：

Android中为什么主线程不会因为Looper.loop()里的死循环卡死？

这里安卓消息机制篇有；

Handler不仅仅可以在主线程处理消息，还可以在子线程，前提是子线程要关联Looper，标准写法为：

```java
class LooperThread extends Thread {
    public Handler mHandler;
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // process incoming messages here
            }
        };
        Looper.loop();
    }
}
```

一定要调用Looper.prepare()和Looper.loop()方法。Looper.prepare()使用ThreadLocal将当前线程与new出来的Looper关联，Looper.loop()开启循环处理消息。这样子，mHandler发送的消息就可以在LooperThread进行处理了。

#### 生产者消费者模式

> 在并发编程中使用生产者和消费者模式能够解决绝大多数并发问题。该模式通过平衡生产线程和消费线程的工作能力来提高程序的整体处理数据的速度。

在线程世界里，生产者就是生产数据的线程，消费者就是消费数据的线程。在多线程开发当中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。为了解决这种生产消费能力不均衡的问题，所以便有了生产者和消费者模式。具体介绍可参考：[聊聊并发（十）生产者消费者模式](https://www.jianshu.com/p/!http://ifeve.com/producers-and-consumers-mode/)

![](https://upload-images.jianshu.io/upload_images/1802251-5d71203adcafd6a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

生产者消费者模式

#### Handler机制与生产者消费者模式

那么Handler机制和生产者消费者模式有什么关系呢？

那么Handler机制和生产者消费者模式有什么关系呢？
**Handler机制就是一个生产者消费者模式。**可以这么理解，Handler发送消息，它就是生产者，生产的是一个个Message。Looper可以理解为消费者，在其loop()方法中，死循环从MessageQueue取出Message进行处理。而MessageQueue就是缓冲区了，Handler产生的Message放到MessageQueen中，Looper从MessageQueue取出消息。
既然Handler机制本质上是一个生产者消费者模式，那么我们就可以脱离Android来实现一个Handler机制。

##### Handler

用来发送和处理消息，因此主要方法就是sendMessage()和handleMessage()：

```java
public abstract class Handler {
    private IMessageQueue messageQueue;
    public Handler(Looper looper) {
        messageQueue = looper.messageQueue;
    }
    public Handler() {
        Looper.myLooper();
    }
    public void sendMessage(Message message) {
        // 指定发送Message的Handler，方便回调
        message.target = this;
        try {
            messageQueue.enqueueMessage(message);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public abstract void handleMessage(Message msg);
}
```

##### Message

相当简单，充当一个使者的角色，重要的是要与对应的Handler绑定：target变量。

```java
public class Message {
    private int code;
    private String msg;
    Handler target;

    public Message() { }
    public Message(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }
    public int getCode() {
        return code;
    }
    public void setCode(int code) {
        this.code = code;
    }
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

##### IMessageQueue

定义的一个接口，由于MessageQueen可以有不同的实现，因此抽象一个接口出来。

```java
public interface IMessageQueue {
    Message next() throws InterruptedException;
    void enqueueMessage(Message message) throws InterruptedException;
}
```

##### Looper

loop()方法是一个死循环，在这里处理消息，没有消息时，绑定线程会处于 wait 状态。使用 ThreadLocal 来与线程进行绑定。

```java
public class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    IMessageQueue messageQueue;
    private static Looper sMainLooper;
    public Looper() {
        messageQueue = new MessageQueue(2);
        // messageQueue = new MessageQueue1(2);
        // messageQueue = new MessageQueue2(2);
    }
    public static void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
    public static void prepareMainLooper() {
        prepare();
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    public static Looper getMainLooper() {
        return sMainLooper;
    }
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        for (;;) {
            // 消费Message，如果MessageQueen为null，则等待
            Message message = null;
            try {
                message = me.messageQueue.next();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if (message != null) {
                message.target.handleMessage(message);
            }
        }
    }
}
```

##### LinkedBlockingQueue实现的MessageQueen

MessageQueue是缓冲区，它需要满足以下功能：

- 当缓冲区满时，挂起执行enqueueMessage的线程。
- 当缓冲区空时，挂起执行next的线程。
- 当缓冲区非空时，唤醒被挂起在next的线程。
- 当缓冲区不满时，唤醒被挂起在enqueueMessage的线程。

所以MessageQueue最简单的实现莫过于使用LinkedBlockingQueue了。（[源码|并发一枝花之BlockingQueue](https://www.jianshu.com/p/!https://www.jianshu.com/p/5dfc81b10e30)）

```java
public class MessageQueue implements IMessageQueue {
    private final BlockingQueue<Message> queue;
    public MessageQueue(int cap) {
        this.queue = new LinkedBlockingQueue<>(cap);
    }
    public Message next() throws InterruptedException {
        return queue.take();
    }
    public void enqueueMessage(Message message) {
        try {
            queue.put(message);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

像这样，一个简易的Handler机制就实现了。Android系统的Handler机制肯定不是那么简单，做了很多鲁棒性相关的处理，还有和底层的交互等等。另外，Android系统MessageQueen是结合Message实现的一个无界队列，意味着发送消息的队列不会阻塞。
Handler机制搭建好了，那么来测试一下吧：

```java
public class Main {
    public static void main(String[] args) {
        MainThread mainThread = new MainThread();
        mainThread.start();
        // 确保mainLooper构建完成
        while (Looper.getMainLooper() == null) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        Handler handler = new Handler(Looper.getMainLooper()) {
            @Override
            public void handleMessage(Message msg) {
                System.out.println("execute in : " + Thread.currentThread().getName());
                switch (msg.getCode()) {
                    case 0 :
                        System.out.println("code 0 : " + msg.getMsg());
                        break;
                    case 1 :
                        System.out.println("code 1 : " + msg.getMsg());
                        break;
                    default :
                        System.out.println("other code : " + msg.getMsg());
                }
            }
        };

        Message message1 = new Message(0, "I am the first message!");
        WorkThread workThread1 = new WorkThread(handler, message1);

        Message message2 = new Message(1, "I am the second message!");
        WorkThread workThread2 = new WorkThread(handler, message2);

        Message message3 = new Message(34, "I am a message!");
        WorkThread workThread3 = new WorkThread(handler, message3);

        workThread1.start();
        workThread2.start();
        workThread3.start();
    }
    /**模拟工作线程**/
    public static class WorkThread extends Thread {
        private Handler handler;
        private Message message;

        public WorkThread(Handler handler, Message message) {
            setName("WorkThread");
            this.handler = handler;
            this.message = message;
        }

        @Override
        public void run() {
            super.run();
            // 模拟耗时操作
            Random random = new Random();
            try {
                Thread.sleep(random.nextInt(10) * 300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 任务执行完，sendMessage
            handler.sendMessage(message);
        }
    }
    /**模拟主线程*/
    public static class MainThread extends Thread {
        public MainThread() {
            setName("MainThread");
        }
        @Override
        public void run() {
            super.run();
            // 这里是不是与系统的调用一样，哈哈
            Looper.prepareMainLooper();
            System.out.println(getName() + " the looper is prepared");
            Looper.loop();
        }
    }
}
```

这里模拟了子线程执行任务，发送消息到主线程处理的场景。那么执行结果就是：

```
MainThread the looper is prepared
execute in : MainThread
other code : I am a message!

execute in : MainThread
code 0 : I am the first message!

execute in : MainThread
code 1 : I am the second message!
```

可以看出handleMessage的执行都是在主线程的，简易Handler机制搞定！！！

本文章到这就应该结束了，但是MessageQueen的实现方式是可以有多种的，因此来看看其不同的实现。

##### wait/notify实现

```java
public class MessageQueue1 implements IMessageQueue {

    private Queue<Message> queue;
    private final AtomicInteger integer = new AtomicInteger(0);
    private volatile int count;
    private final Object BUFFER_LOCK = new Object();
    public MessageQueue1(int cap) {
        this.count = cap;
        queue = new LinkedList<>();
    }

    @Override
    public Message next() throws InterruptedException {
        synchronized (BUFFER_LOCK) {
            while (queue.size() == 0) {
                BUFFER_LOCK.wait();
            }
            Message message = queue.poll();
            BUFFER_LOCK.notifyAll();
            return message;
        }
    }

    @Override
    public void enqueueMessage(Message message) throws InterruptedException {
        synchronized (BUFFER_LOCK) {
            while (queue.size() == count) {
                BUFFER_LOCK.wait();
            }
            queue.offer(message);
            BUFFER_LOCK.notifyAll();
        }
    }
}
```

BUFFER_LOCK 用来处理与锁相关的逻辑。

next()方法中，如果queue的size 为0，说明现在没有消息待处理，因此执行BUFFER_LOCK.wait()挂起线程，当queue.poll();时，queue里就有了消息，需要唤醒因为没有消息而挂起的线程，所以执行BUFFER_LOCK.notifyAll();。

enqueueMessage()方法中，如果queue.size() == count说明消息满了，如果Handler继续sendMessage，queue无法继续装下，因此该线程需要挂起： BUFFER_LOCK.wait();。当执行queue.offer(message);时，queue里头保存的Message就少了一个，可以插入新的Message，因此BUFFER_LOCK.notifyAll();唤醒因为queue满了而挂起的线程。

##### Lock实现

既然wait/notify可以实现MessageQueue，那么ReentrantLock肯定也能实现，下面就是使用ReentrantLock实现的例子，原理上来讲是一致的。

```java
public class MessageQueue2 implements IMessageQueue {
    private final Queue<Message> queue;
    private int cap = 0;
    private final Lock lock = new ReentrantLock();
    private final Condition BUFFER_CONDITION = lock.newCondition();
    public MessageQueue2(int cap) {
        this.cap = cap;
        queue = new LinkedList<>();
    }

    @Override
    public Message next() throws InterruptedException {
        try {
            lock.lock();
            while (queue.size() == 0) {
                BUFFER_CONDITION.await();
            }
            Message message = queue.poll();
            BUFFER_CONDITION.signalAll();
            return message;
        } finally {
            lock.unlock();
        }
    }
    @Override
    public void enqueueMessage(Message message) throws InterruptedException {
        try {
            lock.lock();
            while (queue.size() == cap) {
                BUFFER_CONDITION.await();
            }
            queue.offer(message);
            BUFFER_CONDITION.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

##### Lock实现的优化

上面的实现过程中，入队出队是基于同一个锁的，意味着，如果有一个线程正在入队，其他线程就不能出队；有一个线程出队，其它不能入队。一个时刻只能有一个线程操作缓冲区，这显然在多线程环境下会影响性能。所以对其进行优化。
优化想法是，入队和出队互相不干扰。所以入队和出队分别需要一个锁，入队在队列的尾部进行，出队在头部进行。代码如下：

```java
public class MessageQueue3 implements IMessageQueue {
    private final Lock putLock = new ReentrantLock();
    private final Condition notFull = putLock.newCondition();
    private final Lock takeLock = new ReentrantLock();
    private final Condition notEmpty = takeLock.newCondition();
    private Node head; // 队头
    private Node last; // 队尾
    private AtomicInteger count = new AtomicInteger(0); // 记录大小
    private int cap = 10; // 容量，默认为10

    public MessageQueue3(int cap) {
        this.cap = cap;
    }

    @Override
    public Message next() throws InterruptedException {
        Node node;
        int c = -1;
        takeLock.lock();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            node = head;
            head = head.next;
            c = count.getAndDecrement();
            if (c > 0) {
                notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (count.get() < cap) {
            signalNotFull();
        }
        return node.data;
    }

    @Override
    public void enqueueMessage(Message message) throws InterruptedException {
        Node node = new Node(message);
        int c = -1;
        putLock.lock();
        try {
            while (count.get() == cap) {
                notFull.await();
            }
            // 初始状态
            if (head == null && last == null) {
                head = last = node;
            } else {
                last.next = node;
                last = last.next;
            }

            c = count.getAndIncrement();
            if (c < cap) {
                notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c > 0) {
            signalNotEmpty();
        }
    }

    private void signalNotEmpty() {
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    private void signalNotFull() {
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
    
    static class Node {
        Message data;
        Node next;
        public Node(Message data) {
            this.data = data;
        }
    }
}
```

全文完~

