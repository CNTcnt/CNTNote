[TOC]

# 简单理解 Binder

* 解决之前的一些疑问

## 实现原理

一次完整的 Binder IPC 通信过程通常是这样：

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区；
2. 接着在内核空间开辟一块内核缓存区，建立**内核缓存区**和**内核中数据接收缓存区**之间的映射关系，以及**内核中数据接收缓存区**和**接收进程用户空间地址**的映射关系；
3. 发送方进程通过系统调用 copy_from_user() 将数据 copy 到内核中的**内核缓存区**(从应用进程到内核的一次 copy_from_user就是这里，这里内核也知道了数据的内存位置，等等返回值就可以直接往里写了)，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

## 从 bpBind 的 transact 开始

~~~java
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}

status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;
...
   
    
    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        //这里把数据写进请求
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
    if ((flags & TF_ONE_WAY) == 0) {
     ...
    } else {
         //看这里！！名字就是等待回复了，与 Binder 的通信应该就是这里了
        err = waitForResponse(NULL, NULL);
    }
   
    return err;
}
~~~

* waitForResponse

~~~java
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        //这个talkWithDriver就很明显了！！！他就是去和BInder驱动通信了！！下面去这里看
        if ((err=talkWithDriver()) < NO_ERROR) break;
        //这里的返回值是从 mIn 里读取出来的，也就是说，与 binder 通信完的返回值就是放在这里了
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
        
        cmd = (uint32_t)mIn.readInt32();
        
        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        //根据cmd做处理，
        switch (cmd) {
      	...
            goto finish;

        default:
            //看这里，这个是当收到回复后的事
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

 //这个是本次已经通信完成了，结束了！
finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }
    
    return err;
}
~~~

* talkWithDriver()

~~~java
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }
    
    //这个东西是用来给驱动交换数据的
    binder_write_read bwr;
    
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    bwr.write_size = outAvail;
    //我们刚刚设置好的数据在这里传进去了，将数据设置进 bwr 中
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    IF_LOG_COMMANDS() {
        TextOutput::Bundle _b(alog);
        if (outAvail != 0) {
            alog << "Sending commands to driver: " << indent;
            const void* cmds = (const void*)bwr.write_buffer;
            const void* end = ((const uint8_t*)cmds)+bwr.write_size;
            alog << HexDump(cmds, bwr.write_size) << endl;
            while (cmds < end) cmds = printCommand(alog, cmds);
            alog << dedent;
        }
        alog << "Size of receive buffer: " << bwr.read_size
            << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;
    }
    
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    
   
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(__ANDROID__)
        //看这里，最重要的一步，就是和BInder驱动进行通信了！！！可以看到bwr就是我们设置好的数据（我们要给的数据放在这里，等等 binder 也可以通过这个 bwr 来返回通信，因为 binder 都知道 bwr 的内存位置了），下面我们就到这个最重要的部分去探索，
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    IF_LOG_COMMANDS() {
        alog << "Our err: " << (void*)(intptr_t)err << ", write consumed: "
            << bwr.write_consumed << " (of " << mOut.dataSize()
                        << "), read consumed: " << bwr.read_consumed << endl;
    }

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {//这里可以看到通信结束后的返回数据确实是写在 bwr 中的，我们直接取出数据放进 mIn 里，等等应用程序就可以去 mIn 里读数据了
            //然后把他读出来到mIn中
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        IF_LOG_COMMANDS() {
            TextOutput::Bundle _b(alog);
            alog << "Remaining data size: " << mOut.dataSize() << endl;
            alog << "Received commands from driver: " << indent;
            const void* cmds = mIn.data();
            const void* end = mIn.data() + mIn.dataSize();
            alog << HexDump(cmds, mIn.dataSize()) << endl;
            while (cmds < end) cmds = printReturnCommand(alog, cmds);
            alog << dedent;
        }
        return NO_ERROR;
    }
    
    return err;
}
~~~

* ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)

~~~java
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    struct binder_proc *proc = filp->private_data;
    //这里 ubuf 记录下了客户端缓冲区的位置
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    //将用户空间 ubuf 拷贝到内核空间的 bwr 中，这就是我们说的把用户空间的数据复制到内核空间
    copy_from_user(&bwr, ubuf, sizeof(bwr));
    ...

    if (bwr.write_size > 0) {
        //将数据放入目标服务端进程然后执行，操作 bwr 缓冲区，我们的客户端只需要知道服务端的 handle 值就可以构造出 bp ，然后此时驱动根据这个 handle 就可以知道目标服务然后就在映射区执行即可，那现在就有一个问题，这个 handle 值哪里来的，驱动程序怎么根据这个handle来找到我们的目标服务呢？我们知道我们获取一项服务是通过 ServiceManager.getService()根据服务名获取的，既然通过 ServiceManager ,我们的服务就需要向它注册，向它注册的过程中就可能生成这个 handle 值了。我们到这里去看一下
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        ...
    }
   

    //将内核空间执行结果 bwr 拷贝到用户空间的 ubuf 中
    copy_to_user(ubuf, &bwr, sizeof(bwr));
    ...
}   
~~~

* binder_thread_write

~~~java
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //拿到用户空间的cmd命令，此时为BC_TRANSACTION
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            //拷贝内核空间 的 bwr 到 tr ，目标服务与 tr 沟通
            if (copy_from_user(&tr, ptr, sizeof(tr)))   return -EFAULT;
            ptr += sizeof(tr);
            //看这里，这里
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
        ...
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
~~~

* binder_transaction(proc, thread, &tr, cmd == BC_REPLY);

~~~java
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    struct binder_transaction *t;
   	struct binder_work *tcomplete;
    ...

    if (reply) {
        ...
    }else {
        //这里就是根据handle值去找对应的服务端处理了，如果我们的服务Bp里面有Handle值，然后会取处理走到这里吧，这样就可以找到对应的进程进而走下面的代码去处理数据的交换了！！当然，这一切都要建立在有Handle的情况，我们往下看吧！怎么生成Handle
        if (tr->target.handle) {
            //如果有 handle 不为0 就执行这一部分的代码，handle 为0 指的是 ServiceManager 的 handle
            ...
        } else {
            // handle=0则找到servicemanager实体
            target_node = binder_context_mgr_node;
        }
        //如果 handle 为0 ，那么此时target_proc为servicemanager进程
        target_proc = target_node->proc;
    }

    if (target_thread) {
        ...
    } else {
        //找到servicemanager进程的todo队列
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

    t = kzalloc(sizeof(*t), GFP_KERNEL);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    //非oneway的通信方式，把当前thread保存到transaction的from字段
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;

    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc; //此次通信目标进程为servicemanager进程
    t->to_thread = target_thread;
    t->code = tr->code;  //此次通信code = ADD_SERVICE_TRANSACTION
    t->flags = tr->flags;  // 此次通信flags = 0
    t->priority = task_nice(current);

    //从servicemanager进程中分配buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

    t->buffer->allow_user_free = 0;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;

    //引用计数加1
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL); 
    offp = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    //分别拷贝 binder_transaction_data中ptr.buffer和ptr.offsets到内核
    copy_from_user(t->buffer->data,
        (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size);

    off_end = (void *)offp + tr->offsets_size;

    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        off_min = *offp + sizeof(struct flat_binder_object);
        switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
              struct binder_ref *ref;
             //服务注册过程是在服务所在进程创建binder_node，在servicemanager进程创建binder_ref。 对于同一个binder_node，每个进程只会创建一个binder_ref对象。所以这里会先去查找，如果节点为空，则需要创建的操作
              struct binder_node *node = binder_get_node(proc, fp->binder);
              if (node == NULL)， {
                //服务所在进程 创建binder_node实体
                node = binder_new_node(proc, fp->binder, fp->cookie);
                ...
              }
              //servicemanager进程binder_ref
              ref = binder_get_ref_for_node(target_proc, node);
              ...
              //调整type为HANDLE类型
              if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
              else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
              fp->binder = 0;
              fp->handle = ref->desc; //设置handle值
              fp->cookie = 0;
              binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
            } break;
            case :...
             //代码走到这里我们就可以得出了一个结论：这个handle值就是累加的！！！然后放在了ServiceManager里面去保存，比如说我们A服务先去注册，假设他的Handle为1，那B服务再去注册，B的Handle就为2了吧！然后我们上面也可以看到如果已经注册了是直接binder_get_node，但是具体的累加代码上面没说，因为我也没打开到BInder驱动的源码，但是就是累加的！！！
 //handle值计算方法规律：
               //每个进程binder_proc所记录的binder_ref的handle值是从1开始递增的；
                //所有进程binder_proc所记录的handle=0的binder_ref都指向service manager；
				//同一个服务的binder_node在不同进程的binder_ref的handle值可以不同；
    }

    if (reply) {
        ..
    } else if (!(t->flags & TF_ONE_WAY)) {
        //BC_TRANSACTION 且 非oneway,则设置事务栈信息
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
        ...
    }

    //将BINDER_WORK_TRANSACTION添加到目标队列，本次通信的目标队列为target_proc->todo
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);

    //将BINDER_WORK_TRANSACTION_COMPLETE添加到当前线程的todo队列
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);

    //唤醒等待队列，本次通信的目标队列为target_proc->wait，所以这里可以看出，驱动程序是以队列去处理多个进程并发操作的！！！
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
~~~

### 小结

小结binder的大概通信过程

* 首先，一个进程使用 BINDER_SET_CONTEXT_MGR 命令通过 Binder 驱动将自己注册成为 ServiceManager；
* Server 通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager，ServiceManger 将其填入查找表。
* Client 通过名字，在 Binder 驱动的帮助（查找表）从 ServiceManager 中获取到对 Binder 实体的引用，通过这个引用就能实现和 Server 进程的通信。