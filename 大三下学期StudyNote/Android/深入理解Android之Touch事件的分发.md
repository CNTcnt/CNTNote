[TOC]

# 深入理解Android之Touch事件的分发

* 从我们的手指接触屏幕的瞬间进入Android之Touch事件的分发

##　概述

* 当时输入设备（如触摸屏，键盘等）可用时，Linux Kernel会在/dev/input/下创建名为*event0~eventN*的设备节点; 当输入设备不可用时，会将相应的设备节点删除。
* 当用户操作输入设备时，Linux Kernel会收到相应的硬件中断，然后会将中断加工成*原始输入事件（raw input event）*，并写入到设备节点中。而后在用户空间就可以通过read()函数读取事件数据了。
* Android输入系统会监控/dev/input/下的所有设备节点，当某个结点有数据可读时，将数据读出并进行一系列处理，然后在当前系统中的所有窗口（Window）中寻找合适的接收者，并把事件派发给它。
* 具体来说，Linux Kernel将raw input event写入到设备节点后，InputReader会通过EventHub将原始事件读取出来并翻译加工为Android输入事件，而后把它交给InputDispatcher。InputDispatcher根据WMS（WindowManagerService）提供的窗口信息将事件传递给合适的窗口，若窗口为壁纸/SurfaceView等，则到了终点；否则会由该Window的ViewRoot继续分发到合适的View。

## Android输入事件的派发流程

* 当原始事件进入InputDispatcher时，已被加工为KeyEvent、MotionEvent（或SwitchEvent）。而后InputDispatcher会对事件进行进一步分发。

### 将事件放入派发队列

将输入事件加入到派发队列后，会依次执行以下步骤：

- 派发线程 InputDiapter 开始派发事件；
- 锁定目标窗口，然后向目标Window发送事件。
- InputDispatcher（输入调度）通过InputChannel(在最后面有讲解，这里只需要对它有个模糊的概念)将event发给Window（InputDispatcher运行于system_server进程中，Window运行于应用进程，两者通过InputChannel通信（而不是binder，因为这样速度更快））；
- Window端的Looper被唤醒，从InputChannel中读取一个InputEvent，而后调用onInputEvent(ev)。具体来说是调用ViewRootImpl的mInputEventReceiver的成员的onInputEvent()方法。
- 调用doProcessInputEvents()
- 调用deliverInputEvent()

### deliverInputEvent()

* 这个方法内部会调用processPointerEvent()方法对输入事件进行处理，而processPointerEvent内部会通过mView.dispatchPointerEvent()方法来进一步对输入事件分发。这里的mView即为Window的decorView
* 这里我们只分析Touch事件的情况，对于Touch事件，会进一步调用mView.dispatchTouchEvent()方法来对其进行分发。

### Touch事件的分发

#### mView.dispatchTouchEvent()

我们可以把decorView看作是Window的Callback的实现者（即Activity）在控件树（ViewTree）中安插的一个间谍，decorView所接收到的事件，Activity也都会接收到。实际上，Activity的dispatchTouchEvent()方法会先于控件树中的任一个子View被调用。

#### Activity.dispatchTouchEvent()

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    onUserInteraction();
  } 
  if (getWindow().superDispatchTouchEvent(ev)) {
    return true; 
  } 
  return onTouchEvent(ev);
}
```

我们可以看到，这个方法中调用了Window的superDispatchTouchEvent()方法，这个方法会把Touch事件传递给decorView，从而开始在窗口的控件树（ViewTree）中进行分发。(所以我们可以在activity 直接拦截处理消息)

#### ViewGroup.dispatchTouchEvent()

decorView本质上是一个ViewGroup，因此我们来看一下ViewGroup的dispatchTouchEvent()方法。

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    ......
    boolean handled = false;
    ......
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;

    // 1. 处理初始的ACTION_DOWN
    if (actionMasked == MotionEvent.ACTION_DOWN) {

       // 把ACTION_DOWN作为一个Touch手势（Touch事件序列）的始点，清除之前的手势状态；
       // 将mFirstTouchTarget重置为null（mFirstTouchTarget为最近一次对处理Touch事件的View）
        cancelAndClearTouchTargets(ev); 
       // 重置FLAG_DISALLOW_INTERCEPT，该标记为true则表示禁止拦截本次事件
        resetTouchState();
    }

    // 2. 判断是否拦截此次的Touch事件
    final boolean intercepted;

    if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
        // 本次事件是ACTION_DOWN，或者mFirstTouchTarget不为null(已经找到能够接收该Touch事件的目标组件)
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

        // 判断禁止拦截的FLAG，因为子View可以通过requestDisallowInterceptTouchEvent()
        // 方法可以禁止父View执行“是否需要拦截”的判断
        if (!disallowIntercept) {   
            // 禁止拦截的FLAG为false，说明可以执行拦截判断，则执行此ViewGroup的onInterceptTouchEvent方法
            intercepted = onInterceptTouchEvent(ev);
            // 此方法默认返回false，如果想修改默认的行为，需要override此方法，修改返回值。
            ev.setAction(action);

        } else {
 
            // 禁止拦截的FLAG为ture，设置intercepted为false
            intercepted = false;

        }

    } else {
       // 当不是ACTION_DOWN事件并且mFirstTouchTarget为null时，
       // 这个ViewGroup应该继续执行拦截的操作。
        intercepted = true;
    }

    
    // 通过前面的逻辑处理，得到了是否需要进行拦截的变量值
    final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;

    final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;

    TouchTarget newTouchTarget = null;

    boolean alreadyDispatchedToNewTouchTarget = false;

    if (!canceled && !intercepted) {
        // 不是ACTION_CANCEL且不拦截
        if (actionMasked == MotionEvent.ACTION_DOWN) {

           // 在ACTION_DOWN时去寻找这次DOWN事件新出现的TouchTarget
            final int actionIndex = ev.getActionIndex(); // always 0 for down
            .....
            final int childrenCount = mChildrenCount;
            
            if (newTouchTarget == null && childrenCount != 0) {
                // 根据触摸的坐标寻找能够接收这个事件的子组件
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);
                final View[] children = mChildren;

                // 逆序遍历所有子组
                for (int i = childrenCount - 1; i >= 0; i--) {
                    final int childIndex = i;
                    final View child = children[childIndex];

                    // 寻找可接收这个事件（可见性为VISIBLE或包含动画）并且组件区域内包含点击坐标的子View
                    if (!canViewReceivePointerEvents(child) || !isTransformedTouchPointInView(x, y, child, null)) {
                        continue;
                    }
                    // 找到了符合条件的子View，赋值给newTouchTarget
                    newTouchTarget = getTouchTarget(child);
    
                    ......

                    // 把ACTION_DOWN事件传递给子View进行处理
                    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                       
                        // 如果此子View消费了这个touch事件
                        . . .
                       
                       // 则把mFirstTouchTarget赋值为newTouchTarget，此子View成为新的touch事件的起点
                        newTouchTarget = addTouchTarget(child, idBitsToAssign);
                        alreadyDispatchedToNewTouchTarget = true;
                        break;
                    }
                }
            }

            ......

        }
    }

    // 经过前面对ACTION_DOWN的处理后，有两种情况。
    if (mFirstTouchTarget == null) {
        // 情况1： 没有找到能够消费Touch事件的子View或者是touch事件被拦截了，此时mFirstTouchTarget为null。
        // 会调用ViewGroup的dispatchTransformedTouchEvent方法对Touch事件做处理，
        // 这里传入该方法的第三个参数为null，表示没有子View能消费此事件，
        // 则会通过调用super.dispatchOnTouchEvent()把事件回递给父ViewGroup进行处理，
        // 实际上会调用父View的onTouchEvent()方法来处理
        handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
    } else {

        // 情况2：找到了能够消费touch事件的子组件，那么后续的touch事件都可以传递到子View
        TouchTarget target = mFirstTouchTarget;
   
       // 简单起见，省略了Target List的概念，具体可参考源码
        while (target != null) {
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
              
                // 如果前面找到了能消费ACTION_DOWN事件的View并消费掉了ACTION_DOWN事件，这里直接返回true
                handled = true;
            } else {
                final boolean cancelChild = resetCancelNextUpFlag(target.child) || intercepted;

                // 对于非ACTION_DOWN事件，则继续传递给目标子View进行处理，无需再判断是否进行拦截
                if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                    handled = true;
                }
                // 如果ACTION_DOWN时没有被拦截，而后面的Touch事件被拦截，则需要发送ACTION_CANCEL给之前处理ACTION_DOWN的View
                ......
            }
        }

    }

    if (canceled || actionMasked == MotionEvent.ACTION_UP) {
        // 如果是ACTION_CANCEL或者ACTION_UP，重置FLAG_DISALLOW_INTERCEPT，mFirstTouchTarget赋为null
        resetTouchState();
    }
    ......
    return handled;
}
```

### View对Touch事件的处理

在以上ViewGroup对Touch事件分发的源码中，我们多次看到了ViewGroup的dispatchTransformedEvent()方法，这个方法中调用了View的dispatchTouchEvent()方法来对Touch事件进行处理：

#### View.dispatchTouchEvent()

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    . . .


    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
        . . .
        return result;
}
```

若设置了onTouchListener，则会先调用onTouch()方法，此方法返回false才会调用onTouchEvent()方法。

#### View.onTouchEvent()

只要View的CLICKABLE属性和LONG_CLICKABLE属性有一个为true(View的CLICKABLE属性和具体View有关，LONG_CLICKABLE属性默认为false，setOnClikListener和setOnLongClickListener会分别自动将以上俩属性设为true），那么这个View就会消耗这个touch事件，即使这个View处于DISABLED状态。若当前事件是ACTION_UP，还会调用performClick()方法，该View若设置了OnClickListener，则performClick方法会在其内部调用onClick()方法.

至此，Touch事件的分发流程就分析完了。

------



### 输入系统-InputChannel

* 上面埋了坑，这里填一下

#### InputChannel在哪里创建

* `InputChannel`的创建是在 `ViewRootImpl`中`setView`方法中。

* ~~~java
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      ....
    if ((mWindowAttributes.inputFeatures
                          & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
         //首先是java层创建了一个InputChannel，java层的只是一个空壳，具体实现全依赖jni调用native层的
         mInputChannel = new InputChannel();
     }
    ....
    //将InputChannel添加到WindowManagerService中,创建socketpair（一对socket）用来发送和接受事件,然后将一对中的一个保存在 mInputChannel的mPtr变量中保存
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                              getHostVisibility(), mDisplay.getDisplayId(),
                              mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                              mAttachInfo.mOutsets, mInputChannel);
    ....
  if (mInputChannel != null) {
     if (mInputQueueCallback != null) {
         mInputQueue = new InputQueue();
         mInputQueueCallback.onInputQueueCreated(mInputQueue);
    }
      //会创建 WindowInputEventReceiver，继承于 InputEventReceiver，这个消息接收者就是用来接收消息然后处理消息的,注意，以下文的InputEventReceiver都是指的这个WindowInputEventReceiver对象
     mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                              Looper.myLooper());
  }
    ...
  }
  ~~~

* ~~~java
  //addToDisplay将会把InputChannel添加到WindowManagerService中。会调用WMS的addWindow方法。
  public int addWindow(Session session, IWindow client, int seq,
              WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
              Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
              InputChannel outInputChannel) {
        ....
  
        final boolean openInputChannels = (outInputChannel != null
                      && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
        if  (openInputChannels) {
            //这里打开了InputChannel，具体怎么打开看下面，而为什么要打开呢，可能是因为ＷＭＳ与我们的应用进程要建立起直接连接
            win.openInputChannel(outInputChannel);
        }
        ....
  }
  void openInputChannel(InputChannel outInputChannel) {
      if (mInputChannel != null) {
          throw new IllegalStateException("Window already has an input channel.");
       }
       String name = makeInputChannelName();
      //这里打开了一对InputChannelPair（其实就是一对已连接的无名socket对），它是什么，怎么打开的看下面
       InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
       mInputChannel = inputChannels[0];
       mClientChannel = inputChannels[1];
       mInputWindowHandle.inputChannel = inputChannels[0];
       if (outInputChannel != null) {
           //将mClientChannel与客户端传来的 outInputChannel （中的mPtr属性）连接，就是将outInputChannel 设置到 mClinetChannel 对象里的 mPtr 属性里保存，这样Java层才找得到这个对象
         mClientChannel.transferTo(outInputChannel);
   /*    static void android_view_InputChannel_nativeTransferTo(JNIEnv* env, jobject obj,
  180        jobject otherObj) {
  181    if (android_view_InputChannel_getNativeInputChannel(env, otherObj) != NULL) {
  182        jniThrowException(env, "java/lang/IllegalStateException",
  183                "Other object already has a native input channel.");
  184        return;
  185    }
  186
  187    NativeInputChannel* nativeInputChannel =
  188            android_view_InputChannel_getNativeInputChannel(env, obj);
  		***这里把 outInputChannel 中的 mptr 设置成 mClientChannel 的nativeInputChannel，这里是将java层创建的空壳 inputChannel 与 WSM 创建的用于通信的无名socket对的一端连接起来，就是最关键的这里使得WSM与应用程序在处理TOuch事件的时候有直接独立的通道
  189    android_view_InputChannel_setNativeInputChannel(env, otherObj, nativeInputChannel);
          //既然Java层 inputChannel 对象中的mPtr变量已经指向了 mClientChannel 的nativeInputChannel，那么这个 mClientChannel 就没有存在的必要了，它存在就是一个中转站而已
  190    android_view_InputChannel_setNativeInputChannel(env, obj, NULL);
  191}*/
         mClientChannel.dispose();
         mClientChannel = null;
        } else {
           mDeadWindowEventReceiver = new DeadWindowEventReceiver(mClientChannel);
        }
         //`InputChannel`设置到`InputDispatcher`，即将此window的 创建好的一对中的另一个 InputChannel 向 系统进程的　InputDispatcher 注册，即InputDispatcher只与上面创建好的socket对中的另一个沟通(接收消息吧)，然后另一个（与应用进程创建的InputChannel中的mPtr变量指向的对象中的socket）负责处理消息，这里我们就清楚了，原来WMS创建的WIndow的无名socket对是作为传输带，一端连应用程序的窗口，一端连着InputDispatcher（系统进程输入消息传递者）
         mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
  }
~~~
  
* ~~~c++
  //现在看是怎么创建 openInputChannelPair的
  static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
  125        jclass clazz, jstring nameObj) {
  126    const char* nameChars = env->GetStringUTFChars(nameObj, NULL);
  127    String8 name(nameChars);
  128    env->ReleaseStringUTFChars(nameObj, nameChars);
  129
  130    sp<InputChannel> serverChannel;
  131    sp<InputChannel> clientChannel;
      //这里再去openInputChannelPair
  132    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
  133
  134    if (result) {
  135        String8 message;
  136        message.appendFormat("Could not open input channel pair.  status=%d", result);
  137        jniThrowRuntimeException(env, message.string());
  138        return NULL;
  139    }
  140
  141    jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);
  142    if (env->ExceptionCheck()) {
  143        return NULL;
  144    }
  145//openInputChannelPair创建了native的对象，现在需要返回Java对象给Java层使用，这里就创建 java层的 对象,里面的mPtr指向native层创建的对象<NativeInputChannel>(serverChannel)）
  146    jobject serverChannelObj = android_view_InputChannel_createInputChannel(env,
  147            std::make_unique<NativeInputChannel>(serverChannel));
  148    if (env->ExceptionCheck()) {
  149        return NULL;
  150    }
  151
  152    jobject clientChannelObj = android_view_InputChannel_createInputChannel(env,
  153            std::make_unique<NativeInputChannel>(clientChannel));
  154    if (env->ExceptionCheck()) {
  155        return NULL;
  156    }
  157
  158    env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
  159    env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
      //返回这个Java数组，这样其实等等 Java 层拿到的就是封装了 InputChannelPair（具体实现是封装无名socket对的InputChannel）的 Java对象数组，每个Java对象里面的 mPtr 就指向了 具体的某个jni层创建的 InputChannel
  160    return channelPair;
  161}
  
  static jobject android_view_InputChannel_createInputChannel(JNIEnv* env,
  114        std::unique_ptr<NativeInputChannel> nativeInputChannel) {
      //就是新创建一个 java 对象，然后将 nativeInputChannel 赋值给 Java对象中的 mPtr 属性
  115    jobject inputChannelObj = env->NewObject(gInputChannelClassInfo.clazz,
  116            gInputChannelClassInfo.ctor);
  117    if (inputChannelObj) {
  118        android_view_InputChannel_setNativeInputChannel(env, inputChannelObj,
  119                 nativeInputChannel.release());
  120    }//然后返回这个Java对象
  121    return inputChannelObj;
  122}
  
  static struct {
  36    jclass clazz;
  37
  38    jfieldID mPtr;   // native object attached to the DVM InputChannel
  39    jmethodID ctor;
  40} gInputChannelClassInfo;
  
  static void android_view_InputChannel_setNativeInputChannel(JNIEnv* env, jobject inputChannelObj,
  91        NativeInputChannel* nativeInputChannel) {
  92    env->SetLongField(inputChannelObj, gInputChannelClassInfo.mPtr,
  93             reinterpret_cast<jlong>(nativeInputChannel));
  94}
  //nativeOpenInputChannelPair
  status_t InputChannel::openInputChannelPair(const String8& name,
          sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
      //看名字应该是基于socket实现
      int sockets[2];
      if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {//这里就创建一对已连接的（UNIX族）无名socket
          status_t result = -errno;
          outServerChannel.clear();
          outClientChannel.clear();
          return result;
      }
  
      int bufferSize = SOCKET_BUFFER_SIZE;
      setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
      setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
      setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
      setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
  
      String8 serverChannelName = name;
      serverChannelName.append(" (server)");
      //将无名socket对的一端封装成 S 端的 InputChannel（jni层的对象）
      outServerChannel = new InputChannel(serverChannelName, sockets[0]);
  
      String8 clientChannelName = name;
      clientChannelName.append(" (client)");
      //将无名socket对的一端封装成 C 端的 InputChannel
      outClientChannel = new InputChannel(clientChannelName, sockets[1]);
      return OK;
  }
  //Linux实现了一个源自BSD的socketpair调用可以实现上述在同一个文件描述符中进行读写的功能（该调用目前也是POSIX规范的一部分 。该系统调用能创建一对已连接的（UNIX族）无名socket。在Linux中，完全可以把这一对socket当成pipe返回的文件描述符一样使用，唯一的区别就是这一对文件描述符中的任何一个都可读和可写。
  
  //socketpair产生的文件描述符是一对socket，socket上的标准操作都可以使用，其中也包括shutdown。——利用shutdown，可以实现一个半关闭操作，通知对端本进程不再发送数据，同时仍可以利用该文件描述符接收来自对端的数据。
  
  //native层的 InputChannel 的创建，就是保存者socket的一个对象而已，操作它就是操作socket
  InputChannel::InputChannel(const String8& name, int fd) :
  103        mName(name), mFd(fd) {
  104#if DEBUG_CHANNEL_LIFECYCLE
  105    ALOGD("Input channel constructed: name='%s', fd=%d",
  106            mName.string(), fd);
  107#endif
  108
  109    int result = fcntl(mFd, F_SETFL, O_NONBLOCK);
  110    LOG_ALWAYS_FATAL_IF(result != 0, "channel '%s' ~ Could not make socket "
  111            "non-blocking.  errno=%d", mName.string(), errno);
  112}
  ~~~

  #### InputChannel 的发送和接收消息

  ~~~c++
  //通过写/读socket的方式来写入/读取消息。这里可以看出输入事件传递最终会写进某个窗口的 InputChannel 对象里的socket中
  //发送消息
  status_t InputChannel::sendMessage(const InputMessage* msg) {
      size_t msgLength = msg->size();
      ssize_t nWrite;
      do {
          nWrite = ::send(mFd, msg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
      } while (nWrite == -1 && errno == EINTR);
       .....
      return OK;
  }
  
  //接收消息，
  status_t InputChannel::receiveMessage(InputMessage* msg) {
      ssize_t nRead;
      do {
          nRead = ::recv(mFd, msg, sizeof(InputMessage), MSG_DONTWAIT);
      } while (nRead == -1 && errno == EINTR);
      ......
      return OK;
  }
  ~~~

#### 开启监听InputChannel对象里的socket

* 之前的`setView`中，我们创建了`InputChannel`之后，开启了对于InputChannel中输入事件的监听。

```java
if (mInputChannel != null) {
   if (mInputQueueCallback != null) {
       //这里的 InputQueue 暂时不知道用来干什么的
       mInputQueue = new InputQueue();
       mInputQueueCallback.onInputQueueCreated(mInputQueue);
   。。。。。
       。。。。。
    
  //创建完InputQueue 才创建接收者WindowInputEventReceiver，继承于InputEventReceiver，传入 java层的looper和 mInputChannel（此时mInputChannel已经不是个空壳对象了，在 上面 addWindow 的时候，wsm已经把通信用的soceket对中的一个封装成一个对象然后保存进 mInputChannel 对象中的 mPtr 变量中了）进入，这个接收者就是用来接收消息然后处理消息的
   mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
}
final class WindowInputEventReceiver extends InputEventReceiver {
     public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
         super(inputChannel, looper);
     }
      ....
}
public InputEventReceiver(InputChannel inputChannel, Looper looper) {
     ....
     mInputChannel = inputChannel;
     mMessageQueue = looper.getQueue();
    //调用jni方法,把Java层的消息队列和inputChannel和接受者传进去
     mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);
  
}
```

~~~java
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
   ....
 // sp<InputChannel> android_view_InputChannel_getInputChannel(JNIEnv* env, jobject inputChannelObj) {
 //   NativeInputChannel* nativeInputChannel =
 //           android_view_InputChannel_getNativeInputChannel(env, inputChannelObj);
 //   return nativeInputChannel != NULL ? nativeInputChannel->getInputChannel() : NULL;
 //}     
 //static NativeInputChannel* android_view_InputChannel_getNativeInputChannel(JNIEnv* env,
 //       jobject inputChannelObj) {
      //通过保存在Java层的 InputChannel 对象中的 mPtr 来获取到 jni 层创建的 InputChannel
 //   jlong longPtr = env->GetLongField(inputChannelObj, gInputChannelClassInfo.mPtr);
 //   return reinterpret_cast<NativeInputChannel*>(longPtr);
 //}      
  sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,inputChannelObj);
    
    //这里很重要，通过保存在Java层的消息队列里的mPtr把 jni 层消息队列取出来，源码是
//    sp<MessageQueue> android_os_MessageQueue_getMessageQueue(JNIEnv* env, jobject messageQueueObj) {
//   jlong ptr = env->GetLongField(messageQueueObj, 			                            gMessageQueueClassInfo.mPtr);
//   return reinterpret_cast<NativeMessageQueue*>(ptr);}
  sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    
    //创建native层的输入事件接受者，将Java层的接受者，jni层的消息队列和inputChannel 传进入
  sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    //这个 NativeInputEventReceiver 最主要的是继承了class NativeInputEventReceiver : public LooperCallback {。。。}，这处理了接收消息后的处理动作 handleEvent（），后面处理消息时会用到
    //它的构造函数只是赋值变量操作而已
  //NativeInputEventReceiver::NativeInputEventReceiver(JNIEnv* env,
  //      jobject receiverWeak, const sp<InputChannel>& inputChannel,
  //      const sp<MessageQueue>& messageQueue) :
  //      mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
  //      mInputConsumer(inputChannel), mMessageQueue(messageQueue),
  //      mBatchedInputEventPending(false), mFdEvents(0) {
  //  if (kDebugDispatchCycle) {
  //      ALOGD("channel '%s' ~ Initializing input event receiver.", getInputChannelName());
  //  }}
    //调用native层的事件接受者的initialize();
    status_t status = receiver->initialize();
  .....
}

status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}

//在initialize()方法中，只调用了一个函数setFdEvents，
void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        //从inputConsumer(封装了inputChannel的一个输入消费者对象)拿到channel，再从channel拿到它的socket
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            //然后把socket设置进jni层的looper的epoll机制去监听
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}
~~~

* 

~~~java
//现在进入 Looper.cpp 中查看是怎么 addFd 的
//传递的参数很重要：callback：当有事件发生时，会回调该callback。class NativeInputEventReceiver : public LooperCallback {。。。}
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ addFd - fd=%d, ident=%d, events=0x%x, callback=%p, data=%p", this, fd, ident,
            events, callback.get(), data);
#endif

    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }

        if (ident < 0) {
            ALOGE("Invalid attempt to set NULL callback with ident < 0.");
            return -1;
        }
    } else {
        ident = ALOOPER_POLL_CALLBACK;
    }

    int epollEvents = 0;
    if (events & ALOOPER_EVENT_INPUT) epollEvents |= EPOLLIN;
    if (events & ALOOPER_EVENT_OUTPUT) epollEvents |= EPOLLOUT;

    { // acquire lock
        AutoMutex _l(mLock);

        //创建一个 request 对象，内部封装了 fd 和fd发生事件后执行的 callback
        Request request;
        request.fd = fd;
        request.ident = ident;
        //主要是设置这里的callback(这个callback 真正实现其实就是NativeInputEventReceiver，class NativeInputEventReceiver : public LooperCallback {。。。},而这个接受者f封装了要接收的 InputEvent,而这个 InputEvent 就是 保存在Java层的InputChannel对象里mPtr属性所指向的native层的NativeInputChannel对象，这个NativeInputChannel对象其实就是封装了wsm在addWindow（）时 openInputChannelPair 中生成的无名socket对中的一个，将这个无名socket对中的一个封装成 InputChannel 再封装成 NativeInputChannel )
        request.callback = callback;
        request.data = data;

        struct epoll_event eventItem;
        memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
        eventItem.events = epollEvents;
        eventItem.data.fd = fd;

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            //然后就让 jni 层的消息队列里的 epoll 机制注册这个socket
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult < 0) {
                ALOGE("Error adding epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            //然后就添加进 mRequests 保存，可能用于处理消息时取出来然后执行callback
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult < 0) {
                ALOGE("Error modifying epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}
~~~

#### 管道接收到消息后的处理

* 之前把 监听任务交由给 looper 中的epoll,也就是说进入loop循环中了，looper循环会阻塞在 Looper::pollInner 方法里，我们要去里面看消息来后 做了什么处理

  ~~~c++
  int Looper::pollInner(int timeoutMillis) {
  	......
  	int result = ALOOPER_POLL_WAKE;
  	......
  #ifdef LOOPER_USES_EPOLL
  	struct epoll_event eventItems[EPOLL_MAX_EVENTS];
   //就是在这里阻塞,监听的事件发生后就取消阻塞，向下执行
  	int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
  	bool acquiredLock = false;
  #else
  	......
  #endif
      
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
          if (fd == mWakeReadPipeFd) {//这个文件描述符是用于唤醒主线程处理消息的描述符，我们要找的不是这个
              if (epollEvents & EPOLLIN) {
                  awoken();
              } else {
                  ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
              }
          } else {//我们会跑到这里来
              ssize_t requestIndex = mRequests.indexOfKey(fd);//通过fd找到对应的request,这个request存放我们设置好的东西
              if (requestIndex >= 0) {
                  int events = 0;
                  if (epollEvents & EPOLLIN) events |= ALOOPER_EVENT_INPUT;
                  if (epollEvents & EPOLLOUT) events |= ALOOPER_EVENT_OUTPUT;
                  if (epollEvents & EPOLLERR) events |= ALOOPER_EVENT_ERROR;
                  if (epollEvents & EPOLLHUP) events |= ALOOPER_EVENT_HANGUP;
                  //这里应该是去处理了吧
                  pushResponse(events, mRequests.valueAt(requestIndex));
                  //void Looper::pushResponse(int events, const Request& request) {
      			//						Response response;
      			//						response.events = events;
      			//						response.request = request;
                  		//其实就是把request封装成 response然后丢进mResponses中，所以这里还没有处理，继续往下看
      			//						mResponses.push(response);}
              } else {
                  ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                          "no longer registered.", epollEvents, fd);
              }
          }
      }
  	if (acquiredLock) {
  		mLock.unlock();
  	}
  Done: ;
      // Invoke pending message callbacks.
      mNextMessageUptime = LLONG_MAX;
      while (mMessageEnvelopes.size() != 0) {
          nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
          const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
          if (messageEnvelope.uptime <= now) {
              // Remove the envelope from the list.
              // We keep a strong reference to the handler until the call to handleMessage
              // finishes.  Then we drop it so that the handler can be deleted *before*
              // we reacquire our lock.
              { // obtain handler
                  sp<MessageHandler> handler = messageEnvelope.handler;
                  Message message = messageEnvelope.message;
                  mMessageEnvelopes.removeAt(0);
                  mSendingMessage = true;
                  mLock.unlock();
  
  #if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                  ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                          this, handler.get(), message.what);
  #endif
                  handler->handleMessage(message);
              } // release handler
  
              mLock.lock();
              mSendingMessage = false;
              result = ALOOPER_POLL_CALLBACK;
          } else {
              // The last message left at the head of the queue determines the next wakeup time.
              mNextMessageUptime = messageEnvelope.uptime;
              break;
          }
      }
  
      // Release lock.
      mLock.unlock();
  
      //就是在下面去循环遍历mResponses内的response（刚刚我们把request封装成response放进里面了），然后调用设置好的callback response.request.callback 
      // Invoke all response callbacks.
      for (size_t i = 0; i < mResponses.size(); i++) {
          Response& response = mResponses.editItemAt(i);
          if (response.request.ident == ALOOPER_POLL_CALLBACK) {
              int fd = response.request.fd;
              int events = response.events;
              void* data = response.request.data;
  #if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
              ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                      this, response.request.callback.get(), fd, events, data);
  #endif
              //就是这里处理的，我们再到callback->handleEvent里面看
              int callbackResult = response.request.callback->handleEvent(fd, events, data);
              if (callbackResult == 0) {
                  removeFd(fd);
              }
              // Clear the callback reference in the response structure promptly because we
              // will not clear the response vector itself until the next poll.
              response.request.callback.clear();
              result = ALOOPER_POLL_CALLBACK;
          }
      }
      return result;
  }
  ~~~

  * response.request.callback->handleEvent

  ~~~c++
  //request里的callback 是什么呢？其实就是 native层的消息接受者，因为NativeInputEventReceiver 最主要的是继承了class NativeInputEventReceiver : public LooperCallback {。。。}，这复写了接收消息后的处理动作 handleEvent（）
  int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
  158    if (events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP)) {
  159#if DEBUG_DISPATCH_CYCLE
  160        // This error typically occurs when the publisher has closed the input channel
  161        // as part of removing a window or finishing an IME session, in which case
  162        // the consumer will soon be disposed as well.
  163        ALOGD("channel '%s' ~ Publisher closed input channel or an error occurred.  "
  164                "events=0x%x", getInputChannelName(), events);
  165#endif
  166        return 0; // remove the callback
  167    }
  168
  169    if (events & ALOOPER_EVENT_INPUT) {//如果事件是 INPUT 类型，可读了，如果为ALOOPER_EVENT_INPUT事件，调用consumeEvents继续执行；如果为ALOOPER_EVENT_OUTPUT表示客户端已经收到数据，需要发送一个完成信号给服务端
  170        JNIEnv* env = AndroidRuntime::getJNIEnv();
      					//调用 consumeEvents，下面我们去看这个方法做了什么
  171        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
  172        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
  173        return status == OK || status == NO_MEMORY ? 1 : 0;
  174    }
  175
  				****
  215}
  ~~~

* ~~~c++
  status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
          bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
      ...
  
      ScopedLocalRef<jobject> receiverObj(env, NULL);
      bool skipCallbacks = false;
      for (;;) {
          uint32_t seq;
          InputEvent* inputEvent;
          //调用封装了inputChannel的 mInputConsumer 的consume 处理（接收事件）最后会将事件被inputEvent指针指,
          status_t status = mInputConsumer.consume(&mInputEventFactory,
                  consumeBatches, frameTime, &seq, &inputEvent);
          if (status) {
              if (status == WOULD_BLOCK) {
                  ...
                  return OK; //消费完成
              }
              return status; //消失失败
          }
  
          if (!skipCallbacks) {
              if (!receiverObj.get()) {
                  receiverObj.reset(jniGetReferent(env, mReceiverWeakGlobal));
                  if (!receiverObj.get()) {
                      return DEAD_OBJECT;
                  }
              }
  
              jobject inputEventObj;
              switch (inputEvent->getType()) {
                  case AINPUT_EVENT_TYPE_KEY:
                      //由 inputEvent 来生成Java层的事件对象，
                      inputEventObj = android_view_KeyEvent_fromNative(env,
                              static_cast<KeyEvent*>(inputEvent));
                      break;
                  ...
              }
  
              if (inputEventObj) {
                  //执行Java层的InputEventReceiver.dispachInputEvent,下面就要到这个方法里面看，上面有了事件对象了，现在应该就是要分发这个事件了
                  env->CallVoidMethod(receiverObj.get(),
                          gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
                  if (env->ExceptionCheck()) {
                      skipCallbacks = true; //分发过程发生异常
                  }
                  env->DeleteLocalRef(inputEventObj);
              } else {
                  skipCallbacks = true;
              }
          }
  
          if (skipCallbacks) {
              //发生异常，则直接向InputDispatcher线程发送完成信号。
              mInputConsumer.sendFinishedSignal(seq, false);
          }
      }
  }
  
  status_t InputConsumer::consume(InputEventFactoryInterface* factory,
          bool consumeBatches, nsecs_t frameTime, uint32_t* outSeq, InputEvent** outEvent) {
  
      *outSeq = 0;
      *outEvent = NULL;
  
      //循环遍历所有的Event
      while (!*outEvent) {
          if (mMsgDeferred) {
              mMsgDeferred = false; //上一次没有处理的消息
          } else {
              //调用consume方法会持续的调用InputChannel的receiveMessage方法来从socket中读取数据。到这里，我们已经将写入socket的事件读出来了，mMsg是 InputConsumer 的一个InputMessage变量
              status_t result = mChannel->receiveMessage(&mMsg);
              if (result) {
                  if (consumeBatches || result != WOULD_BLOCK) {
                      
                      result = consumeBatch(factory, frameTime, outSeq, outEvent);
                      if (*outEvent) {
                          break;
                      }
                  }
                  return result;
              }
          }
  
          switch (mMsg.header.type) {
            case InputMessage::TYPE_KEY: {
                //从mKeyEventPool池中取出KeyEvent
                KeyEvent* keyEvent = factory->createKeyEvent();
                if (!keyEvent) return NO_MEMORY;
  
                //将msg封装成KeyEvent
                initializeKeyEvent(keyEvent, &mMsg);
                *outSeq = mMsg.body.key.seq;
                //这一步很重要，将事件赋给 outEvent 对象里
                *outEvent = keyEvent;
                break;
            }
            ...
          }
      }
      return OK;
  }
  //InputChannal 接收数据
  status_t InputChannel::receiveMessage(InputMessage* msg) {
      ssize_t nRead;
      do {
          //读取InputDispatcher发送过来的消息，本函数用于已连接的数据报或流式套接口进行数据的接收。将消息放置于 msg 中
          nRead = ::recv(mFd, msg, sizeof(InputMessage), MSG_DONTWAIT);
      } while (nRead == -1 && errno == EINTR);
      ......
}
  ~~~
  
* 现在到Java层的 InputEventReceiver.dispachInputEvent，其实真正实现是WindowInputEventReceiver

  ~~~java
  //由于 WindowInputEventReceiver 没有复写dispatchInputEvent，所以这个方法在InputEventReceiver 中
  181    // Called from native code.
  182    @SuppressWarnings("unused")
  183    private void dispatchInputEvent(int seq, InputEvent event) {
  184        mSeqMap.put(event.getSequenceNumber(), seq);
  185        onInputEvent(event);//这个方法WindowInputEventReceiver复写了
  186    }
  
  @Override
  6744        public void onInputEvent(InputEvent event) {
  6745            enqueueInputEvent(event, this, 0, true);
  6746        }
  6747
      
           void enqueueInputEvent(InputEvent event,
  6555            InputEventReceiver receiver, int flags, boolean processImmediately) {
  6556        adjustInputEventForCompatibility(event);
      //把event以及对应的处理者封装成一个 QueueInputEvent 对象，后面存放于ViewRootImpl 对象里的一个属性 mPendingInputEventHead链表中
  6557        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
  6558
  6559        // Always enqueue the input event in order, regardless of its time stamp.
  6560        // We do this because the application or the IME may inject key events
  6561        // in response to touch events and we want to ensure that the injected keys
  6562        // are processed in the order they were received and we cannot trust that
  6563        // the time stamp of injected events are monotonic.
  6564        QueuedInputEvent last = mPendingInputEventTail;
  6565        if (last == null) {
  6566            mPendingInputEventHead = q;
  6567            mPendingInputEventTail = q;
  6568        } else {
  6569            last.mNext = q;
  6570            mPendingInputEventTail = q;
  6571        }
  6572        mPendingInputEventCount += 1;
  6573        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
  6574                mPendingInputEventCount);
  6575// 上面的传参是true，调用doProcessInputEvents()
  6576        if (processImmediately) {
  6577            doProcessInputEvents();
  6578        } else {
  6579            scheduleProcessInputEvents();
  6580        }
  6581    }
~~~
  
  ~~~java
          void doProcessInputEvents() {
              //下面就是处理 mPendingInputEventHead 链表里的事件
  6593        // Deliver all pending input events in the queue.
  6594        while (mPendingInputEventHead != null) {
  6595            QueuedInputEvent q = mPendingInputEventHead;
  6596            mPendingInputEventHead = q.mNext;
  6597            if (mPendingInputEventHead == null) {
  6598                mPendingInputEventTail = null;
  6599            }
  6600            q.mNext = null;
  6601
  6602            mPendingInputEventCount -= 1;
  6603            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
  6604                    mPendingInputEventCount);
  6605
  6606            long eventTime = q.mEvent.getEventTimeNano();
  6607            long oldestEventTime = eventTime;
  6608            if (q.mEvent instanceof MotionEvent) {
  6609                MotionEvent me = (MotionEvent)q.mEvent;
  6610                if (me.getHistorySize() > 0) {
  6611                    oldestEventTime = me.getHistoricalEventTimeNano(0);
  6612                }
  6613            }
  6614            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);
  6615			//传递消息
  6616            deliverInputEvent(q);
  6617        }
  6618
  6619        // We are done processing all input events that we can process right now
  6620        // so we can clear the pending flag immediately.
  6621        if (mProcessInputEventsScheduled) {
  6622            mProcessInputEventsScheduled = false;
  6623            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
  6624        }
6625    }
  ~~~
  
  ~~~java
  //这个方法首先进行事件的一个判断，通过shouldSkipIme来判断是否传递给输入法，然后决定使用何种InputStage进行消息的继续传递，这里实现了多种InputStage，对于每一个类型的InputStage都实现了一个方法onProcess方法来针对不同类型的事件做处理，如果是触摸屏类的消息，最终会将事件的处理转交到View的身上。
  
  private void deliverInputEvent(QueuedInputEvent q) {
  6628        Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
  6629                q.mEvent.getSequenceNumber());
  6630        if (mInputEventConsistencyVerifier != null) {
  6631            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
  6632        }
  6633
  6634        InputStage stage;
  6635        if (q.shouldSendToSynthesizer()) {
  6636            stage = mSyntheticInputStage;
  6637        } else {
  6638            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
  6639        }
  6640//这里的stage是ViewPostImeInputStage继承InputStage
  6641        if (stage != null) {
      //执行到这里
  6642            stage.deliver(q);
  6643        } else {
  6644            finishInputEvent(q);
  6645        }
  6646    }
  /**				在 InputStage 中，ViewPostImeInputStage没有复写该方法
  4120         * Delivers an event to be processed.
  4121         */
  4122        public final void deliver(QueuedInputEvent q) {
  4123            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
  4124                forward(q);
  4125            } else if (shouldDropInputEvent(q)) {
  4126                finish(q, false);
  4127            } else {
      //下面回调了 onProcess 方法
  4128                apply(q, onProcess(q));
  4129            }
  4130        }
  
  @Override //ViewPostImeInputStage复写了该方法
  4584        protected int onProcess(QueuedInputEvent q) {
  4585            if (q.mEvent instanceof KeyEvent) {
  4586                return processKeyEvent(q);
  4587            } else {
  4588                final int source = q.mEvent.getSource();
  4589                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
      //这里调用了 processPointerEvent(q);
  4590                    return processPointerEvent(q);
  4591                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
  4592                    return processTrackballEvent(q);
  4593                } else {
  4594                    return processGenericMotionEvent(q);
  4595                }
  4596            }
  4597        }
  
  4771        private int processPointerEvent(QueuedInputEvent q) {
  4772            final MotionEvent event = (MotionEvent)q.mEvent;
  4773
  4774            mAttachInfo.mUnbufferedDispatchRequested = false;
  4775            mAttachInfo.mHandlingPointerEvent = true;
      // 很重要的是这里，这里给 调用了View的dispatchPointerEvent(event);方法，此时ViewRootImpl的mView 指的是什么呢？其实是我们窗口最顶级的View 即 DecorView ，在调用setView的时候赋给mView了
      
  4776            boolean handled = mView.dispatchPointerEvent(event);
  4777            maybeUpdatePointerIcon(event);
  4778            maybeUpdateTooltip(event);
  4779            mAttachInfo.mHandlingPointerEvent = false;
  4780            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
  4781                mUnbufferedInputDispatch = true;
  4782                if (mConsumeBatchedInputScheduled) {
  4783                    scheduleConsumeBatchedInputImmediately();
  4784                }
  4785            }
  4786            return handled ? FINISH_HANDLED : FORWARD;
  4787        }
  
  //这是 DecorView 实现的
  @Override
      public boolean dispatchTouchEvent(MotionEvent ev) {
          //注意这里的Window.Callback就是Activity，因为activity实现了这个WindowCallback接口
          final Window.Callback cb = mWindow.getCallback();
          return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                  ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
      }
  //到这里我们的分析就告一段落，因为下面就是View的事件分发了
  ~~~

####　最后一张图理解整个架构

![](https://upload-images.jianshu.io/upload_images/5982616-b0ed949988847e96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)