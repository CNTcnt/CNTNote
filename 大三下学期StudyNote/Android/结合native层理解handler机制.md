[TOC]

# 结合native 层理解 handler机制

`<https://blog.csdn.net/luoshengyang/article/details/6817933>`

* 之前对于 handler 的一点不懂就是loop中消息队列无消息时就会进入阻塞，那么是怎么进入阻塞，怎么实现阻塞的，以及阻塞之后是怎么唤醒的没有一个全面的认识，这一篇就是为了解决这个疑问

## 接收消息时的阻塞

* ~~~java
  public static void loop() {
      ...
          for (;;) {
              Message msg = queue.next(); // might block就是在这里阻塞住的，我们再去消息队列里看看是怎么阻塞的
      ...
  }
  ~~~

* ~~~java
  Message next() {
         //下面这个变量是干什么的，
          final long ptr = mPtr;
          if (ptr == 0) {
              return null;
          }
  
          int pendingIdleHandlerCount = -1; // -1 only during first iteration
          int nextPollTimeoutMillis = 0;
          for (;;) {
              if (nextPollTimeoutMillis != 0) {
                  Binder.flushPendingCommands();
              }
  			//这个方法用来干什么的，看名字是一个native方法，应该是阻塞在这里了，第一个参数就是我们刚才不知道的那个参数，那这个参数是什么呢？有什么功能呢？
              nativePollOnce(ptr, nextPollTimeoutMillis);
  
        ...
          }
  ~~~

* ~~~java
  //原来是在构造函数中构造的
  MessageQueue(boolean quitAllowed) {
          mQuitAllowed = quitAllowed;
          mPtr = nativeInit();//这个也是native方法
   }
  ~~~

* ~~~java
  //安卓4.2以上实现
  static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
      NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
      if (!nativeMessageQueue) {
          jniThrowRuntimeException(env, "Unable to allocate native queue");
          return 0;
      }
  
      nativeMessageQueue->incStrong(env);
      return reinterpret_cast<jlong>(nativeMessageQueue);
  }
  //安卓4.2以下为下面这样实现
  //在JNI层，创建了一个NativeMessageQueue对象，这个NativeMessageQueue对象保存在Java层的消息队列对象mQueue的成员变量mPtr中；
  static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
      //这里看出来， 在JNI层也创建一个native的消息队列
      NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
      if (! nativeMessageQueue) {
          jniThrowRuntimeException(env, "Unable to allocate native queue");
          return;
      }
   	//这个方法是干什么的？
      android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
  }
  static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv* env, jobject messageQueueObj,
          NativeMessageQueue* nativeMessageQueue) {
      //原来把这个消息队列对象保存在前面我们在Java层中创建的MessageQueue对象的mPtr成员变量里面，这里我们就解决了之前的mpr对象是什么的疑问。
      env->SetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr,
               reinterpret_cast<jint>(nativeMessageQueue));
  }
  NativeMessageQueue::NativeMessageQueue() {
      //可以看到消息队列创建的时候也把looper对象创建出来了
      mLooper = Looper::getForThread();
      if (mLooper == NULL) {
          mLooper = new Looper(false);
          Looper::setForThread(mLooper);
      }
  }
  
  //在C++层，创建了一个Looper对象，保存在JNI层的NativeMessageQueue对象的成员变量mLooper中，这个对象的作用是，当Java层的消息队列中没有消息时，就使Android应用程序主线程进入等待状态，而当Java层的消息队列中来了新的消息后，就唤醒Android应用程序的主线程来处理这个消息。
  Looper::Looper(bool allowNonCallbacks) :
  	mAllowNonCallbacks(allowNonCallbacks),
  	mResponseIndex(0) {
  	int wakeFds[2];
  	int result = pipe(wakeFds);//创建管道，这个管道用来干什么呢，管道就是一个文件，在管道的两端，分别是两个打开文件文件描述符，这两个打开文件描述符都是对应同一个文件，其中一个是用来读的，别一个是用来写的
  	......
   //下面就是拿到创建好的管道的两端
  	mWakeReadPipeFd = wakeFds[0];
  	mWakeWritePipeFd = wakeFds[1];
   
  	......
   
  #ifdef LOOPER_USES_EPOLL
  	// Allocate the epoll instance and register the wake pipe.
      //这里用到了linux的epoll机制，但是这里我们其实只需要监控的IO接口只有mWakeReadPipeFd一个，即前面我们所创建的管道的读端，为什么还需要用到epoll呢？有点用牛刀来杀鸡的味道。其实不然，这个Looper类是非常强大的，它除了监控内部所创建的管道接口之外，还提供了addFd接口供外界面调用，外界可以通过这个接口把自己想要监控的IO事件一并加入到这个Looper对象中去，当所有这些被监控的IO接口上面有事件发生时，就会唤醒相应的线程来处理，不过这里我们只关心刚才所创建的管道的IO事件的发生。
  
  	mEpollFd = epoll_create(EPOLL_SIZE_HINT);
  	......
   
  	struct epoll_event eventItem;
  	memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
	eventItem.events = EPOLLIN;
  	eventItem.data.fd = mWakeReadPipeFd;
      //这里就是告诉mEpollFd，它要监控mWakeReadPipeFd文件描述符的EPOLLIN事件，即当管道中有内容可读时，就唤醒当前正在等待管道中的内容的线程。但是这里还没有开开始去获取监听的结果
  	result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
  	......
  #else
  	......
  #endif
   
  	......
  }
  
  ~~~
  
* 本来只想弄清楚Java层的message对象里的mpr属性是什么，没想到看到了一大堆之前没有注意的东西，现在我们弄清楚 mpr属性是什么了，我们该到 nativePollOnce(mPtr, nextPollTimeoutMillis);中看一下到底是怎么阻塞的

  ~~~java
  //这是一个JNI方法，我们等一下再分析，这里传入的参数mPtr就是指向前面我们在JNI层创建的NativeMessageQueue对象了，而参数nextPollTimeoutMillis则表示如果当前消息队列中没有消息，它要等待的时间，如果消息中没有消息，就进入无限等待状态中去。
  nativePollOnce(mPtr, nextPollTimeoutMillis);
  
  static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
          jint ptr, jint timeoutMillis) {
      NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);//通过ptr拿到jni层创建的消息队列后，调用 pollOnce
      nativeMessageQueue->pollOnce(timeoutMillis);
  }
  void NativeMessageQueue::pollOnce(int timeoutMillis) {
      //这里把任务交由给 looper 执行
      mLooper->pollOnce(timeoutMillis);
  }
  ~~~

  ~~~java
  //下面我们就进入了 looper 中
  int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
  	int result = 0;
      //这里进入死循环，直到pollInner有结果返回
  	for (;;) {
  		......
  		if (result != 0) {
  			......
  			return result;
  		}
  		result = pollInner(timeoutMillis);
  	}
  }
  ~~~

  ~~~java
  int Looper::pollInner(int timeoutMillis) {
  	......
  	int result = ALOOPER_POLL_WAKE;
  	......
  #ifdef LOOPER_USES_EPOLL
  	struct epoll_event eventItems[EPOLL_MAX_EVENTS];
      //调用epoll_wait函数来看看epoll专用文件描述符mEpollFd所监控的文件描述符是否有IO事件发生，它设置监控的超时时间为timeoutMillis：当mEpollFd所监控的文件描述符发生了要监控的IO事件后或者监控时间超时后，线程就从epoll_wait返回了（不再阻塞在这里），没有发生注册的事件的话线程就会在epoll_wait函数中进入睡眠状态了。就是在epoll_wait这里阻塞的
      //前面的Looper的构造函数，我们在里面设置了要监控mWakeReadPipeFd文件描述符的EPOLLIN事件。
  	int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
  	bool acquiredLock = false;
  #else
  	......
  #endif
      //当mEpollFd所监控的文件描述符发生了要监控的IO事件后或者监控时间超时后，线程就从epoll_wait返回了，否则线程就会在epoll_wait函数中进入睡眠状态了。返回后如果eventCount等于0，就说明是超时了，如果eventCount不等于0，就说明发生要监控的事件
  	if (eventCount < 0) {
  		if (errno == EINTR) {
  			goto Done;
  		}
   
  		LOGW("Poll failed with an unexpected error, errno=%d", errno);
  		result = ALOOPER_POLL_ERROR;
  		goto Done;
  	}
   //超时
  	if (eventCount == 0) {
  		......
  		result = ALOOPER_POLL_TIMEOUT;
  		goto Done;
  	}
   
  	......
   
          //如果前面判断没有问题，就说明管道有我们期待的消息来了
  #ifdef LOOPER_USES_EPOLL
  	for (int i = 0; i < eventCount; i++) {
  		int fd = eventItems[i].data.fd;
  		uint32_t epollEvents = eventItems[i].events;
  		if (fd == mWakeReadPipeFd) {//先找到我们关心的管道的读端
  			if (epollEvents & EPOLLIN) {//判定发生的事件
  				awoken();//然后就执行这里
  			} else {
  				LOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
  			}
  		} else {
  			......
  		}
  	}
  	if (acquiredLock) {
  		mLock.unlock();
  	}
  Done: ;
  #else
  	......
  #endif
  	......
  	return result;
  }
  //用来清空管道中的内容，以便下次再调用pollInner函数时，知道自从上次处理完消息队列中的消息后，有没有新的消息加进来
  void Looper::awoken() {
  	......
   
  	char buffer[16];
  	ssize_t nRead;
  	do {
          //清空的逻辑在这里，其实就是把管道内的内容读取出来
  		nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
  	} while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
  }
  ~~~

* 到这里我们可以分析一下，应用程序在消息队列没有消息时，会在消息队列阻塞住，就是在native层的消息队列的mLooper->pollOnce（）方法内部阻塞住，就是在epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);方法阻塞住，当管道有消息来时，就取消阻塞，应用程序就去消息队列里取消息，这里的管道应该就是一个用来传递消息队列里有没有消息的一个信号而已，就是别的地方往我们的应用程序中的消息队列塞消息后，就往我们应用程序的管道里塞一点数据，这样我们的应用程序不再阻塞就从epoll_wait()方法依次返回，java层的looper就可以去消息队列里取消息了。我们现在看看是什么往消息队列里塞数据然后往管道里写数据唤醒我们的应用进程的。

## 发送消息时的唤醒

* 发消息,发消息的工作肯定是由 handler 来完成的啊

  ~~~java
  //经过多层调用我们就会到达这里
  public boolean sendMessageAtTime(Message msg, long uptimeMillis)
  	{
  		boolean sent = false;
  		MessageQueue queue = mQueue;
  		if (queue != null) {
              //这里很重要的一步，向这个消息说你的处理者是我
  			msg.target = this;
              //然后就把消息往消息队列里发送了
  			sent = queue.enqueueMessage(msg, uptimeMillis);
  		}
  		else {
  			......
  		}
  		return sent;
  	}
  ~~~

  ~~~java
  final boolean enqueueMessage(Message msg, long when) {
  		......
   
  		final boolean needWake;
  		synchronized (this) {
  			......
   
  			msg.when = when;
  			//Log.d("MessageQueue", "Enqueing: " + msg);
  			Message p = mMessages;
  			if (p == null || when == 0 || when < p.when) {
                  //当前消息队列为空时，这时候应用程序的主线程一般就是处于空闲等待状态了，这时候就要唤醒它
  				msg.next = p;
  				mMessages = msg;
  				needWake = mBlocked; // new head, might need to wake up
  			} else {
                  //应用程序的消息队列不为空，这时候就不需要唤醒应用程序的主线程了，因为这时候它一定是在忙着处于消息队列中的消息，因此不会处于空闲等待的状态。
  				Message prev = null;
  				while (p != null && p.when <= when) {
  					prev = p;
  					p = p.next;
  				}
  				msg.next = prev.next;
  				prev.next = msg;
  				needWake = false; // still waiting on head, no need to wake up
  			}
   
  		}
      //前面判定最后需不需要唤醒应用程序的主线程
  		if (needWake) {
              //唤醒，往管道写数据的逻辑很可能就在这里了
  			nativeWake(mPtr);
  		}
  		return true;
  }
  ~~~

  ~~~java
  static void android_os_MessageQueue_nativeWake(JNIEnv* env, jobject obj, jint ptr) {
      NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
      //这里调用jni层创建的消息队列的方法，看方法名就知道要唤醒了
      return nativeMessageQueue->wake();
  }
  void NativeMessageQueue::wake() {
      mLooper->wake();
  }
  void Looper::wake() {
  	...... 
  	ssize_t nWrite;
  	do {
          //终于看到了，就是这里，往管道的写入端写入一个“W”字符，这里就验证了我们之前的推论，管道只作为需不需要唤醒的一个信号量，所以这里写入什么字符都是可以的，前面我们在分析应用程序的消息循环时说到，当应用程序的消息队列中没有消息处理时，应用程序的主线程就会进入空闲等待状态，而这个空闲等待状态就是通过调用这个Looper类的pollInner函数来进入的，具体就是在pollInner函数中调用epoll_wait函数来等待管道中有内容可读的。这时候既然管道中有内容可读了，应用程序的主线程就会Looper类的pollInner函数返回到JNI层的nativePollOnce函数，最后返回到Java层中的MessageQueue.next函数中去，这里它就会发现消息队列中有新的消息需要处理了，于就会处理这个消息。
  
  		nWrite = write(mWakeWritePipeFd, "W", 1);
  	} while (nWrite == -1 && errno == EINTR);
  	.......
  }
  ~~~

## 这里native层的消息队列存的是什么

* 这里我也不懂，只是猜测
* 我们可以看到上面，jni 创建的消息队列里，我们都没有看到消息队列里有消息，我们大部分逻辑其实都是在消息队里looper里面，我们接触到的消息只有一个，就是管道里的消息，所以这里大胆的猜测，jni 和 native 的消息机制是，消息为管道里的消息，发送消息其实是发送到 looper 里的管道，looper 处理的也是管道里的消息。消息队列只是一个门面而已，大部分任务全部由 looper 实现。

